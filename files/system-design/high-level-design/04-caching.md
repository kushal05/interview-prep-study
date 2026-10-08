# Caching

> **Learning goal:** Master caching strategies, invalidation patterns, and know when caching helps vs hurts.

---

## Before You Begin — Plain-English Setup

- **A cache** is a smaller, faster copy of data you use a lot. Your fridge is a cache for groceries: faster than the supermarket, smaller, holds the things you use most.
- **RAM (Random Access Memory)** is your computer's short-term memory. Reading from RAM is ~1000× faster than reading from disk. Caches usually live in RAM.
- **A cache hit** = the data you wanted was already in the cache. Fast.
- **A cache miss** = it wasn't, so you have to ask the slow original source.
- **Hit rate** = (hits) / (hits + misses). A 95% hit rate means 95 of every 100 lookups were fast.
- **Eviction** = removing something from the cache to make room for new data.
- **TTL (Time To Live)** = how long data lives in the cache before automatically being removed.
- **Stale data** = data in the cache that is older than what's now in the database. (The cache hasn't been updated yet.)
- **Invalidation** = telling the cache "this data is no longer correct; delete it."
- **Redis / Memcached** = the two most common cache servers — programs you run on dedicated machines to hold cached data in RAM.

> The big idea: caching trades a small risk of seeing slightly old data for a *huge* speed-up.

---

## 1. Why Caching?

```
Without cache:
Client → Server → Database (100ms)

With cache:
Client → Server → Cache HIT (5ms) → Response
                → Cache MISS → Database (100ms) → Store in Cache → Response

Result: 95% cache hit rate = 95% of requests are 20x faster
```

### The Core Trade-off
```
✓ Dramatically faster reads
✓ Reduces database load
✗ Data can become stale (consistency risk)
✗ Extra complexity (invalidation, eviction)
✗ Additional infrastructure cost
```

---

## 2. Caching Layers

```
┌─────────────────────────────────────────────┐
│                   Client                     │
│  [Browser Cache] [Service Worker Cache]      │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│                    CDN                        │
│  [Edge Cache - Static assets, API responses] │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│              API Gateway / LB                 │
│  [Response Cache - Common API responses]      │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│              Application Layer                │
│  [In-Memory Cache - Local process cache]      │
│  [Distributed Cache - Redis / Memcached]      │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│                 Database                      │
│  [Query Cache] [Buffer Pool / Page Cache]     │
└──────────────────────────────────────────────┘
```

---

## 3. Caching Strategies

### Cache-Aside (Lazy Loading) — Most Common
```
Read:
1. App checks cache
2. Cache HIT → return data
3. Cache MISS → read from DB → write to cache → return data

Write:
1. Write to DB
2. Invalidate cache (delete the key)
```
```
Client → App: "Get user 123"
App → Cache: GET user:123
Cache → App: MISS
App → DB: SELECT * FROM users WHERE id=123
DB → App: {name: "Alice", ...}
App → Cache: SET user:123 = {name: "Alice", ...}
App → Client: {name: "Alice", ...}
```
- ✓ Only caches what's actually requested (no wasted memory)
- ✓ Cache failure doesn't break the system (falls back to DB)
- ✗ Cache miss = 3 round trips (cache + DB + write cache)
- ✗ Stale data possible if DB is updated without invalidating cache

### Read-Through
```
1. App reads from cache
2. Cache MISS → CACHE reads from DB (not the app)
3. Cache stores result and returns to app
```
- ✓ Simpler application code (cache manages DB reads)
- ✗ Cache library must support this pattern

### Write-Through
```
1. App writes to cache
2. Cache synchronously writes to DB
3. Return success only after both succeed
```
- ✓ Cache is always consistent with DB
- ✗ Higher write latency (two writes per operation)
- ✗ Caches data that may never be read

### Write-Behind (Write-Back)
```
1. App writes to cache
2. Cache acknowledges immediately
3. Cache asynchronously writes to DB in batches
```
- ✓ Very fast writes (only writes to memory)
- ✓ Batching reduces DB load
- ✗ Risk of data loss if cache crashes before persisting
- ✗ Complex failure handling

### Write-Around
```
1. App writes directly to DB (bypasses cache)
2. Cache is only populated on read (cache-aside for reads)
```
- ✓ Cache not flooded with writes that are never read
- ✗ Cache miss on first read after write

### Strategy Comparison
```
                 Consistency   Write Speed   Read Speed   Complexity
Cache-Aside      Medium        Medium        Fast*        Low
Read-Through     Medium        Medium        Fast*        Medium
Write-Through    High          Slow          Fast         Medium
Write-Behind     Low           Very Fast     Fast         High
Write-Around     Medium        Fast          Slow*        Low

* After first read (warm cache)
```

