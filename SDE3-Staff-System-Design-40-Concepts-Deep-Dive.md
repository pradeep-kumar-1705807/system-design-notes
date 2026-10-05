# SDE-3 / Staff System Design — Deep-Dive Concepts

## Purpose

This document is a practical interview reference for three common distributed-system problems:

1. **Instagram / News Feed**
2. **BookMyShow / Seat Booking**
3. **Distributed Rate Limiter**

It explains the requested concepts with:

- What the concept means
- Why it exists
- A concrete example
- Design choices and trade-offs
- Failure modes
- What to say in an SDE-3 / Staff interview

---

# Part I — Instagram / News Feed

## 1. Fan-out on Write vs Fan-out on Read

### What is fan-out?

Fan-out means distributing one user's newly created post to the users who may consume it.

Suppose:

```text
Alice
  |
  | creates post P1
  v
Post Service
```

Alice has 1,000 followers.

The system can make P1 available to those followers in two ways.

---

## Fan-out on Write

At post creation time, push the post reference into followers' feed stores.

```text
Alice creates P1
      |
      v
Post Service
      |
      v
Kafka
      |
      v
Fanout Workers
      |
      +----> Bob feed
      +----> Carol feed
      +----> Dave feed
      +----> ...
```

For 1,000 followers:

```text
1 post
  -> 1,000 feed writes
```

### Read path

```text
GET /feed
   |
   v
Feed Service
   |
   v
Redis / Feed Store
   |
   v
Return precomputed feed
```

### Advantage

Read is cheap.

If 500K users request feeds per second, we do not have to reconstruct every feed from the follow graph.

### Disadvantage

Write amplification.

If a celebrity has:

```text
50,000,000 followers
```

one post could generate:

```text
1 post
-> 50M feed writes
```

That is unacceptable as a normal synchronous operation.

---

## Fan-out on Read

Do not push the post into follower feeds.

Instead, when Bob requests his feed:

```text
Bob
 |
 | follows Alice, Carol, Dave
 v
Feed Service
 |
 +--> Alice recent posts
 +--> Carol recent posts
 +--> Dave recent posts
 |
 v
Merge + Rank
 |
 v
Bob's feed
```

### Advantage

Post creation is cheap.

```text
Celebrity posts
      |
      v
Store one post
```

### Disadvantage

Read becomes expensive.

For a user following 1,000 accounts:

```text
1 feed request
-> potentially inspect many sources
-> merge
-> rank
```

Caching and precomputation are usually required.

---

## Comparison

| Dimension | Fan-out Write | Fan-out Read |
|---|---|---|
| Post write | Expensive | Cheap |
| Feed read | Cheap | Expensive |
| Celebrity handling | Bad | Good |
| Normal users | Excellent | More work |
| Storage writes | High | Lower |
| Read latency | Predictable | More variable |
| Freshness | Excellent | Depends on read path |

---

# 2. Hybrid Feed Architecture

In a real system, choose neither extreme globally.

Use a hybrid model.

### Typical strategy

```text
Normal user
    |
    | post
    v
Fan-out on write
```

For celebrity:

```text
Celebrity
    |
    | post
    v
Store post only
```

Then on feed read:

```text
User feed request
       |
       +----> Precomputed normal-user feed
       |
       +----> Celebrity recent posts
       |
       v
     Merge
       |
       v
     Rank
       |
       v
     Return
```

### Example

Bob follows:

```text
Alice       -> 2K followers
Carol       -> 5K followers
PopularUser -> 50M followers
```

Alice and Carol use fan-out-on-write.

PopularUser uses fan-out-on-read.

Bob's feed becomes:

```text
Precomputed Feed
+ 
PopularUser recent posts
+
Ranking
```

### Why hybrid is powerful

It handles both:

- high read volume
- extremely high follower counts

without creating massive write amplification.

### Interview answer

> "I would use a hybrid fan-out model. Normal users use fan-out-on-write for predictable low-latency reads. Celebrities or users above a follower threshold use fan-out-on-read, and the feed service merges their posts during reads."

---

# 3. Celebrity / Hot-Key Problem

A celebrity is a user with an unusually large follower count.

Example:

```text
Celebrity = 100M followers
```

One post can create:

```text
100M potential feed updates
```

This is a **hot-key / hot-partition / hot-user** problem.

### Another hot-key example

Suppose:

```text
GET /profile/celebrity
```

receives:

```text
200K RPS
```

A single Redis key may become extremely hot.

```text
profile:celebrity
```

Even if Redis is distributed, one key is normally mapped to one shard.

---

## Solutions

### 1. Fan-out on read

Do not write the celebrity post to 100M feeds.

### 2. Cache recent celebrity posts

```text
celebrity:123:recent_posts
```

