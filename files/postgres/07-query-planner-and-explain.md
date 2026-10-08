# Query Planner & EXPLAIN

> TL;DR: PG is cost-based. `EXPLAIN` shows the plan; `EXPLAIN (ANALYZE, BUFFERS)` shows actual time + I/O. Mismatches between **estimated rows** and **actual rows** are the #1 indicator of a bad plan. Fix with `ANALYZE`, better stats target, multivariate stats, or query rewrite. PG has no real hint syntax — that's a feature.

## Plan tree basics

```
Sort  (cost=20.50..21.00 rows=200 width=12) (actual time=0.301..0.310 rows=187 loops=1)
  Sort Key: created_at DESC
  Sort Method: quicksort  Memory: 27kB
  ->  Index Scan using idx_orders_customer on orders  (cost=0.29..18.00 rows=200 width=12)
        Index Cond: (customer_id = 42)
        Buffers: shared hit=14
Planning Time: 0.121 ms
Execution Time: 0.345 ms
```

Read **bottom-up, inside-out**. Children feed their parents.

## EXPLAIN options

```sql
EXPLAIN SELECT ...;                                    -- estimates only
EXPLAIN ANALYZE SELECT ...;                            -- actually runs it!
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;                 -- + I/O
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON) ...;  -- machine-readable
EXPLAIN (ANALYZE, BUFFERS, SETTINGS) ...;              -- shows non-default GUCs (PG 12+)
```

`EXPLAIN ANALYZE` **executes** the query. For `UPDATE`/`DELETE`, wrap in a transaction you roll back:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS) DELETE FROM logs WHERE created_at < now() - interval '90 days';
ROLLBACK;
```

## What to look for

1. **rows estimate vs actual** — orders-of-magnitude mismatch = bad stats or query antipattern.
2. **Seq Scan on large table** with selective predicate -> missing index.
3. **Hash Join with `Disk: NNkB`** in `Sort Method` -> spill, raise `work_mem` for the session or rewrite.
4. **Nested Loop with huge outer rows** -> wrong join order, often because of wrong cardinality.
5. **Buffers: shared read=...** vs `hit=` -> cold cache; expect first run to be slower.
6. `loops=N` in nested loop inner: time-per-row = total_time / N.

## Plan nodes you'll see

| Node | Meaning |
|------|---------|
| Seq Scan | Full table scan |
| Index Scan | Index probe + heap fetch |
| Index Only Scan | Read only the index leaves (requires VM updated) |
| Bitmap Index Scan + Bitmap Heap Scan | Build TID bitmap, then ordered heap reads |
| Nested Loop / Hash Join / Merge Join | Join algos |
| Hash | Build side for hash join |
| Sort | Explicit sort node |
| Aggregate / HashAggregate / GroupAggregate | Aggregation modes |
| Memoize (PG 14+) | Caches repeated lookups in nested loops |
| Gather / Parallel Seq Scan | Parallel query workers |
| CTE Scan | Materialized CTE result |
| Append / Merge Append | UNION ALL / partitioned scan |

## Statistics — the planner's eyes

The planner consults `pg_stats`. Without good stats, choices are guesses.

```sql
ANALYZE orders;                       -- one table
ANALYZE;                              -- the whole DB
SELECT * FROM pg_stat_user_tables WHERE relname='orders';
SELECT attname, n_distinct, most_common_vals, correlation
FROM pg_stats WHERE tablename='orders';
```

Knobs:

- `default_statistics_target` — sample size per column (default 100). Raise for skewed columns (300, 1000).
- `ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;` — per column.
- `ALTER TABLE orders SET (autovacuum_analyze_scale_factor = 0.02);` — analyze more aggressively.

### Multivariate / extended statistics (PG 10+)

When two columns are correlated, the planner's independence assumption fails:

```sql
-- e.g. country implies language; (country, language) cardinality is far smaller
CREATE STATISTICS s_locale (dependencies, ndistinct)
  ON country, language FROM users;
