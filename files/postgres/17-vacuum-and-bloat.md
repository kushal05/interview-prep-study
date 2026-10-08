# Vacuum & Bloat

> TL;DR: VACUUM exists because of [MVCC](10-mvcc.md). It reclaims dead tuples, updates the visibility map, freezes old XIDs to prevent wraparound, and updates planner stats. Autovacuum handles this automatically — until it doesn't, then you fight bloat with `VACUUM`, `VACUUM FULL`, `REINDEX`, or `pg_repack`.

## What VACUUM does

1. **Scans heap** for dead tuples (no live snapshot needs them).
2. **Removes index entries** pointing to those tuples.
3. **Reclaims line pointers** to reusable state.
4. **Updates visibility map (VM)** — needed for index-only scans and VACUUM skip.
5. **Updates free-space map (FSM)** so future inserts find space.
6. With `ANALYZE`, refreshes `pg_stats` for the planner.
7. **Freezes** tuples older than `vacuum_freeze_min_age` to avoid XID wraparound.

VACUUM does **not** return space to the OS — partially-empty pages remain in the file, available for new tuples.

## Variants

| Command | Lock | Returns space to OS | Use case |
|---------|------|---------------------|----------|
| `VACUUM t` | SHARE UPDATE EXCLUSIVE | No | Routine cleanup, online |
| `VACUUM ANALYZE t` | same | No | + refresh stats |
| `VACUUM FREEZE t` | same | No | Force freeze (e.g. before major upgrade) |
| `VACUUM FULL t` | ACCESS EXCLUSIVE | **Yes** (rewrites table) | Reclaim disk, exclusive lock |
| `pg_repack` extension | brief AE | Yes | Like VACUUM FULL but online |
| `REINDEX [CONCURRENTLY] i` | depends | Yes (rebuilds index) | Rebuild bloated indexes |
| `CLUSTER t USING i` | AE | Yes | Rewrite table in index order |

## Autovacuum

A background launcher schedules workers per database. Configuration lives in `postgresql.conf` plus per-table reloptions.

Key knobs:

| Setting | Default | Notes |
|---------|---------|-------|
| `autovacuum` | on | Don't disable globally |
| `autovacuum_naptime` | 1min | How often the launcher wakes |
| `autovacuum_max_workers` | 3 | Increase for many active DBs |
| `autovacuum_vacuum_threshold` | 50 | Min dead tuples to consider |
| `autovacuum_vacuum_scale_factor` | 0.2 | Fraction of table size that triggers |
| `autovacuum_analyze_scale_factor` | 0.1 | Same idea for ANALYZE |
| `autovacuum_vacuum_cost_limit` | 200 | I/O throttling budget |
| `autovacuum_vacuum_cost_delay` | 2ms | Pause per cost-limit batch |

Trigger condition for VACUUM:
```
dead_tuples > threshold + scale_factor * reltuples
```

Per-table overrides for hot tables:

```sql
ALTER TABLE big_table SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_analyze_scale_factor = 0.01,
  autovacuum_vacuum_cost_limit = 1000
);
```

Watch progress:

```sql
SELECT * FROM pg_stat_progress_vacuum;       -- PG 9.6+
```

## Detecting bloat

Approximate via stats:

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup, 0), 2) AS dead_ratio,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY dead_ratio DESC NULLS LAST;
```

Index bloat — `pgstattuple` extension gives true numbers:

```sql
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstattuple('orders');
SELECT * FROM pgstatindex('orders_pkey');
```

`pgstattuple` is heavy (scans). Approximate views via `pgstattuple_approx` or the popular `bloat_query` (find Tomas Vondra's variant) suffice for monitoring.

## VACUUM FULL caveats

- `ACCESS EXCLUSIVE` lock — full downtime for that table.
- Rewrites table + indexes -> 2x disk briefly.
- Wins back disk; planning to do this in production? Use `pg_repack` instead.

```sql
VACUUM FULL VERBOSE orders;
```

## pg_repack — online rebuild

Extension that rebuilds tables/indexes with minimal locking using triggers + swap. Bin must match server version. Standard ops choice.

```bash
pg_repack -d shop -t orders
pg_repack -d shop --no-superuser-check -i idx_orders_customer
```

## Freezing & wraparound — the doomsday issue

XIDs are 32 bits (~4B). Freezing rewrites tuple xmin to `FrozenXID` (visible to all). Autovacuum runs an "anti-wraparound" vacuum when `age(relfrozenxid) > autovacuum_freeze_max_age` (default 200M).

If you've been ignoring autovacuum, you'll see warnings:

```
WARNING: database "shop" must be vacuumed within 10485760 transactions
HINT: To avoid a database shutdown, execute a database-wide VACUUM in "shop".
```

At ~10M XIDs from wraparound, PG **forces single-user mode** until you VACUUM. Don't be that person.

Monitor:

```sql
SELECT datname, age(datfrozenxid) AS age, datfrozenxid
FROM pg_database
ORDER BY age DESC;

