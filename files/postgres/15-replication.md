# Replication

> TL;DR: PG ships physical **streaming replication** (binary WAL shipping) and **logical replication** (row-level via publications/subscriptions). Streaming gives a byte-for-byte hot standby. Logical lets you replicate select tables, across major versions, or to non-PG sinks. WAL is the lifeblood of both.

## What the WAL is for

The Write-Ahead Log records every change. Uses:
- Crash recovery (replay since last checkpoint).
- Point-in-time recovery (see [backup](19-backup-and-recovery.md)).
- Streaming replication (primary -> replica).
- Logical replication / CDC (decode WAL into row events).

`pg_wal/` contains 16 MB segments. Retained until checkpoint + archive + replication slots release them.

## Physical streaming replication

```
Primary                         Replica (hot standby)
  walsender ----TCP-----> walreceiver -> startup process replays WAL
                                  -> shared_buffers updated
                                  -> read-only queries answered
```

Replica's data files are bit-for-bit identical (same XIDs, same page LSNs). Replicas can serve read-only queries while replaying.

### Basic setup

On the **primary** (`postgresql.conf`):

```
wal_level = replica
max_wal_senders = 10
wal_keep_size = '1GB'         -- or use replication slots
listen_addresses = '*'
```

`pg_hba.conf`:

```
host    replication     replicator     10.0.0.0/8    scram-sha-256
```

Create role:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD '...';
```

On the **replica**:

```bash
pg_basebackup -h primary-host -U replicator -D /var/lib/postgresql/16/data \
              -R -P -W --slot=repl1 --wal-method=stream
```

`-R` writes `standby.signal` and `primary_conninfo` automatically. Start the server — it streams.

### Synchronous vs asynchronous

```
synchronous_commit = on
synchronous_standby_names = 'FIRST 1 (replica_a, replica_b)'
```

| Mode | Behavior |
|------|----------|
| off | Primary doesn't wait for replica; possible data loss on failover |
| on (async) | Default; commit returns when local fsync done |
| remote_write | Wait until replica receives WAL (in OS cache) |
| on (sync) | Wait until replica fsync's WAL |
| remote_apply | Wait until replica has replayed (visible to queries) |

Synchronous costs latency; one stuck replica can halt commits unless quorum is set.

### Replication slots

A **physical replication slot** ensures the primary retains WAL the replica still needs — but if the replica is gone for a long time, slots can fill up `pg_wal`. Monitor:

```sql
SELECT slot_name, slot_type, active, restart_lsn,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM pg_replication_slots;
```

### Replication lag

```sql
-- on primary
SELECT client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)   AS sent_lag,
       pg_wal_lsn_diff(sent_lsn, write_lsn)              AS write_lag,
       pg_wal_lsn_diff(write_lsn, flush_lsn)             AS flush_lag,
       pg_wal_lsn_diff(flush_lsn, replay_lsn)            AS replay_lag