### 3. Request coalescing

If thousands of requests miss cache simultaneously:

```text
10,000 requests
      |
      v
same cache miss
      |
      +----> 1 DB request
      |
      +----> 9,999 wait
```

### 4. Key replication / sharding

For extreme read hot keys:

```text
celebrity:123:0
celebrity:123:1
celebrity:123:2
...
```

Requests can be spread across replicas.

Use carefully because it increases complexity.

---

# 4. Ranking

A feed is not simply:

```text
ORDER BY created_at DESC
```

A modern feed may rank posts using signals such as:

- recency
- relationship strength
- engagement
- predicted relevance
- content type
- user preferences
- negative feedback
- freshness

### Simplified scoring

Example:

```text
score =
    0.40 * relevance
  + 0.25 * freshness
  + 0.20 * engagement
  + 0.15 * relationship
```

This is only an illustrative model.

### Example

Three posts:

```text
P1: 2 minutes old, low engagement
P2: 20 minutes old, high engagement
P3: 5 minutes old, strong relationship
```

Chronological ordering:

```text
P1 -> P3 -> P2
```

Ranking may produce:

```text
P3 -> P2 -> P1
```

### Architecture

```text
Candidate generation
        |
        v
Candidate posts
        |
        v
Ranking service
        |
        v
Top N posts
```

### Staff-level issue

Ranking must not become the availability bottleneck.

If the ML/ranking service fails:

```text
Feed Service
     |
     X Ranking unavailable
     |
     v
Fallback:
chronological ordering
```

A degraded feed is usually better than a completely unavailable feed.

---

# 5. Redis / Feed Cache

Redis is useful for storing a user's precomputed feed.

Example:

```text
feed:user:123
```

Possible structure:

```text
ZSET

post_id       score
P100          10001
P101          10000
P102           9999
```

The score can represent ordering/ranking.

### Read

```text
GET /feed
   |
   v
Redis
   |
   +-- hit --> return
   |
   +-- miss
        |
        v
Feed Store / DB
        |
        v
Rebuild/cache
```

### Important principle

Redis is generally a **performance layer**, not the source of truth.

The durable post metadata should live in durable storage.

### Cache invalidation

When a post is deleted:

```text
Post DB
   |
   v
event
   |
   v
remove/update cached references
```

Do not assume cache invalidation is perfectly synchronous.

The feed service should tolerate stale cache entries where product semantics allow it.

---

# 6. Kafka

Kafka is useful for decoupling asynchronous work.

Example:

```text
Post Service
    |
    v
Kafka topic: post-created
    |
    +--> Fanout workers
    +--> Ranking pipeline
    +--> Notification service
    +--> Analytics
```

### Why Kafka?

Without Kafka:

```text
POST /post
   |
   +--> write DB
   +--> fanout 10K users
   +--> send notifications
   +--> update analytics
```

The request becomes slow and fragile.

With Kafka:

```text
POST /post
   |
   v
DB + durable event
   |
   v
Return quickly

Kafka
 |
 +--> fanout
 +--> notification
 +--> analytics
```

### Partitioning

Partition based on a key that gives useful ordering and distribution.

For example:

```text
key = author_id
```

can preserve event order for a particular author.

But if one author is extremely hot, that key can become a partition hotspot.

For celebrity events, special handling may be needed.

### Consumer lag

Monitor:

```text
consumer lag
```

If lag increases:

```text
producers > consumer capacity
```

Possible responses:

- scale consumers
- increase partitions
- batch processing
- reduce non-critical work
- apply backpressure

---

# 7. Cursor Pagination

Offset pagination:

```text
GET /feed?page=100
```

is problematic for continuously changing feeds.

Example:

```text
page=1
```

contains:

```text
P100
P99
P98
```

A new post arrives.

Now page 2 can shift.

This can cause duplicates or missing records.

---

## Cursor pagination

Return a cursor representing the position.

```text
GET /feed?limit=20
```

Response:

```json
{
  "items": ["P100", "P99", "P98"],
  "nextCursor": "score=1700000000&id=P98"
}
```

Next:

```text
GET /feed?cursor=score=1700000000&id=P98
```

### Why include ID?

If multiple posts have the same timestamp:

```text
P100 timestamp = 1000
P99  timestamp = 1000
```

timestamp alone is not enough.

Use a stable tuple:

```text
(created_at, post_id)
```

### Advantages

- efficient for large datasets
- stable traversal
- avoids expensive OFFSET
- works well with sorted stores

---

# 8. Media + CDN

Do not serve large images/videos directly from the application servers.

Architecture:

```text
Client
  |
  v
API
  |
  v
Metadata Service

Media upload
  |
  v
Object Storage
  |
  v
CDN
  |
  v
Users
```

Example:

