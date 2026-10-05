# 🔴 P0 — Design Distributed Rate Limiter

> Interview Level: SDE-3 / Staff  
> Difficulty: ⭐⭐⭐⭐⭐  
> Interview Time: 45–60 minutes  
> Core Topics: Distributed Counters, Token Bucket, Sliding Window, Redis, Atomicity, Hot Keys, Backpressure, Multi-Region

---

# 1. Problem Statement

Design a distributed rate limiter that protects APIs from excessive traffic.

It should support limits such as:

```text
100 requests / second / user
10,000 requests / minute / merchant
1,000,000 requests / hour / API
```

The rate limiter must:

- Work across many application instances
- Enforce limits consistently enough for the business requirement
- Add very low latency
- Scale horizontally
- Handle Redis/node failures
- Avoid becoming a single point of failure
- Support multiple rate-limit dimensions

---

# 2. Clarifying Questions

Ask:

1. What are we limiting by?
2. User, IP, API key, tenant, endpoint?
3. Fixed or rolling windows?
4. Burst traffic allowed?
5. Global or per-region limits?
6. Fail-open or fail-closed?
7. Expected RPS?
8. Required decision latency?
9. How accurate must enforcement be?
10. What happens if the rate-limit store is unavailable?

### Sample assumptions

```text
Global traffic        = 1M RPS
Peak                  = 5M RPS
Decision latency      = < 5 ms
Instances             = 1,000+
Regions               = 3
Keys                  = 100M+
Typical limit         = 100 req/sec/key
Burst                 = allowed
Availability          = 99.99%
```

---

# 3. Requirements

## Functional

```text
ALLOW
DENY
remaining quota
retry-after
```

Example:

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 23
X-RateLimit-Reset: 1700000000
```

When blocked:

```http
429 Too Many Requests
```

---

# 4. High-Level Architecture

```text
Client
  |
  v
API Gateway
  |
  v
Rate Limiter
  |
  +--> Local cache
  |
  +--> Redis Cluster
  |
  +--> Configuration Store
```

At large scale:

```text
                  Global Traffic
                       |
               Global Load Balancer
                       |
          +------------+------------+
          |                         |
       Region A                  Region B
          |                         |
      Gateway                    Gateway
          |                         |
      RL Nodes                   RL Nodes
          |                         |
          v                         v
      Redis A                    Redis B
```

---

# 5. Where Should Rate Limiting Happen?

Possible layers:

```text
CDN
 |
API Gateway
 |
Service
 |
Database
```

Rate limiting should generally happen as close to the edge as possible for broad protection.

But business-specific limits may belong closer to the application.

Example:

```text
IP limit       -> Edge
API-key limit  -> Gateway
Payment limit  -> Payment service
```

---

# 6. Algorithm 1 — Fixed Window

Example:

```text
10:00:00 - 10:00:59
```

Allow:

```text
100 requests
```

Counter:

```text
rate:user123:10:00
```

### Problem

Boundary burst:

```text
10:00:59 -> 100 requests
10:01:00 -> 100 requests
```

Potentially:

```text
200 requests in ~1 second
```

---

# 7. Algorithm 2 — Sliding Window Log

Store timestamps:

```text
t1
t2
t3
...
```

Remove timestamps older than the window.

Accurate but memory-intensive.

At:

```text
5M RPS
```

storing every request timestamp becomes expensive.

---

# 8. Algorithm 3 — Sliding Window Counter

Approximate a rolling window using counters for adjacent windows.

Example:

```text
Current window
Previous window
```

Weighted calculation:

```text
estimated =
previous_count * overlap
+
current_count
```

Lower memory than a full timestamp log.

---

# 9. Algorithm 4 — Token Bucket

This is often a strong default when bursts are allowed.

Parameters:

```text
capacity = 100
refill = 100 tokens/sec
```

Request:

```text
1 request
=
1 token
```

If a token exists:

```text
ALLOW
```

Otherwise:

```text
DENY
```

---

# 10. Token Bucket Behavior

Suppose:

```text
capacity = 100
refill = 10/sec
```

If the bucket has:

```text
100 tokens
```

a burst of 100 requests can pass immediately.

After that:

```text
10 tokens/sec
```

arrive.

This separates:

```text
average rate
```

from:

```text
maximum burst
```

---

# 11. Redis Implementation

A naive design:

```text
GET counter
INCR
SET expiry
```

is not safe because multiple clients can race.

Use atomic server-side operations, such as:

```text
Lua script
```

or another atomic primitive.

The complete decision:

```text
read state
+
calculate refill
+
consume token
+
write state
```

should be atomic.

---

# 12. Token Bucket State

Conceptually:

```text
key:
rate:{user_id}

