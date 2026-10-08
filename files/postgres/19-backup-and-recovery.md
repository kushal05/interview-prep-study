# Backup & Recovery

> TL;DR: Two flavors: **logical** (`pg_dump`/`pg_restore`, SQL text or custom format) and **physical** (`pg_basebackup` + WAL archive for PITR). Logical is portable, restores schema-by-schema, can't do point-in-time. Physical is the production answer for big clusters. A backup you haven't restored doesn't exist.

## Logical backups — pg_dump

Per-database dumps:

```bash
# Plain SQL (human-readable)
pg_dump -h host -U app -d shop > shop.sql

# Custom format (compressed, parallel-restorable)
pg_dump -h host -U app -d shop -F c -f shop.dump

# Directory format - allows parallel dump & restore
pg_dump -h host -U app -d shop -F d -j 8 -f shop_dir/
```

Selective:

```bash
pg_dump -d shop -t 'public.orders'       # one table
pg_dump -d shop -n 'billing'             # one schema
pg_dump -d shop --data-only -t orders    # data only
pg_dump -d shop --schema-only            # schema only
```

Cluster-wide:

```bash
pg_dumpall > all.sql                 # all DBs + roles
pg_dumpall --globals-only > globals.sql   # just roles / tablespaces
```

### pg_restore

```bash
pg_restore -d shop -j 8 shop.dump          # parallel restore
pg_restore -d shop --clean shop.dump       # drop existing objects first
pg_restore -d shop --section=pre-data shop.dump   # types, tables, functions
pg_restore -d shop --section=data shop.dump
pg_restore -d shop --section=post-data shop.dump  # constraints, indexes, triggers
```

Restore order matters because constraints reference tables. The `--section` split allows: pre-data -> data -> post-data, and lets you skip indexes for a fast load then create later.

### Consistency

`pg_dump` runs at a single snapshot (REPEATABLE READ) so the dump is internally consistent. With parallel directory dump it uses synchronized snapshots (PG 9.2+).

### When logical isn't enough

- 1 TB+ databases — restore is slow (re-creates indexes from scratch).
- Cannot do point-in-time recovery.
- Roles / tablespaces / configs not in `pg_dump` (use `pg_dumpall --globals-only`).

## Physical backups — pg_basebackup

Copies the **entire data directory** while the server is running, plus enough WAL to bring it consistent.

```bash
pg_basebackup -h primary -D /backup/base \
              -U replicator -W -P -R \
              --wal-method=stream \
              --format=tar --gzip
```

Restore: extract to `$PGDATA`, set up `postgresql.auto.conf` for primary_conninfo if becoming a replica, or just `recovery.signal`+`restore_command` for PITR.

### WAL archiving

For PITR, archive every WAL segment somewhere durable.

```ini
wal_level = replica
archive_mode = on
archive_command = 'aws s3 cp %p s3://bucket/wal/%f'    -- or rsync, etc.
archive_timeout = '60s'    -- force segment switch periodically
```

Or use **pgBackRest** / **WAL-G** / **Barman** which handle this for you.

### Point-In-Time Recovery (PITR)

Goal: restore to a moment between backups.

1. Restore base backup to fresh `$PGDATA`.
2. Create `recovery.signal` file (PG 12+; pre-12 used `recovery.conf`).
3. In `postgresql.conf` set:
   ```
   restore_command = 'aws s3 cp s3://bucket/wal/%f %p'
   recovery_target_time = '2024-04-01 03:15:00+00'
   recovery_target_action = 'promote'
   ```
4. Start PG — it replays WAL up to that target and promotes.

Targets: `recovery_target_time`, `recovery_target_xid`, `recovery_target_lsn`, `recovery_target_name` (created with `pg_create_restore_point`).

### Backup tooling