```text
S3-compatible object storage
        |
        v
      CDN
        |
        v
     Mobile App
```

### Upload flow

Prefer direct or pre-signed upload:

```text
Client
  |
  | request upload URL
  v
API
  |
  v
Pre-signed URL
  |
  v
Object Storage
```

The API server does not need to stream a 20 MB video.

### Media processing

```text
Upload
  |
  v
Object Storage
  |
  v
Kafka
  |
  v
Media Workers
  |
  +--> resize
  +--> thumbnails
  +--> transcode
  +--> moderation
  |
  v
CDN
```

### CDN cache

CDN reduces:

- application load
- object-storage load
- bandwidth cost
- latency

---

# 9. Consistency in Feed Systems

Not every operation needs strong consistency.

### Strong consistency candidates

Examples:

- account ownership
- post deletion authorization
- privacy settings
- security-sensitive actions

### Eventual consistency candidates

Examples:

- feed propagation
- like counts
- recommendation changes
- analytics

Suppose Alice posts:

```text
10:00:00
```

Bob sees it at:

```text
10:00:01
```

A one-second propagation delay may be acceptable.

### Key interview statement

> "I would explicitly classify each operation by consistency requirement instead of making the entire system strongly consistent."

---

# 10. Cache Stampede

Suppose a popular key expires:

```text
profile:123
```

At the same moment:

```text
10,000 requests
```

arrive.

All see cache miss.

```text
10,000
   |
   v
DB
```

The DB becomes overloaded.

### Solutions

#### Request coalescing

Only one request rebuilds the key.

#### TTL jitter

Instead of:

```text
TTL = 300 sec
```

use:

```text
TTL = 300 + random(0..30)
```

This prevents synchronized expiration.

#### Refresh ahead

Refresh popular keys before expiry.

#### Stale-while-revalidate

Serve slightly stale data while one worker refreshes it.

---

# 11. Multi-Region

For global systems:

```text
Region A
Region B
Region C
```

Possible architectures:

### Active-active

All regions accept traffic.

Advantages:

- lower latency
- better availability

Challenges:

- cross-region consistency
- conflict resolution
- duplicate processing
- data ownership

### Active-passive

One primary region handles writes.

Advantages:

- simpler consistency

Disadvantages:

- failover complexity
- higher latency for distant users
- capacity must exist in standby

### Feed example

A practical design can use:

```text
Global CDN
      |
      v
Nearest region
      |
      v
Regional Feed Service
```

User-specific feed data can have regional ownership/replicas depending on consistency requirements.

---

# 12. Failure Modes — Feed

| Failure | Impact | Mitigation |
|---|---|---|
| Redis down | feed latency increases | fallback to feed store |
| Kafka down | async propagation delayed | durable retry/outbox |
| Ranking down | ranking unavailable | chronological fallback |
| DB read replica lag | stale feed | tolerate or route critical reads |
| Celebrity post spike | huge fanout | fan-out-on-read |
| CDN failure | media slow | origin fallback / multi-CDN |
| Consumer lag | feed freshness drops | scale consumers |
| Cache stampede | DB overload | coalescing + jitter |
| Region failure | traffic loss | failover |

---

# 13. Staff-Level Trade-offs — Feed

A Staff answer should not be:

> "Use Redis + Kafka + Cassandra."

Instead explain why.

### Trade-off example

**Fan-out on write**

Pros:

- fast reads
- predictable latency

Cons:

- write amplification
- celebrity problem

**Fan-out on read**

Pros:

- cheap writes
- handles celebrities

Cons:

- expensive reads
- more ranking/merge work

**Hybrid**

Pros:

- balances both

Cons:

- more implementation complexity

### Staff-level framing

For every major decision:

```text
Requirement
   |
   v
Constraint
   |
   v
Design choice
   |
   v
Trade-off
   |
   v
Failure fallback
```

---

# Part II — BookMyShow / Seat Booking

## 14. Seat Inventory

The core resource is:

```text
Show + Seat
```

Example:

```text
Movie: Interstellar
Theater: PVR
Screen: 2
Show: 7 PM
Seat: A10
```

The system must maintain the state of A10.

Possible states:

```text
AVAILABLE
HELD
BOOKED
```

### Critical invariant

```text
For a given show:
one seat can have at most one confirmed owner.
```

This invariant is more important than cache speed.

---

# 15. Double-Booking Prevention

Two users:

```text
User A -> A10
User B -> A10
```

arrive simultaneously.

Bad design:

```text
check availability
if available:
    book
```

Both can observe:

```text
AVAILABLE
```

Then both book.

This is a classic check-then-act race.

Correct design requires atomic state transition.

---

# 16. Atomic Conditional Updates

A powerful approach:

```sql
UPDATE show_seats
SET status = 'HELD',
    hold_id = :holdId,
    hold_until = :expiry
WHERE show_id = :showId
  AND seat_id = :seatId
  AND status = 'AVAILABLE';
```

Then check affected rows.

```text
rows = 1
```

means success.

```text
rows = 0
```

means another request won the race.

### Why this works

The database performs the condition and state change atomically.

Two concurrent requests cannot both successfully transition the same row from AVAILABLE.

### Example

Initial:

```text
A10 = AVAILABLE
```

Request A:

```text
UPDATE ... WHERE status=AVAILABLE
```

wins.

Now:

```text
A10 = HELD
```

Request B executes the same condition:

```text
WHERE status=AVAILABLE
```

fails.

---

# 17. Pessimistic vs Optimistic Locking

## Pessimistic locking

Assume conflicts are likely.

Example:

```sql
SELECT *
FROM show_seats
WHERE show_id = ?
AND seat_id = ?
FOR UPDATE;
```

The row is locked during the transaction.

### Pros

- straightforward correctness
- good for highly contended resources

### Cons

- lock contention
- deadlocks
- long transactions can hurt throughput

---

## Optimistic locking

Assume conflicts are relatively rare.

Example:

```text
seat:
version = 10
```

Update:

```sql
UPDATE seat
SET status='HELD',
    version=11
WHERE seat_id=?
  AND version=10;
```

If affected rows = 0:

```text
someone else changed it
```

### Pros

- less lock contention
- good for low-conflict workloads

### Cons

- retries under contention
- bad behavior if contention becomes extreme

---

## Which for ticket booking?

For highly contested seats, atomic conditional update or carefully scoped pessimistic locking is usually easier to reason about.

---

# 18. Redis Locks + Fencing

A distributed Redis lock can look like:

```text
SET lock:show:123:seat:A10 token NX PX 5000
```

But a distributed lock alone does not guarantee correctness.

### Problem

Client A obtains lock.

Then:

```text
network pause
```

The lock expires.

Client B obtains lock.

Now A wakes up and continues.

Both may think they own the lock.

---

## Fencing tokens

Every lock acquisition gets an increasing token:

```text
A -> token 101
B -> token 102
```

The downstream resource accepts only the latest valid token.

```text
token 101 -> rejected
token 102 -> accepted
```

### Important

Fencing is stronger than simply saying:

> "I acquired a Redis lock."

### Interview position

For seat correctness, prefer the database's atomic invariant enforcement.

Use Redis for coordination/performance only when justified.

---

# 19. Seat Holds / Expiry

A user selects:

```text
A10
A11
A12
```

We cannot leave them held forever.

Example:

```text
HELD until 10:05:00
```

At expiry:

```text
HELD -> AVAILABLE
```

### Hold record

```text
hold_id
show_id
seat_id
user_id
expires_at
status
```

### Important race

At 10:05:

```text
Hold expiry
```

and

```text
Payment success
```

may happen concurrently.

Never blindly release the seat.

Use a conditional operation:

```sql
UPDATE show_seats
SET status='AVAILABLE',
    hold_id=NULL
WHERE hold_id=:holdId
  AND status='HELD'
  AND hold_until < now();
```

If payment already converted it to BOOKED, the condition fails.

### Expiry processing

Options:

- scheduled jobs
- Kafka-based delayed events
- sorted sets
- background workers

Do not rely only on an in-memory timer.

---

# 20. Payment Failure Scenarios

Seat booking and payment are separate systems.

Typical flow:

```text
AVAILABLE
   |
   v
HELD
   |
   v
Payment
   |
   +---- success ---> BOOKED
   |
   +---- failure ---> AVAILABLE
```

But real failure modes are harder.

---

## Scenario 1: Payment fails

```text
HELD
  |
  v
payment failed
  |
  v
release hold
```

---

## Scenario 2: Payment succeeds but response is lost

Payment processor:

```text
SUCCESS
```

Booking service does not receive response because of network failure.

Do not create another payment blindly.

Use:

- idempotency key
- payment status query
- webhook
- reconciliation

---

## Scenario 3: Payment timeout

Timeout does not mean payment failed.

It means:

```text
UNKNOWN
```

This distinction is extremely important.

---

# 21. Idempotency

Client sends:

```text
POST /book
Idempotency-Key: abc-123
```

Request times out.

Client retries with the same key.

The system must not create another booking/payment.

### Idempotency table

```text
idempotency_key
request_hash
status
response
created_at
```

Possible states:

```text
IN_PROGRESS
SUCCEEDED
FAILED
```

### Correct behavior

Same key + same request:

```text
return previous result
```

Same key + different request:

```text
reject
```

### Database protection

Also use a unique constraint:

```sql
UNIQUE(user_id, idempotency_key)
```

