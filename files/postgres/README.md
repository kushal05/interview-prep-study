# PostgreSQL Interview Prep

A focused, interview-oriented tour of PostgreSQL. Each file is self-contained, leans on working SQL, and calls out the gotchas interviewers love to probe.

## How to use this folder

- Skim `01-intro-and-setup.md` first to ground the architecture vocabulary (postmaster, backends, shared buffers).
- Work top-down for a structured pass; jump to a file when you need to refresh a specific topic.
- Every file ends with **Interview Questions** and **Common Pitfalls** — drill those before any onsite.
- Cross-links point to MongoDB (`../mongodb/`), Spring JPA (`../spring-detailed/`), Node.js (`../nodejs/`) and System Design (`../system-design/`) for comparative depth.

## Recommended study order

1. Fundamentals: 01 -> 02 -> 03 -> 04 -> 05
2. Performance & internals: 06 -> 07 -> 10 -> 17 -> 18
3. Concurrency: 08 -> 09 -> 11
4. PG-specific power features: 12 -> 13 -> 14
5. Operations: 15 -> 16 -> 19

## Topic index

| # | File | One-liner |
|---|------|-----------|
| 01 | [Intro & Setup](01-intro-and-setup.md) | Process architecture, psql, databases vs schemas |
| 02 | [SQL Basics & DDL](02-sql-basics-and-ddl.md) | CREATE TABLE, data types, constraints, ALTER |
| 03 | [Joins](03-joins.md) | INNER/OUTER/CROSS/LATERAL, semi/anti joins, join algorithms |
| 04 | [Subqueries & CTEs](04-subqueries-and-ctes.md) | Correlated subqueries, WITH, recursive CTEs |
| 05 | [Window Functions](05-window-functions.md) | OVER/PARTITION BY, ROW_NUMBER, LAG/LEAD, frames |
| 06 | [Indexes](06-indexes.md) | B-tree, GIN, GiST, BRIN, partial, expression, covering |
| 07 | [Query Planner & EXPLAIN](07-query-planner-and-explain.md) | EXPLAIN ANALYZE BUFFERS, statistics, scan choices |
| 08 | [Transactions & ACID](08-transactions-and-acid.md) | BEGIN/COMMIT, savepoints, ACID in Postgres |
| 09 | [Isolation Levels](09-isolation-levels.md) | Read committed, repeatable read, serializable (SSI) |
| 10 | [MVCC](10-mvcc.md) | Tuple visibility, xmin/xmax, HOT updates, bloat |
| 11 | [Locking](11-locking.md) | Row/table/advisory locks, FOR UPDATE flavours, deadlocks |
| 12 | [Stored Procedures & Triggers](12-stored-procedures-and-triggers.md) | PL/pgSQL, functions vs procedures, triggers |
| 13 | [JSONB & Arrays](13-jsonb-and-arrays.md) | jsonb operators, GIN indexes, when to skip Mongo |
| 14 | [Full-Text Search](14-full-text-search.md) | tsvector, tsquery, ranking, vs ElasticSearch |
| 15 | [Replication](15-replication.md) | Streaming/logical replication, WAL, failover |
| 16 | [Partitioning](16-partitioning.md) | Range/list/hash, pruning, partitioning vs sharding |
| 17 | [Vacuum & Bloat](17-vacuum-and-bloat.md) | Autovacuum, VACUUM FULL, freezing, wraparound |
| 18 | [Performance Tuning](18-performance-tuning.md) | pg_stat_statements, PgBouncer, memory knobs |
| 19 | [Backup & Recovery](19-backup-and-recovery.md) | pg_dump, pg_basebackup, PITR, logical vs physical |

## Related folders

- [MongoDB notes](../mongodb/) — compare document vs relational
- [Spring databases & SQL](../spring-detailed/PART-08-databases-and-sql.md) and [JDBC](../spring-detailed/PART-09-jdbc.md)
- [System design: databases](../system-design/high-level-design/03-databases.md)
- [Node.js](../nodejs/) — pg, knex, prisma patterns
