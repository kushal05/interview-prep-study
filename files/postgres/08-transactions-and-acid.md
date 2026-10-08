# Transactions & ACID

> TL;DR: A transaction is a unit of work that's Atomic, Consistent, Isolated, Durable. In PG, every statement runs in a transaction (implicit autocommit) unless you open one with `BEGIN`. Use **savepoints** for partial rollback within a transaction. Crash safety comes from WAL + fsync.

## ACID in Postgres

| Letter | What PG guarantees |
|--------|---------------------|
| **A**tomicity | All-or-nothing via WAL + rollback. A crashed transaction leaves no half-applied effects. |
| **C**onsistency | Constraints (PK, FK, CHECK, UNIQUE) hold at commit. Application enforces business rules. |
| **I**solation | Configurable; see [isolation levels](09-isolation-levels.md). Default = Read Committed. |
| **D**urability | Committed data survives crash. WAL fsync'd before commit ack (unless you disable `synchronous_commit`). |

## Transaction syntax

```sql
BEGIN;                    -- aliases: START TRANSACTION
  INSERT INTO accounts(id, balance) VALUES (1, 1000);
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                   -- or ROLLBACK;
```

Set characteristics at start:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ READ WRITE;
SET TRANSACTION READ ONLY;
SET TRANSACTION DEFERRABLE;     -- pairs with READ ONLY SERIALIZABLE
```

## Autocommit

`psql` and most drivers default to autocommit ON: each statement is its own transaction. Inside `BEGIN`/`COMMIT` everything joins one transaction.

JDBC/Spring: `connection.setAutoCommit(false)` to opt out — covered in [Spring JDBC](../spring-detailed/PART-09-jdbc.md).

## Aborted transactions

A failure inside a transaction puts it in **aborted state**. Subsequent statements error with:

```
ERROR: current transaction is aborted, commands ignored until end of transaction block
```

Either `ROLLBACK` or use a `SAVEPOINT`:

```sql
BEGIN;
  SAVEPOINT s1;
  INSERT INTO t VALUES ('bad...');   -- fails
  ROLLBACK TO SAVEPOINT s1;          -- back to known-good state
  INSERT INTO t VALUES ('good');
