# Databases — The Data Layer

> **Learning goal:** Know when to use which database, how to design schemas, and how databases scale.

---

## Before You Begin — Plain-English Setup

- **A database (DB)** is software that stores data on disk and lets you ask it questions ("queries") to get data back.
- **A table** (in SQL) is like a spreadsheet — columns (fields like `name`, `email`) and rows (one per user).
- **A row** = one record. A **column** = one attribute.
- **A schema** = the rules for what columns exist and what types they hold (text, number, date).
- **A query** = a question you send the DB. Example SQL: `SELECT name FROM users WHERE id = 42`.
- **A JOIN** combines rows from two tables based on a matching column (e.g., orders + customers).
- **A primary key** = a column whose value is unique per row (often `id`). The "name tag" of the row.
- **A foreign key** = a column that points to another table's primary key (e.g., `user_id` in `orders` points to `id` in `users`).
- **A transaction** = a group of operations treated as one unit — either all happen or none do.
- **ACID** = the 4 guarantees a traditional SQL DB makes about transactions. Defined in section 2 below.
- **An index** = a data structure that makes searches on a column fast. Like a book index.
- **NoSQL** = "Not Only SQL" — any DB that doesn't use the strict table-based SQL model. 4 flavors (section 1).

If you've never written SQL, that's OK. Understand the concepts, not the syntax.

---

## 1. SQL vs NoSQL — Decision Framework

### SQL (Relational Databases)
```
Structure: Tables with rows and columns
Schema:    Fixed, predefined (ALTER TABLE to change)
Relations: JOINs connect tables
ACID:      Guaranteed transactions
Examples:  PostgreSQL, MySQL, Oracle, SQL Server
```

### NoSQL (Non-Relational)
```
4 Types:
┌─────────────────┬──────────────┬───────────────────────────────────┐
│ Type            │ Examples     │ Best For                          │
├─────────────────┼──────────────┼───────────────────────────────────┤
│ Document        │ MongoDB,     │ Flexible schemas, nested data,    │
│                 │ CouchDB      │ content management                │
├─────────────────┼──────────────┼───────────────────────────────────┤
│ Key-Value       │ Redis,       │ Caching, session storage,         │
│                 │ DynamoDB     │ simple lookups                    │
├─────────────────┼──────────────┼───────────────────────────────────┤
│ Wide-Column     │ Cassandra,   │ Time-series, IoT, event logging,  │
│                 │ HBase        │ write-heavy workloads             │
├─────────────────┼──────────────┼───────────────────────────────────┤
│ Graph           │ Neo4j,       │ Social networks, recommendations, │
│                 │ Amazon Neptune│ fraud detection                  │
└─────────────────┴──────────────┴───────────────────────────────────┘
```

### Decision Matrix
```
Choose SQL when:
✓ Data has clear relationships (foreign keys, JOINs)
✓ You need ACID transactions
✓ Schema is well-defined and unlikely to change rapidly
✓ Complex queries (aggregations, GROUP BY, subqueries)
✓ Data integrity is critical (financial, inventory)

Choose NoSQL when:
✓ Schema changes frequently or varies per record
✓ Massive scale needed (horizontal scaling built-in)
✓ Simple access patterns (key-value lookups)
✓ High write throughput needed
✓ Hierarchical or denormalized data
✓ Low latency at scale matters more than complex queries
```

---

## 2. ACID vs BASE

### ACID (SQL guarantee)
```
Atomicity    → All or nothing. Transaction either fully completes or fully rolls back.
Consistency  → Database moves from one valid state to another.
Isolation    → Concurrent transactions don't interfere with each other.
Durability   → Once committed, data survives crashes (written to disk).
```

### BASE (NoSQL trade-off)
```
Basically Available → System is always available (may return stale data)
Soft state         → State may change over time even without input
Eventually consistent → System will converge to consistent state
```

### Feynman Check
> "ACID is like a bank teller who processes one customer at a time and double-checks every transaction. BASE is like a busy market where everyone trades simultaneously — things might be temporarily messy, but they settle up by end of day."

