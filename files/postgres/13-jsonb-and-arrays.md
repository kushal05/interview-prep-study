# JSONB & Arrays

> TL;DR: `jsonb` is binary, indexable, deduplicates keys, and supports rich operators (`@>`, `?`, `->`, `->>`, `#>`). `json` is text-only — avoid. GIN indexes make containment queries fast. Use these instead of [MongoDB](../mongodb/) when you also need relational guarantees on the same data.

## json vs jsonb

| | json | jsonb |
|---|------|-------|
| Storage | text | parsed binary |
| Key order / duplicates | preserved / kept | not preserved / last wins |
| Index support | functional only | GIN, btree on expressions |
| Read speed | slower (re-parse) | faster |
| Write speed | slightly faster (no parse) | slightly slower |

Default to **jsonb** unless you must preserve exact byte-for-byte input.

## Operators cheatsheet

```sql
SELECT '{"a":1,"b":{"c":2},"d":[10,20]}'::jsonb ->  'a';        -- 1 (jsonb)
SELECT '{"a":1,"b":{"c":2},"d":[10,20]}'::jsonb ->> 'a';        -- '1' (text)
SELECT '{"a":1,"b":{"c":2}}'::jsonb -> 'b' -> 'c';              -- 2
SELECT '{"a":1,"b":{"c":2}}'::jsonb #>  '{b,c}';                -- 2 (jsonb)
SELECT '{"a":1,"b":{"c":2}}'::jsonb #>> '{b,c}';                -- '2' (text)

SELECT '{"a":1,"b":2}'::jsonb @> '{"a":1}';                     -- true (left contains right)
SELECT '{"a":1,"b":2}'::jsonb <@ '{"a":1,"b":2,"c":3}';         -- true (left contained by right)

SELECT '{"a":1,"b":2}'::jsonb ?  'a';                           -- true (has key)
SELECT '{"a":1,"b":2}'::jsonb ?| ARRAY['x','b'];                -- true (any key)
SELECT '{"a":1,"b":2}'::jsonb ?& ARRAY['a','b'];                -- true (all keys)
```

## Modifying jsonb

```sql
-- concat (shallow merge; later keys win)
SELECT '{"a":1}'::jsonb || '{"b":2}'::jsonb;                    -- {"a":1,"b":2}

-- delete key / path
SELECT '{"a":1,"b":2}'::jsonb - 'a';                            -- {"b":2}
SELECT '{"a":{"b":1,"c":2}}'::jsonb #- '{a,b}';                 -- {"a":{"c":2}}

-- set / update path (creates if missing when 4th arg true)
SELECT jsonb_set('{"a":{"b":1}}', '{a,c}', '99', true);          -- {"a":{"b":1,"c":99}}

-- insert without overwriting
SELECT jsonb_insert('{"a":[1,2]}', '{a,0}', '99');               -- {"a":[99,1,2]}
```

## Path expressions (jsonb_path_query, PG 12+)

SQL/JSON path language is SQL-standard:

```sql
SELECT jsonb_path_query('{"items":[{"p":1,"q":2},{"p":3,"q":4}]}',
                        '$.items[*] ? (@.p > 1).q');             -- 4
```

Supports filters, arithmetic, regex.

## Indexing jsonb

### GIN with default `jsonb_ops`

```sql
CREATE INDEX idx_events_payload ON events USING gin (payload);
-- supports @>, ?, ?|, ?&
```

### GIN with `jsonb_path_ops`

```sql
CREATE INDEX idx_events_payload_path ON events USING gin (payload jsonb_path_ops);
-- supports only @> but smaller & faster for that case
```

### Expression / partial indexes for known fields

```sql
CREATE INDEX ON events ((payload->>'type')) WHERE payload ? 'type';
SELECT * FROM events WHERE payload->>'type' = 'signup';   -- uses btree index
```

If you query a specific JSON field constantly, a btree on the expression often beats GIN.

## Typical query patterns

```sql
-- exact value
SELECT * FROM events WHERE payload->>'type' = 'signup';

-- containment (fast with GIN)
SELECT * FROM events WHERE payload @> '{"type":"signup","plan":"pro"}';

-- key presence
SELECT * FROM events WHERE payload ? 'utm_source';

-- numeric comparison: cast first
SELECT * FROM events WHERE (payload->>'amount')::numeric > 100;
```

## Aggregating into JSON

```sql
SELECT customer_id,
       jsonb_agg(jsonb_build_object('id', id, 'amt', amount) ORDER BY placed_at DESC) AS orders
FROM orders
GROUP BY customer_id;
```

