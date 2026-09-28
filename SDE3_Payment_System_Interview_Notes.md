# SDE-3 System Design Interview Notes — Payment / Merchant Transaction System

## Final Rating

**Overall: 4.3 / 5 — Strong SDE-3 / borderline 4.5**

The main improvement area is not learning more patterns. It is presenting the reasoning proactively and systematically:

> **State → Invariant → Decision → Failure → Recovery → Scale / Operational consequence**

---

# 1. Overall Scorecard

| Area | Score | Key feedback |
|---|---:|---|
| Requirements & Scoping | 4.2/5 | Identifies functional and NFR concerns well |
| Architecture | 4.2/5 | Good decomposition across Gateway, Payment Service, DB, Kafka, providers |
| Distributed Systems | 4.4/5 | Strong idempotency, UNKNOWN-state, retry and callback reasoning |
| Financial Correctness | 4.5/5 | Strongest area; understands ledger atomicity and duplicate financial effects |
| Failure Handling | 4.5/5 | Strong provider timeout, reconciliation and recovery reasoning |
| Concurrency | 4.2/5 | Good `SKIP LOCKED`, claiming and bulkhead understanding |
| Scalability | 4.0/5 | Good hot-merchant reasoning; diagnose before sharding |
| Operational Thinking | 4.2/5 | Good circuit breaker, bulkhead, rate-limit thinking |
| Data Modeling | 4.2/5 | Good separation of payment, provider attempt, ledger and idempotency |
| Communication | 4.0/5 | Strong technical knowledge; make the decision tree more explicit |

---

# 2. Core Payment System Model

A useful high-level flow:

```text
Customer / Merchant
        |
        v
   App Gateway
   - Auth/AuthZ
   - Rate limiting
        |
        v
 Payment Service
   - Order/payment creation
   - Idempotency
   - Provider routing
   - Payment state
        |
        +--------------------+
        |                    |
        v                    v
 PostgreSQL              Provider(s)
 Source of Truth              |
        ^                     |
        |                     v
        +------------- Callback / Status
                      |
                    Kafka
                      |
                Payment Service
```

Important principle:

> **PostgreSQL is the source of truth. Redis is an acceleration layer, not the financial source of truth.**

---

# 3. Idempotency

## Idempotency Key

The idempotency key protects against duplicate delivery of the **same logical API request**.

Typical model:

```text
merchant_id + idempotency_key
        |
        v
durable unique constraint
```

Example:

```sql
UNIQUE (merchant_id, idempotency_key)
```

If the same key is received again:

```text
Customer retry
     |
     v
Idempotency lookup
     |
     +---- existing transaction ---> return existing result/status
     |
     +---- no transaction ----------> create transaction
```

### Important distinction

> **Customer retry does not necessarily mean provider retry.**

A customer may retry the API request while the system returns the existing payment's `PENDING` / `UNKNOWN` state.

---

# 4. Idempotency Key vs Business Identity

These are different concepts.

### Idempotency key

Protects against duplicate delivery of the same logical API request.

### Business/payment identity

Identifies the actual payment/business operation.

This distinction matters because:

```text
Same idempotency key
    -> same logical request

Different idempotency key
    -> does NOT automatically mean a safe second charge
```

For financial systems, don't deduplicate purely using:

- amount
- customer
- merchant
- timestamp

Two legitimate payments can have identical values.

---

# 5. Payment State vs Provider Attempt

A useful mental model is:

```text
Logical Payment
      |
      +---- Provider Attempt A
      |
      +---- Provider Attempt B
```

But an UNKNOWN attempt requires special handling.

Example:

```text
Provider A
   |
request sent
   |
timeout
   |
UNKNOWN
```

Do **not** blindly do:

```text
UNKNOWN -> Provider B
```

because Provider A may already have charged the customer.

The safe sequence is:

```text
UNKNOWN
   |
   +--> Provider status API
   |
   +--> Callback / webhook
   |
   +--> Reconciliation
   |
   v
Terminal outcome
   |
   +--> SUCCESS
   |
   +--> FAILED
```

Only after the financial outcome is sufficiently established should another provider attempt become eligible according to the business recovery policy.

---

# 6. PENDING vs UNKNOWN vs FAILED

Do not treat every timeout as FAILED.

### PENDING

Payment is still expected to progress.

### UNKNOWN

The system cannot determine whether the provider-side financial operation succeeded.

### FAILED

There is authoritative evidence that the payment did not succeed.

Key rule:

> **A timeout is not automatically a failure.**

---

# 7. Redis + PostgreSQL

Redis can be used for:

- fast idempotency lookup
- cached payment status
- low-latency reads

PostgreSQL remains the durable source of truth.

Example:

```text
Request
   |
   v
Redis
   |
   +-- HIT --> return cached status
   |
   +-- MISS --> PostgreSQL
```

