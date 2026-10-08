# Subqueries & CTEs

> TL;DR: Subqueries appear in SELECT/FROM/WHERE. Correlated subqueries reference the outer row; uncorrelated ones do not. CTEs (`WITH ...`) name a result set and aid readability. **Recursive CTEs** solve graph/tree traversals. In PG 12+, CTEs are inlined by default (`NOT MATERIALIZED`) — they're no longer optimization fences.

## Subquery shapes

```sql
-- Scalar in SELECT
SELECT id,
  (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS n_orders
FROM customers c;

-- Derived table in FROM
SELECT region, AVG(spend) FROM (
  SELECT customer_id, SUM(amount) AS spend
  FROM orders GROUP BY customer_id
) s
JOIN customers c USING (customer_id) ... ;   -- (would need c.region)

-- IN / EXISTS in WHERE
SELECT * FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE amount > 1000);
```

### Correlated vs uncorrelated

```sql
-- Uncorrelated: subquery runs once
SELECT * FROM orders WHERE amount > (SELECT AVG(amount) FROM orders);

-- Correlated: subquery references the outer row, runs per row (logically)
SELECT * FROM orders o
WHERE amount > (
  SELECT AVG(amount) FROM orders WHERE customer_id = o.customer_id
);
```

The planner often rewrites correlated subqueries to joins. EXPLAIN to confirm.

### Scalar subquery gotcha

Must return at most one row, or you'll get a runtime error:

```
ERROR: more than one row returned by a subquery used as an expression
```

Use `LIMIT 1` with `ORDER BY` to make intent explicit.

## Common Table Expressions (CTEs)

```sql
WITH top_customers AS (
  SELECT customer_id, SUM(amount) AS spend
  FROM orders
  GROUP BY customer_id
  HAVING SUM(amount) > 10000
)
SELECT c.name, t.spend
FROM top_customers t
JOIN customers c ON c.id = t.customer_id;
```

### Multiple CTEs

```sql
WITH
  recent AS (
    SELECT * FROM orders WHERE placed_at > now() - interval '30 days'
  ),
  by_customer AS (
    SELECT customer_id, COUNT(*) c, SUM(amount) s FROM recent GROUP BY customer_id
  )
SELECT * FROM by_customer WHERE c > 5;
```

### Data-modifying CTEs

`INSERT`/`UPDATE`/`DELETE` can appear in `WITH` and feed downstream:

```sql
WITH moved AS (
  DELETE FROM cart WHERE user_id = $1 RETURNING *
)
INSERT INTO order_items (order_id, product_id, qty)
SELECT $2, product_id, qty FROM moved;
```

A single statement; runs atomically inside its transaction.

### MATERIALIZED vs NOT MATERIALIZED (PG 12+)

Before PG 12, CTEs were **always** materialized (computed once, stored). That was a known "optimization fence." From PG 12, the planner inlines simple CTEs by default. You can force either:

```sql
WITH x AS NOT MATERIALIZED ( ... )       -- inline (default if used once, non-volatile, non-recursive)
WITH x AS MATERIALIZED     ( ... )       -- compute once, then reuse
```

Use `MATERIALIZED` when:
- The CTE is referenced multiple times and is expensive.
- You want to fence the planner away from a bad rewrite.

## Recursive CTEs

Generic shape:

```sql
WITH RECURSIVE rcte AS (
  -- anchor (base case)
  SELECT ... FROM ...
  UNION ALL
  -- recursive term: references rcte itself
  SELECT ... FROM rcte JOIN ...
)
SELECT * FROM rcte;
```

### Tree traversal (org chart)

