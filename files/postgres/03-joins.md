# Joins

> TL;DR: INNER and LEFT JOIN cover ~90% of real queries. Know the three physical algorithms (nested loop, hash, merge) and when each wins. LATERAL is the secret weapon for per-row subqueries. NULLs and outer joins are where interview gotchas hide.

## The conceptual zoo

```
INNER JOIN     -- rows that match on both sides
LEFT  JOIN     -- all left rows, NULLs from right when no match
RIGHT JOIN     -- mirror of LEFT
FULL  OUTER    -- both sides; NULLs where either is missing
CROSS JOIN     -- Cartesian product
SELF JOIN      -- table joined to itself with aliases
LATERAL JOIN   -- right side may reference left columns (subquery per row)
SEMI JOIN      -- EXISTS — just "does a match exist?"
ANTI JOIN      -- NOT EXISTS / LEFT JOIN ... IS NULL
```

## Setup tables (used throughout)

```sql
CREATE TABLE customers (
  id bigserial PRIMARY KEY,
  name text,
  region text
);

CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  customer_id bigint REFERENCES customers,
  amount numeric(12,2),
  placed_at timestamptz DEFAULT now()
);
```

## INNER JOIN

```sql
SELECT c.name, o.amount
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.amount > 100;
```

Drops customers with no matching orders.

## LEFT JOIN

```sql
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.name;
```

Keeps customers with **zero** orders (count = 0). Watch out:

```sql
-- BUG: pushes the predicate to the matched side, hiding "no orders" rows
SELECT c.name FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.amount > 100;        -- WRONG if you wanted customers with no orders

-- CORRECT: put condition on the JOIN
SELECT c.name FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.amount > 100;
```

A LEFT JOIN's `WHERE` clause filters the *post-join* rowset; a condition on the joined side effectively turns it into an INNER JOIN.

## FULL OUTER JOIN

```sql
SELECT COALESCE(a.user_id, b.user_id) AS user_id,
       a.last_login, b.last_purchase
FROM logins a FULL OUTER JOIN purchases b USING (user_id);
```

Use when you genuinely need rows present in either side. Rare in OLTP, common in reconciliation jobs.

## CROSS JOIN

```sql
SELECT d.dt, p.id
FROM generate_series(date '2024-01-01', date '2024-12-31', interval '1 day') AS d(dt)
CROSS JOIN products p;
```

Multiplicative — wrap with care. Most real CROSS JOINs are with `generate_series` or `unnest`.

## SELF JOIN

```sql
-- employees with their manager's name
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

Always alias.

## LATERAL — per-row subqueries

```sql
-- top 3 orders per customer
SELECT c.name, o.id, o.amount
FROM customers c
JOIN LATERAL (
  SELECT id, amount
  FROM orders
  WHERE customer_id = c.id
  ORDER BY amount DESC
  LIMIT 3
) o ON true;
```

Without `LATERAL` the subquery couldn't reference `c.id`. Use it for top-N-per-group, JSON aggregation per parent, or expanding `unnest` per row.

```sql
SELECT u.id, t
FROM users u
CROSS JOIN LATERAL unnest(u.tags) AS t;
```

## USING vs ON

```sql
SELECT * FROM a JOIN b USING (id);
-- equivalent to ON a.id = b.id, plus collapses duplicate "id" column in output.
```

## Semi joins & anti joins

There is no `SEMI JOIN` keyword. Express them with EXISTS / NOT EXISTS:

```sql
-- customers who placed at least one order (semi)
SELECT c.* FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- customers with NO orders (anti)
SELECT c.* FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

Equivalent anti-join via outer join:

```sql
SELECT c.*
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;
```

Generally, the planner converts `NOT EXISTS` to an anti-join just like the LEFT JOIN form. **Prefer `NOT EXISTS` to `NOT IN`** — `NOT IN` silently returns no rows if the subquery contains a NULL.

```sql
-- HAZARD: if any order.customer_id is NULL, this returns 0 rows.
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
```

## Join algorithms

When EXPLAIN shows a join, it's one of:

### Nested Loop

