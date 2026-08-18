# System Design Interview Framework

## Core Framework

Memorize:

> **R → S → A → D → H → F → C → T**

**Requirements → Scale → APIs → Data → HLD → Failure → Concurrency → Trade-offs**

Then add:

> **10× Scale → Dynamic Requirement → LLD → Summary**

---

## 1. Opening — Take Control

Start with:

> "I'll first clarify the functional and non-functional requirements, then do a quick capacity estimate. After that I'll define the APIs and data model, build the high-level architecture, and finally we'll look at scaling, failures and concurrency."

This directly addresses the communication weakness identified in the previous mock: hesitation, circular explanations, and lack of a crisp narrative. fileciteturn2file2

---

## 2. Requirements — ~5 Minutes

Ask only questions that can change the architecture.

### Functional

```text
Who?
 ↓
What operation?
 ↓
Critical flow?
 ↓
Out of scope?
```

### Non-functional

```text
Scale
Latency
Availability
Consistency
Durability
Security
Geo
```

Then summarize:

> "So I'll optimize primarily for high read throughput, p95 under 100ms and high availability. Analytics will be asynchronous and out of scope for the critical path."

Then:

> "I'll proceed with these assumptions."

Do not keep requirements open indefinitely.

---

## 3. Capacity Estimation — ~5 Minutes

This is a major improvement area from your previous interview. You stated traffic but did not consistently convert it into QPS and architectural decisions. fileciteturn2file2

Always think:

```text
Daily Requests
      ↓
Average QPS
      ↓
Peak QPS
      ↓
Read / Write Ratio
      ↓
Storage
      ↓
Bandwidth
```

### Formula

```text
Average QPS = Requests/day / 86,400

Peak QPS = Average QPS × Peak Factor
```

Example:

```text
100M requests/day
        ↓
≈ 1,157 QPS average
        ↓
10× peak
        ↓
≈ 11.5K QPS
```

Then make the number useful:

> "At ~12K peak QPS, the stateless application tier can scale horizontally. I now need to check whether the database can sustain the write volume."

### Golden rule

> **Every calculation must lead to an architectural decision.**

---

## 4. API Design — ~5 Minutes

For every important API think:

```text
Method
Endpoint
Request
Response
Authentication
Authorization
Idempotency
Errors
Pagination
Versioning
Rate limiting
```

For every important write API:

> "What happens if the client retries?"

For concurrent operations:

> "What happens if two clients perform this operation simultaneously?"

Mental model:

```text
Normal Request
 ↓
Invalid Request
 ↓
Duplicate Request
 ↓
Concurrent Request
 ↓
Downstream Failure
```

---

## 5. Data Model — ~5 Minutes

Start with:

> "What are my main entities and what are the primary access patterns?"

Then:

```text
Entity
 ↓
Primary Key
 ↓
Important Fields
 ↓
Indexes
 ↓
Access Pattern
 ↓
Growth
 ↓
Partitioning
```

For every important query:

> "What query am I optimizing?"

Then:

> "What index supports that query?"

Then:

> "What happens when this table reaches 10 billion rows?"

Also consider lifecycle where relevant:

```text
Created
 ↓
Active
 ↓
Expired / Disabled
 ↓
Deleted / Archived
```

---

## 6. Database Decision Framework

Whenever choosing Postgres, DynamoDB, Redis, Kafka, Elasticsearch, etc.:

> **Requirement → Choice → Alternative → Trade-off**

Example:

> "I'll use Postgres because I need strong consistency and uniqueness constraints. A distributed KV store would give easier horizontal scaling, but at this scale I don't need that complexity. The trade-off is that the primary could eventually become a write bottleneck."

Avoid:

> "I'll use Postgres because it's reliable."

The previous feedback specifically identified insufficient explicit trade-off reasoning. fileciteturn2file2

---

## 7. High-Level Design

Start simple:

```text
Client
   ↓
API Gateway / Load Balancer
   ↓
Stateless Service
   ↓
Database
```

Add components only when requirements justify them:

```text
                 ┌── Redis
                 │
Client → LB → Service → Postgres
                 │
                 └── Queue → Worker → External Service
```

Do not start by assembling a technology stack.

Think:

> "What problem do I need to solve?"

Then introduce the component.

---

## 8. Defend Every Component

For every box you draw, ask five questions:

### 1. WHY?