Redis can accelerate lookup, but durable storage should enforce the invariant.

---

# 22. Hot Shows

A blockbuster show can receive:

```text
50K requests/sec
```

for only:

```text
200 seats
```

The system has an enormous demand-to-resource ratio.

### Problem

Without protection:

```text
50K RPS
   |
   v
Booking Service
   |
   v
DB
```

The DB becomes a contention hotspot.

### Solutions

- waiting room
- admission control
- per-show rate limit
- queueing
- cached availability
- atomic DB updates
- partitioning by show
- short hold transactions

---

# 23. Waiting Room / Admission Control

Instead of allowing 100K clients to hit the booking DB simultaneously:

```text
Users
  |
  v
Waiting Room
  |
  v
Admission Controller
  |
  v
Booking Service
```

Only a controlled number enter.

Example:

```text
100,000 waiting users
        |
        v
2,000 active booking sessions
```

This protects downstream systems.

### Important distinction

Rate limiting says:

> "You may send at most N requests."

Admission control says:

> "Only N concurrent users may enter this expensive workflow."

### Token example

```text
capacity = 2,000
```

When a user gets an admission token:

```text
token -> expires after 2 minutes
```

This prevents unlimited concurrency.

---

# 24. Outbox / Kafka

Suppose booking transaction does:

```text
DB:
booking = CONFIRMED

then:

Kafka:
BookingConfirmed event
```

What if DB commits but Kafka publish fails?

Now:

```text
Booking exists
Kafka event missing
```

This is the dual-write problem.

---

## Transactional Outbox

Write both booking and event into the same DB transaction:

```text
BEGIN

INSERT booking
INSERT outbox_event

COMMIT
```

Then an outbox worker publishes:

```text
outbox table
    |
    v
Kafka
    |
    +--> notification
    +--> ticket generation
    +--> analytics
```

### Why it works

If the DB transaction commits:

```text
booking + outbox event
```

both exist.

The event can be retried until published.

### Consumer idempotency

Kafka consumers should also tolerate duplicate delivery.

Use:

```text
event_id
```

and deduplicate at the consumer.

---

# 25. Multi-Region — Booking

For seat inventory, active-active writes across regions can be dangerous.

Suppose:

```text
Mumbai region -> A10
Bangalore region -> A10
```

Both process independently.

Conflict.

### Better model

Assign a home region/authority for a show:

```text
show 123 -> Region A
```

All authoritative booking writes for that show go there.

Reads can be served closer to users where safe.

### Why?

Seat inventory requires a single authoritative ordering for conflicting updates.

### Trade-off

You may accept slightly higher latency to preserve correctness.

---

# 26. Reconciliation

Distributed payment/booking systems must periodically compare systems of record.

Example:

```text
Booking DB
Payment Gateway
Ticket Service
Seat Inventory
```

Potential mismatch:

```text
Booking = CONFIRMED
Payment = UNKNOWN
```

or:

```text
Payment = SUCCESS
Booking = PENDING
```

A reconciliation job detects these.

```text
Reconciliation
      |
      +--> compare booking vs payment
      +--> compare payment vs processor
      +--> compare seat state vs booking
      |
      v
Repair / alert / manual review
```

### Staff-level principle

Do not assume:

> "The happy path always completes."

Reconciliation is part of correctness for eventually consistent distributed workflows.

---

# Part III — Distributed Rate Limiter

## 27. Fixed Window

Example policy:

```text
100 requests / minute
```

Windows:

```text
10:00:00 - 10:00:59
10:01:00 - 10:01:59
```

Counter:

```text
rate:user123:10:00
```

Increment on every request.

### Problem: boundary burst

User sends:

```text
100 requests at 10:00:59
100 requests at 10:01:00
```

Potentially:

```text
200 requests in ~1 second
```

even though the configured limit is 100/minute.

### Pros

- simple
- cheap
- easy to implement

### Cons

- boundary burst
- less smooth enforcement

---

# 28. Sliding Window

Sliding window asks:

> "How many requests happened during the previous N seconds?"

Example:

```text
now = 10:01:30
window = previous 60 sec
```

Count requests between:

```text
10:00:30
and
10:01:30
```

### Sliding Window Log

Store timestamps:

```text
[1000, 1002, 1005, 1010, ...]
```

Remove timestamps older than the window.

### Problem

Memory usage can be large.

If:

```text
1M users
x
1000 requests
```

the number of timestamps can become huge.

---

## Sliding Window Counter

Approximate the sliding window using neighboring fixed windows.

For example:

```text
current window count
previous window count
```

Weighted estimate:

```text
estimated =
previous_count * overlap
+
current_count
```

### Pros

- lower memory
- smoother than fixed window

### Cons

- approximate
- more logic

---

