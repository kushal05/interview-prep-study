# Databases in Production

> **TL;DR:** Back up, restore, replicate, fail over, pool connections, watch slow queries, deploy with expand-contract. DBs cause more outages than code does — operate them carefully.

## Backup — Then Verify the Restore

A backup you haven't restored from is a wish. Do quarterly restore drills.

### Logical vs Physical

- **Logical:** `pg_dump`, `mongodump`, `mysqldump`. Portable across versions, slower restore, schema visible. Default for cross-environment moves.
- **Physical:** filesystem snapshot or WAL/binlog (Postgres `pg_basebackup`, MySQL `xtrabackup`, EBS/CSI VolumeSnapshot). Fast restore, version-specific, can do point-in-time.

### Postgres example

```bash
# Logical
pg_dump -Fc -h prod-db -U app appdb > app-$(date +%F).dump
# Restore
pg_restore -d appdb -j 4 app-2026-05-20.dump

# Physical + WAL for PITR
pg_basebackup -h prod-db -U replicator -D /backup/base -Fp -Xs -P
# Continuous archive WAL with archive_command; replay on restore
```

### MongoDB

```bash
mongodump --uri "mongodb+srv://prod/app" --out /backup/$(date +%F)
mongorestore --uri "mongodb+srv://dev/app" /backup/2026-05-20/app
```

### Cloud-Managed

RDS / Aurora / Cloud SQL / Cosmos DB do automated backups + PITR. Configure retention; **also export to a separate account / project** to survive an account compromise.

### Cross-Region / Cross-Account

Cold copies in another region (compliance, full disaster). Tag retention separately. Test cross-region restore drills annually.

## Replication

### Synchronous vs Asynchronous

- **Sync:** primary waits for at least one replica to ack before committing. Zero data loss on failover. Higher write latency. Aurora's storage layer is effectively sync 4-of-6.
- **Async:** primary commits, then ships to replicas. Faster writes, replicas lag by ms-to-seconds. Data loss possible if primary dies mid-replication.

Most cloud-managed HA pairs are synchronous within a region, async across regions.

### Read Replicas

- Offload read load.
- Async replication — beware of **replication lag** (reads return stale data).
- Don't read your own writes from a replica; use the primary or use "read-your-writes" routing logic.
- 1-15 replicas typical for RDS / Aurora; consensus-based systems (Cassandra, Spanner, CockroachDB) work differently.

### Multi-Master

- Cassandra, Cosmos DB, DynamoDB Global Tables — last-writer-wins or CRDTs.
- Tradeoff: app must handle conflicts; not all data fits this model.
- Spanner / CockroachDB / TiDB give multi-region writes with strong consistency at the cost of write latency.

## Failover

### Manual vs Automatic

Automatic failover via the managed service (RDS Multi-AZ, Aurora, Cloud SQL HA, Azure SQL Failover Group) — typical 30-60s.

Critical app-side behavior:
- **Retry transient connection errors** — your app must handle "connection reset" without surfacing 5xx.
- **Short connection timeouts** — don't sit on a dead connection for 30s.
- **TCP keepalive** + tuned LB / pool idle timeouts.

Without these, failover that completes in 30s causes 5+ minutes of 5xx in your app.

### Promotion

When promoting a read replica to primary (cross-region DR):
1. Confirm last applied LSN (replication lag).
2. Stop writes to the old primary (or accept the small data loss window).
3. Promote replica.
4. Repoint app: change connection string, update Secrets Manager / Vault, restart connections.
5. New replicas off the new primary.

DR exercise quarterly. It's the only way to know your runbook works.

## Connection Pooling

DBs handle hundreds of concurrent connections at most — apps can spawn thousands. Without pooling, you exhaust the DB, sockets, or both.

### Patterns

- **App-side pool** — `pg-pool` (Node), HikariCP (Java), SQLAlchemy pool (Python). Easy, per-process.
- **External pooler** — PgBouncer / Odyssey (Postgres), ProxySQL (MySQL). Sits between app and DB, multiplexes many app conns onto few DB conns.

### Postgres + PgBouncer Modes