For each outer row, probe inner side (often via index). Best when outer side is tiny or inner side has an index. Cost scales with `outer_rows * inner_lookup_cost`.

```
Nested Loop
  -> Seq Scan on customers c
  -> Index Scan using orders_customer_idx on orders o
        Index Cond: (customer_id = c.id)
```

### Hash Join

Build a hash table on one side (usually the smaller), stream the other side. Equality joins only. Needs `work_mem`; spills to disk if too large.

```
Hash Join
  Hash Cond: (o.customer_id = c.id)
  -> Seq Scan on orders o
  -> Hash
      -> Seq Scan on customers c
```

### Merge Join

Sort both sides on join keys, walk in lock-step. Great when both sides are already sorted (e.g. arrived via an index). Equality and range joins both work.

### How to influence the choice

You usually don't; the planner picks based on stats. You can nudge:

- `SET enable_hashjoin = off;` (debugging)
- Better stats via `ANALYZE` and higher `default_statistics_target`
- Functional / partial indexes that satisfy the join predicate
- Rewriting the query (push filters earlier)

See [Query planner & EXPLAIN](07-query-planner-and-explain.md).

## NULL semantics in joins

```sql
-- Inner join on NULL never matches
SELECT * FROM a JOIN b ON a.x = b.x;       -- a.x = NULL fails
-- Use IS NOT DISTINCT FROM to treat NULL as equal
SELECT * FROM a JOIN b ON a.x IS NOT DISTINCT FROM b.x;
```

## Performance patterns

- Cover the FK on the child side: `CREATE INDEX ON orders(customer_id);`
- Composite indexes match leading columns only.
- Joins of unrelated tables on text columns: ensure same collation.
- Convert correlated subqueries to JOINs when planner doesn't.
- For "is there at least one" use `EXISTS` (semi-join), not `COUNT(*) > 0`.

## Interview Questions

**Q1. Difference between `WHERE` filter on outer side vs `ON` clause in LEFT JOIN?**
A predicate in `ON` filters the *joined* side before the outer rows are preserved; the same predicate in `WHERE` filters after, effectively turning the LEFT into an INNER for that condition.

**Q2. When does the planner pick a hash join over a nested loop?**
Hash join when the build side fits in `work_mem` and both sides are sizeable. Nested loop when one side is tiny or the inner side has a selective index. Merge join when both inputs are presorted.

**Q3. `NOT IN` vs `NOT EXISTS` — pick one.**
`NOT EXISTS` — it's NULL-safe and the planner can do an anti-join.

**Q4. What is a LATERAL join and when do you need it?**
A LATERAL subquery in FROM can reference earlier FROM items. You need it for per-row subqueries like top-N-per-group, JSON building per parent, or to expand `unnest()` aware of outer columns.

**Q5. Top 3 most recent orders per customer — write it.**
```sql
SELECT c.id, o.id, o.placed_at
FROM customers c
JOIN LATERAL (
  SELECT id, placed_at FROM orders
  WHERE customer_id = c.id
  ORDER BY placed_at DESC LIMIT 3
) o ON true;
```

**Q6. Anti-join on multi-column key with NULLs?**
Use `NOT EXISTS` with explicit predicates:
```sql
WHERE NOT EXISTS (
  SELECT 1 FROM b
  WHERE b.x IS NOT DISTINCT FROM a.x
    AND b.y IS NOT DISTINCT FROM a.y);
```

## Common Pitfalls

- Filter on LEFT-joined side in `WHERE`, accidentally dropping unmatched rows.
- `NOT IN` with a nullable subquery — silent zero rows.
- Equi-join on text columns with mismatched collations (slow, occasionally wrong).
- Forgetting `LATERAL` when subquery references outer column — get "missing FROM-clause entry".
- Treating CROSS JOIN as harmless; it's `n*m` rows.
- `SELECT *` after a join with duplicate column names — be explicit.

## See also

- [Subqueries & CTEs](04-subqueries-and-ctes.md)
- [Indexes](06-indexes.md)
- [Query planner](07-query-planner-and-explain.md)
- [Spring/JPA join fetch](../spring-detailed/PART-08-databases-and-sql.md)