SELECT relname, age(relfrozenxid)
FROM pg_class WHERE relkind IN ('r','m')
ORDER BY age(relfrozenxid) DESC LIMIT 20;
```

## fillfactor & HOT

Setting `fillfactor` below 100 leaves space on each page for new tuple versions, enabling HOT updates (see [MVCC](10-mvcc.md)).

```sql
ALTER TABLE orders SET (fillfactor = 90);
-- new inserts/updates respect; existing pages will gradually conform on rewrites.
```

## Practical playbook

- Tune `autovacuum_*_scale_factor` low on hot tables (0.02, 0.01).
- Set `idle_in_transaction_session_timeout` to bound long open transactions.
- Watch `pg_stat_user_tables.n_dead_tup` and `last_autovacuum` in monitoring.
- For bloat that's already there: `pg_repack` table, `REINDEX CONCURRENTLY` indexes.
- Keep replication slot consumers alive — abandoned slots stall vacuum on the primary.
- `hot_standby_feedback = on` on replicas adds bloat on primary; weigh tradeoff.

## A common bloat post-mortem

Symptoms: query times creeping up, disk usage growing, plan choices odd.

1. Check `pg_stat_user_tables` for high dead-tuple ratio.
2. Check `pg_stat_activity` for `idle in transaction` and long XID ages.
3. Check `pg_replication_slots` for stuck slots.
4. Check `pg_stat_progress_vacuum` to see if AV is making progress.
5. Tune per-table AV settings; consider `pg_repack` for immediate cleanup.

## Interview Questions

**Q1. Why do we need VACUUM?**
MVCC creates dead tuple versions on every UPDATE/DELETE. VACUUM reclaims them, updates the visibility map, freezes old XIDs to avoid wraparound, and refreshes planner stats.

**Q2. Difference between VACUUM and VACUUM FULL?**
VACUUM is online, marks space reusable, doesn't return to OS. VACUUM FULL takes ACCESS EXCLUSIVE, rewrites the table, returns space to OS.

**Q3. What is transaction ID wraparound?**
XIDs are 32-bit. If a tuple's xmin isn't frozen before the counter wraps, it appears to be from the future and becomes "invisible." PG halts the cluster to prevent this. Autovacuum freezes old tuples to keep ahead of it.

**Q4. Symptoms of a bloated table?**
Slow seq scans on what should be small tables, growing disk usage despite stable row count, dead_tuples high in `pg_stat_user_tables`, index sizes far larger than expected.

**Q5. How does an idle-in-transaction session cause bloat?**
It pins the lowest active xmin, preventing vacuum from removing any tuple deleted/updated since that snapshot. Bloat accumulates across the entire database.

**Q6. How would you fix bloat without downtime?**
`pg_repack` for tables, `REINDEX CONCURRENTLY` for indexes, tighten autovacuum per-table thresholds, and address the root cause (long transactions, abandoned slots).

## Common Pitfalls

- Disabling autovacuum because "it's slow" — bloat and wraparound risk.
- Running `VACUUM FULL` during business hours.
- Forgetting that VACUUM doesn't release disk.
- Letting replication slots accumulate retained WAL.
- Massive UPDATEs followed by surprise: storage doubled. (Each UPDATE creates new versions; old space reclaimed by next VACUUM but not returned to OS.)
- Ignoring `last_autovacuum`/`last_autoanalyze` columns in monitoring.

## See also

- [MVCC](10-mvcc.md)
- [Performance tuning](18-performance-tuning.md)
- [Replication (hot_standby_feedback)](15-replication.md)
- [Backup & recovery](19-backup-and-recovery.md)