| Tool | Strengths |
|------|-----------|
| **pgBackRest** | Most production-ready: parallel, incremental, retention, encryption |
| **WAL-G** | Cloud-native (S3/GCS), incremental, supports MySQL/MongoDB too |
| **Barman** | Mature, Italian-flavored CLI, scheduling friendly |
| **pg_basebackup + scripts** | Built-in, simple, no incremental |

In modern production: use one of the above, do **not** roll your own.

## Backup testing

Restore drills are mandatory:

- Routinely restore to a sandbox; verify row counts and a sample of business invariants.
- Time the restore — RTO targets are paper tigers without rehearsal.
- Check WAL archive completeness: `select * from pg_stat_archiver;` (failures > 0?).

## Replication vs backup

Replicas are **not** backups. They faithfully replay everything, including `DROP TABLE` and `DELETE FROM payments`. Always keep an offline backup chain with retention.

## Logical replication for migrations

If you must migrate to new hardware / major version with minimal downtime, see [replication](15-replication.md) — logical sub/pub lets you sync everything online and cut over.

## pg_dump format quick chart

| Format | Flag | Restore tool | Parallel | Notes |
|--------|------|--------------|----------|-------|
| plain SQL | (default) | psql | no | Editable, large |
| custom | `-F c` | pg_restore | restore-only | Compressed |
| directory | `-F d` | pg_restore | dump + restore | Many small files |
| tar | `-F t` | pg_restore | no | Rarely used |

## Encryption

PG doesn't encrypt at rest natively. Options:
- File-system / disk-level (LUKS, EBS encryption).
- Backup-side encryption in pgBackRest/WAL-G.
- For specific columns: `pgcrypto` extension (`pgp_sym_encrypt`/decrypt).

## Sample DR runbook (RPO < 5 min, RTO < 30 min)

1. Continuous WAL archive to S3 (`archive_command`).
2. Daily `pg_basebackup` to S3.
3. Hot standby via streaming replication (preferred failover target).
4. Synthetic restore drill weekly.
5. Failover: promote replica, repoint app, replace lost primary as new replica.

## Interview Questions

**Q1. Logical vs physical backup — when to use each?**
Logical (`pg_dump`) is portable, schema-aware, good for migrations, refresh of dev/staging. Physical (`pg_basebackup` + WAL) is the production answer for large clusters and the only way to do PITR.

**Q2. How does Point-in-Time Recovery work?**
Restore a base backup, then replay archived WAL until a target time/XID/LSN. Set `restore_command` + `recovery_target_time` in `postgresql.conf` with `recovery.signal`.

**Q3. Are replicas a backup?**
No. Destructive commands replicate immediately. Keep independent backups with retention.

**Q4. Why is `pg_restore` of a 500 GB dump slow?**
Re-runs all DDL, loads data, then rebuilds every index from scratch. Use parallel restore (`-j N`) and consider physical backups + WAL replay for big clusters.

**Q5. What does `archive_command` do?**
Runs a shell command for each WAL segment after PG closes it (typically copies to durable storage). Must return 0 on success; failures back PG up — disk fills with un-archived WAL.

**Q6. Tools you'd reach for in production?**
**pgBackRest** or **WAL-G** for backups + WAL archiving. **Patroni** (or cloud-managed) for HA. **Debezium** for CDC. Don't roll your own.

## Common Pitfalls

- "We have a replica — we're safe" — until someone `DROP TABLE`s on the primary.
- Never restore-testing — first attempt happens during a real outage.
- `archive_command` failing silently (no monitoring on `pg_stat_archiver`).
- Forgetting `pg_dumpall --globals-only` — restored DB has no roles.
- Running `pg_dump` on a busy primary without `-j` synchronization — multiple unrelated snapshots.
- Storing backups in the same blast radius as the primary (same region, same account, same disk).

## See also

- [Replication](15-replication.md)
- [Vacuum & bloat](17-vacuum-and-bloat.md)
- [Performance tuning](18-performance-tuning.md)
- [System design: backup & DR](../system-design/high-level-design/03-databases.md)
