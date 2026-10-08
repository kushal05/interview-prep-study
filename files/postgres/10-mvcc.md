# MVCC — Multi-Version Concurrency Control

> TL;DR: PG never updates a row in place. UPDATE creates a new tuple version; DELETE marks the old one dead. Each tuple has `xmin` (creating txn) and `xmax` (deleting/updating txn); visibility is computed per snapshot. Garbage collection happens via VACUUM. Side effects: table bloat, mandatory autovacuum, and the famous "why is count(*) slow?"

## Why MVCC

Goal: readers don't block writers, writers don't block readers. Achieved by keeping multiple **versions** of each row. Each transaction sees the version that was committed at the time its snapshot was taken (consistent with its [isolation level](09-isolation-levels.md)).

## Tuple layout (the bits that matter)

Each heap tuple header contains:

- **xmin** — XID of the transaction that **inserted** this version. Visible only after that XID committed.
- **xmax** — XID of the transaction that **deleted/updated** this version. If 0 or aborted, still alive.
- **cmin/cmax** — command IDs within the transaction.
- **ctid** — physical pointer to the tuple (block, offset). Updates may point to a newer version.
- **t_infomask** — flags (xmin committed, xmax committed, hint bits).

```sql
SELECT ctid, xmin, xmax, * FROM orders LIMIT 5;
```

## Visibility rules (simplified)

Tuple T is visible to snapshot S if:
- `T.xmin` is committed and **<= snapshot's xmax** AND not in snapshot's in-flight list.
- `T.xmax` is 0, or aborted, or > snapshot's xmin in a way that means it's still live for us.

Hint bits cache committed/aborted decisions on the tuple to avoid re-checking CLOG (commit log).

## UPDATE behavior

UPDATE doesn't overwrite — it:
1. Marks old tuple's `xmax = current_xid`.
2. Inserts a new tuple with `xmin = current_xid`.
3. Updates all indexes to point to the new tuple (unless **HOT** — see below).
4. On COMMIT both versions exist; old one becomes garbage once no snapshot needs it.

```
Heap page:
  [tuple v1: xmin=100, xmax=200, ctid=(0,1)]   <- old, dead once tx 200 visible
  [tuple v2: xmin=200, xmax=0,   ctid=(0,2)]   <- new
```

## HOT updates (Heap-Only Tuple)

If no indexed column changed AND the new tuple fits in the same page, PG performs a **HOT update**:
- The new tuple chains off the old (`t_ctid` points from old to new in-page).
- Indexes still point to the old tuple — no index updates needed.
- VACUUM (or HOT pruning during access) can later prune the dead version.

Big throughput win for status-flag updates etc. Requires **fillfactor** < 100 so there's room in the page:

```sql
ALTER TABLE orders SET (fillfactor = 90);
```

`pg_stat_user_tables.n_tup_hot_upd` shows HOT update count.

## Snapshots

A snapshot captures:
- `xmin` — lowest active XID (everything older was committed or aborted).
- `xmax` — next XID to assign (everything >= is invisible).
- list of in-progress XIDs in the [xmin, xmax) range.

A transaction in Read Committed gets a fresh snapshot per statement; RR / Serializable take one snapshot for the whole txn.

```sql
SELECT * FROM pg_snapshot_xmin(txid_current_snapshot()), txid_current();
```

## Dead tuples and VACUUM

A tuple is "dead" when no snapshot needs it anymore — i.e. its `xmax` is committed and all snapshots have advanced past it. VACUUM:

1. Scans heap, finds dead tuples, marks line pointers reusable.
2. Updates indexes (removes pointers to dead tuples).
3. Frees space within pages for new inserts/updates.

VACUUM doesn't return space to the OS (use `VACUUM FULL` or `pg_repack` for that). It also updates the **visibility map** — required for index-only scans and for `VACUUM FREEZE` skipping.

Details in [vacuum & bloat](17-vacuum-and-bloat.md).

## XID freezing & wraparound

