# Performance Tuning

> TL;DR: Most wins come from indexes + plan analysis + connection pooling + sane memory settings, not exotic tweaks. Measure first (`pg_stat_statements`, `auto_explain`), then change one thing at a time. PgBouncer is mandatory at any real concurrency.

## Workflow

1. **Measure** with `pg_stat_statements`, `auto_explain`, slow log.
2. **Reproduce** offending query, capture `EXPLAIN (ANALYZE, BUFFERS)`.
3. **Hypothesize** — missing index, bad join order, bloat, lock waits, plan instability.
4. **Fix one thing**.
5. **Re-measure**.

## pg_stat_statements

```sql
CREATE EXTENSION pg_stat_statements;

SELECT calls,
       round(total_exec_time::numeric, 1) AS total_ms,
       round(mean_exec_time::numeric, 2)  AS mean_ms,
       rows,
       query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Find by:
- `total_exec_time` — biggest total CPU/wall consumers.
- `mean_exec_time` — individually slow queries.
- `rows / calls` — pagination going wrong.
- `shared_blks_hit / shared_blks_read` — cache hit ratio.

Reset:

```sql
SELECT pg_stat_statements_reset();
```

## auto_explain

Logs the plan of any query slower than a threshold.

```ini
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze     = on
auto_explain.log_buffers     = on
auto_explain.log_timing      = on
auto_explain.log_format      = json
auto_explain.log_nested_statements = on
```

Beware: `log_analyze` adds per-row instrumentation overhead. Restrict to staging or use a higher threshold.

## Memory settings

| Setting | Typical | What it does |
|---------|---------|--------------|
| `shared_buffers` | 25% of RAM | PG's page cache |
| `effective_cache_size` | 50-75% of RAM | Planner hint (not actual allocation) |
| `work_mem` | 16-64 MB | Per-node sort/hash; multiplies per query node! |
| `maintenance_work_mem` | 512 MB-2 GB | VACUUM, CREATE INDEX |
| `wal_buffers` | -1 (auto) | Usually fine |
| `temp_buffers` | 8 MB | Temp tables |

`work_mem` is the most-misunderstood: a single complex query with 5 hash/sort nodes will use `5 * work_mem`. With 200 connections that's `200 * 5 * work_mem` worst case. Pool connections and keep `work_mem` modest, raise per-session for heavy reports.

```sql
SET LOCAL work_mem = '256MB';   -- in a session/txn
```

## Cost settings

```
seq_page_cost = 1.0
random_page_cost = 4.0      -- lower on SSD (1.1)
cpu_tuple_cost = 0.01
cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025
```

Tune `random_page_cost` down on SSD to encourage index scans.

## Parallel query

```
max_worker_processes = 16
max_parallel_workers = 8
max_parallel_workers_per_gather = 4
min_parallel_table_scan_size = 8MB
parallel_setup_cost = 1000
```

Parallel seq scan, parallel index scan, parallel hash join, parallel aggregate are available. Help on big scans; cost setup is real, so small queries shouldn't parallelize.

## Connection pooling — PgBouncer

Each PG connection costs ~5-10 MB and a backend process. Apps with thousands of clients must pool. **PgBouncer** is the standard.

```ini
[databases]
shop = host=10.0.0.10 dbname=shop

