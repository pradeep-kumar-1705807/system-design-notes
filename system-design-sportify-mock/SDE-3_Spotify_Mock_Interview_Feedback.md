# SDE-3 System Design Mock Interview Feedback

## Problem
**Design Spotify — System Design Interview**

## Overall Assessment

Current performance is around **3.3/5 — Borderline / inconsistent SDE-3 level**.

The mock showed clear improvement in distributed-systems and failure-handling depth compared with the previous feedback. The strongest improvement was the ability to reason through a chain of failures instead of stopping at the initial architecture.

The main remaining gap is **precision + proactive communication**. You often identify a relevant technology or pattern, but only after the interviewer narrows the problem. At SDE-3 level, the expectation is to proactively explain:

> **Decision → Why → Alternative → Failure/Trade-off**

---

# Scorecard

| Area | Score | SDE-3 Target | Assessment |
|---|---:|---:|---|
| Requirements | 3.5/5 | 4.5 | Core requirements covered, but some required prompting |
| Capacity / Estimation | 3.5/5 | 4.5 | Good recovery from the initial QPS mistake |
| API Design | 3.5/5 | 4.5 | Reasonable endpoints and idempotency discussion |
| Data Modeling | 3/5 | 4.5 | Good playlist-song mapping direction, but limited depth |
| Database / Storage | 3/5 | 4.5 | PostgreSQL/read replicas reasonable; sharding/consistency depth limited |
| Caching | 4/5 | 4.5 | One of the strongest areas in this mock |
| Distributed Systems | 3.5/5 | 4.5 | Request coalescing, distributed locks, circuit breakers |
| Fault Tolerance | 3.5/5 | 4.5 | Noticeable improvement over previous mock |
| Concurrency | 3.5/5 | 4.5 | Good lock ownership reasoning; thread-exhaustion reasoning needs work |
| Trade-offs | 2.5/5 | 4.5 | Often named a mechanism without explaining why |
| Communication | 2.5/5 | 4.5 | Biggest remaining weakness |

---

# 1. Requirements

### What went well

You identified the main Spotify-style requirements:

- Play a song
- Recommendations
- Create/manage playlists
- Add/remove songs
- Follow artists/playlists
- Like songs
- Popular songs
- Search songs/playlists

You also identified useful NFRs:

- High availability
- Low latency
- Read-heavy workload
- Scalability
- Durable playlists/likes
- Eventual consistency for popularity/recommendations

### Improvement

At the beginning, you needed prompting to structure requirements.

### Target behavior

Start with:

> "I'll first clarify functional requirements, then NFRs, then scale assumptions."

Then explicitly separate:

```text
Functional
1.
2.
3.

Non-functional
1. Availability
2. Latency
3. Consistency
4. Scalability
```

---

# 2. Capacity Estimation

### What went well

You initially stated an incorrect very-high QPS number, but recovered by calculating:

```text
50M DAU
× 100 API requests/day
= 5B requests/day

5B / 86,400
≈ 58K average QPS

10× peak
≈ 580K peak API QPS
```

You also correctly separated:

```text
API/control-plane QPS
```

from:

```text
Audio streaming bandwidth
```

and moved audio delivery toward CDN/object storage.

### Improvement

Don't introduce arbitrary capacity limits such as:

> "50 RPS if cache/CDN is down"

without deriving them.

Instead:

```text
DB safe capacity
→ safety margin
→ allowed cache-miss traffic
→ rate limit
```

The interviewer should understand why the number exists.

---

# 3. API Design

### What went well

You produced a reasonable set of APIs:

```text
GET  /api/v1/songs/feed
GET  /api/v1/songs/{songId}

POST   /api/v1/playlists
PATCH  /api/v1/playlists/{playlistId}/songs
DELETE /api/v1/playlists/{playlistId}/songs/{songId}

POST /api/v1/songs/{songId}/like

GET /api/v1/playlists
GET /api/v1/playlists/{playlistId}/songs

GET /api/v1/playlists/search?q={query}
GET /api/v1/songs/search?q={query}
```

You also identified idempotency keys for POST APIs.

You correctly stated that the DB should remain the source of truth for correctness, while Redis can hold a cached copy of the response.

### Improvement

At SDE-3 level, add:

- Request/response examples
- Authentication
- Error semantics
- Pagination
- Idempotency behavior
- Versioning
- Concurrency behavior