tokens
last_refill_timestamp
```

On request:

```text
elapsed = now - last_refill

new_tokens =
min(
  capacity,
  tokens + elapsed * refill_rate
)

if new_tokens >= 1:
    tokens = new_tokens - 1
    ALLOW
else:
    DENY
```

This calculation must be performed atomically.

---

# 13. Hot Key Problem

Suppose:

```text
Celebrity API key
```

receives:

```text
100K RPS
```

All requests hit:

```text
rate:celebrity
```

One Redis shard can become a hot key.

Mitigations:

- Local rate limiting
- Hierarchical rate limiting
- Key partitioning where semantics allow
- Dedicated capacity
- Per-node token buckets
- Approximate enforcement

Do not blindly shard one counter because doing so changes the global limit semantics.

---

# 14. Local + Global Rate Limiting

A scalable pattern:

```text
Request
   |
   v
Local limiter
   |
   +--> reject immediately
   |
   v
Distributed quota
```

Example:

```text
Global = 1,000 req/sec
10 gateway nodes

Allocate approximately:
100 req/sec/node
```

Local decisions avoid hitting Redis for every request.

But allocations must account for:

- Uneven traffic
- Node churn
- Autoscaling
- Burstiness

---

# 15. Hierarchical Rate Limiting

Example:

```text
                    Global
                      |
              1M requests/sec
                      |
          +-----------+-----------+
          |                       |
       Region A                Region B
          |                       |
      Tenant limit           Tenant limit
          |
       User limit
          |
       Endpoint
```

This can reduce centralized contention.

---

# 16. Multiple Dimensions

A request may need:

```text
IP limit
AND
User limit
AND
API-key limit
AND
Endpoint limit
```

Example:

```text
IP      = 1000/min
User    = 100/min
API key = 10K/min
Payment = 10/sec
```

The request is allowed only if all required policies pass.

Avoid partially consuming quota and then failing another dimension without compensating carefully.

---

# 17. Policy Configuration

Do not hardcode:

```text
100 req/sec
```

in application code.

Use configuration:

```text
policy:
  tenant=123
  endpoint=/payments
  limit=100
  refill=100/sec
  burst=200
```

Configuration should be versioned and distributed safely.

---

# 18. Fail-Open vs Fail-Closed

Critical question:

> What happens when Redis is unavailable?

### Fail-open

Allow traffic.

Pros:

- Better availability
- Less user impact

Cons:

- Abuse can overload downstream systems

### Fail-closed

Reject traffic.

Pros:

- Strong protection

Cons:

- Rate limiter outage becomes application outage

---

# 19. Hybrid Failure Strategy

A practical approach:

```text
Redis unavailable
      |
      v
Local limiter
      |
      +--> conservative fallback
```

For security-sensitive endpoints:

```text
Fail closed
```

For less critical public APIs:

```text
Fail open / degraded
```

The decision is business-specific.

---

# 20. Redis Failure

If one Redis node fails:

```text
Redis Cluster
     |
     +--> shard failover
```

But if the whole regional Redis cluster fails:

```text
Region
   |
   X
```

the rate limiter needs a regional fallback.

Possible:

```text
Local token bucket
+
Regional quota
+
Conservative limits
```

---

# 21. Multi-Region Rate Limiting

Global limit:

```text
1M requests/sec
```

Regions:

```text
A = 500K
B = 300K
C = 200K
```

There are two broad strategies.

### Centralized global counter

Pros:

- Accurate global limit

Cons:

- Cross-region latency
- Dependency on central service
- Reduced availability

### Regional quotas

Allocate budgets to regions.

Pros:

- Low latency
- High availability

Cons:

- Approximate global enforcement
- Unused quota can create inefficiency

---

# 22. Clock Problems

Distributed rate limiting often depends on time.

Problems:

```text
Node A clock = 10:00:00.100
Node B clock = 09:59:59.900
```

Avoid assuming perfect clock synchronization.

For centralized Redis token-bucket calculations, server-side time or consistent time semantics can reduce client clock differences.

---

# 23. Rate Limiter Placement

For a payment system:

```text
Internet
   |
   v
API Gateway
   |
   +--> global rate limit
   |
   v
Payment Service
   |
   +--> payment-specific limit
   |
   v
DB / Processor
```

This gives layered protection.

---

# 24. Backpressure Relationship

Rate limiting is one part of downstream protection.

```text
Rate Limit
    |
    v
