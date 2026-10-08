# Partitioning

> TL;DR: Declarative partitioning (PG 10+) splits one logical table into many physical tables by **range**, **list**, or **hash**. Queries with predicates on the partition key prune partitions. Great for time-series and archival. Not the same as sharding — partitioning is single-cluster; sharding crosses clusters (Citus, application sharding).

## Why partition

- Drop a month of data instantly with `DROP TABLE p_2023_01` instead of slow `DELETE`.
- Smaller working set per partition; indexes stay shallow.
- Maintenance (VACUUM, REINDEX) parallelized across partitions.
- Plan time benefits when constraint exclusion or partition pruning eliminates partitions.

## Strategies

### Range — by time (most common)

```sql
CREATE TABLE events (
  id bigserial,
  occurred_at timestamptz NOT NULL,
  user_id bigint,
  payload jsonb,
  PRIMARY KEY (id, occurred_at)              -- PK must include partition key
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2024_01 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

Bounds are `[from, to)` — half-open.

### List — by enum/discrete value

```sql
CREATE TABLE sales (
  id bigserial,
  region text NOT NULL,
  amount numeric,
  PRIMARY KEY (id, region)
) PARTITION BY LIST (region);

CREATE TABLE sales_us PARTITION OF sales FOR VALUES IN ('US');
CREATE TABLE sales_eu PARTITION OF sales FOR VALUES IN ('FR','DE','ES');
CREATE TABLE sales_other PARTITION OF sales DEFAULT;
```

### Hash — even distribution

```sql
CREATE TABLE accounts (
  id bigint NOT NULL,
  ...,
  PRIMARY KEY (id)
) PARTITION BY HASH (id);

CREATE TABLE accounts_p0 PARTITION OF accounts FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE accounts_p1 PARTITION OF accounts FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE accounts_p2 PARTITION OF accounts FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE accounts_p3 PARTITION OF accounts FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

Use when you have no natural range key but want to spread load.

## Sub-partitioning

```sql
CREATE TABLE events_2024_01 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')
  PARTITION BY HASH (user_id);

CREATE TABLE events_2024_01_p0 PARTITION OF events_2024_01
  FOR VALUES WITH (MODULUS 4, REMAINDER 0);
```

## Indexes on partitioned tables

```sql
CREATE INDEX ON events (user_id, occurred_at);
-- creates an "index template" + a real index on each partition (PG 11+).
```

Each partition has its own index file; queries route to the right one.

`UNIQUE` constraints **must include the partition key** (no global uniqueness across partitions without it). Same for primary keys.

## Partition pruning

The planner / executor skips partitions whose constraints exclude the query predicates.

```sql
SET enable_partition_pruning = on;   -- default on
EXPLAIN SELECT * FROM events WHERE occurred_at >= '2024-02-01' AND occurred_at < '2024-03-01';
-- Should show only events_2024_02 being scanned.
```

Works for constants, parameters, and many ranges. Fails for `WHERE date_trunc('month', occurred_at) = ...` — wrap your predicate to match the partition key directly.

## Attach / Detach (rolling windows)

Add a new partition every month, drop the oldest. To do this without locks:

```sql
-- Pre-create partition matching the bounds
CREATE TABLE events_2025_01 (LIKE events INCLUDING ALL);
ALTER TABLE events_2025_01 ADD CONSTRAINT chk
  CHECK (occurred_at >= '2025-01-01' AND occurred_at < '2025-02-01') NOT VALID;
ALTER TABLE events_2025_01 VALIDATE CONSTRAINT chk;

-- Cheap attach (PG 12+ uses the CHECK to skip a scan)
ALTER TABLE events ATTACH PARTITION events_2025_01
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- Detach old without long lock (PG 14+):
ALTER TABLE events DETACH PARTITION events_2022_01 CONCURRENTLY;
DROP TABLE events_2022_01;
```

## INSERT routing

Insert into the parent; PG routes to the correct partition automatically. Inserts that match no partition (and no DEFAULT) fail.

For huge bulk loads, you can write directly to the leaf partition.

## When NOT to partition

- Table is small (< 10s of GB). Overhead > benefit.
- Queries don't use the partition key — pruning fails and you get many seq scans.
- You need cross-partition unique constraints that can't include the partition key.
- You truly need horizontal scaling — partitioning doesn't add servers.

## Partitioning vs sharding

| | Partitioning | Sharding |
|---|--------------|----------|
| Granularity | Single cluster | Multiple clusters / nodes |
| Transparency | DB handles routing | App or middleware routes |
| Tools | Built-in | Citus, custom app logic |
| Cross-shard joins | Trivial | Hard |
| Failover | Standard PG | Per-shard plus orchestration |

Partition first, shard only when one cluster's writes can't scale.

## Materialized views & partitions

Partitioned MV not directly supported; emulate with a partitioned table populated by jobs.

## Maintenance script — monthly partition creator

```sql
DO $$
DECLARE
  d date := date_trunc('month', now())::date;
  start_d date;
  end_d date;
  pname text;
BEGIN
  FOR i IN 0..2 LOOP                         -- create next 3 months
    start_d := (d + (i || ' months')::interval)::date;
    end_d   := (start_d + interval '1 month')::date;
    pname := 'events_' || to_char(start_d, 'YYYY_MM');
    EXECUTE format(
      'CREATE TABLE IF NOT EXISTS %I PARTITION OF events FOR VALUES FROM (%L) TO (%L);',
      pname, start_d, end_d);
  END LOOP;
END$$;
```

Schedule with `pg_cron` or external scheduler.

## Interview Questions

**Q1. Range vs list vs hash partitioning?**
Range — partition key falls in `[a,b)` (time-series). List — discrete values (region, tenant id). Hash — modulus of key (even load distribution when there's no natural range).

**Q2. What is partition pruning?**
At plan time or runtime, PG eliminates partitions whose constraints can't match the query predicates. Requires queries to filter on the partition key (or an equivalent expression).

**Q3. Why must PRIMARY KEY include the partition key?**
A unique index lives within each partition. To guarantee global uniqueness, every row routed to the partition must be distinguishable using its own index — that requires the partition key in the key.

**Q4. How do you rotate a daily partition without downtime?**
Pre-create with `LIKE` and a matching `CHECK NOT VALID + VALIDATE`, then `ATTACH PARTITION`. Detach the old with `DETACH ... CONCURRENTLY` (PG 14+) and drop.

**Q5. Partitioning vs sharding — when do I switch?**
Partition when one server can hold the data but one big table is unwieldy. Shard when writes / data exceed a single primary; consider Citus or app-level sharding.

**Q6. What's the perf risk if queries don't include the partition key?**
The planner scans all partitions — N x the cost. With hundreds of partitions you pay heavy planning + executor overhead.

## Common Pitfalls

- Querying with `date_trunc(...)` instead of `occurred_at >= ... AND occurred_at < ...` — pruning fails.
- Forgetting to include partition key in PK / UNIQUE constraints.
- Hundreds of partitions slow planning (`partition_pruning` helps a lot, but raise `from_collapse_limit` & don't go nuts).
- Bulk loading via parent triggers extra routing cost; load into leaves.
- ATTACH PARTITION without a matching CHECK — PG scans the table (slow).
- Dropping the parent table without realizing it drops every partition.

## See also

- [Indexes](06-indexes.md)
- [Replication](15-replication.md)
- [System design: sharding](../system-design/high-level-design/03-databases.md)
