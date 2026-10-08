# Isolation Levels

> TL;DR: PG supports three real isolation levels: **Read Committed** (default), **Repeatable Read**, **Serializable** (via SSI). PG accepts `READ UNCOMMITTED` syntax but treats it as Read Committed — there are no dirty reads in PG. Each level prevents progressively more anomalies, at the cost of more serialization failures you must retry.

## The four anomalies

| Anomaly | What goes wrong |
|---------|-----------------|
| **Dirty read** | See uncommitted data from another transaction |
| **Non-repeatable read** | Re-read a row, find it changed |
| **Phantom read** | Re-run a range query, see new rows |
| **Serialization anomaly** | Outcome differs from any serial execution (write skew) |

## What each level prevents

| Level | Dirty | Non-repeatable | Phantom | Serialization anomaly |
|-------|-------|----------------|---------|-----------------------|
| Read Uncommitted (PG: same as RC) | — | possible | possible | possible |
| **Read Committed** (default) | impossible | possible | possible | possible |
| **Repeatable Read** | impossible | impossible | impossible* | possible |
| **Serializable** (SSI) | impossible | impossible | impossible | impossible |

*PG's Repeatable Read uses **snapshot isolation**, which prevents phantoms by design — stronger than the SQL standard requires.

## Setting it

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- or
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

Session default:

```sql
SET default_transaction_isolation = 'repeatable read';
```

## Read Committed

Each **statement** sees a fresh snapshot of committed data at its start. Within one transaction, two `SELECT`s can return different rows.

```sql
BEGIN;
SELECT balance FROM accounts WHERE id=1;     -- 1000
-- another txn commits: UPDATE accounts SET balance=900 WHERE id=1;
SELECT balance FROM accounts WHERE id=1;     -- 900 (non-repeatable read!)
COMMIT;
```

Writes block on conflicting writes (row locks). UPDATE re-reads the latest version of the row (a special "read-committed mode" inside the executor).

## Repeatable Read (snapshot isolation)

The transaction sees a snapshot taken at the **first non-transaction-control statement** and uses it for every read.

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM orders;        -- snapshot taken
-- concurrent INSERTs are invisible no matter how many times you re-read
COMMIT;
```

If two RR transactions modify the same row, the second to commit gets:

```
ERROR: could not serialize access due to concurrent update
```

Application must catch and retry.

### Write skew under RR

RR does *not* prevent write skew. Two transactions reading disjoint rows then writing based on the assumption that the other didn't change them can both commit:

```
T1: SELECT SUM(amount) FROM withdrawals WHERE acct=1;   -- sees 0
T2: SELECT SUM(amount) FROM withdrawals WHERE acct=1;   -- sees 0
T1: INSERT INTO withdrawals (acct, amount) VALUES (1, 80);
T2: INSERT INTO withdrawals (acct, amount) VALUES (1, 80);
-- both commit; overdraft permitted
```

To fix: use SERIALIZABLE, or lock explicitly with `SELECT ... FOR UPDATE`/predicate locks via constraints.

## Serializable (SSI — Serializable Snapshot Isolation)

PG's serializable uses **SSI** (Cahill 2008). It tracks read/write dependencies between concurrent transactions and aborts one if a serialization conflict would occur.

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(amount) FROM withdrawals WHERE acct=1;
INSERT INTO withdrawals (acct, amount) VALUES (1, 80);
COMMIT;
-- if a concurrent transaction would make the schedule non-serial, one of them errors:
-- ERROR: could not serialize access due to read/write dependencies among transactions
```

Pros: prevents write skew & phantoms cleanly. Cons: serialization failures, retry logic mandatory. Most replicas use snapshot too — `READ ONLY DEFERRABLE` waits for a safe snapshot to avoid retries.

### When SSI helps and hurts

- Helps: low-conflict workloads where correctness > raw throughput.
- Hurts: hot-row contention — many retries, throughput collapses.

PG tracks "SI Read locks" in shared memory; high concurrency may overflow them, degrading to coarser locks.

## Retry boilerplate

```sql
-- In application code (pseudocode):
for attempt in 1..5:
  try:
    BEGIN ISOLATION LEVEL SERIALIZABLE;
    -- ... business logic ...
    COMMIT;
    break;
  except SerializationFailure:    -- SQLSTATE 40001
    ROLLBACK;
    sleep(jitter())
```

Handle both `40001` (serialization failure) and `40P01` (deadlock).

## Read-only optimizations

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
```

PG waits until it can take a snapshot that's guaranteed not to cause serialization failure, then runs the read-only transaction at full speed. Great for reporting on a primary.

## Practical defaults

- Web app: stay on **Read Committed**, use row locks (`FOR UPDATE`) on hot transitions.
- Money-touching critical sections: **Serializable** + retry loop.
- ETL / reports: **Repeatable Read** for consistent snapshot.

## How locks fit in

Row locks (`SELECT ... FOR UPDATE`) and isolation level are complementary. Even Read Committed can enforce critical-section semantics with explicit locking:

```sql
BEGIN;                                       -- read committed
SELECT * FROM accounts WHERE id=1 FOR UPDATE;
-- nobody else can update id=1 until we commit
UPDATE accounts SET balance = balance - 100 WHERE id=1;
COMMIT;
```

See [locking](11-locking.md).

## Inspecting

```sql
SHOW transaction_isolation;
-- current per session
SHOW default_transaction_isolation;
-- per-database default via ALTER DATABASE
```

## Interview Questions

**Q1. PG's isolation levels and the default?**
Read Committed (default), Repeatable Read, Serializable. Read Uncommitted is accepted syntactically but treated as Read Committed — PG has no dirty reads.

**Q2. What's the difference between Repeatable Read and Serializable in PG?**
RR uses snapshot isolation — prevents phantoms and non-repeatable reads but allows write skew. Serializable adds SSI: tracks read/write dependencies and aborts transactions that would create non-serializable schedules. Application must retry on serialization failure.

**Q3. Give a write-skew example RR doesn't prevent.**
Two doctors both check "is at least one doctor on call?" — both see yes — both go off duty. RR snapshots permit it; Serializable detects it.

**Q4. What error code indicates a serialization failure and how should the app react?**
SQLSTATE `40001`. Catch, ROLLBACK, retry with backoff. Don't surface as a 500 to users.

**Q5. Is `SELECT FOR UPDATE` enough to avoid race conditions at Read Committed?**
For row-level updates of an existing row, yes — it blocks writers and serializes the critical section. For decisions made over **ranges** of rows (e.g. "no overlapping booking"), use Serializable, or an EXCLUDE constraint, or explicit `LOCK TABLE` (heavy).

**Q6. Why might Serializable kill throughput?**
SSI tracks dependencies and aborts conflicting transactions; under contention, retries cascade. Mitigate by minimizing transaction length and keeping the critical section narrow.

## Common Pitfalls

- Assuming `READ UNCOMMITTED` actually gives dirty reads in PG (it doesn't).
- Using RR and being surprised by write skew.
- Using Serializable and not implementing retry logic.
- Re-reading a row under Read Committed and reasoning as if RR.
- Holding RR/Serializable transactions open across user think-time — concurrent commits will start failing.

## See also

- [Transactions & ACID](08-transactions-and-acid.md)
- [MVCC](10-mvcc.md)
- [Locking](11-locking.md)
