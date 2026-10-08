# Locking

> TL;DR: PG has row-level locks, table-level locks, advisory locks, and predicate locks (SSI). Most of the time you write `SELECT ... FOR UPDATE` or `FOR NO KEY UPDATE` on a row. Deadlocks happen; PG detects them in ~1s and kills one transaction.

## Lock granularity

| Level | Acquired by | Visible in |
|-------|-------------|-----------|
| Row | UPDATE/DELETE/SELECT FOR UPDATE | `pg_locks` (with care) |
| Tuple | Buffer pin | Internal |
| Table | DDL, LOCK, implicit on DML | `pg_locks` |
| Page | Index ops | Internal |
| Advisory | `pg_advisory_lock` | `pg_locks` |

## Row-level locks (the four FOR clauses)

```sql
SELECT ... FROM t WHERE id=1 FOR UPDATE;          -- strongest
SELECT ... FROM t WHERE id=1 FOR NO KEY UPDATE;   -- weaker; allows concurrent KEY SHARE
SELECT ... FROM t WHERE id=1 FOR SHARE;           -- prevents updates
SELECT ... FROM t WHERE id=1 FOR KEY SHARE;       -- weakest; prevents key change only
```

### Compatibility matrix

| Requested \\ Held | KEY SHARE | SHARE | NO KEY UPDATE | UPDATE |
|--------------------|-----------|-------|----------------|--------|
| **KEY SHARE**      | OK | OK | OK | conflict |
| **SHARE**          | OK | OK | conflict | conflict |
| **NO KEY UPDATE**  | OK | conflict | conflict | conflict |
| **UPDATE**         | conflict | conflict | conflict | conflict |

### FOR UPDATE vs FOR NO KEY UPDATE — the famous interview question

- **UPDATE** changes any column. Acquires `FOR UPDATE`.
- **UPDATE** that changes a column referenced by a UNIQUE/PK or FK is a "key update".
- A plain UPDATE that doesn't change keys acquires the weaker **NO KEY UPDATE** under the hood.
- Why it matters: foreign-key checks on the **referencing side** acquire `KEY SHARE` on the parent row. If your parent UPDATE held `FOR UPDATE` (strong), inserts on the child would block; with `FOR NO KEY UPDATE` they don't.

Practical rule: use `FOR NO KEY UPDATE` when you're locking a parent row to compute or update a non-key column, and don't want child inserts to wait.

### NOWAIT and SKIP LOCKED

```sql
SELECT * FROM jobs
WHERE status='pending'
ORDER BY id LIMIT 1
FOR UPDATE SKIP LOCKED;     -- ignore already-locked rows
```

Common for queue workers — each consumer grabs unlocked rows and processes them.

```sql
SELECT * FROM t WHERE id=1 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "t"
```

## Table-level locks

Acquired implicitly by DML/DDL or explicitly with `LOCK TABLE`. Listed from weak to strong:

| Mode | Used by |
|------|---------|
| ACCESS SHARE | SELECT |
| ROW SHARE | SELECT FOR UPDATE/SHARE |
| ROW EXCLUSIVE | INSERT/UPDATE/DELETE |
| SHARE UPDATE EXCLUSIVE | VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY |
| SHARE | CREATE INDEX (not concurrently) |
| SHARE ROW EXCLUSIVE | special DDL |
| EXCLUSIVE | refresh materialized view |
| ACCESS EXCLUSIVE | DROP, TRUNCATE, REINDEX, most ALTER TABLE, VACUUM FULL |

ACCESS EXCLUSIVE blocks *everything*, including SELECTs. Avoid on hot tables; favor CONCURRENTLY variants.

```sql
LOCK TABLE orders IN SHARE MODE;   -- inside a transaction
```

## Advisory locks

User-defined locks not tied to objects — useful for app-level mutexes / cron job coordination.

```sql
SELECT pg_advisory_lock(42);             -- session-scoped, blocks
SELECT pg_try_advisory_lock(42);         -- non-blocking, returns bool
SELECT pg_advisory_unlock(42);
SELECT pg_advisory_xact_lock(42);        -- released on COMMIT/ROLLBACK
```

Common pattern: ensure single instance of a job runs.

```sql
DO $$
BEGIN
  IF NOT pg_try_advisory_lock(hashtext('nightly-report'))::int::boolean THEN
    RAISE NOTICE 'another worker is running';
    RETURN;
  END IF;
  -- ... do work ...
  PERFORM pg_advisory_unlock(hashtext('nightly-report'));
END$$;
```

## Inspecting locks