Why do I need it?

### 2. ALTERNATIVE?

What could I use instead?

### 3. FAILURE?

What happens if it goes down?

### 4. SCALE?

What happens at 10×?

### 5. SECOND ORDER?

Does its failure overload something else?

This is one of the most important SDE-3 habits.

---

## 9. Cache Framework

Whenever you use a cache:

```text
What?
 ↓
Key?
 ↓
TTL?
 ↓
Eviction?
 ↓
Invalidation?
 ↓
Cache Miss?
 ↓
Hot Key?
 ↓
Stampede?
 ↓
Failure?
```

Example:

> "Redirect traffic is read-heavy and latency-sensitive, so I'll cache the URL mapping in Redis to reduce database reads."

Then test failure:

```text
Redis Failure
 ↓
DB Fallback
 ↓
DB QPS Increases
 ↓
DB Saturation?
 ↓
Latency Increases
 ↓
Timeouts?
 ↓
Retries?
```

Then consider:

```text
Bounded Timeouts
+
Bounded Retries
+
Exponential Backoff
+
Circuit Breaker
+
Load Shedding
```

This addresses the cascading-failure gap identified in the previous mock. fileciteturn2file2

---

## 10. Messaging Framework

When introducing Kafka or a queue:

```text
Why Async?
     ↓
Producer
     ↓
Delivery Semantics
     ↓
Consumer
     ↓
Duplicate
     ↓
Retry
     ↓
DLQ
     ↓
Ordering
     ↓
Backpressure
```

Always ask:

> "What happens if this message is delivered twice?"

Do not casually say:

> "Kafka gives exactly-once."

Explain where correctness is actually enforced.

---

## 11. Failure Framework

For every major component:

> **Failure → Detection → Impact → Fallback → Secondary Impact → Recovery**

Example:

```text
DB Primary Fails
      ↓
Health Failure Detected
      ↓
Failover
      ↓
Writes Temporarily Affected
      ↓
Application Reconnects
      ↓
Check Replication Lag / Possible Data Loss
      ↓
Resume Writes
```

Then ask:

> "What does the user experience?"

Depending on the system, consider:

- Multi-AZ
- Multi-region
- Health checks
- Failover
- Timeouts
- Retries
- Backoff
- Circuit breakers
- Load shedding
- Backpressure
- Replication
- Backups
- RPO
- RTO
- Disaster recovery

Do not blindly mention everything.

---

## 12. Concurrency Framework

For every important write operation ask:

> **"What if two requests happen simultaneously?"**

Examples:

```text
Two payments
Two bookings
Two workers
Two inventory updates
Two requests with same idempotency key
```

Possible mechanisms:

```text
Database Constraint
Transaction
Optimistic Lock
Pessimistic Lock
Atomic Operation
Idempotency
Queue Serialization
Distributed Lock
```

Prefer the simplest mechanism that guarantees correctness.

Do not automatically use distributed locks.

---

## 13. Scaling — The 10× Rule

When the interviewer says:

> "Traffic increased 10×."

Do not immediately redesign everything.

Say:

> "Let me identify which component becomes the bottleneck first."

Walk through:

```text
Traffic
 ↓
API
 ↓
Service CPU
 ↓
Cache
 ↓
DB Reads
 ↓
DB Writes
 ↓
Network
 ↓
Storage
```

Find the bottleneck.

Then change only what is necessary.

Your previous feedback specifically identified gaps around larger-scale behavior, DB partitioning/sharding, Redis hot keys, and application-layer scaling. fileciteturn2file2

---

## 14. Hot Key / Hot Partition

Example:

```text
Normal:
A → 10 requests
B → 20 requests
C → 15 requests

Hot:
X → 10M requests
```

Think:

```text
Skew
 ↓
Hot Key
 ↓
Single Partition Overloaded
```

Possible solutions depend on the architecture:

- Replication
- Key spreading
- Local cache
- Request coalescing
- Partition strategy
- Workload isolation

Do not automatically say "shard."

Explain why it solves the bottleneck.

---

## 15. Dynamic Requirement

When the interviewer changes the requirement:

> "Now we're going global."

Use:

```text
Current Architecture
        ↓
What assumption changed?
        ↓
What breaks?
        ↓
What must change?
        ↓
What can remain?
```

Strong response:

> "I don't need to redesign the entire system. The new global requirement primarily affects data locality and traffic routing."

