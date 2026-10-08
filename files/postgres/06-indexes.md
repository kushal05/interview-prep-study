# Indexes

> TL;DR: B-tree is the default. GIN for JSONB/arrays/full-text, GiST for geometry/ranges, BRIN for huge append-only tables. Partial and expression indexes are the "cheap wins" interviewers love. Indexes are not free — they slow writes and eventually bloat.

## Index types at a glance

| Type | Best for | Notes |
|------|----------|-------|
| **B-tree** | `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, prefix `LIKE 'foo%'` | Default; covers most cases |
| **Hash** | `=` only | Now WAL-logged (PG 10+); rarely better than B-tree |
| **GIN** | Composite values: jsonb, arrays, tsvector, trigram | "Generalized Inverted iNdex" |
| **GiST** | Geometry, ranges, FTS, nearest-neighbor | "Generalized Search Tree" |
| **SP-GiST** | Non-balanced trees: quadtrees, radix tries | Niche |
| **BRIN** | Huge tables where physical order correlates with value (e.g. time-series append-only) | Tiny on disk |

## B-tree fundamentals

```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_orders_customer_date ON orders (customer_id, placed_at DESC);
```

A composite index `(a, b, c)` supports queries that filter on:
- `a`
- `a` and `b`
- `a` and `b` and `c`

But **not** `b` alone, `c` alone, or `b` and `c`. Leading column rule.

It can also support `ORDER BY a, b DESC` if the index direction matches (or the planner can read backwards).

### B-tree size & locality

- Sequential PKs (`bigserial`) -> dense, tight indexes.
- Random UUIDs -> sparse insertions, more page splits, larger indexes. Consider UUIDv7 / ULID for time-ordered randomness.

## Partial indexes

Only index rows matching a predicate — smaller, cheaper, often faster.

```sql
CREATE INDEX idx_orders_open
  ON orders (placed_at)
  WHERE status = 'open';
```

Great for: soft-deleted rows excluded, status = 'pending', a small "hot" subset of a giant table.

## Expression / functional indexes

```sql
CREATE INDEX idx_users_email_lower ON users (lower(email));
SELECT * FROM users WHERE lower(email) = lower('A@B.com');     -- uses index
```

You must call the function the same way in the query. For case-insensitive equality consider `citext` instead.

## Covering (INCLUDE) indexes — PG 11+

```sql
CREATE INDEX idx_orders_customer_inc
  ON orders (customer_id)
  INCLUDE (status, amount);
```

`INCLUDE` columns aren't part of the B-tree key (no ordering) but live in the leaf pages — enabling **index-only scans** for queries that need just those columns. Must combine with up-to-date visibility map (i.e. table well-vacuumed).

## Unique indexes & constraints

```sql
CREATE UNIQUE INDEX idx_users_email ON users (lower(email));
-- or
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);
```

A `UNIQUE` constraint is implemented by a unique index. With NULLs, distinct NULLs are allowed by default (PG <= 14); PG 15 adds `UNIQUE NULLS NOT DISTINCT`.

## GIN — JSONB, arrays, FTS, trigrams

```sql
-- jsonb containment
CREATE INDEX idx_events_payload ON events USING gin (payload jsonb_path_ops);
SELECT * FROM events WHERE payload @> '{"type":"signup"}';

-- arrays
CREATE INDEX idx_users_tags ON users USING gin (tags);
SELECT * FROM users WHERE tags @> ARRAY['vip'];

-- full text
CREATE INDEX idx_docs_fts ON docs USING gin (to_tsvector('english', body));