---

## 3. Indexing

### What It Does
Without index: Database scans every row (O(n)) — **full table scan**
With index: Database jumps directly to matching rows (O(log n)) — **index lookup**

### B-Tree Index (Most Common)
```
                    [50]
                   /    \
             [20,30]    [70,80]
            /  |  \    /  |  \
          [10][25][35][60][75][90]  ← Leaf nodes point to actual rows
```
- Balanced tree structure
- Good for: range queries, equality, sorting
- O(log n) for search, insert, delete

### Hash Index
```
key "user_123" → hash() → bucket 7 → row pointer
```
- O(1) lookups (fastest for exact match)
- Cannot do range queries
- Good for: exact equality lookups only

### Types of Indexes
| Type | Use Case |
|------|----------|
| **Primary index** | Auto-created on primary key |
| **Secondary index** | Created on non-PK columns for faster queries |
| **Composite index** | Index on multiple columns (order matters!) |
| **Covering index** | Index contains all columns needed (no table lookup) |
| **Full-text index** | For text search (LIKE, CONTAINS) |

### Index Trade-offs
```
✓ Faster reads (much faster for indexed columns)
✗ Slower writes (must update index on every INSERT/UPDATE/DELETE)
✗ Extra storage (index takes disk space)

Rule of thumb:
- Index columns used in WHERE, JOIN, ORDER BY
- Don't over-index (every index slows writes)
- Composite index on (A, B) helps queries on A, or (A AND B), but NOT just B
```

---

## 4. Sharding (Horizontal Partitioning)

### What It Does
Splits data across multiple database servers, each holding a subset.

```
Before sharding (one DB holds everything):
┌─────────────────────────────┐
│   Users: 100M rows          │  ← Single point of failure
│   All queries hit one DB    │  ← Bottleneck
└─────────────────────────────┘

After sharding (data split across DBs):
┌───────────┐  ┌───────────┐  ┌───────────┐
│ Shard 1   │  │ Shard 2   │  │ Shard 3   │
│ Users A-H │  │ Users I-P │  │ Users Q-Z │
└───────────┘  └───────────┘  └───────────┘
```

### Sharding Strategies

**1. Range-Based Sharding**
```
Shard 1: user_id 1 - 1,000,000
Shard 2: user_id 1,000,001 - 2,000,000
Shard 3: user_id 2,000,001 - 3,000,000

✓ Simple to implement
✓ Range queries are efficient
✗ Hotspots if distribution is uneven (e.g., new users all go to last shard)
```

**2. Hash-Based Sharding**
```
shard_number = hash(user_id) % num_shards

✓ Even distribution
✗ Range queries require hitting ALL shards
✗ Resharding is painful (all data must be redistributed)
```

**3. Consistent Hashing** (the smart way)
```
           0°
          ╱   ╲
     330°│     │30°
        S3    S1
     300°│     │60°
          ╲   ╱
          270°
         S2

- Servers and keys are mapped to a ring (0-360°)
- A key is stored on the next server clockwise
- Adding/removing a server only affects neighbors
- Virtual nodes: each server gets multiple positions for even distribution

✓ Minimal data movement when adding/removing servers
✓ Even distribution with virtual nodes
Used by: DynamoDB, Cassandra, Discord
```

### Sharding Challenges
| Problem | Explanation |
|---------|-------------|
| **JOINs across shards** | Can't easily JOIN data on different servers |
| **Resharding** | Adding shards requires data migration |
| **Hotspots** | One shard gets disproportionate traffic |
| **Referential integrity** | Foreign keys don't work across shards |
| **Distributed transactions** | ACID across shards is very hard |

---

## 5. Replication

### Why Replicate?
- **Availability:** If one server dies, another can serve
- **Read performance:** Distribute reads across replicas
- **Geo-distribution:** Put data close to users

