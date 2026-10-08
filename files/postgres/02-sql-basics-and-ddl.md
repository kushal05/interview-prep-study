# SQL Basics & DDL

> TL;DR: Postgres has a rich type system (numeric, text, jsonb, uuid, timestamptz, arrays, enums, ranges). Constraints are enforced at write time. `ALTER TABLE` is mostly metadata-only but `ADD COLUMN ... DEFAULT non_constant` historically rewrote the table — PG 11+ avoids this for constant defaults.

## Creating tables

```sql
CREATE TABLE users (
  id           bigserial PRIMARY KEY,                    -- bigint + sequence
  email        text       NOT NULL UNIQUE,
  full_name    text,
  age          int        CHECK (age >= 0 AND age < 130),
  status       text       NOT NULL DEFAULT 'active',
  metadata     jsonb      NOT NULL DEFAULT '{}'::jsonb,
  tags         text[]     NOT NULL DEFAULT '{}',
  created_at   timestamptz NOT NULL DEFAULT now(),
  updated_at   timestamptz NOT NULL DEFAULT now()
);
```

### Identity columns (modern alternative to `serial`)

```sql
CREATE TABLE orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id bigint NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  amount numeric(12,2) NOT NULL CHECK (amount > 0),
  placed_at timestamptz NOT NULL DEFAULT now()
);
```

`GENERATED ALWAYS AS IDENTITY` is SQL-standard, blocks accidental client-supplied IDs, and avoids the ownership quirks of `bigserial`.

## Data types worth knowing

### Numerics

| Type | Use when |
|------|----------|
| `smallint` (int2) | Counters, enums encoded as int |
| `integer` (int4) | Default integer; range ~+/-2.1B |
| `bigint` (int8) | IDs in any system that might scale |
| `numeric(p, s)` | **Money, anything where rounding matters** |
| `real`, `double precision` | Scientific data; never money |

Never store money in `float`. `numeric(12, 2)` for cents — exact arithmetic.

### Strings

- `text` — variable-length, no length limit; same internal storage as `varchar(n)`.
- `varchar(n)` — text with a check constraint on length. No performance benefit over `text`.
- `char(n)` — **avoid**; right-pads with spaces and compares with weird semantics.
- `citext` (extension) — case-insensitive text. Use for emails, usernames.

```sql
CREATE EXTENSION IF NOT EXISTS citext;
ALTER TABLE users ALTER COLUMN email TYPE citext;
```

### UUIDs

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();          -- random v4
-- PG 13+:
SELECT gen_random_uuid();           -- requires pgcrypto OR built-in in PG 13+
```

UUIDs are 16 bytes vs 8 for bigint — fine for surface IDs, expensive as PKs at scale because random UUIDs trash B-tree locality. Consider ULID / UUIDv7 if you need both.

### Dates & times

| Type | Notes |
|------|-------|
| `date` | Calendar date, no time |
| `time` / `time with time zone` | Rarely useful |
| `timestamp` | "Wall clock", no TZ awareness — **dangerous** |
| `timestamptz` | Stored in UTC, displayed in session TZ — use this |
| `interval` | `'1 day 02:30'::interval` |

```sql
SELECT now() AT TIME ZONE 'America/New_York';
SELECT now() - interval '7 days';
SELECT date_trunc('month', now());
SELECT extract(epoch FROM (t2 - t1));   -- seconds between
```

### JSON & JSONB

```sql
SELECT '{"a":1, "b":[2,3]}'::jsonb -> 'b';            -- jsonb [2,3]
SELECT '{"a":1}'::jsonb @> '{"a":1}';                  -- true (contains)
```

Detailed coverage in [JSONB & arrays](13-jsonb-and-arrays.md).

### Arrays

```sql
SELECT ARRAY[1,2,3] || ARRAY[4,5];       -- {1,2,3,4,5}
SELECT 3 = ANY(ARRAY[1,2,3]);            -- true
SELECT unnest(ARRAY['a','b','c']);       -- expand to rows
```

### Enums

```sql
CREATE TYPE order_status AS ENUM ('pending','paid','shipped','cancelled');
ALTER TABLE orders ADD COLUMN status order_status NOT NULL DEFAULT 'pending';
```

Enums are storage-cheap (4 bytes) but reordering values requires `ALTER TYPE ... ADD VALUE` (cannot reorder/remove without recreate). Many shops prefer `text` + `CHECK (status IN (...))`.

## Constraints

```sql
ALTER TABLE orders
  ADD CONSTRAINT chk_amount_positive CHECK (amount > 0),
  ADD CONSTRAINT uq_user_external UNIQUE (user_id, external_ref),
  ADD CONSTRAINT fk_user FOREIGN KEY (user_id)
      REFERENCES users(id) ON DELETE CASCADE
      DEFERRABLE INITIALLY IMMEDIATE;