# 29. Token Bucket

Token bucket is often a strong general-purpose algorithm.

State:

```text
capacity = 100
refill_rate = 10 tokens/sec
```

Initially:

```text
tokens = 100
```

Each request consumes one token.

```text
tokens > 0
    |
    v
allow
```

Otherwise:

```text
reject with 429
```

### Burst support

If bucket contains 100 tokens:

```text
100 requests
```

can pass immediately.

Then refill:

```text
10 tokens/sec
```

So sustained throughput is approximately 10 requests/sec.

### Example

At t=0:

```text
tokens = 100
```

100 requests:

```text
tokens = 0
```

After 1 second:

```text
tokens = 10
```

10 more requests can pass.

### Why token bucket is attractive

It controls sustained rate while allowing bounded bursts.

---

# 30. Redis Atomicity / Lua

Suppose the limiter does:

```text
GET tokens
calculate
SET tokens
```

This is unsafe under concurrency.

Two requests:

```text
A GET -> 1 token
B GET -> 1 token
A SET -> 0
B SET -> 0
```

Both may be allowed even though only one token existed.

---

## Lua script

Perform the entire token operation atomically in Redis.

Conceptually:

```text
read state
calculate refill
if token available:
    decrement
    return ALLOW
else:
    return DENY
```

Redis executes the Lua script atomically relative to other Redis commands.

### Important

Atomicity solves concurrent updates to one limiter key.

It does not magically solve:

- Redis failure
- multi-region consistency
- hot keys
- policy propagation

---

# 31. Hot Keys — Rate Limiter

Suppose:

```text
API key = enterprise-client-123
```

sends:

```text
100K RPS
```

Every request touches:

```text
rate:enterprise-client-123
```

One Redis shard may become a bottleneck.

### Solutions

#### Local pre-check

Use local counters for coarse protection.

#### Key sharding

Split state carefully.

But sharding a token bucket changes semantics because tokens are distributed.

#### Hierarchical limits

Use local + global controls.

#### Dedicated capacity

Move very high-volume tenants to dedicated infrastructure.

---

# 32. Local + Global Limiting

A powerful architecture:

```text
Client
  |
  v
Local limiter
  |
  v
Global limiter
```

### Local limiter

Runs inside each gateway/service instance.

Advantages:

- extremely fast
- no network round trip
- protects local CPU

### Global limiter

Redis or distributed store enforces tenant-wide limits.

Example:

```text
Global limit = 10K RPS
```

Across:

```text
100 application instances
```

A local-only limiter cannot accurately enforce the global limit.

### Trade-off

Local:

```text
fast + approximate
```

Global:

```text
accurate + network cost
```

---

# 33. Hierarchical Rate Limiting

Apply limits at multiple levels.

Example:

```text
Global
  |
  +-- Tenant
        |
        +-- User
              |
              +-- API
                    |
                    +-- Endpoint
```

Example policies:

```text
Global: 1M RPS
Tenant: 50K RPS
User: 100 RPS
POST /payment: 10 RPS
```

A request must pass all applicable limits.

### Why hierarchical limiting?

Different layers protect different resources.

For example:

```text
global limit
```

protects infrastructure.

```text
tenant limit
```

prevents one customer from consuming everything.

```text
endpoint limit
```

protects an expensive operation.

---

# 34. Multi-Dimensional Limits

Real systems rarely have one limiter.

Examples:

```text
per IP
per user
per API key
per tenant
per endpoint
per region
```

A payment endpoint might enforce:

```text
IP       = 20 RPS
user     = 5 RPS
merchant = 1,000 RPS
endpoint = 10K RPS
global   = 100K RPS
```

### Request evaluation

```text
Request
   |
   +--> IP limiter
   +--> user limiter
   +--> tenant limiter
   +--> endpoint limiter
   +--> global limiter
```

If any mandatory limit rejects:

```text
429 Too Many Requests
```

### Design issue

More dimensions mean:

- more Redis operations
- higher latency
- more state
- more policy complexity

Batching or a single atomic script can reduce round trips.

---

# 35. Fail-Open vs Fail-Closed

What happens if Redis is unavailable?

## Fail-open

Allow requests.

```text
Redis DOWN
    |
    v
ALLOW
```

### Pros

- better availability
- users can continue

### Cons

- abuse may overwhelm system
- rate limit temporarily ineffective

---

## Fail-closed

Reject requests.

```text
Redis DOWN
    |
    v
REJECT
```

### Pros

- protects backend
- strong enforcement

### Cons

- legitimate users may be blocked
- creates availability dependency

---

## Hybrid approach

Not every endpoint should behave the same.

For example:

```text
Public read API:
    fail-open with local emergency limit

Payment:
    conservative local limit

Expensive admin operation:
    fail-closed
```

