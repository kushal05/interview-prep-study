# High-Level Design (HLD) Study Guide

> **What is High-Level Design?**
> HLD is about designing **the architecture of a whole system** — how big building blocks (servers, databases, caches, queues, services) fit together. You decide what tech to use, how data flows, and how the system scales to millions of users.
>
> **HLD answers questions like:**
> - Should I use SQL or NoSQL?
> - How do I handle 1 million users?
> - Where do I put a cache?
> - How do servers stay in sync when one fails?
>
> **HLD is the opposite of Low-Level Design (LLD)**, which is about designing classes, methods, and interactions *inside* one service. (See `../low-level-design/`)

---

## How to Use This Guide

This folder follows a **study-mode** approach using four proven learning techniques:

1. **Spaced Repetition** — Review topics on days 1, 3, 7, 14, 30
2. **Interleaving** — Mix topics instead of binge-studying one
3. **Feynman Technique** — Explain each concept in simple words (kid-friendly)
4. **Active Recall** — Close the book, write what you remember

### Suggested Order (Beginners)

1. Read `00-90-day-blueprint.md` — your study plan
2. Read `01-fundamentals.md` — the vocabulary
3. Read `02-networking-and-protocols.md` — how data moves
4. Read `03-databases.md` — how data is stored
5. Continue through `04` to `09` for infrastructure topics
6. Practice with `10-real-world-system-designs.md` (actual interview problems)
7. Test yourself with `11-active-recall-cards.md`
8. Use `12-feynman-explanations.md` to write your own simple explanations
9. Consult `13-resources.md` for books, blogs, mock interviews

---

## File Map

| # | File | What You Learn |
|---|------|---------------|
| 00 | 90-day-blueprint | Day-by-day study plan |
| 01 | Fundamentals | Scalability, latency, CAP, availability — the vocabulary |
| 02 | Networking & Protocols | DNS, HTTP, TCP/UDP, WebSockets, REST |
| 03 | Databases | SQL vs NoSQL, indexing, sharding, replication |
| 04 | Caching | Redis, cache patterns, eviction, common pitfalls |
| 05 | Message Queues & Streaming | Kafka, RabbitMQ, event-driven design |
| 06 | Load Balancing & Proxies | How traffic is distributed |
| 07 | Storage & CDN | Object storage (S3), CDN, media pipelines |
| 08 | Microservices & Patterns | Monolith vs microservices, CQRS, Saga |
| 09 | Consistency & Consensus | CAP deep dive, Raft, distributed transactions |
| 10 | Real-World Designs | URL shortener, Twitter, Uber, etc. |
| 11 | Active Recall Cards | Self-test questions |
| 12 | Feynman Explanations | Simple-words explanations of concepts |
| 13 | Resources | Books, channels, papers, problem bank |

---

## Beginner's Master Glossary

These terms appear throughout. Refer back here when stuck.

| Term | What it means (plain English) |
|------|------------------------------|
| **Server** | A computer that responds to requests over the internet |
| **Client** | The computer asking the question (your phone, browser, etc.) |
| **Request** | A message from client to server (e.g., "send me my profile") |
| **Response** | The reply from the server |
| **Database (DB)** | Software that stores data on disk and lets you query it |
| **Cache** | A fast, temporary store for data you use a lot (usually in RAM) |
| **Node** | One single computer/server in a group |
| **Cluster** | A group of nodes working together |
| **Replica** | A copy of data stored on another node for safety/speed |
| **Replication** | The process of keeping replicas in sync |
| **Latency** | Time to get one answer (measured in milliseconds — `ms`) |
| **Throughput** | How many answers per second you can give |
| **QPS** | Queries Per Second — the request rate |
| **DAU** | Daily Active Users — people who used your app today |
| **API** | A defined set of endpoints the client can call (`GET /users/42`) |
| **Endpoint** | One specific URL the client can call |
| **HTTP** | The text-based protocol your browser uses |
| **HTTPS** | HTTP plus encryption |
| **TCP/UDP** | The two main "transport" protocols. TCP = reliable, UDP = fast |
| **DNS** | The "phone book" that turns `google.com` into an IP address |
| **CDN** | A network of servers worldwide that cache your static files near users |
| **Load Balancer (LB)** | A server that spreads incoming requests across multiple backend servers |
| **Proxy** | A middleman server that forwards requests |
| **Microservice** | A small, focused service (e.g., just "users" or just "payments") |
| **Monolith** | One big app that handles everything |
| **Queue** | A line where messages wait their turn to be processed |
| **Message Broker** | The software that holds and routes queue messages (Kafka, RabbitMQ) |
| **Async** | Don't wait — do the work later in the background |
| **Sync** | Wait until the work is done before continuing |
| **CAP** | The rule: a distributed DB can only fully guarantee 2 of {Consistent, Available, Partition-tolerant} |
| **ACID** | The 4 promises a traditional SQL DB makes about transactions |
| **Eventual consistency** | "It'll catch up eventually" — replicas may briefly disagree |
| **Sharding** | Splitting one big database into smaller pieces by some key |
| **Partition** | A piece of split data (a shard) |
| **Index** | A data structure that makes lookups fast (like a book's index) |
| **JSON** | A text format for sending structured data: `{"name": "Alice"}` |
| **SLA / SLO / SLI** | Service-Level Agreement / Objective / Indicator (uptime/quality promises) |

If a term appears in a file and isn't in this glossary, the file itself should explain it. If not, that's a bug — please add it.

---

## The Interview Framework (Use Every Time)

```
Step 1: REQUIREMENTS (5 min)
├── Functional: What does the system DO?
├── Non-functional: Scale, latency, availability, consistency?
├── Constraints: Users? QPS? Storage? Budget?
└── Out of scope: What are we NOT building?

Step 2: ESTIMATION (5 min)
├── Users → QPS (read/write ratio)
├── Storage (per item × items × retention)
├── Bandwidth (QPS × payload size)
└── Memory for caching (80/20 rule)

Step 3: HIGH-LEVEL DESIGN (10 min)
├── Core components (draw boxes + arrows)
├── Data flow (write path + read path)
├── API design (key endpoints)
└── Data model (key tables/collections)

Step 4: DEEP DIVE (15 min)
├── Pick 2-3 most interesting/challenging components
├── Discuss trade-offs for each decision
├── Address failure scenarios
└── Show depth of knowledge

Step 5: SCALING & WRAP-UP (10 min)
├── Bottlenecks and solutions
├── Monitoring and alerting
├── Future improvements
└── Summary of key trade-offs
```