```

- `NOT NULL` — most performant constraint; checked in C.
- `UNIQUE` — backed by a unique B-tree index.
- `CHECK` — arbitrary boolean expression; not enforced across rows.
- `FOREIGN KEY` — referential integrity; consider performance impact on bulk loads.
- `EXCLUDE` — generalized constraint, e.g. "no two bookings overlap":

```sql
CREATE EXTENSION btree_gist;
CREATE TABLE bookings (
  id bigserial PRIMARY KEY,
  room_id int NOT NULL,
  during tstzrange NOT NULL,
  EXCLUDE USING gist (room_id WITH =, during WITH &&)
);
```

## ALTER TABLE — what's cheap, what's painful

| Operation | Cost |
|-----------|------|
| `ADD COLUMN` (no default, or constant default in PG 11+) | Metadata only — fast |
| `ADD COLUMN ... DEFAULT volatile_fn()` | Full table rewrite |
| `DROP COLUMN` | Metadata; space reclaimed by VACUUM |
| `ALTER COLUMN TYPE` (compatible) | Sometimes metadata, often rewrite |
| `ADD CONSTRAINT NOT VALID` then `VALIDATE` | Skips full scan during ADD, validates later |
| `ALTER TABLE ... SET TABLESPACE` | Rewrites |

### Adding a NOT NULL column safely on a hot table

```sql
ALTER TABLE big ADD COLUMN flag boolean DEFAULT false;             -- PG 11+: instant
-- backfill if computed:
UPDATE big SET flag = compute(id) WHERE flag IS NULL;              -- in batches
ALTER TABLE big ALTER COLUMN flag SET NOT NULL;
```

`ALTER TABLE ... SET NOT NULL` scans the table; PG 12+ can skip the scan if a `CHECK (col IS NOT NULL) NOT VALID` was added & validated first.

### Adding a CHECK constraint without long locks

```sql
ALTER TABLE big ADD CONSTRAINT chk_pos CHECK (qty > 0) NOT VALID;
ALTER TABLE big VALIDATE CONSTRAINT chk_pos;                       -- shared lock only
```

## Generated columns (PG 12+)

```sql
CREATE TABLE products (
  id bigserial PRIMARY KEY,
  price_cents int NOT NULL,
  tax_cents   int GENERATED ALWAYS AS (price_cents * 18 / 100) STORED
);
```

Only `STORED` is supported (no virtual yet).

## INSERT, UPDATE, DELETE essentials

```sql
INSERT INTO users(email, full_name) VALUES ('x@y.com', 'X Y')
RETURNING id, created_at;

-- Upsert
INSERT INTO users(email) VALUES ('x@y.com')
ON CONFLICT (email) DO UPDATE
  SET full_name = EXCLUDED.full_name
RETURNING id;

UPDATE orders SET status='shipped'
WHERE id = ANY($1::bigint[])
RETURNING id;

DELETE FROM orders WHERE placed_at < now() - interval '1 year';
```

`RETURNING` is a Postgres superpower — you can pipe inserted rows into a CTE chain. See [CTEs](04-subqueries-and-ctes.md).

## Truncate vs delete

- `DELETE FROM t` — row-by-row, fires triggers, writes WAL per row, leaves dead tuples for VACUUM.
- `TRUNCATE t` — instant, no triggers fired by default (`TRUNCATE ... CASCADE`), resets sequences with `RESTART IDENTITY`, but requires `ACCESS EXCLUSIVE` lock.

## NULL semantics — the classic trap

```sql
SELECT NULL = NULL;          -- NULL, not true
SELECT NULL <> NULL;         -- NULL
SELECT NULL IS NULL;         -- true
SELECT a IS NOT DISTINCT FROM b;   -- treats NULL as equal
```

`WHERE x = NULL` matches nothing; you must use `IS NULL`. UNIQUE constraints treat NULLs as distinct (PG 14 and earlier). PG 15 adds `UNIQUE NULLS NOT DISTINCT`.

## Interview Questions

**Q1. `varchar(n)` vs `text` — which is faster?**
Identical. Both use the `varlena` representation. `varchar(n)` just adds a length check. Use `text` + `CHECK` if you really need a limit.

**Q2. Why prefer `timestamptz` over `timestamp`?**
`timestamptz` normalizes to UTC on store and converts on display. `timestamp` silently drops TZ context, causing DST/timezone bugs that surface only in production.

**Q3. `bigserial` vs `GENERATED ALWAYS AS IDENTITY`?**
Both create a sequence and a default. Identity is SQL-standard, owned by the column (drops with it), and rejects explicit inserts unless you say `OVERRIDING SYSTEM VALUE`. Prefer identity for new code.

**Q4. How would you add a NOT NULL column to a 200 GB table without downtime?**
Add with a constant default (PG 11+ is instant). If the value is computed, add nullable, backfill in batches, then add `CHECK ... NOT VALID`, `VALIDATE`, then `SET NOT NULL` (PG 12+ can skip the scan thanks to the validated CHECK).

**Q5. What's the difference between `UNIQUE` and `PRIMARY KEY`?**
PK = UNIQUE + NOT NULL + cluster-default identity for FKs. A table can have many UNIQUE constraints but only one PK.

**Q6. When would you use `EXCLUDE` constraints?**
For "no two rows overlap" semantics on ranges (bookings, schedules) — something UNIQUE cannot express.

## Common Pitfalls

- Using `float` for money. Use `numeric`.
- Forgetting that empty string `''` is **not** NULL.
- `ADD COLUMN ... DEFAULT random()` rewrites the entire table.
- Enum values can't be removed; design enums conservatively.
- `WHERE col = NULL` is a silent bug — use `IS NULL`.
- `TRUNCATE` requires ACCESS EXCLUSIVE and won't queue politely behind running transactions.

## See also

- [Joins](03-joins.md)
- [JSONB & arrays](13-jsonb-and-arrays.md)
- [Indexes](06-indexes.md)
- [Spring JPA mapping](../spring-detailed/PART-08-databases-and-sql.md)