### Leader-Follower (Master-Slave)
```
  Writes → [Leader/Master]
              │
     ┌────────┼────────┐
     ▼        ▼        ▼
  [Follower] [Follower] [Follower] ← Reads distributed here
```
- All writes go to leader
- Leader replicates to followers
- Followers serve reads
- If leader dies: promote a follower (failover)

### Replication Methods
| Method | How | Trade-off |
|--------|-----|-----------|
| **Synchronous** | Leader waits for ALL followers to confirm | Strong consistency, higher latency |
| **Asynchronous** | Leader doesn't wait for followers | Lower latency, risk of data loss |
| **Semi-synchronous** | Leader waits for at least 1 follower | Balance of both |

### Multi-Leader Replication
```
[Leader A] ←→ [Leader B]    (both accept writes)
    │              │
[Followers]   [Followers]
```
- Used in multi-datacenter setups
- **Challenge:** Write conflicts (both leaders modify same row)
- Conflict resolution: Last Write Wins, merge, custom logic

### Leaderless Replication (Quorum)
```
Write to 3 of 5 nodes → W = 3
Read from 3 of 5 nodes → R = 3
W + R > N (5) → Guaranteed to read latest write

Common quorum: N=3, W=2, R=2
```
- Used by: DynamoDB, Cassandra
- No single leader = no failover needed

---

## 6. Normalization vs Denormalization

### Normalization (Eliminate Redundancy)
```
Users Table:          Posts Table:
┌────┬───────┐       ┌────┬─────────┬─────────┐
│ id │ name  │       │ id │ user_id │ content │
├────┼───────┤       ├────┼─────────┼─────────┤
│ 1  │ Alice │       │ 1  │ 1       │ Hello   │
│ 2  │ Bob   │       │ 2  │ 1       │ World   │
└────┴───────┘       └────┴─────────┴─────────┘

To get post with author: SELECT * FROM posts JOIN users ON posts.user_id = users.id
```
✓ No duplicate data, easier updates, data integrity
✗ Requires JOINs (slower reads), doesn't scale well across shards

### Denormalization (Optimize for Reads)
```
Posts Table (denormalized):
┌────┬─────────┬───────────┬─────────┐
│ id │ user_id │ user_name │ content │
├────┼─────────┼───────────┼─────────┤
│ 1  │ 1       │ Alice     │ Hello   │
│ 2  │ 1       │ Alice     │ World   │
└────┴─────────┴───────────┴─────────┘

No JOIN needed! But "Alice" is stored twice.
```
✓ Faster reads (no JOINs), works well with NoSQL/sharding
✗ Data duplication, harder to update (must update everywhere)

### When to Use Each
- **Normalize** for write-heavy, data-integrity-critical systems (banking, inventory)
- **Denormalize** for read-heavy, performance-critical systems (news feed, product catalog)

---

## 7. Database Scaling Patterns Summary

```
Level 1: Single Server
         → Optimize queries, add indexes

Level 2: Read Replicas
         → Separate read/write traffic
         ┌──────────┐
Writes → │  Master  │
         └────┬─────┘
              │ replication
     ┌────────┼────────┐
     ▼        ▼        ▼
  [Replica] [Replica] [Replica] ← Reads

Level 3: Caching Layer
         → Cache hot data in Redis/Memcached

Level 4: Vertical Partitioning
         → Split by feature (users DB, posts DB, payments DB)

Level 5: Horizontal Sharding
         → Split data within a feature across servers
```

---

## Active Recall Questions

1. You're building a social media app. Would you use SQL or NoSQL? Defend your choice.
2. Explain ACID. Give a real-world example where each property matters.
3. What happens to write performance when you add an index? Why?
4. You have 1 billion users. Design a sharding strategy. What are the trade-offs?
5. Explain consistent hashing. Why is it better than hash(key) % N?
6. What's the difference between synchronous and asynchronous replication? When would you choose each?
7. A table has columns (user_id, country, created_at). You query often by country + created_at. What index would you create?
8. Explain the quorum formula W + R > N. What does it guarantee?
9. When would you denormalize a database? What problems does it create?
10. Walk through the 5 levels of database scaling. When do you move from one to the next?
