# 🔴 P0 — Design BookMyShow / Movie Ticket Booking

> Interview Level: SDE-3 / Staff  
> Difficulty: ⭐⭐⭐⭐⭐  
> Interview Time: 45–60 minutes  
> Core Topics: Concurrency, Inventory, Distributed Locking, Consistency, Transactions, Idempotency, Expiry, Payment

---

# 1. Problem Statement

Design a movie-ticket booking system similar to BookMyShow.

Users should be able to:

- Search movies
- View theatres
- View shows
- View seat availability
- Select seats
- Temporarily hold seats
- Pay
- Confirm booking
- Cancel/refund where supported

The critical problem is:

> Two users must not successfully book the same seat.

---

# 2. Clarifying Questions

Ask:

1. How many theatres?
2. How many shows/day?
3. Average seats/show?
4. Peak booking TPS?
5. How large is the traffic spike for a popular movie?
6. Is seat selection real-time?
7. How long should a seat hold last?
8. What happens when payment fails?
9. Are cancellations supported?
10. Is multi-region required?
11. What availability and latency SLOs are required?

### Sample assumptions

```text
Users/day              = 10M
Peak browse RPS        = 200K
Peak booking requests  = 20K/sec
Seats/show             = 200
Popular show spike     = 50K users
Seat hold              = 5 minutes
Availability           = 99.99%
Booking correctness    = no double booking
```

---

# 3. Core Invariant

The most important invariant:

```text
One seat + one show
=
At most one confirmed booking
```

This must hold even during:

```text
Traffic spike
Concurrent requests
Retries
Payment timeout
Service restart
DB failover
```

---

# 4. State Model

Seat state:

```text
AVAILABLE
    |
    v
HELD
    |
    +----> AVAILABLE   (expiry/payment failure)
    |
    v
BOOKED
```

Booking:

```text
CREATED
   |
   v
PAYMENT_PENDING
   |
   +----> PAYMENT_FAILED
   |
   v
CONFIRMED
   |
   v
CANCELLED
```

---

# 5. High-Level Architecture

```text
Client
  |
  v
API Gateway
  |
  +-------------------+
  |                   |
  v                   v
Catalog Service    Booking Service
                      |
              +-------+-------+
              |               |
              v               v
          Redis Cache      Booking DB
                              |
                              v
                           Kafka
                              |
                +-------------+-------------+
                |             |             |
                v             v             v
          Payment Service  Notification  Expiry Worker
```

---

# 6. Browse Flow

```text
Client
  |
  v
API Gateway
  |
  v
Catalog Service
  |
  +--> Redis
  |
  +--> DB
```

Movie/show metadata is highly cacheable.

---

# 7. Seat Availability

For:

```text
show_id = S1
```

we need seat states:

```text
A1 AVAILABLE
A2 BOOKED
A3 HELD
A4 AVAILABLE
```

The seat inventory is highly contended for popular shows.

---

# 8. Critical Concurrency Problem

Two users:

```text
User A -> A1
User B -> A1
```

Both read:

```text
A1 = AVAILABLE
```

Both attempt booking.

A naive implementation:

```text
READ A1
if AVAILABLE:
    UPDATE A1
```

is unsafe.

---

# 9. Atomic Conditional Update

One approach:

```sql
UPDATE show_seats
SET status = 'HELD',
    hold_id = ?,
    hold_expires_at = ?
WHERE show_id = ?
AND seat_id = ?
AND status = 'AVAILABLE';
```

Check affected rows:

```text
1 row -> hold acquired
0 rows -> someone else owns it
```

This is an excellent correctness primitive because the condition and state transition happen atomically.

---

# 10. Database Locking

Alternative:

```sql
SELECT *
FROM show_seats
WHERE show_id = ?
AND seat_id = ?
FOR UPDATE;
```

Then:

```text
check AVAILABLE
     |
     v
mark HELD
     |
     v
commit
```

This uses a DB transaction.

Trade-off:

- Strong correctness
- But high contention can create DB lock pressure

---

# 11. Distributed Lock

Redis distributed locking can be used as an optimization/control mechanism.

But do not rely solely on:

```text
Redis lock
```

for financial/inventory correctness.

A process can:

```text
acquire lock
pause
lock expires
another process acquires lock
first process resumes
```

Therefore, where a distributed lock is used, consider:

```text
lease
+
fencing token
+
database validation
```

---

# 12. Seat Hold

A seat should not be permanently booked before payment.

Example:

```text
A1
 |
 v
HELD
 |
 +--> expires in 5 minutes
```

During hold:

```text
User A can pay
User B cannot select A1
```

---

# 13. Hold Expiration

Avoid scanning every seat constantly.

Use:

```text
hold_id
expires_at
```

and an expiry mechanism.

Possible design:

```text
Booking Service
      |
      v
Kafka / Delay Queue
      |
      v
Expiry Worker
      |
      v
Release seat
```

At release time:

```sql
UPDATE show_seats
SET status = 'AVAILABLE'
WHERE show_id = ?
AND seat_id = ?
AND status = 'HELD'
AND hold_id = ?;
```

This prevents an old expiry event from releasing a newer hold.

---

# 14. Payment Flow

```text
User
 |
 v
Create Hold
 |
 v
Payment
 |
 +----> FAILURE -> Release
 |
 v
SUCCESS
 |
 v
Confirm Booking
```

Critical problem:

```text
Payment succeeds
but booking confirmation fails
```

Do not simply charge again.

Use:

- Payment idempotency
- Booking state machine
- Retry
- Reconciliation

---

# 15. Idempotency

Booking request:

```http
POST /bookings
Idempotency-Key: abc123
```

Retry must return the original booking.

Database:

```text
UNIQUE(user_id, idempotency_key)
```

Payment should also use its own idempotency key.

---

# 16. Booking Transaction

A robust sequence:

```text
BEGIN
  |
  +--> Validate hold
  |
  +--> Validate expiry
  |
  +--> Validate payment state
  |
  +--> Mark seats BOOKED
  |
  +--> Mark booking CONFIRMED
  |
  +--> Insert outbox event
  |
COMMIT
```

Then:

```text
Outbox -> Kafka
```

---

# 17. Why Not 2PC?

Do not coordinate:

```text
Booking DB
     +
Payment DB
```

with distributed 2PC unless absolutely necessary.

External payment systems do not behave like a single transactional database.

Prefer:

```text
Local transactions
+
Idempotency
+
Outbox
+
Saga/reconciliation
```

---

# 18. Popular Show Problem

Suppose:

```text
50K users
    |
    v
same show
    |
    v
same 20 seats
```

The bottleneck is not the entire system.

It is:

```text
hot show / hot seat partitions
```

Protect the system with:

- Admission control
- Rate limiting
- Per-show concurrency limits
- Queueing
- Atomic seat updates
- Short hold operations

---

# 19. Waiting Room

For extreme launches:

```text
50K requests
     |
     v
Waiting Room
     |
     v
Booking Service
```

Instead of allowing all requests to hit the inventory DB simultaneously.

This converts an uncontrolled spike into controlled concurrency.

---

# 20. Redis Seat Cache

Redis can expose availability quickly:

```text
show:{show_id}:seats
```

But cached availability can become stale.

Therefore:

```text
Cache -> fast display
DB    -> final booking correctness
```

Never trust a stale cache to finalize a booking.

---

# 21. Sharding

Possible shard key:

```text
show_id
```

This is attractive because seat operations are naturally show-scoped.

But a blockbuster show can become a hot shard.

Possible mitigation:

- Admission control
- Dedicated capacity for hot shows
- Logical partitioning
- Queueing
- Avoid cross-show transactions

---

# 22. Read/Write Separation

Browse traffic:

```text
200K RPS
```

Booking traffic:

```text
20K/sec
```

Do not let browse traffic overload the booking database.