-- trigram for LIKE / ILIKE / similarity
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING gin (name gin_trgm_ops);
SELECT * FROM users WHERE name ILIKE '%ravi%';
```

`jsonb_path_ops` is smaller and faster for pure containment queries; default `jsonb_ops` supports more operators.

## GiST — geometry, ranges, KNN

```sql
CREATE INDEX idx_bookings_during ON bookings USING gist (during);
SELECT * FROM bookings WHERE during && '[2024-01-01, 2024-01-31)'::tstzrange;
```

KNN search ("nearest neighbour"):

```sql
SELECT * FROM places ORDER BY geom <-> ST_Point(77.6, 12.97) LIMIT 5;
```

## BRIN — block-range index

```sql
CREATE INDEX idx_events_ts_brin ON events USING brin (event_ts) WITH (pages_per_range = 128);
```

Stores min/max value per range of pages. Great for naturally-sorted huge tables (server logs, sensor data) where each query touches a contiguous time slice. Tiny — maybe a few MB for billions of rows.

## Hash indexes

```sql
CREATE INDEX idx_users_session_hash ON sessions USING hash (token);
```

Only useful for `=`. B-tree on the same column is typically the same speed and supports more. Skip unless you really know why.

## Reading the index in EXPLAIN

```
-- Index Scan: probe index, jump to heap for each match.
-- Index Only Scan: read everything from the index leaves.
-- Bitmap Index Scan + Bitmap Heap Scan: collect TIDs in a bitmap, then sorted heap reads.
-- Seq Scan: full table scan, not always bad on small / mostly-matching queries.
```

Bitmap is chosen when many index matches are expected — turns random I/O into sequential.

## When indexes are NOT used

- Function applied to indexed column: `WHERE date(created_at) = ...` -> add an expression index, or rewrite predicate.
- Type mismatch: indexed `bigint`, parameter sent as text.
- `LIKE '%foo'` with leading wildcard -> needs trigram GIN.
- Tiny table: seq scan wins.
- Stale stats -> ANALYZE.
- `OR` across columns -> planner may not combine; consider two queries + UNION ALL, or a multi-column index.

## CREATE INDEX vs CREATE INDEX CONCURRENTLY

```sql
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders (customer_id);
```

- Default `CREATE INDEX` takes `SHARE` lock — blocks writers.
- `CONCURRENTLY` does not block reads/writes; two-pass scan, slower; cannot run inside a transaction; if it fails, leaves an `INVALID` index that you must `DROP` and retry.

```sql
SELECT * FROM pg_index WHERE NOT indisvalid;
```

## REINDEX & maintenance

- Bloated indexes — `REINDEX INDEX CONCURRENTLY idx_x;` (PG 12+).
- Rebuild PK / unique: `REINDEX TABLE CONCURRENTLY orders;`.
- See [vacuum & bloat](17-vacuum-and-bloat.md).

## Identifying unused / duplicate indexes

```sql
SELECT relname, indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;          -- never-scanned indexes
```

```sql
SELECT pg_size_pretty(pg_relation_size(indexrelid)) AS size, *
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

Duplicate detection: `(a, b)` is redundant if `(a, b, c)` already exists (for B-tree).

## Interview Questions

**Q1. When is a `LIKE` query indexable?**
Anchored prefix (`'foo%'`) is supported by B-tree with proper collation (`text_pattern_ops` if locale is non-C). Infix/`%foo%` needs `pg_trgm` GIN.

**Q2. B-tree composite index `(a, b, c)` — which queries can use it?**
Queries that filter from the leading column: `a`; `a AND b`; `a AND b AND c`. Also useful for ORDER BY when the prefix matches.

**Q3. What's an index-only scan and what does it require?**
Reads all needed columns from index leaves without touching heap. Requires: all selected columns covered by the index (key or INCLUDE), and the visibility map up to date (i.e. recent VACUUM).

**Q4. GIN vs GiST?**
GIN: faster for read-mostly, slower writes, exact lookups (jsonb containment, arrays, FTS). GiST: balanced read/write, supports ranges, KNN, geometry.

**Q5. Difference between CREATE INDEX and CREATE INDEX CONCURRENTLY?**
CONCURRENTLY doesn't block writes; runs two passes; cannot be in a transaction; can leave an INVALID index on failure.

**Q6. Why might the planner choose a seq scan over an index scan?**
The query selects a large fraction of rows; bitmap scan would still touch most pages; random I/O cost is higher than sequential. Tune with stats, `random_page_cost`, `effective_cache_size`.

## Common Pitfalls

- Adding indexes "just in case" — they hurt INSERT/UPDATE throughput and bloat WAL.
- Forgetting that `WHERE lower(email)=...` skips an index on `email`.
- Random UUID PKs causing B-tree fragmentation at scale.
- Composite index column order picked alphabetically instead of by query selectivity.
- Running `CREATE INDEX` (not CONCURRENTLY) on production -> writers block.
- Expecting an index-only scan when the table has lots of recent dead tuples (visibility map stale).

## See also

- [Query planner & EXPLAIN](07-query-planner-and-explain.md)
- [Full-text search](14-full-text-search.md)
- [JSONB & arrays](13-jsonb-and-arrays.md)
- [Vacuum & bloat](17-vacuum-and-bloat.md)