For important APIs, explain the exact consistency expectation.

---

# 4. Data Modeling

### What went well

You moved away from embedding a large `Set<Songs>` directly inside a playlist and introduced:

```text
playlist_song
----------------
playlist_id
song_id
song_added_at
```

You also identified:

```text
UNIQUE(playlist_id, song_id)
```

and the query pattern:

```sql
WHERE playlist_id = ?
ORDER BY song_added_at DESC
```

This was a good direction.

### Improvement

Go deeper into:

- Primary key choice
- Composite indexes
- Pagination
- Large playlists
- Partitioning
- Hot playlists
- Replication
- Consistency
- Write amplification

For example, don't merely say:

> "I'll add an index."

Say:

> "The dominant query filters by playlist_id and orders by song_added_at, so the index should be designed around that access pattern."

---

# 5. Caching

## Score: 4/5

This was one of your strongest areas.

You discussed:

- CDN
- Redis
- Cache miss
- Cache stampede
- Request coalescing
- Distributed locking
- Lock timeout
- Lock ownership
- Rate limiting

The strongest answer was:

> Store a unique UUID/token as the Redis lock value and use an atomic compare-and-delete operation, such as a Lua script, before releasing the lock.

This demonstrates understanding of the important problem where:

```text
Instance A lock expires
        ↓
Instance B acquires lock
        ↓
Instance A finishes later
        ↓
A must NOT delete B's lock
```

### Improvement

Your biggest caching weakness was initially treating request coalescing as:

> "100K requests wait on application threads."

That can exhaust the thread pool.

You need to distinguish:

```text
Request coalescing
≠
100K blocked threads
```

The next level is bounded/asynchronous coordination where only one request performs the DB fetch and the rest do not independently overload the DB or consume unbounded server threads.

---

# 6. Distributed Locking

This was a strong part of the mock.

You correctly reasoned about:

```text
Lock TTL
Lock ownership
Unique token
Atomic release
```

The final formulation was strong:

```text
SET lock:key <unique-token> NX EX <ttl>
```

and release only when:

```text
current_value == my_token
```

with an atomic operation.

### Improvement

Be prepared to discuss:

- What happens when DB latency exceeds lock TTL
- Lock renewal/lease extension
- Duplicate DB fetches
- Whether duplicate fetches are acceptable
- What happens if Redis itself fails
- Whether locking is actually necessary for every workload

---

# 7. Fault Tolerance

## Score: 3.5/5

This showed significant improvement.

You covered:

```text
Redis failure
↓
Rate limiting

Cache stampede
↓
Request coalescing

Cross-instance cache miss
↓
Distributed lock

Replica timeout
↓
Circuit breaker

Replica unhealthy
↓
Route to healthy replicas

All replicas unhealthy
↓
Controlled writer fallback

Writer capacity exceeded
↓
Rate limiting / rejection
```

This is much stronger than simply saying:

> "Use Redis cluster and DB replicas."

### Improvement

For every major component, use:

```text
Failure
→ Detection
→ Immediate behavior
→ Recovery
→ User impact
```

For example:

```text
Replica timeout
→ circuit breaker detects repeated failures
→ remove replica from routing
→ route to healthy replicas
→ HALF_OPEN probes after cooldown
→ re-add if healthy
```

This should become automatic in your thinking.

---

# 8. Circuit Breaker

You initially mentioned monitoring/CloudWatch, but eventually reached the runtime mechanism:

```text
CLOSED
   ↓ repeated failures
OPEN
   ↓ cooldown
HALF_OPEN
   ↓ successful probes
CLOSED
```

You also recognized that normal requests should go to other healthy replicas while one replica's circuit is OPEN.

### Improvement

Do not require 100% of traffic to pass through during HALF_OPEN.

Think:

```text
OPEN
 ↓
small number of probe requests
 ↓
success → gradually restore
failure → OPEN again
```

And define the trigger using actual request behavior:

- Timeout rate
- Error rate
- Consecutive failures
- Latency threshold

---

# 9. Concurrency

## Score: 3.5/5

### Strong point

Distributed lock ownership was good.

### Weak point

Your initial response to 100K concurrent cache misses was:

> "100K requests will wait on application server."

Then:

> "configure thread pool with blocking queue and rejection policy."

This provides backpressure, but doesn't solve the fundamental problem.

The SDE-3 thought process should be:

