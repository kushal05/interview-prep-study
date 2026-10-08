# System Design Fundamentals

> **Learning goal:** Understand the core vocabulary and mental models that every system design builds upon.

---

## Before You Begin — Plain-English Setup

If any of these are new, read them first. The rest of this file builds on them.

- **A server** is just a computer that listens for messages from other computers (clients) and replies. Your phone is a client; the YouTube computer that sends you videos is a server.
- **A request** is the message a client sends ("show me video #42"). A **response** is the reply.
- **Latency** = how long ONE request takes. Measured in time (`ms` = milliseconds, `μs` = microseconds, `ns` = nanoseconds). Lower is better.
  - 1 second = 1,000 ms = 1,000,000 μs = 1,000,000,000 ns.
- **Throughput** = how many requests you can handle per second. Higher is better. Often written as **QPS** (Queries Per Second) or **RPS** (Requests Per Second).
- **A distributed system** is one where several computers work together, often pretending to be one big computer. They talk over a network.
- **A network partition** means the connection between some computers breaks. They are still running but cannot reach each other.
- **A replica** is a copy of your data stored on a different computer (so if one dies, you still have the data).
- **A node** is one computer in the group. **A cluster** is the whole group.
- **DAU** = Daily Active Users — distinct users who used your app in one day.

You do not need to know HTTP, databases, or caching to read this file. Each has its own file (02, 03, 04).

---

## 1. Scalability

### What is it?
The ability of a system to handle growing amounts of work by adding resources.

### Two Types