```sql
CREATE TABLE employees (
  id bigserial PRIMARY KEY,
  manager_id bigint REFERENCES employees,
  name text
);

-- All reports under id = 1
WITH RECURSIVE subordinates AS (
  SELECT id, manager_id, name, 1 AS depth
  FROM employees WHERE id = 1
  UNION ALL
  SELECT e.id, e.manager_id, e.name, s.depth + 1
  FROM employees e JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

### Generate a series of dates

```sql
WITH RECURSIVE days AS (
  SELECT date '2024-01-01' AS d
  UNION ALL
  SELECT d + 1 FROM days WHERE d < '2024-01-31'
)
SELECT * FROM days;
```

(`generate_series` is simpler — prefer it when applicable.)

### Cycle detection (PG 14+)

```sql
WITH RECURSIVE g AS (
  SELECT id, parent_id, ARRAY[id] AS path FROM nodes WHERE id = 1
  UNION ALL
  SELECT n.id, n.parent_id, g.path || n.id
  FROM nodes n JOIN g ON n.parent_id = g.id
  WHERE NOT n.id = ANY(g.path)
)
SELECT * FROM g;
-- Or PG 14:
-- ... CYCLE id SET is_cycle USING path
```

## Subquery patterns worth memorizing

### Top-N per group (LATERAL is usually cleanest)

```sql
SELECT * FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY placed_at DESC) AS rn
  FROM orders
) t WHERE rn <= 3;
```

vs LATERAL form in [joins](03-joins.md).

### Anti-pattern: count to check existence

```sql
-- BAD
IF (SELECT COUNT(*) FROM orders WHERE user_id=$1) > 0 ...
-- GOOD
IF EXISTS (SELECT 1 FROM orders WHERE user_id=$1) ...
```

### IN-list vs JOIN to a value list

```sql
-- pass an array param and use ANY
SELECT * FROM users WHERE id = ANY($1::bigint[]);
-- or VALUES + JOIN for big lists
SELECT u.* FROM users u JOIN (VALUES (1),(2),(3)) v(id) ON v.id = u.id;
```

## Aggregate and grouping refresher

```sql
SELECT region,
       COUNT(*)                     AS n,
       COUNT(DISTINCT customer_id)  AS uniq_customers,
       SUM(amount) FILTER (WHERE status='paid') AS paid_total,
       jsonb_agg(jsonb_build_object('id', id, 'amt', amount)) AS line_items
FROM orders
GROUP BY region
HAVING SUM(amount) > 0;
```

- `FILTER (WHERE ...)` — conditional aggregate, beats `SUM(CASE WHEN ... )`.
- `HAVING` filters groups; `WHERE` filters rows before grouping.
- `GROUPING SETS`, `ROLLUP`, `CUBE` produce multiple aggregation levels in one pass.

```sql
SELECT region, status, SUM(amount)
FROM orders
GROUP BY ROLLUP (region, status);
```

## Interview Questions

**Q1. Are CTEs a performance optimization?**
In PG 11 and earlier they were optimization **fences** (materialized always). PG 12+ inlines them by default. So use CTEs for readability, not as a hint. Use `MATERIALIZED` if you actually want fencing.

**Q2. Recursive CTE structure?**
`WITH RECURSIVE name AS (anchor UNION ALL recursive-term) SELECT ... FROM name;` The recursive term references `name` and must converge.

**Q3. EXISTS vs IN vs JOIN — which is fastest?**
Modern PG plans them equivalently for non-NULL cases. Prefer `EXISTS` for semantics (NULL-safe) and readability. Avoid `NOT IN` with nullable subqueries.

**Q4. Difference between WHERE and HAVING?**
`WHERE` filters rows before grouping; `HAVING` filters groups after. You can use a column not in GROUP BY only inside an aggregate in `HAVING`.

**Q5. How would you find the second-highest salary per department?**
```sql
SELECT DISTINCT ON (department_id) ...   -- not what you want
-- Window-function approach:
SELECT * FROM (
  SELECT *, DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS r
  FROM employees
) t WHERE r = 2;
```

**Q6. Detect cycles in a graph using a recursive CTE.**
Carry the visited path as an array, stop when the current node appears in it (or use PG 14's `CYCLE` clause).

## Common Pitfalls

- Scalar subqueries returning >1 row at runtime.
- Believing CTEs always materialize (no longer true).
- Recursive CTE missing convergence condition - infinite recursion until `max_recursion` or memory.
- `NOT IN` + NULL = empty result.
- Forgetting `UNION ALL` (cheap) vs `UNION` (de-dup, requires sort/hash).
- Using `COUNT(*) > 0` for existence checks.

## See also

- [Window functions](05-window-functions.md)
- [Joins](03-joins.md)
- [Query planner](07-query-planner-and-explain.md)
