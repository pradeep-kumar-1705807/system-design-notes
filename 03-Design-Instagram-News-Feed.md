# 🔴 P0 — Design Instagram / News Feed

> Interview Level: SDE-3 / Staff  
> Difficulty: ⭐⭐⭐⭐⭐  
> Interview Time: 45–60 minutes  
> Core Topics: Fan-out, caching, ranking, sharding, consistency, hot keys, media storage, async processing

---

# 1. Problem Statement

Design an Instagram-like social platform focused on:

- User profiles
- Following / unfollowing
- Creating posts
- Viewing a personalized home feed
- Likes and comments
- Image/video metadata
- Feed ranking
- High read scalability

The primary interview focus is the **News Feed**.

---

# 2. Clarifying Questions

Ask:

1. How many users?
2. DAU / MAU?
3. Average follows per user?
4. Posts per second?
5. Feed reads per second?
6. Are posts public/private?
7. Is chronological ordering required?
8. Do we need ranking/recommendations?
9. How fresh must the feed be?
10. How quickly must a new post appear?
11. Do we support videos?
12. What is the availability target?

### Sample assumptions

```text
Users                  = 500M
DAU                    = 100M
Feed reads             = 500K/sec peak
Post creation          = 10K/sec peak
Average follows        = 300
Celebrity followers    = 50M
Feed p99               = < 200 ms
Availability           = 99.99%
Feed freshness         = seconds
```

---

# 3. Core Requirements

## Functional

```text
Create Post
Follow User
Unfollow User
Get Home Feed
Like Post
Comment
Delete Post
```

## Non-functional

- High availability
- Low feed latency
- Horizontal scalability
- Eventual consistency acceptable for feed
- No duplicate posts
- Resilient to celebrity/hot-key traffic

---

# 4. High-Level Architecture

```text
                    Client
                       |
                       v
                API Gateway / LB
                       |
              +--------+--------+
              |                 |
              v                 v
        Post Service       Feed Service
              |                 |
              v                 v
        Metadata DB        Feed Cache
              |
              v
        Object Storage
              |
              v
         Media Pipeline

Post Service
     |
     v
   Kafka
     |
     +--------------------+
     |                    |
     v                    v
Fanout Workers       Ranking Pipeline
     |                    |
     v                    v
Feed Store             ML/Ranking
```

---

# 5. Post Creation Flow

```text
Client
  |
  v
Post Service
  |
  +--> Store post metadata
  |
  +--> Store media in object storage
  |
  +--> Publish PostCreated event
             |
             v
           Kafka
             |
             v
        Fanout workers
```

The post write should not synchronously update millions of followers.

---

# 6. The Central Design Decision

The key question:

> Do we fan out posts on write or on read?

There are three common approaches.

---

# 7. Fan-out on Write

When Alice posts:

```text
Alice
 |
 v
Post Service
 |
 v
Kafka
 |
 +--> Feed Worker
        |
        +--> Bob feed
        +--> Carol feed
        +--> David feed
        +--> ...
```

Each follower's feed is updated ahead of time.

### Pros

- Very fast feed reads
- Simple read path
- Good for normal users

### Cons

- Expensive for celebrities
- Huge write amplification
- Celebrity with 50M followers creates enormous fanout

---

# 8. Fan-out on Read

Store only the post.

When Bob opens the feed:

```text
Bob
 |
 v
Feed Service
 |
 +--> Get followed users
 |
 +--> Get recent posts
 |
 +--> Merge
 |
 +--> Rank
 |
 v
Response
```

### Pros

- Cheap writes
- No massive fanout

### Cons

- Expensive feed reads
- Ranking/merge work on every request
- Higher latency

---

# 9. Hybrid Approach

Recommended interview direction:

```text
Normal Users
     |
     v
Fan-out on write

Celebrities
     |
     v
Fan-out on read
```

Example:

```text
Post
 |
 +--> Normal author
 |       |
 |       +--> push to follower feeds
 |
 +--> Celebrity
         |
         +--> keep in author-post store
         +--> merge during feed read
```

This avoids the celebrity problem.

---

# 10. Feed Read Flow

```text
Client
  |
  v
Feed Service
  |
  +--> Feed Cache
  |
  +--> Celebrity Post Store
  |
  +--> Ranking Service
  |
  v
Top N posts
```

Example:

```text
Precomputed Feed
      +
Celebrity Posts
      +
Freshness
      +
Ranking
      |
      v
Top 20
```

---

# 11. Feed Storage

Possible model:

```text
user_feed
------------------------
user_id
post_id
author_id
created_at
score
```

Partition by:

```text
user_id
```

This gives efficient:

```text
GET feed(user_id)
```

---

# 12. Redis Feed Cache

Example:

```text
feed:{user_id}
```

Value:

```text
sorted set
----------------
post_id -> score
```

The score can represent:

```text
ranking score
```

or:

```text
timestamp
```

Use Redis for the hot feed.

Persistent storage remains available for reconstruction.

---

# 13. Pagination

Avoid offset pagination:

```text
?page=10
```

because the feed changes continuously.

Prefer cursor pagination:

```http
GET /feed?cursor=eyJzY29yZSI6...
```

Conceptually:

```text
(score, post_id)
```

Use a stable ordering key to avoid duplicates/missing posts.

---

# 14. Ranking

Chronological:

```text
score = timestamp
```

Ranking can use:

```text
score =
  freshness
+ engagement
+ relationship
+ content affinity
```

Example conceptual model:

```text
score =
  w1 * freshness
+ w2 * engagement
+ w3 * relationship
+ w4 * predicted_interest
```

The exact ML model is outside the core HLD unless explicitly required.

---

# 15. Hot Users / Celebrities

A celebrity may have:

```text
50M followers
```

A single post cannot synchronously update 50M feed entries.

Do not do:

```text
POST
 |
 +--> update 50M feeds synchronously
```

Instead:

```text
POST
 |
 v
Kafka
 |
 +--> asynchronous fanout
```

or hybrid read-time merge.

---

# 16. Backpressure

Suppose:

```text
Celebrity post
      |
      v
50M fanout tasks
```

Do not create unlimited in-memory work.

Use:

```text
Kafka
 |
 v
Bounded consumer concurrency
 |
 v
Feed Store
```

Monitor:

```text
consumer lag
queue depth
processing latency
```

---

# 17. Kafka Partitioning

Potential partition key:

```text
author_id
```

This groups an author's post events.

For feed updates, another strategy may be:

```text
recipient_user_id
```

depending on the desired ordering and consumer architecture.

The key should be selected based on the dominant processing requirement.

---

# 18. Consistency

Feed systems usually tolerate eventual consistency.

Example:

```text
Alice posts
     |
     v
Kafka
     |
     v
Fanout
     |
     v
Bob feed
```

Bob might see the post a few seconds later.

This is usually acceptable.

But social actions such as:

```text
follow/unfollow
delete post
privacy changes
```

may require stronger handling to avoid exposing content incorrectly.

---

# 19. Unfollow Problem

Bob follows Alice.

Alice's posts are already in:

```text
feed:bob
```

Bob unfollows Alice.

Old Alice posts may still exist in Bob's precomputed feed.

Options:

### Option A

Remove feed entries asynchronously.

### Option B

Filter author relationship during feed read.

### Option C

Use tombstones/version checks.

For correctness-sensitive privacy changes, filtering must not rely solely on eventual cleanup.

---

# 20. Delete Post

Flow:

```text
Delete Post
    |
    v
Post DB -> DELETED
    |
    v
Event
    |
    v
Feed cleanup workers
```

Feed entries may temporarily remain, so the feed service should verify deletion/tombstone state where necessary.

---

# 21. Media Storage

Do not store large images/videos directly in the relational database.

Use:

```text
Client
  |
  v
Object Storage
  |
  v
CDN
```

Metadata DB stores:

```text
post_id
media_id
object_key
media_type
size
created_at
```

---

# 22. Media Upload Pipeline

```text
Client
  |
  v
Object Storage
  |
  v
Upload Event
  |
  v
Kafka
  |
  +--> Thumbnail
  +--> Transcoding
  +--> Moderation
  +--> Metadata extraction
```

---

# 23. CDN

Images/videos are excellent CDN candidates.

```text
Client
  |
  v
CDN
  |
  +--> Cache HIT
  |
  +--> Cache MISS
          |
          v
     Object Storage
```

Benefits:

- Lower origin load
- Lower latency
- Lower bandwidth cost
- Global distribution

---

# 24. Database Choices

Possible split:

```text
User/Profile      -> SQL / distributed SQL
Relationships     -> graph-like/key-value model
Posts             -> distributed KV / wide-column store
Feed              -> Redis + durable feed store
Media             -> object storage
Analytics         -> OLAP/data lake
```

Do not choose a database by habit.

Choose based on access pattern.

---

# 25. Sharding

Feed data can be partitioned by:

```text
user_id
```

Post data may be partitioned by:

```text
author_id
```

or:

```text
post_id
```

Watch for:

```text
celebrity partition
hot author
```

A naive `author_id` partition can become hot.

---

# 26. Cache Stampede

Suppose:

```text
feed:alice
```

expires.

10,000 requests arrive simultaneously.

Without protection:

```text
10K requests
     |
     v
DB
```

Use:

- TTL jitter
- Request coalescing
- Single-flight
- Background refresh
- Stale-while-revalidate

---

# 27. Request Coalescing

```text
1000 requests
      |
      v
same feed cache MISS
      |
      v
one request -> DB
      |
      v
populate cache
      |
      v
999 requests receive result
```

This protects the database during cache misses.

---

# 28. Availability

If ranking service is unavailable:

```text
Feed Service
     |
     X
Ranking
```

Fallback:

```text
Chronological ranking
```

The feed should degrade gracefully rather than fail completely.

---

# 29. Observability

Track:

```text
Feed p50/p95/p99
Cache hit ratio
Fanout lag
Kafka consumer lag
Post creation latency
Feed freshness
Ranking latency
DB latency
Hot partition rate
CDN hit ratio
```

Business metrics:

```text
Feed engagement
CTR
Session duration
Post visibility delay
```

---

# 30. Failure Scenarios

| Failure | Strategy |
|---|---|
| Redis down | Rebuild/read from durable feed store |
| Kafka delayed | Feed freshness degrades |
| Ranking unavailable | Chronological fallback |
| Celebrity spike | Hybrid fanout |
| DB shard hot | Repartition/shard |
| CDN down | Origin fallback with protection |
| Fanout worker crash | Kafka replay |
| Duplicate event | Idempotent feed update |
| Post deleted | Tombstone/filter |
| Unfollow event delayed | Read-time relationship check where required |

---

# 31. Interview Traps

### Trap 1

> Fanout every post to every follower.

Ask:

> What happens when a celebrity with 50M followers posts?

### Trap 2

> Build the entire feed at read time.

Ask:

> What happens at 500K feed reads/sec?

### Trap 3

> Store images in PostgreSQL.

Ask:

> How does the DB scale when media reaches petabytes?

### Trap 4

> Use offset pagination.

Ask:

> What happens when new posts arrive between page requests?

### Trap 5

> Cache the feed.

Ask:

> What happens on cache expiry for millions of users?

---

# 32. Staff-Level Trade-offs

Be explicit about:

```text
Write amplification
vs
Read latency

Freshness
vs
Cost

Ranking quality
vs
Latency

Precomputation
vs
Storage

Consistency
vs
Availability
```

The most important architectural decision is usually:

> Hybrid fanout: fan-out-on-write for normal users and fan-out-on-read for high-follower/cold cases.

---

# 33. Interview Follow-Up Questions

1. How would you handle a celebrity with 100M followers?
2. How would you guarantee feed pagination does not duplicate posts?
3. What happens when Redis loses all feed entries?
4. How would you handle unfollow immediately?
5. How do you prevent one user's posts from creating a hot partition?
6. How would you migrate ranking algorithms without breaking the feed?
7. How would you support multiple regions?
8. What happens if Kafka is unavailable for 30 minutes?
9. How would you rebuild feeds after a data corruption event?
10. How would you control storage cost?

---

# 34. 30-Second Answer

> I would use a hybrid news-feed architecture. Posts and media metadata are durably stored, media itself goes to object storage behind a CDN, and post creation publishes an event to Kafka. For normal users, asynchronous fan-out-on-write precomputes follower feeds for low-latency reads. For celebrities and extremely high-follower accounts, I would use fan-out-on-read or hybrid merging to avoid massive write amplification. Redis can cache hot feed pages while a durable feed store allows reconstruction. Feed reads use cursor pagination and can call a ranking service, with chronological fallback if ranking fails. The design accepts eventual consistency for normal feed propagation while using stronger checks for privacy, delete, and relationship changes. Kafka replay, idempotent consumers, backpressure, cache stampede protection, hot-partition handling, and observability make the system operationally resilient.
