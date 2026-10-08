# Intro & Setup

> TL;DR: PostgreSQL is a process-per-connection RDBMS. A single **postmaster** parent forks **backend** processes for clients and helper workers (writer, WAL writer, autovacuum, checkpointer). Data lives in pages cached in **shared_buffers**.

## Architecture cheat sheet

```
              +------------------+
   psql ----> | postmaster (PID) |  listens on 5432
              +---------+--------+
                        | fork()
   +--------------------+--------------------+
   |          |         |          |        |
backend   backend   bgwriter   autovac   walwriter
(client)  (client)  (flushes)  (gc)      (fsync WAL)
   \         |         /
    \        |        /
        shared_buffers (RAM)
              |
        OS page cache
              |
            disk (base/, pg_wal/)
```

Key processes:

- **postmaster** — listener + supervisor; never serves queries directly.
- **backend** — one OS process per client connection. Hence pooling matters.
- **bgwriter** — trickles dirty buffers to disk to smooth checkpoints.
- **checkpointer** — flushes everything at checkpoint, advances WAL.
- **walwriter** — fsyncs WAL records.
- **autovacuum launcher / workers** — reclaim dead tuples ([MVCC](10-mvcc.md), [vacuum](17-vacuum-and-bloat.md)).
- **logical/physical replication walsender, walreceiver** — see [replication](15-replication.md).

## Memory layout

| Area | Knob | Purpose |
|------|------|---------|
| Shared buffers | `shared_buffers` | Page cache (~25% of RAM rule of thumb) |
| WAL buffers | `wal_buffers` | In-flight WAL before flush |
| Work mem | `work_mem` | Per-sort/hash node; can multiply per query |
| Maintenance work mem | `maintenance_work_mem` | VACUUM, CREATE INDEX, REINDEX |
| Effective cache size | `effective_cache_size` | Hint to planner about OS cache |

`work_mem` is per **plan node**, not per query. A query with 3 sorts and 2 hash joins can consume `5 * work_mem` per backend.

## On-disk layout

```
$PGDATA/
  base/<dboid>/<relfilenode>   -- heap & index files, 1 GB segments
  global/                       -- shared catalogs (pg_database, pg_authid)
  pg_wal/                       -- WAL segments (16 MB)
  pg_stat/, pg_stat_tmp/        -- runtime stats
  pg_tblspc/                    -- tablespace symlinks
  postgresql.conf, pg_hba.conf
```

- **pg_hba.conf** — host-based auth rules; checked top to bottom.
- **postgresql.conf** — server settings; many require restart, many are `SIGHUP`-reloadable.

## Installing & connecting

```bash
# Ubuntu
sudo apt-get install postgresql-16
sudo -u postgres psql

# macOS via Homebrew
brew install postgresql@16
brew services start postgresql@16

# Docker
docker run --name pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -d postgres:16
```

Connect:

```bash
psql "postgres://app:secret@localhost:5432/shop"
# or
psql -h localhost -U app -d shop
```

## psql essentials

```sql
\l                -- list databases
\c shop           -- connect to db
\dn               -- list schemas
\dt               -- list tables in current schema
\dt app.*         -- list tables in schema "app"
\d users          -- describe table
\d+ users         -- describe with sizes & comments
\di               -- list indexes
\df               -- list functions
\du               -- list roles
\x                -- toggle expanded display
\timing on        -- show query times
\e                -- edit last query in $EDITOR
\watch 2          -- re-run last query every 2s
\copy users TO 'users.csv' CSV HEADER
```

## Databases vs schemas

- **Cluster** = one postmaster instance, lives at `$PGDATA`, listens on one port.
- **Database** = isolated namespace inside a cluster. Cross-DB queries need FDW or dblink — they are **not** transparent like MySQL.
- **Schema** = lightweight namespace inside a database. Use schemas for modules in a single app DB (`auth.users`, `billing.invoices`).

```sql
CREATE DATABASE shop;
\c shop
CREATE SCHEMA billing;
CREATE TABLE billing.invoice (id bigserial PRIMARY KEY, amount numeric(12,2));
SET search_path = billing, public;   -- resolve unqualified names
```

`public` schema is created by default but in PG 15+ only owners can create objects in it.

## Roles, users, privileges

Roles are unified — a `USER` is a `ROLE` with `LOGIN`.

```sql
CREATE ROLE app LOGIN PASSWORD 'secret';
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE shop TO readonly;
GRANT USAGE ON SCHEMA billing TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA billing TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA billing
  GRANT SELECT ON TABLES TO readonly;
GRANT readonly TO app;
```

## A minimal first query

```sql
CREATE TABLE users (
  id          bigserial PRIMARY KEY,
  email       citext UNIQUE NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

INSERT INTO users(email) VALUES ('a@x.com'), ('b@y.com');

SELECT id, email, created_at AT TIME ZONE 'UTC' AS created_utc
FROM users
ORDER BY created_at DESC;
```

## Interview Questions

**Q1. Process model vs thread model — why does PG use one process per connection?**
Isolation and crash safety. A misbehaving backend cannot corrupt another. Cost: ~5-10 MB per connection + context switches, so production deployments front PG with **PgBouncer** (see [performance tuning](18-performance-tuning.md)).

**Q2. Difference between database and schema?**
A schema is a namespace inside one database. Cross-schema queries are trivial (`auth.users JOIN billing.invoices`). Cross-database queries require FDW/dblink and don't share transactions.

**Q3. What does `shared_buffers` cache, and how is it different from the OS page cache?**
`shared_buffers` is PG's own LRU cache of 8 KB pages (heap + index). The OS also caches the same files. Setting `shared_buffers` too high starves the OS cache; ~25% of RAM is the conventional sweet spot.

**Q4. What does pg_hba.conf do?**
Controls *which* clients can authenticate, *how* (md5, scram-sha-256, peer, trust), and to which DB/user. First matching row wins — order matters.

**Q5. How do you change a server setting?**
Some settings (`work_mem`, `search_path`) are SIGHUP-reloadable (`pg_reload_conf()`); others (`shared_buffers`, `max_connections`) need a restart. Check with `SHOW context FROM pg_settings`.

**Q6. What's the difference between `template0` and `template1`?**
New databases clone `template1` by default; admins can pre-load extensions there. `template0` is a pristine clone-source, used when you need a database that *isn't* contaminated by `template1` customizations (e.g. different encoding).

## Common Pitfalls

- **Forgetting to grant on future objects** — `GRANT SELECT ON ALL TABLES` only covers existing tables. Use `ALTER DEFAULT PRIVILEGES` for new ones.
- **`search_path` surprises** — if `public` is first and you forget to qualify, a malicious user-created function can shadow built-ins.
- **Connection overhead** — running 5000 lambdas each opening a direct connection will crush PG. Pool.
- **Confusing `timestamptz` with `timestamp`** — `timestamptz` stores UTC and converts on display; `timestamp` ignores TZ entirely. Use `timestamptz` 99% of the time.
- **Trying to query across databases** in one statement and expecting it to work like MySQL.

## See also

- [SQL basics & DDL](02-sql-basics-and-ddl.md)
- [Performance tuning](18-performance-tuning.md)
- [Spring datasource config](../spring-detailed/PART-08-databases-and-sql.md)
- [System design: relational vs NoSQL](../system-design/high-level-design/03-databases.md)