Use:

```text
Catalog
  |
  +--> CDN
  +--> Redis
  +--> Read replicas
```

while booking inventory uses a protected write path.

---

# 23. Cache Stampede

Popular show cache expires:

```text
10K requests
     |
     v
same cache MISS
     |
     v
DB overload
```

Use:

- TTL jitter
- Single-flight
- Request coalescing
- Background refresh

---

# 24. Failure Scenarios

| Failure | Strategy |
|---|---|
| Redis down | Read from DB / degraded browse |
| Booking DB down | Stop accepting bookings rather than corrupt inventory |
| Payment timeout | Query payment status before retry |
| Payment succeeds, booking fails | Reconcile/retry confirmation |
| Hold expiry worker down | Recovery sweep + expiry validation |
| Duplicate booking request | Idempotency |
| Duplicate payment webhook | Idempotent handler |
| User disconnects | Hold remains until expiry |
| Region failure | Preserve booking ownership / failover carefully |
| Kafka down | Outbox buffers events |

---

# 25. Multi-Region

Seat inventory creates a strong ownership problem.

Avoid:

```text
Region A -> seat A1
Region B -> seat A1
```

concurrently.

Possible design:

```text
show_id -> home region
```

All booking mutations for a show route to its owning region.

Reads can be served more broadly.

---

# 26. Consistency

Browse:

```text
eventually consistent
```

Booking:

```text
strong consistency
```

Payment:

```text
strong financial correctness
```

Notifications:

```text
eventual
```

This is an important interview trade-off.

---

# 27. Observability

Track:

```text
Seat hold success rate
Booking success rate
Payment success rate
Booking latency
DB lock wait time
Hot show traffic
Queue depth
Hold expiry lag
Kafka lag
```

Business metrics:

```text
Tickets sold
Conversion rate
Payment failure rate
Inventory mismatch
```

---

# 28. Reconciliation

Run reconciliation between:

```text
Booking DB
     |
     +
Payment provider
```

Detect:

- Payment succeeded but booking missing
- Booking confirmed but payment missing
- Duplicate payment
- Incorrect amount
- Expired hold incorrectly retained

---

# 29. Interview Traps

### Trap 1

> "Check availability in Redis and then book."

Ask:

> What prevents two users from booking the same seat?

### Trap 2

> "Use Redis lock."

Ask:

> What happens if the lock expires while the original process is still running?

### Trap 3

> "Payment success means booking success."

Ask:

> What if booking DB fails immediately after payment?

### Trap 4

> "Use read replicas for seat availability."

Ask:

> Can stale availability be used for final booking?

Answer: no.

---

# 30. Staff-Level Trade-offs

The key split:

```text
Browsing
   |
   v
Highly scalable + eventually consistent

Booking
   |
   v
Strongly consistent + protected capacity
```

The system should not optimize browsing at the expense of inventory correctness.

---

# 31. Interview Follow-Ups

1. How would you prevent double booking?
2. Why is Redis not enough?
3. How would you implement five-minute holds?
4. What if payment succeeds but booking confirmation fails?
5. How would you handle a blockbuster release?
6. How would you shard seat inventory?
7. How would you fail over between regions?
8. What happens if the expiry worker is down for one hour?
9. How would you recover from inventory corruption?
10. Would you use pessimistic or optimistic locking and why?

---

# 32. 30-Second Answer

> I would separate catalog browsing from the strongly consistent booking path. Movie and show metadata can be heavily cached, while seat inventory is stored in a transactional system partitioned primarily by show. Booking uses an atomic conditional state transition from AVAILABLE to HELD, with a five-minute expiry and an idempotent booking key. Payment is handled independently with its own idempotency key, and payment success followed by booking failure is resolved through retries and reconciliation rather than charging again. Popular shows require admission control and per-show concurrency limits to prevent a hot show from overwhelming inventory storage. Outbox events, Kafka, expiry workers, observability, and carefully scoped multi-region ownership complete the design.