```text
100K requests
      ↓
same key
      ↓
ONE DB fetch
      ↓
ONE cache population
      ↓
bounded number of waiters
      ↓
timeouts / rejection if necessary
```

The key question is:

> "How do I avoid 100K blocked application threads?"

This should become a major practice topic.

---

# 10. Database Failure / Read Replica Routing

You eventually arrived at:

```text
Read replica failure
→ circuit breaker
→ remove unhealthy replica
→ route to other replicas
→ HALF_OPEN health/probe
→ restore when healthy
```

This is the right direction conceptually.

### Improvement

Don't rely only on:

> "weighted round robin"

because CPU/memory may remain normal even while a replica is returning timeouts.

Routing should incorporate **request health**, not just infrastructure utilization.

---

# 11. Writer Fallback

You proposed sending some cache misses to the writer when all read replicas fail, with rate limiting.

You also recognized that if:

```text
Writer capacity = 20K QPS
Incoming cache misses = 100K QPS
```

then the remaining traffic must be rejected or otherwise shed rather than overwhelming the writer.

That is an important SDE-3 principle:

> Availability does not mean accepting unlimited traffic.

### Improvement

State the degradation policy explicitly:

```text
Healthy path:
CDN → Redis → replicas

Degraded path:
Redis → replicas

Severe degradation:
writer with strict admission control

Beyond capacity:
load shed / reject / fail fast
```

---

# 12. Communication

## Score: 2.5/5

This remains the largest improvement area.

Your technical knowledge is often better than your verbal presentation.

Examples of short answers:

> "all req to writer"

> "we can use circuit breaker"

> "it will handled by other read replicas"

These force the interviewer to ask multiple follow-ups.

### Replace this:

> "Circuit breaker."

### With:

> "I'll use a circuit breaker per read replica. Repeated timeouts will open the circuit and remove that replica from routing. After a cooldown, I'll allow a small number of HALF_OPEN probes and restore it only after successful responses."

Same technical knowledge.

Much stronger SDE-3 communication.

---

# 13. Your New Answer Framework

For every major design decision, practice saying:

## Decision → Why → Alternative → Failure

Example:

> **Decision:** I'll use Redis for metadata caching.  
> **Why:** The workload is heavily read-oriented and we need low latency at high QPS.  
> **Alternative:** Direct PostgreSQL reads are simpler but don't provide enough headroom at peak traffic.  
> **Failure:** If Redis fails, I'll use bounded DB reads and load shedding rather than allowing unlimited cache misses.

This single structure will improve:

- Communication
- Trade-offs
- Fault tolerance
- Technical depth

simultaneously.

---

# 14. Top 5 Areas to Practice

## Priority 1 — Capacity → Architecture

Don't just calculate:

```text
580K QPS
```

Immediately ask:

```text
Where does the 580K go?
How much does CDN absorb?
How much does Redis absorb?
How much reaches DB?
What's the DB capacity?
What happens at 10×?
```

---

## Priority 2 — Cache Stampede / Request Coalescing

Master:

```text
Cache miss
→ stampede
→ coalescing
→ distributed coordination
→ bounded wait
→ timeout
→ load shedding
```

---

## Priority 3 — DB Scaling

Practice:

```text
Read replicas
→ replication lag
→ partitioning
→ sharding
→ hot partitions
→ write bottleneck
→ cross-shard queries
→ failover
```

---

## Priority 4 — Failure Handling

For every component:

```text
Client
Gateway
Application
Redis
Kafka
DB
Read Replica
CDN
```

ask:

```text
How does it fail?
How do I detect it?
What happens immediately?
How do I recover?
What does the user see?
```

---

## Priority 5 — Speaking Structure

Before answering, spend **2–3 seconds organizing the answer**.

Then speak:

> "I'll handle this in three steps..."

This will be much better than starting with:

> "So... we can... maybe use..."

---

# Final Takeaway

Your **technical foundation is stronger than your current interview communication makes it appear**.

The next improvement should not be learning another 50 system-design components.

Focus on making your existing knowledge more structured:

```text
Requirement
    ↓
Scale
    ↓
Decision
    ↓
WHY
    ↓
Alternative
    ↓
Failure
    ↓
Recovery
    ↓
Trade-off
```

Your goal for the next mock should be to move from:

**"I know the technology"**

to:

**"I can independently defend the architecture."**
