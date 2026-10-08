# Message Queues & Streaming

> **Learning goal:** Understand async communication, event-driven architecture, and when to use queues vs streams.

---

## Before You Begin — Plain-English Setup

- **Synchronous (sync)** = "wait for the answer before continuing." Like phoning someone — you both have to be on the line.
- **Asynchronous (async)** = "drop a note and keep going." Like leaving a voicemail.
- **A message** = a small chunk of data (JSON, text, etc.) sent from one program to another.
- **A queue** = a line. First-in, first-out. Messages wait their turn.
- **A producer** = the program that puts messages into the queue.
- **A consumer** = the program that pulls messages out of the queue to process them.
- **A broker** = the software that owns the queue and routes messages (e.g., Kafka, RabbitMQ).
- **A topic** = a named "channel" of messages. Anyone subscribed to the topic gets the messages.
- **Pub/Sub (Publish / Subscribe)** = one producer publishes to a topic, many consumers subscribe and each get a copy.
- **Event-driven** = the system reacts to "something happened" notifications instead of being polled.
- **Idempotent** = doing it twice has the same effect as doing it once. Essential for safe retries (full section in this file).
- **Stream** = an unbounded sequence of events that keeps coming (versus a queue, which can be emptied).

> Mental model: a queue is a drive-through. You hand over your order and drive away — somebody else makes the food.

---

## 1. Why Message Queues?

### The Problem with Synchronous Communication
```
Synchronous (tightly coupled):
User → API → Payment Service → Email Service → SMS Service → Response
                                                              (slow, fragile)
If Email Service is down → entire request fails

Asynchronous (decoupled):
User → API → Payment Service → Response (fast!)
              │
              └──→ [Message Queue]
                      ├──→ Email Service (processes independently)
                      └──→ SMS Service (processes independently)
```

### Benefits
- **Decoupling:** Producer doesn't need to know about consumers
- **Resilience:** If consumer is down, messages wait in queue
- **Buffering:** Absorbs traffic spikes (producer can be faster than consumer)
- **Scalability:** Add more consumers to process faster
- **Guaranteed delivery:** Messages aren't lost

---

## 2. Core Concepts

### Queue vs Topic (Pub/Sub)

```
QUEUE (Point-to-Point):
Each message is consumed by exactly ONE consumer

Producer → [Queue: M1, M2, M3] → Consumer A gets M1
                                → Consumer B gets M2
                                → Consumer C gets M3
Use: Task distribution, job processing

TOPIC (Publish/Subscribe):
Each message is delivered to ALL subscribers

Producer → [Topic: M1, M2, M3] → Subscriber A gets M1, M2, M3
                                → Subscriber B gets M1, M2, M3
                                → Subscriber C gets M1, M2, M3
Use: Event broadcasting, notifications
```

### Delivery Guarantees
```
At-most-once:
  Fire and forget. Message might be lost.
  ✓ Fastest, no overhead
  ✗ Messages can be lost
  Use: Metrics, logs (losing some is OK)

At-least-once:
  Retry until acknowledged. Message might be delivered multiple times.
  ✓ No message loss
  ✗ Duplicates possible → consumer must be IDEMPOTENT
  Use: Most systems (with idempotency)

Exactly-once:
  Each message processed exactly once.
  ✓ Ideal semantics
  ✗ Very hard/expensive to implement (often = at-least-once + deduplication)
  Use: Financial transactions
```

### Idempotency — Critical Concept
```
An operation is idempotent if doing it multiple times has the same effect as doing it once.

Idempotent:     SET balance = 100    (safe to retry)
NOT idempotent: ADD 100 to balance   (retrying doubles the addition!)

Strategy: Assign each message a unique ID. Consumer tracks processed IDs.
If ID already seen → skip (deduplicate).
```

---

## 3. Message Queue Systems

### RabbitMQ
```
Architecture:
Producer → Exchange → [Binding Rules] → Queue → Consumer

Exchange types:
- Direct:  Route by exact routing key match
- Topic:   Route by pattern match (e.g., "order.*")
- Fanout:  Route to ALL bound queues (broadcast)
- Headers: Route by message headers
```
- **Protocol:** AMQP (Advanced Message Queuing Protocol)
- **Model:** Smart broker, dumb consumer (broker manages delivery)
- **Strengths:** Rich routing, priority queues, message TTL, dead letter queues
- **Weakness:** Lower throughput than Kafka
- **Best for:** Complex routing, RPC-style messaging, task queues

### Apache Kafka
```
Architecture:
                    Kafka Cluster
Producer → [Topic: user-events]
            ├── Partition 0: [M1, M4, M7, ...]
            ├── Partition 1: [M2, M5, M8, ...]
            └── Partition 2: [M3, M6, M9, ...]
                                    ↑
                            Consumer Group A (each consumer reads from assigned partitions)
                            Consumer Group B (gets ALL messages independently)
```

**Key Concepts:**
```
Topic:           Named feed of messages (like a table)
Partition:       Ordered, immutable sequence of messages
Offset:          Position of a message within a partition
Consumer Group:  Set of consumers that divide partition processing
Broker:          Single Kafka server
Replication:     Each partition is replicated across brokers
```