ANALYZE users;
```

Three kinds: `dependencies`, `ndistinct`, `mcv`. Fix planning errors for correlated filters and group-bys.

## Choosing scan types

| Predicate selectivity | Scan |
|-----------------------|------|
| 0-1% of table | Index Scan / Bitmap |
| 1-30% | Bitmap Index Scan |
| 30%+ | Seq Scan (often cheaper) |

The cost model uses `random_page_cost` (default 4.0) vs `seq_page_cost` (1.0). On SSDs lower `random_page_cost` to ~1.1.

## Why no hints?

Postgres deliberately omits hint syntax to avoid stale plans rotting in queries. Instead:

- Improve statistics
- Add/refine indexes
- Rewrite the query (push predicates, materialize CTEs, split queries)
- Toggle planner GUCs **per session** for debugging only:
  - `SET enable_hashjoin = off;`
  - `SET enable_nestloop = off;`
  - `SET join_collapse_limit = 1;` (preserve written join order)
  - `SET from_collapse_limit = 1;`

The `pg_hint_plan` extension adds Oracle-style hints if you must.

## Common rewrites

### Push predicates into subqueries

```sql
-- Bad: join everything then filter
SELECT * FROM (SELECT * FROM big_table) t JOIN small s ON ... WHERE t.id=1;
-- Better: filter early
SELECT * FROM (SELECT * FROM big_table WHERE id=1) t JOIN small s ON ...;
```

(In modern PG the planner often does this automatically; not always.)

### Replace `OR` on different columns with `UNION ALL`

```sql
-- Hard to index together:
SELECT * FROM users WHERE email=$1 OR phone=$2;
-- Faster with two indexes:
SELECT * FROM users WHERE email=$1
UNION ALL
SELECT * FROM users WHERE phone=$2 AND email IS DISTINCT FROM $1;
```

### LIMIT 1 with ORDER BY index match

```sql
-- Add index (customer_id, placed_at DESC); LIMIT 1 reads one index entry.
SELECT * FROM orders WHERE customer_id=$1 ORDER BY placed_at DESC LIMIT 1;
```

## auto_explain — capture slow plans

```ini
# postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_format = json
```

Restart, then any query slower than 500ms has its plan logged.

## pg_stat_statements

Aggregated cost per normalized query — find the worst queries.

```sql
CREATE EXTENSION pg_stat_statements;
SELECT calls, total_exec_time, mean_exec_time, rows, query
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 20;
```

More in [performance tuning](18-performance-tuning.md).

## Why `count(*)` is "slow"

Because of [MVCC](10-mvcc.md), PG can't keep a single row count — different transactions might see different sets of rows. `COUNT(*)` triggers an index-only or seq scan + per-row visibility check.

Mitigations:
- `SELECT reltuples FROM pg_class WHERE relname='orders';` — fast estimate.
- Trigger-maintained counter table for hot counters.
- `EXPLAIN` to read the planner's estimate.

## Worked example

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.name, SUM(o.amount)
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.placed_at > now() - interval '7 days'
GROUP BY c.name;
```

Things to check:
- Is `orders(placed_at)` indexed? Maybe partial on recent rows.
- Hash Aggregate vs Group Aggregate — depends on `work_mem`.
- If rows estimate is far off, ANALYZE orders or create stats on (customer_id, placed_at).

## Interview Questions

**Q1. What's the difference between EXPLAIN and EXPLAIN ANALYZE?**
EXPLAIN shows the planned execution plan with cost estimates. EXPLAIN ANALYZE actually runs the query and shows real times, row counts, and (with BUFFERS) I/O.

**Q2. What does `Rows Removed by Filter` mean in EXPLAIN output?**
A node scanned more rows than it returned because its filter discarded some. High numbers suggest a missing index or wrong index choice.

**Q3. What's a bitmap index scan?**
Index returns a bitmap of TIDs; planner sorts and visits heap pages in physical order — sequential I/O instead of random. Better than index scan when many rows match.

**Q4. Why does `SELECT count(*) FROM big` take so long?**
MVCC requires checking visibility for each tuple. PG cannot maintain a single accurate counter cheaply. Use `pg_class.reltuples` for estimates.

**Q5. How would you debug a query that suddenly got slower after a deploy?**
Capture the new plan with EXPLAIN ANALYZE, compare to the previous plan (from `pg_stat_statements` or logs), check if statistics changed, autovacuum running, parameter sniffing for prepared statements, or new data distribution.

**Q6. What is `random_page_cost` and when do you tune it?**
Cost the planner uses for random I/O vs sequential. Default 4.0 (HDD-era). On SSDs lower it to ~1.1 to encourage index scans.

## Common Pitfalls

- Trusting EXPLAIN (without ANALYZE) on a slow query — estimates lie.
- Running `EXPLAIN ANALYZE` on an INSERT/UPDATE without a transaction (it actually writes).
- Ignoring large `Rows Removed by Filter` numbers.
- Not running `ANALYZE` after a bulk load.
- Assuming caching - first run hits cold cache; re-run to warm.
- Forgetting parallel queries can warp wall-clock vs CPU time.
- Using prepared statements where generic plans hurt (PG falls back to custom plan after 5 executions when generic seems worse).

## See also

- [Indexes](06-indexes.md)
- [Performance tuning](18-performance-tuning.md)
- [Joins (join algos)](03-joins.md)
- [MVCC](10-mvcc.md)