| Mode | Behavior |
|------|----------|
| **session** | Client keeps a server conn for the whole session. Like no pooling for server side. |
| **transaction** | Server conn released after each transaction. Best balance. Most apps work. |
| **statement** | Released after each statement. Doesn't allow transactions across statements. |

Transaction mode is the standard. Caveat: features that rely on session state (prepared statements, advisory locks, temp tables) need workarounds.

### Sizing

Don't oversize. Rule of thumb for Postgres: `pool_size ≈ ((core_count * 2) + effective_spindle_count)` for the DB; app-side total connections should fit under DB `max_connections`. With PgBouncer, app-pool can be small (each conn ≈ 1 short transaction at a time).

## Slow Queries

```sql
-- Postgres: top slow queries (pg_stat_statements)
SELECT query, calls,
       total_exec_time::int AS total_ms,
       (mean_exec_time)::int AS mean_ms,
       (max_exec_time)::int AS max_ms,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Explain
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ;

-- Active long-running queries
SELECT pid, age(clock_timestamp(), query_start), state, query
FROM pg_stat_activity
WHERE state != 'idle' AND query_start < now() - interval '5 seconds';

-- Kill
SELECT pg_cancel_backend(pid);     -- polite
SELECT pg_terminate_backend(pid);  -- forceful
```

Common fixes:
- Missing index on join / where column.
- Sequential scan on a big table — `EXPLAIN` shows it.
- N+1 query pattern in the app (use joins / dataloaders / batching).
- Function on indexed column kills the index (`WHERE LOWER(email) = ...` → either add functional index or normalize).
- `OFFSET` paginating millions of rows — use keyset pagination.

## Migrations & Deploys

### Expand-Contract — the Production Pattern

Backward-incompatible schema changes break old code while the new code rolls out. Solution: split into two deploys.

```
Goal: rename column user.fname -> user.first_name

Deploy 1 (expand):
  - ALTER TABLE user ADD COLUMN first_name text;
  - Backfill: UPDATE user SET first_name = fname WHERE first_name IS NULL;
  - Deploy code: writes to BOTH, reads from first_name (fall back to fname).

Deploy 2 (contract):
  - Code now writes only first_name, reads only first_name.
  - ALTER TABLE user DROP COLUMN fname;
```

Other expand-contracts:
- Add nullable column → backfill → make NOT NULL.
- Add new index `CONCURRENTLY` (Postgres) — no table lock.
- Rename → add new + dual-write + cut over reads + drop old.

### Tools

| Tool | Style |
|------|-------|
| **Flyway** | SQL files + state in DB |
| **Liquibase** | XML/YAML/SQL changesets |
| **node-pg-migrate / Knex** | Node ecosystem |
| **Alembic** | Python (SQLAlchemy) |
| **golang-migrate** | Go |
| **Atlas** | Schema-as-code, declarative |

Run migrations as a separate job/step in the pipeline — **not** inside the app container's startup (race conditions across N replicas).

## Locking & DDL Gotchas

### Postgres

- `ALTER TABLE` takes ACCESS EXCLUSIVE lock — blocks all reads + writes. Even seemingly cheap operations can rewrite tables.
- `CREATE INDEX CONCURRENTLY` avoids the lock — slower but safe in prod.
- `ALTER TABLE ... ADD COLUMN ... DEFAULT 'x' NOT NULL` — fine since Postgres 11 (no rewrite for default constant). Earlier: split into add nullable → backfill → set NOT NULL.
- `lock_timeout` setting prevents DDL from waiting forever (and queueing every other query behind it).

```sql
SET lock_timeout = '5s';
SET statement_timeout = '30s';
CREATE INDEX CONCURRENTLY ON orders (user_id);
```

### MySQL

- Online DDL since 5.6 reduces blocking, but `ALTER TABLE` patterns still need care.
- `pt-online-schema-change` or `gh-ost` for safe, copy-based migrations under load.

## Postgres Operational Knobs (quick list)

```
max_connections           # match app pool * replicas
shared_buffers            # 25% of RAM typical
effective_cache_size      # ~75% of RAM
work_mem                  # per-sort/hash; small (4-64MB) to avoid memory blowup
maintenance_work_mem      # VACUUM / CREATE INDEX scratch; 1-2GB OK
checkpoint_completion_target  0.9
wal_compression           on
autovacuum_*              # tune for high-update tables
log_min_duration_statement = 1000   # log queries > 1s
```

