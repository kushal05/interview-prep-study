# Real-World System Designs

> **Learning goal:** Apply all concepts to design actual systems. Practice the framework. Build pattern recognition.

---

## How to Use This File
Each design follows the interview framework. Study the approach, then **close this file and redesign from memory**. Compare your design with the reference. Repeat until fluent.

> **Read files 01–09 first.** This file uses every concept from earlier chapters. If something here feels confusing, the explanation is in the earlier chapter — flip back.

### The 5-Step Framework (Use Every Time)

1. **Requirements** — What does it do? What's the scale? What's out of scope?
2. **Estimation** — QPS, storage, bandwidth, memory.
3. **High-Level Design** — Draw the boxes and arrows. Define the API. Sketch the data model.
4. **Deep Dive** — Pick the 2–3 hardest components. Discuss trade-offs.
5. **Scaling & Wrap-up** — Bottlenecks, monitoring, future improvements.

---

## 1. URL Shortener (TinyURL)

### Requirements
```
Functional: Shorten URL, redirect to original, custom aliases (optional), expiration
Non-functional: Low latency redirect (<50ms), high availability, 100M URLs/day
```

### Estimation
```
Writes: 100M URLs/day = ~1,200 writes/sec
Reads: 10:1 read:write = 12,000 reads/sec
Storage: 100M × 500 bytes × 365 days × 5 years = ~90 TB
URL key: 7 chars from [a-zA-Z0-9] = 62^7 = 3.5 trillion (enough)
```

### High-Level Design
```
Write path:
Client → API Gateway → URL Service → Generate short key → Store in DB → Return short URL

Read path:
Client → API Gateway → URL Service → Lookup DB/Cache → 301 Redirect

┌────────┐    ┌───────────┐    ┌───────┐    ┌──────────┐
│ Client │───→│API Gateway│───→│URL Svc│───→│  Cache   │
└────────┘    └───────────┘    └───┬───┘    │ (Redis)  │
                                   │        └────┬─────┘
                                   │         miss│
                                   ▼             ▼
                               ┌──────────────────┐
                               │   Database        │
                               │  (NoSQL - fast    │
                               │   key-value)      │
                               └──────────────────┘
```

### Key Design Decisions
```
Key generation options:
1. Hash (MD5/SHA256) → take first 7 chars → collision possible
2. Base62 encoding of auto-increment ID → predictable
3. Pre-generated key pool → random, no collision ✓ Best

Database: Key-value store (DynamoDB/Cassandra)
  - Simple access pattern: key → URL
  - Massive scale needed
  - No complex queries

Caching: Redis with LRU
  - 20% of URLs get 80% of traffic
  - Cache top URLs → 80% hit rate

Redirect: 301 (permanent) vs 302 (temporary)
  - 301: Browser caches → less server load, but can't track clicks
  - 302: Always hits server → can track analytics ✓ For analytics
```

---

## 2. Twitter / News Feed

### Requirements
```
Functional: Post tweet, follow users, news feed (home timeline)
Non-functional: 300M MAU, feed loads in <500ms, eventually consistent
```

### Estimation
```
DAU: 200M, avg 2 tweets/day = 400M tweets/day ≈ 4,600 writes/sec
Feed reads: 200M × 10 reads/day = 2B reads/day ≈ 23,000 reads/sec
Tweet size: 280 chars + metadata ≈ 1KB
Storage: 400M × 1KB × 365 = 146 TB/year
```

### The Core Challenge: Fan-out
```
Problem: User opens app → show feed from all people they follow
Approach 1: Pull (Fan-out on Read)
  User opens app → query all followed users' tweets → merge → sort → return
  ✗ Slow for users following 1000+ people (1000 queries)

Approach 2: Push (Fan-out on Write)
  User tweets → push to ALL followers' feed caches
  ✗ Celebrity problem: user with 50M followers → 50M cache writes per tweet!

Approach 3: Hybrid ✓
  Regular users (<10K followers): Push (fan-out on write)
  Celebrities (>10K followers): Pull (fan-out on read, merge at query time)
```

### Architecture
```
Write path:
User → API → Tweet Service → Write to Tweet DB
                │
                └→ Fan-out Service → for each follower:
                     │                push tweet ID to their feed cache
                     ▼
                [Feed Cache (Redis sorted sets by timestamp)]

Read path:
User → API → Feed Service → Read from Feed Cache (pre-computed)
                │              + merge celebrity tweets (pull on read)
                └→ Tweet Service → Hydrate tweet IDs with full data

                    ┌──────────────┐
                    │   Client     │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ API Gateway  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Tweet   │ │  Feed    │ │  User    │
        │ Service  │ │ Service  │ │ Service  │
        └─────┬────┘ └─────┬────┘ └──────────┘
              │             │
              ▼             ▼
        ┌──────────┐ ┌──────────┐
        │Tweet DB  │ │Feed Cache│
        │(sharded) │ │ (Redis)  │
        └──────────┘ └──────────┘
```

