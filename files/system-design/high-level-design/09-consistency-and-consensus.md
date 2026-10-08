# Consistency & Consensus in Distributed Systems

> **Learning goal:** Understand how distributed systems agree on state — the hardest problems in system design.

---

## Before You Begin — Plain-English Setup

This is the hardest chapter. Read 01-fundamentals.md first if you haven't.

- **Consistency** (in this file) = "all replicas agree on what the current value is." Different from ACID's C — see 01-fundamentals section 4.
- **Replica** = a copy of the data on another node.
- **Leader / follower** = in many systems, ONE node is the leader (handles writes), and the rest are followers (copy the leader's data). Same as "master / slave" in old terminology.
- **Quorum** = a majority. For N nodes, quorum = > N/2. Example: 3 of 5 = quorum.
- **Consensus** = a procedure where all nodes agree on a value, even when some nodes are slow or fail.
- **Split brain** = bad scenario where the cluster splits and TWO sides each think they're in charge.
- **Term / epoch** = a number that monotonically increases each time a new leader is elected. Used to reject messages from a dead old leader.
- **Log** = an ordered list of operations, like a journal. Replicating the log = replicating the data.
- **Commit** = mark an entry as durable. Once committed, it won't be undone.
- **Idempotent** = doing it twice has the same effect as once.
- **Clock skew** = different computers' clocks disagree (they always do, by milliseconds).
- **Logical clock** = a counter that orders events without using real time (Lamport, Vector clocks).
- **Compensation** = an action that undoes a previous action ("refund the payment we just made").

---

## 1. Why Is This Hard?

### The Fundamental Problem
```
In a single machine:
  Thread 1: SET x = 5  →  Thread 2: GET x  →  returns 5  ✓

In a distributed system:
  Node A: SET x = 5  →  Node B: GET x  →  returns ???

  Could return: 5 (if replicated), 3 (stale), error (if partitioned)
  Depends on: replication method, consistency model, network state
```

### What Can Go Wrong
```
1. Network delay:     Message takes longer than expected
2. Network partition: Nodes can't communicate at all
3. Node crash:        Server dies mid-operation
4. Clock skew:        Different nodes have different times
5. Split brain:       Two nodes both think they're the leader
```

---

## 2. Consistency Models (Ordered by Strength)

```
Strongest ────────────────────────────────────────── Weakest
   │                                                    │
Linearizable → Sequential → Causal → Eventual
   │              │            │          │
 "Real-time     "Some       "Cause     "Eventually
  ordering"     ordering"   before     all agree"
                            effect"
```

### Linearizability (Strongest)
```
As if there's a single copy of data, operations happen atomically.

Timeline:
Client A: ───write(x=1)─────────────────────
Client B: ─────────────read(x)──→ MUST return 1
Client C: ────────────────────read(x)──→ MUST return 1

If any read sees the new value, ALL subsequent reads must too.

Where used: Leader-based replication (when reading from leader)
Cost: High latency (must confirm across nodes)
```

### Sequential Consistency
```
All operations appear in SOME sequential order that is consistent
with the program order of each individual node.

Node A operations: A1 then A2
Node B operations: B1 then B2
Valid: A1, B1, A2, B2
Valid: B1, A1, B2, A2
Invalid: A2, A1, B1, B2 (violates A's program order)

Weaker than linearizable because real-time order isn't preserved.
```

### Causal Consistency
```
If operation A CAUSES operation B, then everyone sees A before B.
Concurrent operations (no causal link) can be seen in any order.

Example:
Alice posts: "I got the job!"           (Event A)
Bob sees Alice's post, replies: "Congrats!" (Event B, caused by A)

Causal consistency guarantees:
Everyone sees Alice's post BEFORE Bob's reply.
But unrelated posts can appear in different orders for different users.
```

### Eventual Consistency
```
If no new writes occur, all replicas will EVENTUALLY converge
to the same value. No guarantee on when.

Real-world examples:
- DNS propagation (can take hours)
- Social media like counts (temporary discrepancies)
- Shopping cart across devices (syncs when connected)

Most NoSQL databases default to this.
```

---

## 3. Consensus Algorithms

### What Is Consensus?
Getting N distributed nodes to agree on a single value, even if some nodes fail.

### Paxos (Theoretical Foundation)
```
Roles:
- Proposers: Suggest values
- Acceptors: Vote on proposals
- Learners: Learn the decided value

Two Phases:
Phase 1 (Prepare):
  Proposer → ALL Acceptors: "Will you consider proposal #5?"
  Acceptors → Proposer: "Yes, and the highest proposal I've accepted is #3 with value X"

Phase 2 (Accept):
  Proposer → ALL Acceptors: "Accept proposal #5 with value X"
  Acceptors → Proposer: "Accepted" (if no higher proposal seen)

Majority required: Need > N/2 acceptors to agree = consensus!

Why it's hard: Multiple proposers can conflict (livelock possible)
In practice: Multi-Paxos uses a stable leader to avoid conflicts
```

### Raft (Practical, Understandable Consensus)
```
Designed to be UNDERSTANDABLE (unlike Paxos).

Three roles:
- Leader: Handles all client requests, replicates to followers
- Follower: Passively replicates from leader
- Candidate: Trying to become leader (during election)

Leader Election:
┌──────────┐     timeout      ┌───────────┐   wins election   ┌────────┐
│ Follower │ ──────────────→  │ Candidate │ ────────────────→  │ Leader │
└──────────┘                  └───────────┘                    └────────┘
                                    │                               │
                                    │ loses election                │ discovers
                                    └──────────────────────────────→│ higher term
                                                                    │
                                                              ┌─────▼────┐
                                                              │ Follower │
                                                              └──────────┘

Election rules:
1. Each node has a random election timeout (150-300ms)
2. First to timeout becomes candidate, requests votes
3. Candidate needs majority of votes to become leader
4. Each node can only vote once per term
5. Leader sends heartbeats to prevent new elections

Log Replication:
Client → Leader: "SET x = 5"
Leader: Appends to its log
Leader → All Followers: "Append this entry"
Followers: Append and acknowledge
Leader: Once majority acknowledge → commit → respond to client
Leader → Followers: "Entry committed" (in next heartbeat)
```

### Where Consensus Is Used
| System | Algorithm | Purpose |
|--------|-----------|---------|
| etcd | Raft | Kubernetes state, distributed config |
| ZooKeeper | ZAB (Paxos variant) | Service coordination, leader election |
| CockroachDB | Raft | Distributed SQL consensus |
| Google Spanner | Paxos | Global distributed transactions |
| Consul | Raft | Service discovery, config |

---

## 4. Distributed Transactions

### The Problem
```
Transfer $100 from Account A (Service 1) to Account B (Service 2):
1. Deduct $100 from A
2. Add $100 to B

What if step 1 succeeds but step 2 fails?
Money disappeared! We need BOTH to succeed or BOTH to fail.
```

### Two-Phase Commit (2PC)
```
                  Coordinator
                      │
           Phase 1: PREPARE
              ┌───────┼───────┐
              ▼       ▼       ▼
          [Node A] [Node B] [Node C]
          "Ready?" "Ready?" "Ready?"
          "Yes!"   "Yes!"   "Yes!"
              │       │       │
           Phase 2: COMMIT
              ┌───────┼───────┐
              ▼       ▼       ▼
          [Node A] [Node B] [Node C]
          "Commit!" "Commit!" "Commit!"
          "Done!"   "Done!"   "Done!"

If ANY node says "No" in Phase 1 → Coordinator sends ABORT to all
```

**Problems with 2PC:**
```
✗ Blocking: If coordinator crashes after Phase 1, nodes are stuck
✗ Single point of failure: Coordinator
✗ Slow: Synchronous, high latency
✗ Doesn't scale well
```

### Three-Phase Commit (3PC)
Adds a "pre-commit" phase to reduce blocking. Rarely used in practice.

### Saga Pattern (Preferred for Microservices)
```
Instead of distributed ACID → sequence of local transactions + compensations

Happy path:
Order Created → Payment Charged → Inventory Reserved → Shipping Created
     T1              T2                T3                   T4

Failure at T3:
Order Created → Payment Charged → Inventory FAILED
     T1              T2               T3 fails
                                        │
Compensation:                           ▼
Payment Refunded ← Order Cancelled
     C2                C1

Each step is a local ACID transaction.
Each step has a compensating transaction for rollback.
```

---

## 5. Clocks & Ordering

### Physical Clocks
```
Problem: Clocks on different machines are NEVER perfectly synchronized.

NTP (Network Time Protocol): Syncs clocks to ~1-10ms accuracy
  Still too imprecise for ordering events in a distributed system!

Google's TrueTime: Uses atomic clocks + GPS
  Accuracy: ~7ms uncertainty
  Spanner uses this for globally consistent timestamps
```

### Logical Clocks

**Lamport Timestamps:**
```
Simple counter, incremented on each event.

Node A: [1] ───→ send msg ───→ [2]
Node B:          receive msg → [max(local, received) + 1 = 3] → [4]

Rule: If A happened before B, then timestamp(A) < timestamp(B)
Limitation: timestamp(A) < timestamp(B) does NOT mean A happened before B
            (could be concurrent events)
```

**Vector Clocks:**
```
Each node maintains a vector of counters (one per node).

Node A: [A:1, B:0]
Node A sends to B: [A:1, B:0]
Node B receives: merge → [A:1, B:1]
Node B local event: [A:1, B:2]

Comparison:
[A:2, B:1] vs [A:1, B:2]
Neither dominates → events are CONCURRENT (conflict!)

[A:2, B:3] vs [A:1, B:2]
First dominates → first happened AFTER second

Used by: DynamoDB, Riak for conflict detection
```

---

## 6. Split Brain Problem

```
Problem:
Network partition splits cluster into two halves.
Both halves elect their own leader → TWO leaders!
Both accept writes → data diverges → INCONSISTENCY

┌─────────────┐    PARTITION    ┌─────────────┐
│ Node A (L1) │ ──── ✗ ────── │ Node C (L2) │
│ Node B      │                │ Node D      │
└─────────────┘                └─────────────┘

Solutions:
1. Quorum: Only the partition with majority (>N/2) can elect leader
   4 nodes: Left has 2 (no majority), Right has 2 (no majority) → neither leads
   5 nodes: One side has 3 (majority → leads), other has 2 (gives up)

2. Fencing token: New leader gets a monotonically increasing token
   Old leader's requests with lower token are rejected by storage

3. STONITH (Shoot The Other Node In The Head):
   The winning partition physically powers off losing nodes
```

---

## 7. Real-World Consistency Decisions

| System | Consistency Choice | Why |
|--------|-------------------|-----|
| Bank transfers | Strong (linearizable) | Money can't be created/destroyed |
| Social media likes | Eventual | Temporary wrong count is acceptable |
| Shopping cart | Eventual + merge | Must always be available; merge on sync |
| Inventory | Strong for decrement | Can't sell item you don't have |
| DNS | Eventual (TTL-based) | Availability > freshness |
| Chat messages | Causal | Reply must appear after original |
| Leader election | Consensus (Raft/Paxos) | Must agree on exactly one leader |

---

## Active Recall Questions

1. Explain the difference between linearizability and eventual consistency with examples.
2. Walk through Raft leader election. What happens when the leader crashes?
3. What is the split brain problem? How does quorum prevent it?
4. Explain 2PC. What is its main weakness? Why do microservices prefer Saga?
5. What are vector clocks? How do they detect concurrent writes?
6. Why can't we use physical clocks to order events in distributed systems?
7. Explain the Saga pattern with a concrete e-commerce example (order → pay → ship).
8. What is causal consistency? Give an example where it matters.
9. A system has 5 replicas. Write quorum W=3, Read quorum R=3. Why does this guarantee consistency?
10. You're designing a global chat app. Which consistency model? How would you handle message ordering?