```sql
-- who is blocking whom
SELECT blocked.pid AS blocked_pid,
       blocked_activity.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking_activity.query AS blocking_query
FROM pg_locks blocked
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked.pid
JOIN pg_locks blocking ON blocking.locktype = blocked.locktype
   AND blocking.database IS NOT DISTINCT FROM blocked.database
   AND blocking.relation IS NOT DISTINCT FROM blocked.relation
   AND blocking.page IS NOT DISTINCT FROM blocked.page
   AND blocking.tuple IS NOT DISTINCT FROM blocked.tuple
   AND blocking.virtualxid IS NOT DISTINCT FROM blocked.virtualxid
   AND blocking.transactionid IS NOT DISTINCT FROM blocked.transactionid
   AND blocking.classid IS NOT DISTINCT FROM blocked.classid
   AND blocking.objid IS NOT DISTINCT FROM blocked.objid
   AND blocking.objsubid IS NOT DISTINCT FROM blocked.objsubid
   AND blocking.pid <> blocked.pid
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking.pid
WHERE NOT blocked.granted;
```

Shortcut in PG 9.6+:

```sql
SELECT pid, pg_blocking_pids(pid), query, state
FROM pg_stat_activity WHERE wait_event_type = 'Lock';
```

## Deadlocks

PG detects deadlocks after `deadlock_timeout` (default 1s) and kills one transaction:

```
ERROR: deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 555; blocked by process 1235.
        Process 1235 waits for ShareLock on transaction 556; blocked by process 1234.
```

SQLSTATE `40P01`. Retry. Avoid by:
- Acquire locks in **consistent order** across transactions.
- Use `FOR UPDATE` early to acquire all needed locks up front.
- Smaller transactions.

## Lock timeouts

```sql
SET lock_timeout = '2s';          -- abort if lock not acquired in 2s
SET statement_timeout = '30s';    -- abort statement after 30s
SET idle_in_transaction_session_timeout = '5min';
```

`lock_timeout` is gold for migration scripts — don't wait forever on a hot table.

## Migration safety patterns

- Use `CONCURRENTLY` for index ops.
- `ALTER TABLE ... ADD CONSTRAINT ... NOT VALID; VALIDATE CONSTRAINT;`
- Wrap risky DDL with low `lock_timeout` and a retry loop.
- For column type change, prefer add-new-column + backfill + swap, not `ALTER TYPE`.

## Common scenarios

### Money transfer

```sql
BEGIN;
SELECT * FROM accounts WHERE id IN (1,2) ORDER BY id FOR UPDATE;
UPDATE accounts SET balance=balance-100 WHERE id=1;
UPDATE accounts SET balance=balance+100 WHERE id=2;
COMMIT;
```

`ORDER BY id` ensures consistent lock order between transactions touching the same pair — avoids deadlocks.

### Job queue

```sql
SELECT id FROM jobs
WHERE status='pending'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- mark in progress, do work, COMMIT
```

`SKIP LOCKED` lets N workers consume without coordinating.

## Interview Questions

**Q1. `FOR UPDATE` vs `FOR NO KEY UPDATE`?**
Both lock the row against concurrent UPDATEs. `FOR UPDATE` also conflicts with the `KEY SHARE` that FK checks need on the parent row, so child inserts referencing this row will wait. `FOR NO KEY UPDATE` doesn't — preferred when you're not changing keys.

**Q2. How do you implement a queue table?**
`SELECT ... FOR UPDATE SKIP LOCKED LIMIT N` plus a status column. Multiple workers can dequeue concurrently without contention.

**Q3. How does PG detect deadlocks?**
After `deadlock_timeout` (default 1s) a waiting backend runs the deadlock detector that builds a wait-for graph; if a cycle is found, one transaction is aborted with SQLSTATE 40P01.

**Q4. What lock does TRUNCATE take?**
ACCESS EXCLUSIVE — blocks everything, including SELECTs.

**Q5. Difference between table lock and row lock visibility?**
Row locks are stored inside the tuple header (xmax) plus a multixact for shared lockers, not directly listed in pg_locks. Table locks are visible via pg_locks.

**Q6. Why use advisory locks?**
For application-level coordination not tied to a row/table — cron jobs, ensuring single execution of a task, distributed leader election in single-DB topologies.

## Common Pitfalls

- Inconsistent lock order across code paths -> deadlocks.
- Holding locks across user-think-time or external IO.
- `LOCK TABLE` ACCESS EXCLUSIVE on a busy table — every reader stalls.
- Forgetting `SET lock_timeout` in migrations — one long-running query halts deploys.
- Believing `FOR UPDATE` propagates through joins automatically — it locks rows of the **first** table by default; use `FOR UPDATE OF t1, t2`.
- Using `pg_advisory_lock` (session-scoped) when you meant `pg_advisory_xact_lock` (transaction-scoped); the former leaks if you forget to unlock.

## See also

- [Transactions](08-transactions-and-acid.md)
- [Isolation levels](09-isolation-levels.md)
- [MVCC](10-mvcc.md)