Redis entries can expire.

For important unresolved payments:

```text
Redis MISS
    |
    v
PostgreSQL
    |
    v
PENDING / UNKNOWN
```

Do not make correctness depend on Redis durability.

---

# 8. Financial Correctness: Payment + Ledger

Bad design:

```text
1. UPDATE payment = COMPLETED
2. INSERT ledger
3. crash between them
```

This can create:

```text
Payment = COMPLETED
Ledger  = missing
```

## Correct approach when both are in the same database

Use one PostgreSQL transaction:

```sql
BEGIN;

UPDATE payments
SET status = 'COMPLETED'
WHERE payment_id = 'P123';

INSERT INTO merchant_ledger
    (payment_id, amount, ...)
VALUES
    ('P123', 1000, ...);

COMMIT;
```

Either:

```text
BOTH COMMIT
```

or:

```text
BOTH ROLLBACK
```

Therefore the inconsistent intermediate state cannot be committed.

---

# 9. Ledger Idempotency

A database transaction alone does not prevent duplicate callbacks from producing duplicate financial effects.

Use database-level uniqueness.

For example:

```sql
UNIQUE (payment_id, entry_type)
```

or another immutable business reference appropriate to the ledger model.

Conceptually:

```text
Provider SUCCESS
      |
      v
Payment transition + ledger insert
      |
      v
DB commit
```

If the same event is redelivered:

```text
Duplicate event
      |
      v
Existing state / unique constraint
      |
      v
No second financial effect
```

Important principle:

> **State-transition idempotency is not sufficient; financial-effect idempotency is also required.**

---

# 10. Kafka Redelivery

Kafka can redeliver when:

```text
DB COMMIT succeeds
       |
consumer crashes
       |
Kafka ACK not sent
       |
message redelivered
```

Therefore:

```text
Kafka exactly-once delivery
```

should not be treated as the sole guarantee for financial correctness.

Protect the effect at the database level using:

- conditional state transitions
- unique event IDs
- unique ledger/business references
- one DB transaction for related financial state

Example conditional transition:

```sql
UPDATE transactions
SET status = 'COMPLETED',
    updated_at = NOW()
WHERE transaction_id = :transactionId
  AND status = 'PENDING';
```

Interpretation:

```text
1 row updated
    -> valid state transition

0 rows updated
    -> already processed / invalid transition
```

---

# 11. Transaction + Ledger + Outbox

Outbox is useful when a committed DB state needs to produce an event reliably.

Correct model when all belong to the same DB:

```text
BEGIN

UPDATE payment -> COMPLETED

INSERT ledger

INSERT outbox_event -> PAYMENT_COMPLETED

COMMIT
```

Then:

```text
Outbox Worker
      |
      v
Kafka / notification / downstream service
```

The outbox solves reliable event publication.

It does **not** replace the need for atomic payment + ledger state when those are in the same database.

---

# 12. Reconciliation

Reconciliation is a **safety net**, not the primary transaction mechanism.

Useful evidence includes:

- provider status API
- provider transaction ID
- provider settlement report
- webhook/callback
- internal payment state
- ledger state

Example:

```text
UNKNOWN payment
      |
      v
Provider status / settlement report
      |
      v
Resolve financial outcome
      |
      +--> SUCCESS
      |
      +--> FAILED
      |
      +--> initiate business-defined recovery
```

Avoid inventing an SLA such as “5 hours” unless the requirements specify it.

---

# 13. Scheduler / Batch Processing

A global distributed scheduler lock such as ShedLock can prevent multiple scheduler instances from executing the same scheduled job, but it can be too coarse for high-scale work.

Better pattern:

```sql
SELECT payment_id
FROM payments
WHERE status IN ('PENDING', 'UNKNOWN')
  AND next_check_at <= NOW()
ORDER BY next_check_at
LIMIT 1000
FOR UPDATE SKIP LOCKED;
```

Use short transactions to claim work.

### Important

Do **not** hold DB locks while making external provider calls.

Better:

```text
BEGIN
   claim rows
   set claimed_by
   set lease_until
COMMIT

       |
       v

external provider calls

       |
       v

update payment result
```

---

# 14. Lease-Based Recovery

If a worker claims 1,000 transactions and crashes after processing 500:

```text
1000 claimed
   |
   +--> 500 completed
   |
   +--> 500 still owned by crashed worker
```

Use:

```text
claimed_by
lease_until
```

When the lease expires:

```text
another worker
      |
      v
reclaims unfinished work
```

This is more robust than relying only on DB row locks.

Key distinction:

> **DB lock lifetime is not the same thing as business processing ownership.**

---

# 15. ShedLock vs Row Claiming

### ShedLock

Good for:

```text
Only one scheduler instance should execute a job.
```

But a global scheduler lock can serialize scheduling work.

### `FOR UPDATE SKIP LOCKED`

Good for:

```text
Many workers can safely claim different rows.
```

A scalable model is:

```text
Scheduler workers
   |
   +--> claim batch 1
   +--> claim batch 2
   +--> claim batch 3
```

rather than:

```text
One global lock
   |
   v
One scheduler does everything
```

---

# 16. Scheduler Batch Size vs Concurrency

Claiming:

```text
LIMIT 5000
```

does NOT mean:

```text
5000 concurrent provider calls
```

Separate:

- DB claim batch size
- worker concurrency
- provider rate limit
- timeout
- retry policy

Example:

```text
Claim 5000
     |
     v
Worker pool = 100
     |
     v
Provider rate limit = 500 RPS
```

These are independent controls.

---

# 17. Hot Merchant / Scalability

Suppose:

```text
50K requests/sec peak
30% writes
```

Therefore overall write traffic is approximately:

```text
50K × 30%
= 15K write requests/sec
```

But if one merchant generates:

```text
20K requests/sec
```

ordinary merchant-ID sharding can still create a hot shard.

### Do not immediately jump to sharding

First diagnose:

```text
Postgres saturation
       |
       +--> CPU?
       +--> I/O?
       +--> locks?
       +--> connections?
       +--> slow queries?
       +--> index contention?
       +--> hot rows?
       +--> WAL / commit latency?
```

Then choose the mitigation.

Possible responses:

```text
Bad query
    -> query/index optimization

Read overload
    -> caching / replicas

Connection exhaustion
    -> pool/workload isolation

Hot rows
    -> reduce contention / redesign access

Write capacity exhausted
    -> partitioning / sharding / batching / schema optimization
```

---

# 18. Hot Merchant Virtual Partitioning

For a genuinely hot merchant:

```text
Normal merchant
merchant_id -> one partition

Hot merchant
merchant_id + virtual_partition
        |
        +--> P0
        +--> P1
        +--> P2
        +--> P3
```

A partition can be selected using something like:

```text
hash(payment_id)
```

The exact mechanism depends on query patterns.

### Trade-off

Merchant-wide history queries can become:

```text
scatter -> gather -> merge
```

Example:

```text
Aggregator
  |
  +--> shard 1
  +--> shard 2
  +--> shard 3
  +--> shard 4
  |
  v
merge by created_at
  |
  v
paginated response
```

For large historical exports, asynchronous export can be more appropriate.

---

# 19. Callback Routing in a Sharded System

Once a merchant/payment spans multiple partitions, callbacks need deterministic routing.

Useful identifiers:

```text
payment_id
provider_attempt_id
provider_transaction_id
```

A routing layer can determine the owning partition.

Do not rely only on merchant ID if the hot merchant's transactions are spread across virtual partitions.

---

# 20. Provider Resilience

Suppose Provider A changes from:

```text
p99 = 200ms
```

to:

```text
p99 = 8 seconds
error rate = 25%
```

Use multiple layers.

### Bulkhead

Isolate provider resources:

```text
Payment Service
   |
   +--> Provider A → pool/limit A
   +--> Provider B → pool/limit B
   +--> Provider C → pool/limit C
```

Provider A cannot consume all shared resources.

### Circuit Breaker

Typical lifecycle:

```text
CLOSED
   |
failure threshold
   v
OPEN
   |
cooldown
   v
HALF-OPEN
   |
   +--> success -> CLOSED
   |
   +--> failure -> OPEN
```

### Other controls

- strict timeouts
- concurrency limits
- provider-specific rate limits
- bounded retries
- exponential backoff
- health metrics
- traffic routing

---

# 21. Correct Order of Provider Resilience

A useful flow:

```text
Incoming payment
      |
      v
Provider routing
      |
      v
Per-provider concurrency / bulkhead
      |
      v
Circuit breaker check
      |
      v
Strict timeout
      |
      v
Provider
      |
      v
metrics / failure signals
      |
      v
circuit/routing adjustment
```

The circuit breaker and bulkhead solve different problems:

> **Bulkhead protects resources.**

> **Circuit breaker protects the service from continuing to call an unhealthy dependency.**

---

# 22. Critical Payment-Specific Caveat

A circuit breaker opening does NOT mean:

```text
UNKNOWN Provider A payment
       ↓
Provider B
```

That can create a double charge.

Instead:

```text
New payment
   |
Provider A circuit OPEN?
   |
   +--> route to healthy provider

Existing UNKNOWN attempt
   |
   +--> status inquiry
   +--> callback
   +--> reconciliation
   |
   v
resolve financial outcome
```

Key invariant:

> **Availability mechanisms must not violate financial correctness.**

---

# 23. Important Distinctions