---

## 3. Chat System (WhatsApp/Slack)

### Requirements
```
Functional: 1:1 chat, group chat (up to 500), online status, read receipts
Non-functional: Real-time (<100ms), 50M DAU, message persistence, ordering
```

### Architecture
```
Client ←── WebSocket ──→ Chat Server (stateful connection)
                              │
                    ┌─────────┼──────────┐
                    ▼         ▼          ▼
              [Message    [Presence   [Group
               Queue]     Service]   Service]
                 │
                 ▼
           [Message DB]

Message flow (1:1):
1. Alice sends message → her Chat Server via WebSocket
2. Chat Server → Message Queue (Kafka) topic for recipient
3. Bob's Chat Server consumes from queue → pushes via WebSocket
4. If Bob offline → store in DB → push notification → deliver when online

Message ordering:
- Unique message ID = timestamp + sender_id + sequence_number
- Per-conversation ordering (not global)
- Client-side sorting by message ID

Group chat:
- Group has member list in Group Service
- Message sent to group → fan-out to all members
- For large groups: use message queue per group
```

### Key Decisions
```
Protocol: WebSocket (full-duplex, persistent connection)
Database: Cassandra or HBase
  - Write-heavy (messages append-only)
  - Partition key: (chat_id, bucket) — bucket = date for time-range queries
  - Sort key: message_timestamp

Online presence:
  - Heartbeat every 5 seconds via WebSocket
  - No heartbeat for 30s → mark offline
  - Fan-out status only to users who are online and have recipient in view
```

---

## 4. YouTube / Video Platform

### Requirements
```
Functional: Upload video, stream video, search, recommendations
Non-functional: 1B DAU, smooth streaming, low startup latency
```

### Architecture
```
Upload flow:
                    ┌─────────────┐
  Upload API ─────→ │ Object Store│ (raw video)
                    │   (S3)      │
                    └──────┬──────┘
                           │ S3 event
                           ▼
                    ┌─────────────┐
                    │Transcoding  │ → Multiple resolutions (360p-4K)
                    │  Pipeline   │ → Multiple formats (H.264, VP9, AV1)
                    │  (Workers)  │ → Segment into chunks (2-10 sec)
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │Thumbnails│ │Transcoded│ │ Metadata │
        │  (S3)    │ │ Videos(S3)│ │   (DB)   │
        └──────────┘ └────┬─────┘ └──────────┘
                          │
                          ▼
                    ┌──────────┐
                    │   CDN    │ (serves to viewers)
                    └──────────┘

Streaming flow:
Client → CDN → Manifest file (.m3u8)
Client reads manifest → requests video segments from CDN
Adaptive bitrate: client monitors bandwidth → requests appropriate quality
```

### Key Decisions
```
Storage: S3/GCS for videos, CDN for delivery
  - 1 minute of video = ~50MB (compressed, multiple resolutions)
  - 500 hours uploaded/minute × 50MB × 5 resolutions = 150TB/day

Transcoding: Distributed worker pool
  - Split video into chunks → parallel transcoding → reassemble
  - Use DAG (Directed Acyclic Graph) for pipeline: decode → filter → encode → package

Deduplication:
  - Hash first few frames → detect re-uploads
  - Save storage + transcoding costs
```

---

## 5. Uber / Ride-Sharing

### Requirements
```
Functional: Request ride, match with driver, real-time tracking, ETA, pricing
Non-functional: Low latency matching (<30s), real-time location, 20M rides/day
```

### Architecture
```
                    ┌───────────────┐
    Rider App ────→ │  API Gateway  │ ←──── Driver App
                    └───────┬───────┘
                            │
         ┌──────────────────┼──────────────────┐
         ▼                  ▼                  ▼
   ┌──────────┐      ┌───────────┐      ┌──────────┐
   │  Ride    │      │ Location  │      │ Matching │
   │ Service  │      │ Service   │      │ Service  │
   └──────────┘      └─────┬─────┘      └──────────┘
                            │
                     ┌──────▼──────┐
                     │ Geo-Spatial │
                     │   Index     │
                     │(QuadTree/   │
                     │ Geohash)    │
                     └─────────────┘
```

### Geo-Spatial Indexing
```
Geohash: Encode lat/lng into a string
  (37.7749, -122.4194) → "9q8yy"
  Nearby locations share prefix → efficient proximity search

QuadTree: Recursively divide 2D space into 4 quadrants
  ┌────┬────┐
  │ NW │ NE │   Each cell subdivides when it has too many drivers
  ├────┼────┤   Leaf nodes = areas with few drivers (low density)
  │ SW │ SE │   Deep nodes = areas with many drivers (high density)
  └────┴────┘

Matching algorithm:
1. Rider requests ride at location (lat, lng)
2. Find nearby drivers using geospatial index (radius search)
3. Rank by: distance, ETA, driver rating, acceptance rate
4. Send request to top driver → accept/reject → next driver
```