This demonstrates architectural maturity.

---

## 16. LLD Transition

At the end:

> "I'll take the X component and go one level deeper."

Then:

```text
Interface
 ↓
Classes
 ↓
Responsibilities
 ↓
Methods
 ↓
State
 ↓
Concurrency
 ↓
Persistence
 ↓
Extension
```

For Java:

```java
interface PaymentProvider {
    PaymentResult charge(PaymentRequest request);
}
```

Then:

```text
StripeProvider
RazorpayProvider
AdyenProvider
```

Ask:

> "If a fourth provider is added, what changes?"

This tests extensibility rather than pattern memorization.

---

## 17. Final 60-Second Summary

Always close with:

> "To summarize, the system is optimized for ___. Requests enter through ___. The core service handles ___. ___ is used for ___, while ___ remains the source of truth. Asynchronous work is handled through ___. The system scales horizontally at ___. For failures, we handle ___ using ___. The major trade-off is ___ in exchange for ___."

This gives the interviewer a clean mental model.

---

# Complete Interview Cheat Sheet

```text
                    SYSTEM DESIGN
                          ↓
                       PROBLEM
                          ↓
                 1. REQUIREMENTS
                          ↓
              What are we optimizing?
                          ↓
                     2. SCALE
                          ↓
                 QPS / Storage / BW
                          ↓
                      3. APIs
                          ↓
           Retry / Idempotency / Errors
                          ↓
                     4. DATA
                          ↓
          Access Pattern / Index / Growth
                          ↓
                      5. HLD
                          ↓
            Why? Alternative? Trade-off?
                          ↓
                   6. FAILURE
                          ↓
       Failure → Impact → Fallback → Recovery
                          ↓
              7. SECOND-ORDER FAILURE
                          ↓
                What does failure cause?
                          ↓
                  8. CONCURRENCY
                          ↓
             What if two happen together?
                          ↓
                      9. SCALE
                          ↓
                     10× / 100×
                          ↓
                Find the bottleneck
                          ↓
               10. DYNAMIC CHANGE
                          ↓
                What assumption changed?
                          ↓
                       11. LLD
                          ↓
              Interface → Class → State
                          ↓
                    12. SUMMARY
```

---

# 18. Personal Priority for Your Next 5 Interviews

Based on your actual mock feedback:

## Priority 1 — Communication

Previous: **2/5**

Target: **4/5**

Practice:

- Start with structure.
- Stop circling back.
- Make decisions clearly.
- Summarize sections.
- Explicitly state trade-offs.

## Priority 2 — Scalability

Previous: **3/5**

Target: **4/5**

Practice:

- QPS calculation
- Read/write ratio
- Storage estimation
- 10× / 100× analysis
- DB bottlenecks
- Partitioning
- Sharding
- Hot keys
- Application-layer scaling

## Priority 3 — Fault Tolerance

Previous: **3/5**

Target: **4/5**

Practice:

> Failure → Impact → Fallback → Second-order failure → Recovery

Also practice:

- Multi-AZ
- Multi-region
- Timeouts
- Retries
- Backoff
- Circuit breakers
- Graceful degradation
- RPO/RTO

## Priority 4 — Trade-offs

For every major decision:

> Why this?

> Why not X?

> What trade-off am I accepting?

## Priority 5 — Technical Depth

Focus on:

- Key generation
- Partitioning
- Hot keys
- Cache stampede
- TTL / invalidation
- Idempotency
- Retries
- Partial failures
- Multi-AZ / multi-region
- RPO / RTO

These priorities map directly to the weaknesses identified in your actual mock rather than generic system-design preparation. fileciteturn2file2

---

# 19. Practice Loop

For every system-design problem:

```text
Design
  ↓
Record yourself
  ↓
Review
  ↓
Identify 3 gaps
  ↓
Redo the design
  ↓
Run failure drill
  ↓
Run 10× scaling drill
  ↓
Redo under pressure
```

The objective is not to memorize architectures.

The objective is to make this reasoning automatic:

```text
Requirement
     ↓
Quantify
     ↓
Identify bottleneck
     ↓
Choose design
     ↓
Explain trade-off
     ↓
Failure
     ↓
Second-order failure
     ↓
Concurrency
     ↓
10× scale
     ↓
Dynamic requirement
     ↓
LLD
```