Concurrency Limit
    |
    v
Bounded Queue
    |
    v
Timeout
    |
    v
Circuit Breaker
```

Example:

```text
DB can safely process 5K concurrent operations
```

Do not allow:

```text
100K concurrent requests
```

just because the rate limiter allows 100K RPS.

---

# 25. Retry Interaction

Bad design:

```text
Request -> 429
      |
      v
Client immediately retries
      |
      v
More 429
      |
      v
Traffic amplification
```

Return:

```http
429
Retry-After: 2
```

Clients should use:

```text
exponential backoff
+
jitter
```

---

# 26. Distributed Counter Consistency

A distributed rate limiter does not necessarily need perfect linearizability.

Ask:

> What is the business requirement?

For:

```text
security login attempts
```

stricter enforcement may be necessary.

For:

```text
analytics API
```

a small amount of over-admission may be acceptable.

This is a key staff-level trade-off.

---

# 27. Storage Model

Conceptual Redis state:

```text
rate:{policy_key}
-----------------------
tokens
last_refill
policy_version
```

For fixed window:

```text
rate:{key}:{window}
-----------------------
count
TTL
```

Keep TTLs aligned with the policy.

---

# 28. Configuration Propagation

Suppose policy changes:

```text
100 req/sec
     ->
10 req/sec
```

All gateway nodes must eventually receive the new policy.

Possible:

```text
Configuration Store
       |
       v
Pub/Sub / Kafka
       |
       v
Gateway local caches
```

Version policies to avoid stale configurations overwriting newer ones.

---

# 29. Observability

Track:

```text
Allowed requests
Rejected requests
Rate-limit decision latency
Redis latency
Redis errors
Hot keys
Policy distribution
Fallback decisions
```

Business metrics:

```text
Blocked abusive traffic
False-positive rate
Customer throttling
Quota utilization
```

---

# 30. Important Alerts

```text
Redis latency > threshold

Rate limiter error rate > threshold

Fallback rate > threshold

One key consumes abnormal quota

429 rate spikes

Decision latency > SLO

Redis shard CPU saturation
```

---

# 31. Testing

Test:

- Concurrent requests
- Boundary windows
- Burst traffic
- Redis failover
- Node restart
- Clock skew
- Policy changes
- Hot keys
- Region failure
- Autoscaling
- Duplicate/replayed requests

Load-test at:

```text
1x
2x
5x
10x
```

expected traffic.

---

# 32. Interview Traps

### Trap 1

> Use Redis INCR.

Ask:

> What happens when GET and INCR are not atomic?

### Trap 2

> Use one global counter.

Ask:

> What happens at 5M RPS and three regions?

### Trap 3

> Just shard the counter.

Ask:

> Does sharding change the semantics of the global limit?

### Trap 4

> Fail closed.

Ask:

> What happens if your rate limiter itself becomes unavailable?

### Trap 5

> Use local counters everywhere.

Ask:

> How do you enforce a global tenant limit?

---

# 33. Staff-Level Trade-offs

The key dimensions are:

```text
Accuracy
   vs
Latency

Centralization
   vs
Availability

Global enforcement
   vs
Regional independence

Burst tolerance
   vs
Downstream protection

Strict limits
   vs
False positives
```

The correct algorithm depends on the business semantics.

---

# 34. Interview Follow-Up Questions

1. Why Token Bucket over Sliding Window?
2. How do you make the Redis operation atomic?
3. What happens if Redis is down?
4. How do you handle a hot key?
5. How do you enforce a global limit across regions?
6. How do you allocate quota during autoscaling?
7. How do you handle policy changes?
8. How do you avoid retry storms after 429?
9. What happens when a node's local quota is exhausted but another node has unused quota?
10. How would you design rate limiting for a payment endpoint where double spending/abuse is high risk?

---

# 35. 30-Second Answer

> I would implement rate limiting at multiple layers. The gateway provides broad protection while services can apply business-specific limits. Token Bucket is a strong default when controlled bursts are required, with state maintained in a distributed store such as Redis and the refill/consume operation executed atomically. At very high scale, I would reduce centralized traffic using local or hierarchical quotas, accepting bounded approximation where the business allows it. Hot keys require special handling because blindly sharding a counter changes global-limit semantics. For multi-region limits, I would choose between centralized global coordination and regional quota allocation based on the required accuracy. Redis failure requires an explicit fail-open, fail-closed, or conservative local fallback policy. Observability must include rejected traffic, decision latency, hot keys, fallback rate, and downstream saturation.
