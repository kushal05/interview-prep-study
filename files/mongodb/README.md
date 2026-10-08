# MongoDB Interview Prep

## How to use this folder

A MongoDB course that starts from zero and ends at the internals senior interviewers ask about. Every chapter follows the same shape:

1. **TL;DR**, plus the level, prerequisites and reading time
2. **What you'll learn**, **In plain English** (an analogy) and **Key terms**
3. The main content, moving from simple to complex. Sections marked **(Advanced)** can be skipped on a first pass.
4. Mermaid diagrams, worked examples and **Tip / Warning / Interview** callouts
5. **Try it yourself** exercises, **Common Pitfalls**, **Interview Questions** and **Check yourself**, with answers in collapsible blocks

Code targets MongoDB 7.0/8.0, Node driver 6.x and Mongoose 8.x unless noted.

### Interactive study app

Open [`study/mongodb-study.html`](study/mongodb-study.html) in any browser. It's a single offline file with:

- rendered diagrams
- full-text search
- progress tracking
- flashcards built from every Q&A
- a combined glossary
- dark mode

After editing any chapter, rebuild it:

```bash
node mongodb/study/build.mjs
```

## Recommended Study Order

0. Never used MongoDB? Start with setup and your first commands (00).
1. Start with the document model and CRUD to ground intuition (01, 02, 03).
2. Master indexes and aggregation — these dominate interviews (04, 05).
3. Internalize schema design tradeoffs (06, 07) — the most discussed design topic.
4. Operational concepts: transactions, replication, sharding (08, 09, 10).
5. Tooling and applied use: Mongoose, performance tuning, security (11, 12, 13).
6. Production concerns: backup/recovery and advanced features (14, 15).
7. Deep dives and wrap-up: storage engine internals, Atlas Search / Vector Search, then the capstone and interview bank (16, 17, 18).

**Short on time?** Skip the (Advanced) sections, do 00 → 07, then read the Interview Questions in 08–12 and the interview bank in 18.

## Topics

| # | Topic | Description |
|---|-------|-------------|
| 00 | [Getting Started](00-getting-started.md) | Install (Docker/Atlas), mongosh, connection strings, first commands, course map. |
| 01 | [Intro and Document Model](01-intro-and-document-model.md) | What is MongoDB, BSON vs JSON, documents, collections, databases. |
| 02 | [CRUD Operations](02-crud-operations.md) | insert/find/update/delete, update operators, upserts, bulkWrite. |
| 03 | [Query Operators](03-query-operators.md) | Comparison, logical, element, array, regex, projection. |
| 04 | [Indexes](04-indexes.md) | Index types, ESR rule, explain plans, index intersection. |
| 05 | [Aggregation Pipeline](05-aggregation-pipeline.md) | Pipeline stages, expressions, $expr, $lookup, optimization. |
| 06 | [Schema Design](06-schema-design.md) | Embedded/referenced patterns, attribute, bucket, computed, polymorphic, subset. |
| 07 | [Embedded vs Referenced](07-embedded-vs-referenced.md) | When to embed vs reference, denormalization tradeoffs, DBRefs. |
| 08 | [Transactions](08-transactions.md) | Multi-doc ACID, sessions, retryable writes, isolation, cost. |
| 09 | [Replication](09-replication.md) | Replica sets, elections, oplog, read prefs, write concerns. |
| 10 | [Sharding](10-sharding.md) | Shard key choice, chunks, balancer, mongos, hashed vs ranged. |
| 11 | [Mongoose ODM](11-mongoose-odm.md) | Schemas, models, validation, hooks, virtuals, populate, lean. |
| 12 | [Performance Tuning](12-performance-tuning.md) | explain(), profiler, query optimization, working set, pooling. |
| 13 | [Security](13-security.md) | Auth (SCRAM, x.509), RBAC, TLS, field-level encryption, auditing. |
| 14 | [Backup and Recovery](14-backup-and-recovery.md) | mongodump/restore, oplog, PITR, snapshots, Atlas backups. |
| 15 | [Change Streams and Time Series](15-change-streams-and-time-series.md) | Change streams, time series, capped collections, pub/sub patterns. |
| 16 | [Storage Engine Internals](16-storage-engine-internals.md) | WiredTiger cache, MVCC, journal, checkpoints, compression, tickets. |
| 17 | [Atlas Search and Vector Search](17-atlas-search-and-vector-search.md) | Lucene-based $search, analyzers, embeddings, $vectorSearch, RAG, hybrid search. |
| 18 | [Capstone and Interview Bank](18-capstone-and-interview-bank.md) | End-to-end e-commerce design, 40+ graded questions, one-page cheat sheet. |

## Related folders

- Node.js drivers and async patterns: [../nodejs/](../nodejs/)
- Spring Data and JPA comparison: [../spring-detailed/PART-08-databases-and-sql.md](../spring-detailed/PART-08-databases-and-sql.md)
- Postgres (relational counterpart): [../postgres/](../postgres/)
- System design context for DB choice: [../system-design/high-level-design/03-databases.md](../system-design/high-level-design/03-databases.md)