## Idempotency vs Circuit Breaker

```text
Idempotency
    -> prevents duplicate logical operations

Circuit breaker
    -> prevents calls to unhealthy providers
```

## Idempotency vs Reconciliation

```text
Idempotency
    -> prevents duplicate processing

Reconciliation
    -> resolves mismatches / unknown outcomes
```

## Payment State vs Ledger

```text
Payment state
    -> lifecycle status

Ledger
    -> financial record / effect
```

A completed payment does not by itself prove the ledger is correct unless the required DB atomicity exists.

## Event ID vs Provider Transaction ID

```text
Provider event ID
    -> identifies callback/event delivery

Provider transaction ID
    -> identifies provider-side transaction
```

Both can be useful for mapping and idempotency.

---

# 24. Common Mistakes to Avoid

### Mistake 1: Treating timeout as FAILED

Wrong:

```text
timeout -> FAILED -> retry another provider
```

Better:

```text
timeout -> UNKNOWN -> resolve outcome
```

### Mistake 2: Using Redis as the financial source of truth

Redis can disappear or expire.

Use PostgreSQL as the durable source of truth.

### Mistake 3: Holding DB locks during provider calls

Bad:

```text
BEGIN
lock row
call provider
wait 8 seconds
COMMIT
```

This increases lock contention.

### Mistake 4: Global scheduler lock at high scale

A single distributed lock can serialize work unnecessarily.

Use row/batch claiming where appropriate.

### Mistake 5: Assuming Kafka exactly-once guarantees financial exactly-once

Financial correctness should be enforced at the database/effect boundary.

### Mistake 6: Assuming circuit breaker automatically permits failover

Failover is safe for new operations, but not automatically safe for UNKNOWN payment attempts.

### Mistake 7: Sharding before diagnosing the bottleneck

First determine:

```text
CPU / I/O / lock / query / connection / hot row
```

Then choose the mitigation.

### Mistake 8: Saying “customer cannot retry”

The customer can retry the API request.

What the system prevents is:

> **The retry creating an unintended second financial operation.**

---

# 25. SDE-3 Answer Framework

For complex system-design questions, structure the answer as:

## 1. State

What state is the system currently in?

```text
P123 = UNKNOWN
```

## 2. Invariant

What must never be violated?

```text
Never create two financial effects for one logical payment.
```

## 3. Decision

What should the system do now?

```text
Do not create Provider B attempt yet.
```

## 4. Failure

What can fail?

```text
Provider timeout
consumer crash
DB crash
callback duplication
worker crash
```

## 5. Recovery

How do we recover?

```text
status API
reconciliation
lease expiry
idempotent replay
outbox
```

## 6. Scale / Operational Consequence

What happens under load?

```text
bulkheads
rate limits
batching
partitioning
hot-key handling
observability
```

---

# 26. Strong SDE-3 Phrases

Use these naturally when appropriate:

> “Let me establish the state and invariant first.”

> “A timeout is UNKNOWN, not necessarily FAILED.”

> “Customer retry and provider retry are separate concerns.”

> “State-transition idempotency is not enough; the financial effect also needs to be idempotent.”

> “I would diagnose the bottleneck before choosing sharding.”

> “I don't want to hold a DB lock while making an external provider call.”

> “The lease gives us crash recovery after the DB transaction has released its row locks.”

> “The outbox makes state change and event publication durable together; it doesn't replace payment-ledger atomicity.”

> “The circuit breaker protects dependency calls, but it cannot override the financial invariant.”

---

# 27. Final Interview Assessment

## Final score: **4.3 / 5**

### Strongest areas

- Financial correctness
- UNKNOWN-state reasoning
- Provider failure handling
- Idempotency
- Distributed failure modes
- Resilience patterns

### Main improvement area

**Proactive structured reasoning.**

Instead of starting with:

> “We can use Redis / Kafka / circuit breaker / sharding…”

start with:

```text
Current state
     ↓
Invariant
     ↓
Bottleneck / failure
     ↓
Decision
     ↓
Recovery
     ↓
Scale implications
```

Your technical knowledge is already close to the desired level. The biggest improvement now is making that reasoning visible **before the interviewer has to probe for it**.

---

# 28. Target for Next Mock

To move from **4.3 → 4.5+**, focus on:

1. Quantify before proposing architecture.
2. Diagnose before choosing a scalability mechanism.
3. State invariants explicitly.
4. Separate logical payment, provider attempt, and financial ledger effect.
5. Explain ownership and crash recovery for asynchronous workers.
6. Explain **signal → action → recovery** for operational mechanisms.
7. Avoid assumptions not present in the requirements.
8. Keep answers structured and concise before going deep.

## One-line mental model

> **State → Invariant → Decision → Failure → Recovery → Scale.**