### Location Updates
```
Drivers send location every 3-5 seconds
- 1M active drivers × 1 update/4s = 250,000 updates/sec
- Store in: Redis (for real-time) + Kafka (for analytics)
- Update geospatial index on each location change
```

---

## 6. Rate Limiter

### Requirements
```
Functional: Limit requests per user/IP, configurable limits, return 429 when exceeded
Non-functional: Low latency (<1ms overhead), distributed, accurate
```

### Algorithms

**1. Token Bucket** (most common)
```
Bucket capacity: 10 tokens
Refill rate: 1 token/second

Request arrives:
  Tokens > 0 → consume token, allow request
  Tokens = 0 → reject (429 Too Many Requests)

┌──────────────┐
│ Bucket: 7/10 │  ← Tokens available
│ ○○○○○○○···   │
│ Refill: 1/s  │
└──────────────┘

✓ Allows bursts (up to bucket size)
✓ Simple to implement
```

**2. Sliding Window Log**
```
Keep timestamp of each request in a sorted set.

Request at time T:
1. Remove all entries older than (T - window_size)
2. Count remaining entries
3. If count < limit → allow and add timestamp
4. If count >= limit → reject

✓ Most accurate
✗ Memory-heavy (stores every timestamp)
```

**3. Sliding Window Counter**
```
Combine fixed window counts with weighted average.

Previous window: 42 requests (in 0:00-1:00)
Current window:  18 requests (in 1:00-1:30, 50% through)

Weighted count = 42 × 50% + 18 = 39

If limit = 50 → allow (39 < 50)

✓ Memory efficient
✓ Smooth (no boundary spikes)
```

### Distributed Rate Limiting
```
Challenge: Multiple API servers need shared rate limit state

Solution: Centralized Redis
  API Server 1 ─┐
  API Server 2 ──┼──→ Redis (INCR key, EXPIRE)
  API Server 3 ─┘

Redis commands (Token Bucket):
  local tokens = redis.get(user_key)
  if tokens > 0 then
    redis.decr(user_key)
    return ALLOWED
  else
    return REJECTED
  end

Use Lua script for atomic check-and-decrement.
```

---

## 7. Distributed Key-Value Store

### Requirements
```
Functional: PUT(key, value), GET(key), DELETE(key)
Non-functional: High availability, tunable consistency, partition tolerance
```

### Architecture (DynamoDB-style)
```
┌─────────────────────────────────────────────┐
│             Consistent Hash Ring             │
│                                             │
│    Node A ── Node B ── Node C ── Node D     │
│      │                              │       │
│      └──────────────────────────────┘       │
│                                             │
│  Key "user:123" → hash → lands on Node B   │
│  Replicate to next N-1 nodes (C and D)     │
└─────────────────────────────────────────────┘

Write path:
Client → Coordinator (any node) → Write to N replicas
  Respond after W acknowledgments (tunable)

Read path:
Client → Coordinator → Read from R replicas
  Return value with highest version
  If conflict → return all versions (client resolves)

Tunable consistency:
  N=3, W=2, R=2 → Strong consistency (W+R > N)
  N=3, W=1, R=1 → Eventual consistency (fast but stale possible)
```

### Conflict Resolution
```
Using vector clocks:
  Node A writes: {value: "v1", clock: [A:1]}
  Node B writes: {value: "v2", clock: [B:1]}

  Both clocks are concurrent (neither dominates)
  → Conflict! Keep both versions
  → Client resolves on next read (e.g., merge shopping carts)

Using Last-Write-Wins (LWW):
  Compare timestamps → keep latest
  Simple but can lose data
```

---

## 8. Design Patterns Quick Reference

| Pattern | When You See | Reach For |
|---------|-------------|-----------|
| High read volume | → Caching (Redis) + Read replicas |
| High write volume | → Message queue + Async processing |
| Large files | → Object storage (S3) + CDN |
| Real-time data | → WebSockets + Pub/Sub |
| Search | → Elasticsearch / Inverted index |
| Geospatial | → Geohash / QuadTree |
| Rate limiting | → Token bucket + Redis |
| Fan-out | → Push for small fan-out, Pull for large |
| Ordering | → Message queue with partitioning |
| Unique ID gen | → Snowflake ID / UUID / Pre-generated pool |
| Transactions | → Saga pattern (microservices) |
| Leader election | → Raft / ZooKeeper |

---

## Active Recall Questions

1. Design a URL shortener from scratch. What's your key generation strategy?
2. Explain the fan-out problem in Twitter. What's the hybrid approach?
3. Design a chat system. How do you handle offline users?
4. How does YouTube handle video transcoding at scale?
5. Explain how Uber finds nearby drivers. What data structure?
6. Compare Token Bucket vs Sliding Window for rate limiting.
7. Design a distributed key-value store. How do you handle consistency?
8. You need to design a notification system. Walk through the architecture.
9. Design Google Docs collaborative editing. How do you handle conflicts?
10. Pick any system above and add: monitoring, failure handling, and scaling strategy.