---

## 4. Cache Eviction Policies

When cache is full, which entry do we remove?

### LRU (Least Recently Used) — Most Common
```
Cache (capacity 3):
Access: A B C D B

[A] → [B,A] → [C,B,A] → [D,C,B] (A evicted) → [B,D,C] (B moved to front)

✓ Good for: Most workloads (recent = likely needed again)
```

### LFU (Least Frequently Used)
```
Tracks access count. Evicts item with lowest count.

✓ Good for: Workloads with stable hot items
✗ Problem: New items always have low count → evicted too soon
```

### FIFO (First In First Out)
```
Evicts oldest entry regardless of access pattern.
✓ Simple
✗ Doesn't consider access patterns
```

### TTL (Time To Live)
```
Every entry has an expiration time.
After TTL, entry is automatically removed.

SET user:123 = "Alice" EX 3600  (expires in 1 hour)

✓ Guarantees staleness is bounded
✓ Works alongside LRU/LFU
```

### Decision Table
| Policy | Use When |
|--------|----------|
| LRU | General purpose, don't know access pattern |
| LFU | Some items are consistently popular |
| TTL | Data has a natural freshness period |
| LRU + TTL | Most production systems (belt + suspenders) |

---

## 5. Cache Invalidation

> "There are only two hard things in CS: cache invalidation and naming things." — Phil Karlton

### Strategies
```
1. TTL-based: Set expiration time
   ✓ Simple, guaranteed freshness bound
   ✗ Stale during TTL window

2. Event-based: Invalidate on write
   ✓ Immediate consistency
   ✗ Must track all write paths

3. Version-based: Store version with data
   ✓ Detects staleness on read
   ✗ Requires version comparison logic
```

### Common Problems

**Thundering Herd / Cache Stampede**
```
Problem:
Popular key expires → 1000 simultaneous requests → all miss cache
→ all hit DB simultaneously → DB overloaded

Solutions:
1. Lock/Mutex: First miss acquires lock, others wait
2. Early refresh: Refresh cache BEFORE TTL expires
3. Stale-while-revalidate: Serve stale data while refreshing in background
```

**Cache Penetration**
```
Problem:
Requests for data that doesn't exist → always misses cache
→ always hits DB → DB overloaded

Solutions:
1. Cache negative results: SET user:999 = NULL EX 300
2. Bloom filter: Check if key COULD exist before querying DB
```

**Cache Avalanche**
```
Problem:
Many keys expire at the same time → massive DB load spike

Solutions:
1. Random TTL jitter: TTL = base_ttl + random(0, 300)
2. Staggered expiration
3. Never expire (refresh in background)
```

---

## 6. Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Strings, Lists, Sets, Hashes, Sorted Sets, Streams | Strings only |
| Persistence | RDB snapshots + AOF logs | None (pure cache) |
| Replication | Built-in (master-slave) | None |
| Clustering | Redis Cluster (built-in) | Client-side sharding |
| Pub/Sub | Yes | No |
| Lua scripting | Yes | No |
| Memory efficiency | Less efficient | More efficient for simple strings |
| Multi-threading | Single-threaded (6.0+ has I/O threads) | Multi-threaded |

### When to Choose
- **Redis:** Need data structures, persistence, pub/sub, or advanced features
- **Memcached:** Simple key-value caching, multi-threaded performance, memory efficiency

---

## 7. Caching in System Design Interviews

### Cache Sizing Estimation
```
Example: Cache user profiles for a social media app
- 100M DAU
- 20% of users are "hot" (80/20 rule) = 20M profiles to cache
- Average profile size = 1 KB
- Cache needed = 20M × 1 KB = 20 GB
- With overhead = ~30 GB → fits in a single Redis instance (or small cluster)
```

### When NOT to Cache
- Write-heavy workloads (cache invalidation overhead dominates)
- Data that changes every request (cache hit rate too low)
- When consistency is critical and stale data is unacceptable
- Large objects accessed infrequently (wastes memory)

---

## Active Recall Questions

1. Explain cache-aside pattern. Draw the read and write flow.
2. What's the difference between write-through and write-behind? When would you use each?
3. Your cache has a 70% hit rate. Is that good? How would you improve it?
4. Explain the thundering herd problem. Give 3 solutions.
5. What is cache penetration? How does a Bloom filter help?
6. You need to cache 10M user sessions (500 bytes each). How much memory? Redis or Memcached?
7. Why is LRU the most common eviction policy? When would LFU be better?
8. How would you add caching to a URL shortener? Where in the stack?
9. Explain the "stale-while-revalidate" pattern.
10. Name the 5 caching layers from client to database.