Plus enable: `pg_stat_statements`, `auto_explain`.

## Connection Strings That Don't Lie

```
postgres://user:pass@db.example.com:5432/appdb?\
  sslmode=verify-full&\
  application_name=api-prod&\
  connect_timeout=5&\
  statement_timeout=30000&\
  pool_max_conns=20
```

- `sslmode=require` minimum in prod; `verify-full` for proper cert validation.
- `application_name` shows up in `pg_stat_activity` — invaluable for diagnostics.
- Timeouts so a stuck connection doesn't hang forever.

## Cloud-Specific Notes

- **RDS:** Multi-AZ for HA, read replicas for scale, snapshot + PITR. Use `Aurora` for storage decoupling + faster failover + Serverless v2.
- **Cloud SQL (GCP):** HA = regional with sync replica. Read replicas + cross-region replicas for DR.
- **Azure SQL / Postgres Flexible Server:** Zone-redundant HA, geo-replication, auto-failover groups.
- **DynamoDB / Cosmos / Firestore:** managed scaling, but you pay for RCUs/RUs. Design data model around access pattern — no joins.

## Interview Questions

**Q: How do you do a zero-downtime schema migration with a breaking change?**
A: Expand-contract: deploy 1 adds the new shape and the code writes to both (and reads new with fallback). Backfill. Deploy 2 reads/writes only the new shape and drops the old. The DB schema never breaks the currently-deployed code.

**Q: Sync vs async replication tradeoffs?**
A: Sync = no data loss on primary failure, higher write latency, can't write if replica unreachable. Async = lower latency, possible data loss window, replication lag. Most prod uses sync within a region, async cross-region.

**Q: PgBouncer transaction vs session pooling?**
A: Session = client owns the server connection for the whole client session (defeats pooling for long-lived clients). Transaction = server connection released at COMMIT/ROLLBACK — massive multiplexing benefit. Caveat: session-state features (prepared statements, advisory locks, temp tables, LISTEN/NOTIFY) need handling. Most apps work fine in transaction mode.

**Q: A query that used to be fast is now slow. How do you diagnose?**
A: `EXPLAIN ANALYZE` to see the current plan. Check: is the index still being used? Has the table grown / stats become stale (run `ANALYZE`)? Is there a long-running transaction holding locks? `pg_stat_activity` for blocking queries. `pg_stat_statements` for the historical baseline.

**Q: How would you handle a connection pool exhaustion incident?**
A: Find the culprit query in `pg_stat_activity` (long-running). Kill it. Then root cause: missing index? N+1? Leak (connections checked out but never released)? Pool sized wrong vs DB `max_connections`. Add pooler (PgBouncer) so app-pool can grow without DB cost.

**Q: How do you back up and recover Postgres for disaster recovery?**
A: Logical (`pg_dump`) + physical (`pg_basebackup` + continuous WAL archive) + cross-region replica. Restore drill quarterly. RTO = how fast can we restore (minutes via replica promotion, hours via dump restore). RPO = how much data can we lose (0 with sync replica, minutes with async, hours with daily dump). Document RTO/RPO and verify against backups.

## Common Pitfalls

- No restore drills — backup is unverified.
- `ALTER TABLE ADD COLUMN NOT NULL` without `DEFAULT` (or with non-constant default on old PG versions) → full table rewrite under exclusive lock.
- App startup runs migrations across N replicas → race conditions, duplicate errors.
- No `statement_timeout` / `lock_timeout` → runaway query holds locks for hours.
- Reading from a replica for read-your-own-writes flows → stale data shown to user.
- DB connection pool larger than DB `max_connections` → mysterious "too many clients."
- `:latest` image for the DB → silent major version upgrade on next pull = corrupted data.

## Related

- [08-kubernetes-storage.md](08-kubernetes-storage.md)
- [15-cloud-aws-basics.md](15-cloud-aws-basics.md)
- [18-monitoring-and-observability.md](18-monitoring-and-observability.md)
- [20-incident-response.md](20-incident-response.md)
- [../postgres/](../postgres/)
- [../mongodb/](../mongodb/)