`jsonb_build_object('k', v, ...)` is cleaner than building strings. `row_to_json` / `to_jsonb(row_var)` convert composites.

## Arrays

### Literals & access

```sql
SELECT ARRAY[1,2,3];
SELECT '{1,2,3}'::int[];
SELECT (ARRAY[1,2,3])[2];           -- 2 (1-based!)
SELECT array_length(ARRAY[1,2,3], 1);
SELECT cardinality(ARRAY[[1,2],[3,4]]);
```

Arrays in PG are **1-indexed**. Multi-dimensional but uniform.

### Useful operators

```sql
SELECT ARRAY[1,2,3] || ARRAY[4,5];           -- {1,2,3,4,5}
SELECT 2 = ANY(ARRAY[1,2,3]);                -- true
SELECT 5 = ALL(ARRAY[5,5,5]);                -- true
SELECT ARRAY[1,2,3] @> ARRAY[2,3];           -- true (contains)
SELECT ARRAY[1,2] && ARRAY[2,3];             -- true (overlap)
```

### Functions

```sql
SELECT unnest(ARRAY['a','b','c']);            -- expand to rows
SELECT array_agg(id ORDER BY id) FROM users;
SELECT array_position(ARRAY['x','y','z'], 'y'); -- 2
SELECT array_remove(ARRAY[1,2,3,2], 2);         -- {1,3}
```

### Index arrays with GIN

```sql
CREATE INDEX idx_users_tags ON users USING gin (tags);
SELECT * FROM users WHERE tags @> ARRAY['admin'];
```

## When to use jsonb (and when not)

**Use jsonb when**:
- Schema varies per row (attributes per product, event payloads).
- You need to query nested structure with containment.
- You want one DB for relational + flex columns.

**Don't use jsonb for**:
- Stable fields you query/join frequently — promote to columns.
- Anything financial (use `numeric`).
- Foreign keys — store IDs in normal columns, optionally JSONB metadata next to them.

## Comparison vs MongoDB

- jsonb has full transactional ACID (Mongo single-doc only in classic, multi-doc since 4.0).
- Joins available; Mongo needs `$lookup`.
- Schema enforcement via CHECK constraints + JSON Schema (`jsonb_typeof`).
- Mongo wins on flexible sharding & built-in aggregation pipeline ergonomics.

More: [MongoDB notes](../mongodb/).

## Performance tips

- Prefer `payload @> '{"k":"v"}'` over `payload->>'k' = 'v'` for GIN-indexed columns.
- Cast once: `(payload->>'amount')::numeric` is OK; doing it 5 times in one query duplicates work.
- TOAST: jsonb columns are stored out-of-line if large — UPDATE rewrites the entire jsonb value, even for a single key change.
- `jsonb_set` allocates a new value; for huge JSON, consider modeling as a child table.

## Interview Questions

**Q1. Why jsonb over json?**
Binary storage = faster reads, indexable, supports containment operators. JSON keeps text exactly as written; only useful if exact byte preservation matters.

**Q2. How do you index a jsonb column?**
GIN with `jsonb_ops` (more operators) or `jsonb_path_ops` (smaller, only `@>`). For specific paths, btree on an expression like `(payload->>'type')`.

**Q3. `->` vs `->>` vs `#>>`?**
`->` returns jsonb (sub-document or array element). `->>` returns text. `#>` and `#>>` take a path array (`'{a,b,c}'`).

**Q4. Update one key inside a 1 MB jsonb — is that cheap?**
No. PG rewrites the whole jsonb value (TOAST entry replaced). For frequently-updated nested data, model as a child table.

**Q5. When would you migrate a jsonb field to a column?**
When the field is always present, queried/filtered often, indexed, or part of joins. Stable, hot fields belong as columns.

**Q6. Arrays vs a join table?**
Arrays for small, capped, mostly-read sets that move with the parent (tags). Join table when items have their own lifecycle, attributes, or many rows.

## Common Pitfalls

- Comparing `->>` value as a string instead of casting.
- Using `?` operator inside parameter binding without escaping in JDBC.
- Storing money/timestamps as JSON values, losing types.
- GIN index unused because the query uses an unsupported operator (use `jsonb_ops` for richer ops).
- Huge jsonb blobs rewritten per UPDATE.
- Forgetting arrays are 1-indexed.

## See also

- [Indexes](06-indexes.md)
- [Stored procedures & triggers (audit jsonb pattern)](12-stored-procedures-and-triggers.md)
- [MongoDB comparison](../mongodb/)