COMMIT;
```

`RELEASE SAVEPOINT s1` removes it without rolling back.

## Implicit and SQL-level transactions

- DDL is transactional in PG (a major edge over MySQL) — `CREATE TABLE`, `ALTER`, `DROP` can be inside a transaction and rolled back. Notable exceptions: `CREATE INDEX CONCURRENTLY`, `CREATE DATABASE`, `VACUUM`.

```sql
BEGIN;
ALTER TABLE orders ADD COLUMN new_col int;
-- changed your mind?
ROLLBACK;          -- column never existed for any other session
```

## Long-running transactions hurt

A transaction that stays open holds an old `xmin`, preventing VACUUM from reclaiming dead tuples behind it — across the entire database. See [MVCC](10-mvcc.md) and [vacuum](17-vacuum-and-bloat.md).

```sql
-- snapshots in flight
SELECT pid, state, xact_start, query
FROM pg_stat_activity
WHERE state='idle in transaction'
ORDER BY xact_start;
```

Kill long idle-in-tx with `idle_in_transaction_session_timeout` (e.g. `'5min'`).

## WAL, commit, and durability

1. Backend writes change records to the WAL buffer.
2. On `COMMIT`, walwriter flushes WAL up to the commit LSN with `fsync()`.
3. Once flush is acknowledged, COMMIT returns success.
4. Dirty data pages are written later (by bgwriter/checkpointer); on crash recovery, PG replays WAL.

Knobs:
- `synchronous_commit` = on/off/local/remote_write/remote_apply
  - off = "fire and forget"; up to ~600ms of committed-but-lost data on crash. Useful for low-stakes data; never for money.
- `fsync` = off — only ever for ephemeral test DBs; one crash = corrupted cluster.
- `full_page_writes` = on — protects against torn writes during checkpoints.
- `wal_level` = replica / logical — needed for replication.

## Transaction IDs (XIDs)

Each transaction gets a 32-bit XID. After ~2B transactions PG would wrap; **freezing** prevents wraparound — see [vacuum](17-vacuum-and-bloat.md). Background autovacuum freezes old rows; if you ignore it long enough, PG forces single-user mode at ~10M XIDs from wraparound.

64-bit XIDs are an ongoing project; until then, **monitor `pg_stat_activity` and `datfrozenxid`**.

## Transaction-scoped state

Some session state is transaction-scoped:

```sql
BEGIN;
SET LOCAL work_mem = '128MB';        -- reverts on COMMIT/ROLLBACK
CREATE TEMP TABLE staging ON COMMIT DROP (...);
```

`SET` without `LOCAL` is session-scoped (survives commit).

## Two-phase commit (2PC)

For distributed transactions:

```sql
BEGIN;
INSERT ...;
PREPARE TRANSACTION 'order-42';
-- later, by any session:
COMMIT PREPARED 'order-42';   -- or ROLLBACK PREPARED
```

Requires `max_prepared_transactions > 0`. Orphan prepared txns hold locks forever — clean up with a heartbeat / coordinator.

## Patterns

### Idempotent insert with `ON CONFLICT`

```sql
INSERT INTO inbox(message_id, payload)
VALUES ($1, $2)
ON CONFLICT (message_id) DO NOTHING;
-- safe to retry after network errors
```

### Optimistic concurrency

```sql
UPDATE orders SET status='shipped', version=version+1
WHERE id=$1 AND version=$2;
-- if RETURNING gives 0 rows, retry from re-read
```

### Compare-and-set inside a transaction

```sql
BEGIN;
SELECT balance FROM accounts WHERE id=1 FOR UPDATE;
-- ... business logic ...
UPDATE accounts SET balance = ... WHERE id=1;
COMMIT;
```

See [locking](11-locking.md) for `FOR UPDATE` flavors.

## Interview Questions

**Q1. What does ACID mean, and how does PG implement durability?**
Atomicity (WAL + rollback), Consistency (constraints), Isolation (MVCC + isolation levels), Durability (WAL fsync at commit). Durability rests on writing & fsync'ing the WAL record before COMMIT returns.

**Q2. What's the difference between `ROLLBACK` and `ROLLBACK TO SAVEPOINT`?**
Full ROLLBACK aborts the entire transaction; ROLLBACK TO SAVEPOINT undoes work since that savepoint and leaves the transaction open.

**Q3. Are DDL statements transactional in PG?**
Yes, mostly. `CREATE/ALTER/DROP TABLE` participate in transactions. Exceptions: `CREATE INDEX CONCURRENTLY`, `CREATE DATABASE`, `VACUUM`.

**Q4. What happens if a backend crashes mid-transaction?**
On restart, PG replays WAL up to the last checkpoint and discards uncommitted work — the half-applied transaction is rolled back as if it never started.

**Q5. Why is `idle in transaction` bad?**
It pins an old snapshot, blocking VACUUM from cleaning dead rows across the database, leading to bloat and possibly XID wraparound risk. Set `idle_in_transaction_session_timeout`.

**Q6. `synchronous_commit = off` — when is it acceptable?**
Workloads where losing a few hundred ms of latest commits on crash is acceptable (telemetry, analytics ingest). Never for financial data.

## Common Pitfalls

- Holding a transaction open across long external calls (HTTP, message brokers).
- Forgetting `ROLLBACK` after an error — connection stays in aborted state, every subsequent statement fails.
- Using `READ UNCOMMITTED` expecting dirty reads — PG silently maps it to READ COMMITTED.
- Bulk operations in one giant transaction — long-running, blocks vacuum, big rollback cost.
- 2PC orphans holding locks forever.
- Assuming truncate is rolled back — it is, but it reset sequences too unless you used `CONTINUE IDENTITY`.

## See also

- [Isolation levels](09-isolation-levels.md)
- [MVCC](10-mvcc.md)
- [Locking](11-locking.md)
- [Spring transaction management](../spring-detailed/PART-08-databases-and-sql.md)