**Why Kafka is Special:**
```
1. Append-only log: Messages are NOT deleted after consumption
   → Multiple consumers can read the same data
   → Replay from any offset (great for debugging/reprocessing)

2. High throughput: ~1M messages/sec per broker
   → Sequential disk I/O (faster than random memory access!)
   → Zero-copy transfer (kernel sends data directly to network)

3. Horizontal scaling: Add partitions → more parallelism

4. Retention: Keep messages for days/weeks/forever
```

- **Best for:** Event streaming, log aggregation, real-time pipelines, event sourcing

### Amazon SQS
```
Standard Queue:
- At-least-once delivery
- Best-effort ordering (no guaranteed order)
- Nearly unlimited throughput

FIFO Queue:
- Exactly-once processing
- Guaranteed order
- 300 messages/sec (3,000 with batching)
```
- **Best for:** Simple task queues on AWS, decoupling microservices

### Comparison
```
                RabbitMQ         Kafka              SQS
Throughput      ~50K msg/s       ~1M msg/s          ~unlimited*
Ordering        Per queue        Per partition       FIFO variant
Retention       Until consumed   Configurable        4-14 days
Replay          No               Yes (offset-based)  No
Routing         Rich             Topic/partition     Simple
Protocol        AMQP             Custom binary       HTTP
Managed?        Self-host/Cloud  Self-host/Cloud     Fully managed

*SQS throughput is virtually unlimited for standard queues
```

---

## 4. Event-Driven Architecture

### Event Types
```
1. Event Notification:
   "Something happened" (minimal data)
   Example: { "event": "order_placed", "order_id": "123" }
   Consumer must query for details.

2. Event-Carried State Transfer:
   "Something happened, and here's all the data"
   Example: { "event": "order_placed", "order": { "id": "123", "items": [...], "total": 99.99 } }
   Consumer has everything it needs.

3. Event Sourcing:
   Store every state change as an event. Current state = replay of all events.
   [Created] → [ItemAdded] → [ItemAdded] → [ItemRemoved] → [Placed]
   Replay all → Current state
```

### Event Sourcing + CQRS Pattern
```
Command Side (Write):                    Query Side (Read):
  Command → Validate → Store Event       Events → Projection → Read Model
                          │                                        │
                    Event Store                              Read Database
                    (append-only)                            (optimized for queries)

Example: E-commerce order
Events stored: OrderCreated, ItemAdded, PaymentProcessed, OrderShipped
Read model: Denormalized order view (pre-computed for fast queries)
```

---

## 5. Common Patterns

### Dead Letter Queue (DLQ)
```
Main Queue → Consumer → Processing fails
                 │
                 └──→ Retry 3 times → Still fails
                                          │
                                          ▼
                                   [Dead Letter Queue]
                                   (failed messages stored for investigation)
```

### Outbox Pattern (Reliable Event Publishing)
```
Problem: How to atomically update DB AND publish event?

DB write succeeds, event publish fails → inconsistency!

Solution:
1. Write to DB table + write event to "outbox" table in SAME transaction
2. Background process reads outbox table → publishes to queue → marks as sent

┌──── DB Transaction ────┐
│ UPDATE orders ...       │
│ INSERT INTO outbox ...  │
└─────────────────────────┘
        │
Background process → reads outbox → publishes to Kafka → deletes from outbox
```

### Saga Pattern (Distributed Transactions)
```
Problem: A business process spans multiple services. How to handle failures?

Choreography (event-driven):
Order Service → [OrderCreated] → Payment Service → [PaymentDone] → Inventory Service
                                 [PaymentFailed] → Order Service → compensate (cancel order)

Orchestration (central coordinator):
Saga Orchestrator → "Pay" → Payment Service → success/fail
                  → "Reserve" → Inventory Service → success/fail
                  → "Ship" → Shipping Service → success/fail
                  If any fails → execute compensation for all completed steps
```

---

## 6. Choosing the Right Tool

```
"I need to distribute tasks among workers"
→ RabbitMQ or SQS

"I need to broadcast events to multiple services"
→ Kafka (if replay needed) or RabbitMQ Fanout (if simple)

"I need to process a high-volume event stream"
→ Kafka

"I need exactly-once processing for financial transactions"
→ SQS FIFO + idempotency or Kafka with exactly-once semantics

"I need complex message routing"
→ RabbitMQ

"I need minimal infrastructure overhead"
→ SQS (fully managed)

"I need to replay events from the past"
→ Kafka (only option with built-in replay)
```

---

## Active Recall Questions

1. What are the 3 delivery guarantees? Which is most commonly used and why?
2. Explain the difference between a queue and a topic. Give a use case for each.
3. What makes Kafka different from a traditional message queue?
4. Explain Kafka partitions. How do consumer groups work?
5. What is idempotency? Why is it critical in message queue systems?
6. Describe the dead letter queue pattern. When would you use it?
7. What is the outbox pattern? What problem does it solve?
8. Explain the saga pattern. What's the difference between choreography and orchestration?
9. When would you choose RabbitMQ over Kafka? Vice versa?
10. What is event sourcing? How does it differ from traditional CRUD?