XIDs are 32 bits (~4B values). To avoid wrap, VACUUM eventually replaces old xmin with a special "FrozenXID" marker that's visible to all. Without freezing, after 2B transactions PG would think old rows are in the future and lose them. The system enforces this with progressively louder warnings, ultimately forcing single-user mode.

```sql
SELECT datname, age(datfrozenxid)
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
-- Compare to autovacuum_freeze_max_age (default 200M).
```

## The "long transaction = bloat" trap

A transaction that stays open for hours pins `xmin` at its start. VACUUM cannot remove tuples that are dead but **might** still be visible to that snapshot. Result: bloat balloons until the transaction ends.

```sql
SELECT pid, age(backend_xmin), state, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC;
```

Set `idle_in_transaction_session_timeout` and put SLOs on long open transactions.

## Why count(*) is slow

Each tuple's visibility depends on the snapshot — there is no global row count. `COUNT(*)` reads every tuple (heap or index-only) and runs visibility checks. Faster approximations:

```sql
SELECT reltuples::bigint AS approx
FROM pg_class WHERE relname='orders';
```

## Concurrency without locking conflicts (mostly)

- Reader doesn't block writer (writer creates new version).
- Writer doesn't block reader (reader sees older version).
- Two writers on the **same row** do block; one waits, then sees the latest committed row (re-reads in Read Committed) and may proceed.

Cross-row writes don't conflict at all, which is why PG scales well for OLTP.

## TOAST — overflow storage

Large field values (>2 KB-ish) are compressed and stored in a side table (`pg_toast.pg_toast_NNNN`). MVCC applies to TOAST too — large updates create new TOAST entries. `text`/`jsonb` work transparently with TOAST.

## Practical implications checklist

- Schedule autovacuum tight enough for hot tables.
- Use `fillfactor` < 100 for tables with frequent UPDATEs to enable HOT.
- Don't run hours-long transactions on a busy DB.
- Index strategically — every index slows UPDATEs.
- Avoid frequent updates to columns covered by many indexes.

## Interview Questions

**Q1. What is MVCC?**
A concurrency model where each write creates a new version of a row; reads see a consistent snapshot of committed data. Writers don't block readers, and vice versa, at the cost of garbage tuples requiring VACUUM.

**Q2. Walk me through what happens during an UPDATE.**
Old tuple's xmax is set to current XID. A new tuple is inserted with xmin=current XID. Indexes are updated to point at the new tuple (unless HOT applies). On COMMIT the old version becomes garbage once no snapshot needs it.

**Q3. What's a HOT update?**
Heap-Only Tuple — when no indexed column changes and the new tuple fits in the same page, PG chains the new tuple in-page and skips updating indexes. Improves write throughput dramatically.

**Q4. Why does PG need VACUUM?**
To reclaim dead tuples produced by MVCC, update the visibility map, and freeze old XIDs to prevent wraparound.

**Q5. What's transaction ID wraparound and how is it prevented?**
XIDs are 32 bits; old tuples must be "frozen" (marked as visible to all snapshots) before the counter wraps. Autovacuum freezes; failure to do so leads to a forced shutdown ~10M XIDs before wraparound.

**Q6. Why is `COUNT(*) FROM big_table` slow even when nothing is locked?**
MVCC means there's no maintained global row count — visibility must be checked per tuple. Use `pg_class.reltuples` for an estimate.

## Common Pitfalls

- Treating UPDATE as an in-place operation when reasoning about disk usage.
- Letting `idle in transaction` connections accumulate, blocking vacuum cleanup.
- Hot table with many indexes -> every UPDATE rewrites indexes; throughput tanks.
- Forgetting that VACUUM FULL takes an `ACCESS EXCLUSIVE` lock and rewrites the table.
- Assuming the visibility map is current — stale VM disables index-only scans.

## See also

- [Transactions](08-transactions-and-acid.md)
- [Vacuum & bloat](17-vacuum-and-bloat.md)
- [Indexes](06-indexes.md)
- [Isolation levels](09-isolation-levels.md)
