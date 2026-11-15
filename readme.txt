POSITION BOOK APPLICATION - ASSUMPTIONS AND RATIONALE
======================================================

This file records every assumption made while building the Position Book
application, why each one was made, and where it is enforced in the code.
The specification leaves these points open. Each one should be confirmed
with the interviewer before the solution is considered final.


1. STORAGE REQUIREMENT
----------------------

Requirement: "The system should keep the data in memory (not in a database
or in the file system)."

How it is met:
  - All positions and the event index are held in plain Java collections
    (HashMap) inside the PositionBook class (src/main/java/.../domain/PositionBook.java).
  - No database driver, ORM, JDBC, JPA, H2 or other persistence library is
    on the classpath. pom.xml contains only Spring Boot web, validation,
    actuator, springdoc (Swagger UI) and test libraries.
  - Nothing is written to the file system. The application does not create
    files, and application.yml configures no data source.
  - The only file-system writes are build outputs in target/, created by Maven.

Consequence: all data is lost when the application stops. This is the
expected behaviour for an in-memory requirement and is assumption A12.


2. EVENT IDENTIFIERS (A1, A2)
-----------------------------

A1. Event IDs are positive 64-bit integers. BUY and SELL events must have
    IDs that are unique across the whole Position Book, not only within one
    trading account. A CANCEL event reuses the ID of the event it cancels.

    Rationale: Example 4 of the specification submits BUY 21 and then
    CANCEL 21. The cancel must therefore be identified by the same ID as its
    target. Global uniqueness keeps each cancel resolvable to exactly one
    trade without also needing the account in the request.

    Enforced in: PositionBook.trade() and PositionBook.cancel().

A2. A BUY or SELL whose ID is already in use is rejected with HTTP 409
    Conflict. The ID stays reserved even after the original event has been
    cancelled.

    Rationale: if a cancelled ID could be reused, a later CANCEL with that
    ID would be ambiguous. Keeping IDs reserved means every CANCEL refers to
    exactly one trade for the lifetime of the application.

    Enforced in: PositionBook.trade() (eventIndex.containsKey check).


3. CANCEL RULES (A3, A4, A5)
----------------------------

A3. A CANCEL must refer to an existing BUY or SELL and that target must not
    already be cancelled. A missing target returns 404. A second cancel of
    the same target returns 409.

    Rationale: a cancel with no target has no effect and cannot be
    meaningfully recorded. Cancelling twice would reverse the quantity twice,
    corrupting the position.

    Enforced in: PositionBook.cancel().

A4. A CANCEL cannot target another CANCEL. Because cancels are not stored as
    trade targets, a cancel of a cancel is treated as a cancel of an
    already-cancelled ID and returns 409.

    Rationale: follows directly from A3. Cancellation is not a reversible
    operation in this specification.

A5. The quantity on a CANCEL is ignored. The trading account and security ID
    on a CANCEL are optional. If supplied, they must match the target's
    values, otherwise the request is rejected with HTTP 422. If omitted, the
    target's values are used.

    Rationale: the specification says quantity and security ID of a cancel
    are "meaningless", yet Examples 4 supplies them (ACC1 SEC1 0). Accepting
    them and checking them when present lets the examples work and catches
    client mistakes, without making them mandatory. Rejecting a mismatch,
    rather than silently correcting it, avoids cancelling the wrong position.

    Enforced in: TradeEventRequest (format only), ValidTradeEventValidator
    (CANCEL is exempt from the BUY/SELL rules) and PositionBook.cancel()
    (mismatch check).


4. POSITION ARITHMETIC (A6, A7, A8)
-----------------------------------

A6. Net positions may be negative. A SELL larger than the current holding is
    accepted and creates a short position.

    Rationale: securities can be sold short, and the specification does not
    forbid it. The rule is easy to change: reject the SELL in
    PositionBook.trade() if the resulting net would be below zero.

A7. For BUY and SELL, the quantity must be a whole number from 1 to
    1,000,000,000 inclusive.

    Rationale: zero and negative quantities have no meaning for a buy or
    sell. The upper bound is far below the overflow point of a 64-bit
    integer, and it blocks obviously erroneous input. The limit is defined
    once in Limits.MAX_QUANTITY.

A8. GET of a position returns 404 when no event has ever referenced the
    trading account and security pair. A pair whose events net to zero
    returns 200 with quantity 0 and its drilldown.

    Rationale: this distinguishes "never traded" from "traded and now flat",
    which is useful information for a position book.


5. DRILLDOWN CONTENT (A9)
-------------------------

A9. The drilldown lists every event for the pair in arrival order. This
    includes:
      - the original BUY or SELL, marked cancelled=true once it is cancelled,
      - each CANCEL event, reported with quantity 0 and the target's account
        and security.

    Rationale: Example 4 shows the cancelled BUY and the CANCEL record both
    appearing in the output. Keeping both gives a complete audit trail.


6. IDENTIFIER FORMAT (A10)

A10. Trading account and security IDs are 1 to 20 characters, using only
     letters, digits, hyphen and underscore. They are case-sensitive.

     Rationale: a bounded, simple character set keeps the values safe to use
     as map keys and as URL path segments, and avoids encoding problems. The
     specification gives examples such as ACC1, SEC1 and SECXYZ, all of which
     fit. The length limit is a choice, not a requirement.

     Enforced in: Limits.ID_PATTERN, used by the request DTO and by path and
     query parameter validation.


7. ORDERING AND TIME (A11)

A11. Events are applied in the order they are received. No timestamps are
     required or stored.

     Rationale: the specification does not supply timestamps. The write lock
     serialises concurrent submissions, so the order is well defined.


8. DEPLOYMENT (A12)

A12. The application runs as a single instance in a single JVM. Restarting
     it discards all state.

     Rationale: consistent with the in-memory requirement. Multiple instances
     would each hold a separate, inconsistent Position Book.


9. CONCURRENCY
--------------

Not a numbered assumption, but a design decision that affects behaviour:

Writes are serialised with one ReentrantReadWriteLock. Reads share the lock.

Rationale: this is the simplest model that keeps event IDs globally unique
and makes each CANCEL atomic with the trade it targets. Throughput is limited
to one writer at a time, which is acceptable for this exercise. The
concurrency tests confirm that 32,000 concurrent BUYs produce the exact total
and that racing duplicate IDs accept exactly one event.


10. VALIDATION TIMING
---------------------

Validation runs before any state changes. A request that fails validation
leaves the Position Book untouched.

Rationale: a partially applied event would corrupt the position. Validation
at the boundary, plus the domain checks inside the lock, ensures each
accepted event is applied completely or not at all.


11. ERROR RESPONSE FORMAT
-------------------------

Errors use RFC 7807 Problem Details (application/problem+json).

Rationale: a standard format that clients can parse consistently, and that
Spring Boot supports natively.


OPEN QUESTIONS FOR THE INTERVIEWER
----------------------------------

These questions would change the design if the answers differ from the
assumptions above.

  Q1. Should short (negative) positions be allowed? (A6)
  Q2. Should a never-traded position return 404 or a zero-quantity position? (A8)
  Q3. Should a CANCEL with a mismatched account or security be rejected or
      silently corrected? (A5)
  Q4. Are event IDs unique across all accounts, or only within one account? (A1)
  Q5. What throughput and latency are expected? A single writer lock is
      sufficient for the exercise but may not be for production.
  Q6. Is authentication required? The current solution has none.