FROM pg_stat_replication;
```

### Failover

PG has no built-in autofailover. Common stacks:
- **Patroni** (etcd/Consul/ZK).
- **repmgr**.
- **PG cluster managers in clouds** (Aurora, AlloyDB, Crunchy).

After promoting a replica:
```sql
SELECT pg_promote();   -- PG 12+
```

Old replicas need `pg_rewind` (or rebuild) before reattaching.

### Read replicas — caveats

- **Replication lag** means stale reads. Read-after-write on a replica can be wrong.
- Long-running queries on replica can pause replay (or get cancelled). Tune `hot_standby_feedback`, `max_standby_streaming_delay`.
- Cannot write — only `SELECT` and temp tables.

## Logical replication

WAL is decoded into logical change records (rows) and shipped to subscribers. Each subscriber applies them via INSERT/UPDATE/DELETE.

### Setup

Primary:

```ini
wal_level = logical
```

```sql
CREATE PUBLICATION orders_pub FOR TABLE orders, line_items;
-- or FOR ALL TABLES
```

Subscriber:

```sql
CREATE SUBSCRIPTION orders_sub
CONNECTION 'host=primary dbname=shop user=repl password=...'
PUBLICATION orders_pub
WITH (copy_data = true);
```

Initial sync copies the data, then catches up via WAL.

### Uses

- Replicate a subset of tables (multi-tenant carve-out).
- Cross-major-version upgrades (replicate from PG 13 -> 16 with near-zero downtime).
- Logical CDC into Kafka via **Debezium** / `pgoutput`.
- Selective sharding (route writes per tenant to a different DB).

### Constraints / gotchas

- Tables need a **REPLICA IDENTITY**. PK is the default; without one, UPDATEs/DELETEs are rejected.
  - `ALTER TABLE t REPLICA IDENTITY FULL;` if no PK (logs whole row, expensive).
- DDL is **not** replicated — you must run it on both sides.
- Sequences are **not** replicated values; subscribers reset on first write.
- TRUNCATE replication is opt-in.
- Logical slots also retain WAL — a dead subscriber can fill the disk.

## CDC pipeline pattern

```
Postgres -> pgoutput (logical decoding) -> Debezium -> Kafka -> downstream
```

Combine with the **outbox pattern** in your application:
- App writes business row + outbox row in one transaction.
- Debezium streams outbox to Kafka. No dual-write problem.

## Monitoring checklist

- `pg_stat_replication` — physical lag.
- `pg_replication_slots.active` — slot is being consumed.
- WAL on disk: `du -sh $PGDATA/pg_wal/`.
- Long-running queries on standby: `pg_stat_activity`.
- `synchronous_standby_names` is mis-set so syncs hang.

## Logical vs physical at a glance

| | Physical | Logical |
|---|----------|---------|
| Cluster-wide | Whole cluster | Per-table |
| Same version required | Yes (same major) | No |
| DDL | Replicated implicitly | Not replicated |
| Hot standby | Yes | No (subscriber is a normal primary) |
| Failover candidate | Yes | Possible but DIY |
| Throughput | Highest | Lower (decoding overhead) |

## Interview Questions

**Q1. Physical vs logical replication?**
Physical = byte-level WAL streamed; whole cluster; same major version. Logical = decoded row events per publication/table; cross-version capable; subscriber is independent.

**Q2. What's `synchronous_commit = remote_apply` and when would you use it?**
Primary waits until replica has *applied* the WAL (visible to queries) before COMMIT returns. Use when read-after-write on the replica must see the write — at the cost of write latency.

**Q3. What's a replication slot and what risk does it pose?**
A persistent reservation on the primary that retains WAL the replica still needs. If the replica is gone, WAL accumulates and can fill the disk. Always monitor slot lag.

**Q4. How would you upgrade PG from 13 to 16 with minimal downtime?**
Logical replication: set up subscriber on PG 16, let it catch up, switch app over, decommission old primary. (Alternative: `pg_upgrade` with downtime.)

**Q5. What is REPLICA IDENTITY?**
The set of columns logical replication uses to identify a row for UPDATE/DELETE. Defaults to PK; can be set to FULL (entire row) or a UNIQUE index.

**Q6. Why might a hot standby cancel your read query?**
To replay WAL the standby may need to remove rows your snapshot still references. PG cancels long readers (`max_standby_streaming_delay`) to keep up. `hot_standby_feedback = on` makes the primary keep those rows around — at the cost of bloat on the primary.

## Common Pitfalls

- Orphaned replication slot eating disk.
- Logical replication missing DDL — schema drift between sides.
- Expecting transactional read-after-write on a replica.
- Synchronous replication with only one synchronous replica and no quorum — that replica becomes a SPOF for commits.
- `wal_keep_size` too small; replica disconnects and can never catch up.
- Forgetting to set REPLICA IDENTITY on a table without PK.

## See also

- [Backup & recovery](19-backup-and-recovery.md)
- [Partitioning](16-partitioning.md)
- [Vacuum & bloat (hot_standby_feedback tradeoffs)](17-vacuum-and-bloat.md)
- [System design: replication patterns](../system-design/high-level-design/03-databases.md)