**Vertical Scaling (Scale Up)**
```
Before:  [Server: 4 CPU, 16GB RAM]
After:   [Server: 32 CPU, 128GB RAM]
```
- Simpler — no code changes needed
- Has a ceiling (you can't buy an infinitely powerful machine)
- Single point of failure
- Expensive at the top end

**Horizontal Scaling (Scale Out)**
```
Before:  [Server A]
After:   [Server A] [Server B] [Server C] [Server D]
         ↑ Load Balancer distributes traffic ↑
```
- Virtually unlimited scaling
- Requires distributed system design (harder)
- Better fault tolerance (one dies, others continue)
- More cost-effective at scale

### Feynman Check
> "If my lemonade stand gets too popular, vertical scaling = buying a bigger table. Horizontal scaling = opening more stands."

---

## 2. Latency vs Throughput

### Latency
**Time taken for a single operation to complete** (measured in ms/s)

```
Latency Numbers Every Engineer Should Know:
─────────────────────────────────────────────
L1 cache reference .................. 0.5 ns
L2 cache reference .................... 7 ns
Main memory reference ............... 100 ns
SSD random read .................. 16,000 ns  (16 μs)
HDD random read .............. 2,000,000 ns  (2 ms)
Send packet CA → Netherlands → CA  150,000,000 ns  (150 ms)
─────────────────────────────────────────────
```

**Key insight:** There's a ~1000x gap between memory and disk, and a ~1000x gap between disk and network. This is WHY caching exists.

### Throughput
**Number of operations completed per unit time** (measured in QPS, RPS, TPS)

```
Example:
- Latency: Each request takes 50ms
- Throughput with 1 thread: 20 requests/sec
- Throughput with 100 threads: 2,000 requests/sec
```

### The Relationship
- Low latency ≠ high throughput (and vice versa)
- A highway analogy: Latency = speed limit. Throughput = number of lanes.
- You can have high throughput with high latency (batch processing)

---

## 3. Availability

### Definition
**Percentage of time a system is operational and accessible**

```
Availability Table (The "Nines"):
──────────────────────────────────────────
99%      (two 9s)    = 3.65 days downtime/year
99.9%    (three 9s)  = 8.77 hours downtime/year
99.99%   (four 9s)   = 52.6 minutes downtime/year
99.999%  (five 9s)   = 5.26 minutes downtime/year
──────────────────────────────────────────
```

### How to Achieve High Availability
1. **Redundancy** — No single point of failure
2. **Replication** — Data exists in multiple places
3. **Failover** — Automatic switching to backup
4. **Health checks** — Detect failures fast
5. **Graceful degradation** — Serve partial results rather than failing completely

### Availability in Series vs Parallel

```
Series (both must work):
A(99.9%) → B(99.9%) = 99.9% × 99.9% = 99.8%
↓ Availability DECREASES

Parallel (either works):
A(99.9%)  }
          } = 1 - (0.001 × 0.001) = 99.9999%
B(99.9%)  }
↓ Availability INCREASES
```

---

## 4. CAP Theorem

> **First-time reader's warning:** "Consistency" in CAP is NOT the same as the "C" in ACID. CAP's C means "everyone sees the same data right now." ACID's C means "the database stays in a valid state after a transaction." Don't mix them up.

### The Three Properties
A distributed system can guarantee at most **2 out of 3**:

```
        Consistency
           /\
          /  \
         /    \
        / PICK \
       /  TWO   \
      /          \
     /____________\
Availability — Partition Tolerance
```

**Consistency (C):** Every read receives the most recent write or an error
**Availability (A):** Every request receives a response (no errors, no timeouts)
**Partition Tolerance (P):** System continues despite network partitions between nodes

### The Real Choice
In distributed systems, **network partitions WILL happen**. So P is non-negotiable. The real choice is:

| Choice | Behavior During Partition | Examples |
|--------|--------------------------|----------|
| **CP** | Returns error or waits (sacrifices availability) | MongoDB, HBase, Redis Cluster |
| **AP** | Returns potentially stale data (sacrifices consistency) | Cassandra, DynamoDB, CouchDB |

### PACELC Theorem (Extended CAP)
```
If Partition → choose A or C
Else (normal operation) → choose Latency or Consistency

Example:
- DynamoDB: PA/EL (available during partition, low latency normally)
- MongoDB:  PC/EC (consistent during partition, consistent normally)
```

### Feynman Check
> "Imagine 3 friends keeping a shared shopping list. If the phone network goes down (partition), they can either: wait until it's back to stay in sync (CP), or keep adding items independently and merge later with possible duplicates (AP)."

---

## 5. Consistency Models

### Strong Consistency
- After a write completes, ALL subsequent reads return that value
- Feels like a single machine
- Higher latency (must wait for all replicas to agree)
- Example: Bank account balance

### Eventual Consistency
- After a write, reads MAY return stale data for a while
- Eventually, all replicas converge to the same value
- Lower latency, higher availability
- Example: Social media like count, DNS propagation

### Causal Consistency
- Operations that are causally related are seen in the same order by all nodes
- Concurrent operations may be seen in different orders
- Middle ground between strong and eventual

```
Timeline showing Eventual Consistency:

Client A writes: balance = 100
  │
  ├── Replica 1: balance = 100  ✓ (immediate)
  ├── Replica 2: balance = 50   ✗ (stale, hasn't replicated yet)
  └── Replica 3: balance = 100  ✓ (replicated)

  ... time passes ...

  ├── Replica 1: balance = 100  ✓
  ├── Replica 2: balance = 100  ✓ (eventually consistent)
  └── Replica 3: balance = 100  ✓
```

---

## 6. Reliability vs Fault Tolerance

### Reliability
The system performs its intended function correctly over time.

### Fault Tolerance
The system continues operating properly even when components fail.

**Types of Failures:**
```
1. Crash failure     → Process/server dies completely
2. Omission failure  → Messages are lost (network drops)
3. Timing failure    → Response comes too late
4. Byzantine failure → Node behaves arbitrarily/maliciously (hardest to handle)
```

**Strategies:**
| Strategy | How it Works |
|----------|-------------|
| Replication | Multiple copies of data/services |
| Checkpointing | Save state periodically, restore on failure |
| Circuit Breaker | Stop calling a failing service, fail fast |
| Bulkhead | Isolate failures to prevent cascade |
| Retry with backoff | Retry failed operations with increasing delay |

---

## 7. Back-of-the-Envelope Estimation

### Powers of 2 (Memory)
```
2^10 = 1 Thousand    = 1 KB
2^20 = 1 Million     = 1 MB
2^30 = 1 Billion     = 1 GB
2^40 = 1 Trillion    = 1 TB
```

### Common Estimates
```
QPS from DAU:
- 1M DAU, each makes 10 requests/day
- QPS = (1M × 10) / 86,400 ≈ 100 QPS
- Peak = 2-3x average ≈ 250 QPS

Storage:
- 1 tweet ≈ 250 bytes (text only)
- 1 photo ≈ 200 KB (compressed)
- 1 minute of video ≈ 50 MB (compressed)
- 1M users × 1 KB profile = 1 GB

Bandwidth:
- QPS × average response size = bandwidth
- 1000 QPS × 10 KB = 10 MB/s
```

### Quick Math Tricks
```
Seconds in a day: ~100,000 (actually 86,400)
Seconds in a year: ~30 million (actually 31.5M)
1 server can handle: ~1,000 QPS (simple), ~100 QPS (complex DB)
```

---

## 8. System Design Trade-offs

Every design decision is a trade-off. Master these pairs:

| Trade-off | When to favor Left | When to favor Right |
|-----------|-------------------|-------------------|
| Consistency ↔ Availability | Financial, inventory | Social media, analytics |
| Latency ↔ Throughput | Real-time (gaming, chat) | Batch processing, ETL |
| Read-optimized ↔ Write-optimized | Read-heavy (99:1) | Write-heavy (logging) |
| SQL ↔ NoSQL | Complex queries, ACID | Flexible schema, scale |
| Monolith ↔ Microservices | Small team, early stage | Large team, scale needs |
| Push ↔ Pull | Few producers, many consumers | Many producers, few consumers |
| Sync ↔ Async | Need immediate result | Can tolerate delay |

---

## Active Recall Questions

1. What's the difference between horizontal and vertical scaling? When would you choose each?
2. A system has 99.9% availability. How much downtime per year? What about 99.99%?
3. Explain CAP theorem. Why is the real choice between CP and AP?
4. What's the difference between strong and eventual consistency? Give a real-world example of each.
5. You have 10M DAU, each user makes 20 requests/day. What's the average QPS? Peak QPS?
6. A system stores 500 bytes per record, gets 1M new records/day. How much storage per year?
7. Explain the difference between latency and throughput using an analogy.
8. What is PACELC? How does it extend CAP?
9. Name 3 strategies for achieving fault tolerance.
10. When would you sacrifice consistency for availability? Give a concrete example.