### Staff-level answer

> "I would choose fail-open versus fail-closed based on the cost of abuse versus the cost of blocking legitimate traffic."

---

# 36. Multi-Region Quotas

Suppose global tenant limit:

```text
10K RPS
```

Regions:

```text
US
EU
APAC
```

If every region independently allows 10K:

```text
30K global RPS
```

The global limit is violated.

---

## Strategy 1: Global strongly consistent counter

Accurate but expensive.

Every request may require global coordination.

Bad for very high RPS.

---

## Strategy 2: Quota allocation

Allocate:

```text
US   = 5K
EU   = 3K
APAC = 2K
```

Total:

```text
10K
```

Each region enforces its local budget.

### Problem

Traffic may move.

Suppose APAC needs 4K but has only 2K quota.

Dynamic quota rebalancing can solve this.

### Trade-off

You exchange perfect instantaneous global precision for:

- low latency
- availability
- regional independence

---

# 37. Backpressure

Backpressure means slowing or rejecting upstream work when downstream capacity is exhausted.

Example:

```text
Payment Service
      |
      v
DB
```

DB capacity:

```text
20K writes/sec
```

Incoming:

```text
50K writes/sec
```

If the application continues sending all 50K:

```text
DB queue grows
latency grows
connections exhaust
timeouts occur
retries increase
```

This can become a cascading failure.

### Better

```text
Incoming 50K
      |
      v
Admission / Queue
      |
      v
DB at safe 20K
```

Excess requests can be:

- queued
- rejected
- rate limited
- degraded
- retried later

### Backpressure mechanisms

- bounded queues
- concurrency limits
- rate limiting
- circuit breakers
- load shedding
- connection pool limits
- Kafka consumer controls

### Key principle

> "Do not allow an overloaded downstream dependency to turn bounded overload into unbounded queue growth."

---

# 38. Retry Storms

Suppose DB becomes slow.

Requests timeout.

Clients retry.

Original:

```text
10K RPS
```

After retry:

```text
20K RPS
```

Then:

```text
40K
80K
```

This is a retry storm.

### Why dangerous?

The system is already overloaded.

Retries increase the load precisely when the system has the least capacity.

---

## Protection

### Exponential backoff

Example:

```text
100ms
200ms
400ms
800ms
```

### Jitter

Add randomness:

```text
delay = exponential_backoff + random_jitter
```

This prevents synchronized retries.

### Retry budget

Do not allow unlimited retries.

### Retry only transient errors

Do not retry:

```text
400 Bad Request
```

Usually retry:

```text
503
timeout
connection reset
```

depending on operation semantics.

### Idempotency

Never blindly retry a payment without idempotency.

---

# 39. Policy Propagation

Rate-limit policies change.

Example:

```text
old:
100 RPS

new:
200 RPS
```

There may be:

```text
1,000 gateway instances
```

How does the new policy reach them?

Architecture:

```text
Admin
  |
  v
Policy Service
  |
  v
Kafka / Config Stream
  |
  +--> Gateway A
  +--> Gateway B
  +--> Gateway C
```

### Requirements

- version policies
- validate configuration
- make updates idempotent
- monitor propagation lag
- handle missed events
- support snapshot reload

Example policy:

```json
{
  "policyVersion": 42,
  "tenant": "T123",
  "limit": 200,
  "window": "1s"
}
```

Gateway should know:

```text
current version = 42
```

### Failure scenario

Gateway misses version 42.

On restart:

```text
load latest snapshot
```

This prevents permanent stale configuration.

---

# 40. Staff-Level Trade-offs — Rate Limiter

A Staff answer should compare choices rather than blindly choosing one algorithm.

## Fixed Window

Best when:

- simplicity matters
- approximation is acceptable

Weakness:

- boundary burst

## Sliding Window

Best when:

- smoother enforcement matters

Weakness:

- memory/complexity

## Token Bucket

Best when:

- bounded bursts are desirable
- sustained rate needs control

Weakness:

- distributed state is harder

---

## Local vs Redis

### Local

```text
+ very fast
+ no network
- not globally accurate
```

### Redis

```text
+ shared global state
+ stronger enforcement
- network dependency
- hot-key risk
```

---

## Strong consistency vs availability

For a global limiter:

```text
strong global counter
```

provides accuracy but introduces coordination.

A regional quota model:

```text
regional allocation
```

provides better latency and availability but can temporarily deviate from exact global usage.

---

# Staff-Level Interview Framework

For any of these 40 concepts, answer using this structure.

## 1. State the invariant

Examples:

### Feed

```text
A user's feed should be available with predictable latency.
```

### Booking

```text
One show-seat can have at most one confirmed booking.
```

### Rate limiter

```text
Requests should not exceed the configured policy beyond the accepted consistency/error bound.
```