[pgbouncer]
pool_mode = transaction
listen_port = 6432
max_client_conn = 5000
default_pool_size = 50
reserve_pool_size = 5
```

Modes:

- **session** — client gets a backend for its session; no benefit if clients hold sessions.
- **transaction** — backend handed back to pool between transactions. Best general fit. **No session state across queries** (no `SET` outside transaction, no prepared statements unless `pgbouncer_prepared_statements` is on, no `LISTEN/NOTIFY`).
- **statement** — backend reused per statement. Rarely useful.

PgCat is a modern alternative supporting more features (sharding, prepared statements transparently).

## Driver-level tips

- Use prepared statements / parameterized queries (avoid string concat).
- Batch INSERTs via `COPY` (fastest), multi-row INSERT, or libpq batching.
- Avoid round-trip storms — `SELECT ... WHERE id = ANY($1::bigint[])` beats N selects.
- For Node/Spring, see [Node.js](../nodejs/) and [Spring JDBC](../spring-detailed/PART-09-jdbc.md).

## Bulk load with COPY

```sql
COPY orders FROM '/path/in.csv' WITH (FORMAT csv, HEADER);
-- or from psql client:
\copy orders FROM 'in.csv' CSV HEADER
```

Faster than INSERTs by 10-100x. Drop indexes during massive loads, recreate after.

```sql
ALTER TABLE big SET UNLOGGED;   -- skip WAL during load
-- ... COPY ...
ALTER TABLE big SET LOGGED;     -- requires a rewrite
```

Use UNLOGGED only for ephemeral data — UNLOGGED tables are truncated on crash.

## Common query rewrites

- Replace `OFFSET 100000 LIMIT 20` with keyset pagination: `WHERE id > $last_id ORDER BY id LIMIT 20`.
- Replace `count(*)` for pagination with estimate from `EXPLAIN` or a separate counter.
- Replace `NOT IN` with `NOT EXISTS`.
- Push predicates inside subqueries.
- Convert correlated subquery to JOIN if planner picks the wrong order.

## Locking / contention tuning

- `idle_in_transaction_session_timeout`
- `lock_timeout` in migrations
- `statement_timeout`
- Smaller transactions; index FK child columns to avoid FK validation locks.

## Disk & WAL

- Put `pg_wal/` on a fast device.
- `checkpoint_timeout = 15min`, `max_wal_size = 8GB` (typical) to smooth checkpoints.
- `wal_compression = on` saves WAL space on full-page writes (cheap CPU).
- `synchronous_commit = off` for non-critical data trades durability for throughput.

## Monitoring queries to keep on a dashboard

```sql
-- cache hit ratio (target > 0.99)
SELECT sum(blks_hit) * 1.0 / NULLIF(sum(blks_hit) + sum(blks_read), 0)
FROM pg_stat_database;

-- top tables by sequential scans
SELECT relname, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
ORDER BY seq_tup_read DESC LIMIT 10;

-- waits
SELECT wait_event_type, wait_event, COUNT(*)
FROM pg_stat_activity WHERE state = 'active'
GROUP BY 1,2 ORDER BY 3 DESC;
```

## Common bottlenecks & fixes

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| High `seq_tup_read` on a big table | Missing index | Add appropriate index |
| High cache miss ratio | `shared_buffers` too small / cold cache | Increase, or pre-warm with `pg_prewarm` |
| `Sort: external merge Disk: NNkB` | `work_mem` too low | Raise per-session |
| `idle in transaction` piling up | App leaks transactions | Pool + timeouts |
| Many parallel workers slow plans | Plan instability | Tune `parallel_*` costs, or `SET max_parallel_workers_per_gather = 0` for that query |
| Lock waits in `pg_stat_activity` | Long writes / DDL | Smaller txns, CONCURRENTLY |

## pg_prewarm

```sql
CREATE EXTENSION pg_prewarm;
SELECT pg_prewarm('orders', 'buffer');         -- load into shared_buffers
SELECT pg_prewarm('orders_pkey', 'prefetch');  -- OS-level prefetch
```

Useful after a failover or restart.

## Interview Questions

**Q1. How do you find slow queries in PG?**
Enable `pg_stat_statements`, sort by `total_exec_time` and `mean_exec_time`. For one-off slow events, `auto_explain` logs full plans.

**Q2. Why do I need PgBouncer?**
Each PG connection is a process (~5-10 MB). Thousands of app connections directly to PG starve CPU/memory. PgBouncer multiplexes thousands of clients onto tens of PG backends.

**Q3. PgBouncer transaction-mode caveats?**
No session state across transactions: `SET`, advisory locks, `LISTEN`, server-side cursors, prepared statements (unless the pooler version supports them).

**Q4. `work_mem` — how do you size it?**
Modest globally (16-64 MB). Override per-session for heavy reports. Remember it's per plan node, so a complex query consumes a multiple.

**Q5. How to bulk-load 100M rows quickly?**
Use COPY. Drop non-essential indexes, raise `maintenance_work_mem`, set table UNLOGGED if acceptable, run COPY in parallel chunks. Recreate indexes after.

**Q6. Cache hit ratio is 0.85 — good or bad?**
Below 0.99 is suspicious for OLTP. Likely under-sized `shared_buffers`, or workload genuinely larger than RAM (consider partitioning / archiving cold data).

## Common Pitfalls

- Tuning blindly without measurements.
- Raising `work_mem` globally and OOMing the box.
- Session-mode PgBouncer with thousands of long-lived app connections — no pooling benefit.
- Forgetting that prepared statements through `pg_stat_statements` are normalized — your query text in stats may differ from what you ran.
- Setting `synchronous_commit=off` for financial data.
- Adding indexes without measuring write impact.

## See also

- [Indexes](06-indexes.md)
- [Query planner](07-query-planner-and-explain.md)
- [Vacuum & bloat](17-vacuum-and-bloat.md)
- [Spring datasource pooling](../spring-detailed/PART-08-databases-and-sql.md)
- [Node.js pg patterns](../nodejs/)