---

## 2. Identify the bottleneck

Examples:

```text
Feed:
celebrity fan-out

Booking:
hot show / seat contention

Rate limiter:
hot Redis key
```

---

## 3. Choose the architecture

Do not jump directly to technology.

Say:

```text
Requirement
   ->
bottleneck
   ->
architecture
```

---

## 4. Explain the failure mode

Example:

```text
Redis fails
```

Then explain:

```text
What happens?
How do we detect it?
How do we degrade?
How do we recover?
```

---

## 5. Explain the trade-off

Example:

> "I can make the feed strongly consistent, but that would increase cross-region coordination and latency. Since feed freshness can tolerate seconds of delay, I prefer eventual consistency."

---

# Cross-System Cheat Sheet

| Concept | Feed | Booking | Rate Limiter |
|---|---|---|---|
| Hot key | celebrity | hot show | tenant/API key |
| Cache | feed cache | availability cache | limiter state |
| Kafka | fanout | events/outbox | policy propagation |
| Idempotency | writes/events | booking/payment | config/event handling |
| Strong consistency | selective | critical | policy/invariant dependent |
| Eventual consistency | common | notifications | quota propagation |
| Backpressure | fanout workers | waiting room | reject/load shed |
| Multi-region | feed reads | show ownership | regional quota |
| Reconciliation | data pipelines | payment/booking | policy/config drift |
| Failure fallback | chronological feed | hold/reconcile | local limiter |

---

# Interview Examples

## Example 1 — Feed

Interviewer:

> "A celebrity has 50 million followers. How do you handle a new post?"

Strong answer:

> "I would not synchronously fan out to 50 million feeds. I would classify the user as a celebrity based on follower count or fan-out cost. Their posts would remain in the author's post store and be fetched during feed generation. Normal users can use fan-out-on-write. The feed service merges precomputed entries with celebrity candidates and ranks them. This gives us a hybrid architecture and avoids catastrophic write amplification."

---

## Example 2 — Booking

Interviewer:

> "Two users click the same seat at exactly the same time."

Strong answer:

> "I would make the database enforce the invariant atomically. For example, update the seat from AVAILABLE to HELD only where the current state is AVAILABLE and check affected rows. Exactly one request can win. Redis can accelerate availability display, but it is not the source of truth for booking correctness."

---

## Example 3 — Rate Limiter

Interviewer:

> "How do you implement 1,000 RPS with burst support?"

Strong answer:

> "I would use a token bucket. The bucket has a capacity representing the maximum burst and a refill rate representing sustained throughput. For a distributed implementation, I can store bucket state in Redis and perform refill and decrement atomically using Lua. For very high throughput, I would add local limiting before the global Redis limiter and use hierarchical limits to reduce pressure on the shared store."

---

# Common Staff-Level Traps

## Trap 1 — "Redis solves everything"

It does not.

Ask:

```text
What happens when Redis fails?
What happens to hot keys?
What is the source of truth?
What is the consistency model?
```

---

## Trap 2 — "Kafka guarantees exactly once"

Kafka can provide strong delivery/processing semantics in specific architectures, but business-level exactly-once effects still require idempotency and correct transaction boundaries.

Use:

```text
event_id
idempotent consumers
transactional outbox
```

where appropriate.

---

## Trap 3 — "Distributed lock solves booking"

A lock is not automatically a correctness proof.

Prefer:

```text
database invariant
+
atomic transition
+
idempotency
```

and use distributed locks only where they provide a justified coordination benefit.

---

## Trap 4 — "Just retry"

Retries can cause:

```text
overload
+
duplicate operations
+
retry storms
```

Use:

```text
backoff
jitter
retry budget
idempotency
```

---

## Trap 5 — "Make everything strongly consistent"

Strong consistency has cost.

Ask:

```text
Does the business actually require it?
```

For example:

```text
Seat ownership -> yes
Feed freshness -> usually no
Analytics -> no
```

---

# Final Mental Model

For SDE-3 / Staff interviews, think in this order:

```text
1. Requirements
       |
       v
2. Scale
       |
       v
3. Invariants
       |
       v
4. Bottlenecks
       |
       v
5. Data model
       |
       v
6. Concurrency
       |
       v
7. Cache
       |
       v
8. Async processing
       |
       v
9. Backpressure
       |
       v
10. Failure modes
       |
       v
11. Multi-region
       |
       v
12. Observability
       |
       v
13. Trade-offs
```

The strongest Staff-level answers repeatedly connect every technology choice to:

```text
scale
correctness
latency
availability
failure handling
operational complexity
cost
```

That is the difference between:

```text
"Here is my architecture"
```

and:

```text
"Here is why this architecture remains correct and operable under scale and failure."
```
