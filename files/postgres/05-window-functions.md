# Window Functions

> TL;DR: Window functions compute per-row results over a "window" of related rows — without collapsing them like `GROUP BY` does. Master `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, `SUM/AVG OVER`, and frame clauses (`ROWS BETWEEN`).

## Anatomy

```sql
function() OVER (
  PARTITION BY <cols>            -- reset window per group; optional
  ORDER BY <cols>                -- order within partition; required for ranking/LAG/LEAD
  ROWS BETWEEN <start> AND <end> -- the frame; default depends on function
)
```

## The classics

```sql
SELECT id, customer_id, amount,
  ROW_NUMBER()       OVER w AS rn,    -- 1,2,3,4 (no ties)
  RANK()             OVER w AS rk,    -- 1,2,2,4 (ties skip)
  DENSE_RANK()       OVER w AS drk,   -- 1,2,2,3 (ties don't skip)
  PERCENT_RANK()     OVER w AS pr,    -- (rank-1)/(n-1)
  NTILE(4)           OVER w AS quartile
FROM orders
WINDOW w AS (PARTITION BY customer_id ORDER BY amount DESC);
```

The `WINDOW w AS (...)` form lets you reuse a window definition.

### Top N per group

```sql
SELECT *
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY placed_at DESC) AS rn
  FROM orders
) t
WHERE rn <= 3;
```

## LAG and LEAD

```sql
SELECT id, customer_id, placed_at,
  LAG(placed_at)  OVER (PARTITION BY customer_id ORDER BY placed_at) AS prev_order_at,
  LEAD(placed_at) OVER (PARTITION BY customer_id ORDER BY placed_at) AS next_order_at,
  placed_at - LAG(placed_at) OVER (PARTITION BY customer_id ORDER BY placed_at) AS gap
FROM orders;
```

Signatures: `LAG(col, offset DEFAULT 1, default_value DEFAULT NULL)`.

### Detect resumption gaps / sessions

```sql
SELECT id, ts, ts - LAG(ts) OVER (PARTITION BY user_id ORDER BY ts) AS gap,
  SUM(CASE WHEN ts - LAG(ts) OVER (PARTITION BY user_id ORDER BY ts) > interval '30 min' THEN 1 ELSE 0 END)
    OVER (PARTITION BY user_id ORDER BY ts) AS session_id
FROM events;
```

## FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
SELECT customer_id, placed_at, amount,
  FIRST_VALUE(amount) OVER w AS first_amt,
  LAST_VALUE(amount)  OVER w AS last_amt
FROM orders
WINDOW w AS (PARTITION BY customer_id ORDER BY placed_at
             ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING);
```

**Without an explicit frame, `LAST_VALUE` looks "wrong"** because the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Always set the frame for last-value queries.

## Aggregates as window functions

```sql
-- running totals
SELECT id, placed_at, amount,
  SUM(amount) OVER (PARTITION BY customer_id ORDER BY placed_at) AS running_total,
  AVG(amount) OVER (PARTITION BY customer_id ORDER BY placed_at
                    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_7
FROM orders;

-- ratio of total
SELECT customer_id, SUM(amount) AS total,
  SUM(amount) * 1.0 / SUM(SUM(amount)) OVER () AS share_of_all
FROM orders GROUP BY customer_id;
```

Note the trick: window functions can wrap aggregates from the GROUP BY.

## Frames in depth

| Frame | Meaning |
|-------|---------|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | All prior rows + current |
| `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` | 7-row moving window |
| `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` | Current + all after |
| `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | Default for ordered aggregates; **peers** (same ORDER BY key) treated together |
| `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` | Time-based (PG 11+) |
| `GROUPS` | Frame in terms of peer groups (PG 11+) |

### Time-based moving sum (PG 11+)

```sql
SELECT id, placed_at, amount,
  SUM(amount) OVER (
    ORDER BY placed_at
    RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
  ) AS sum_last_7d
FROM orders;
```

## Order of evaluation

```
FROM -> WHERE -> GROUP BY -> HAVING -> WINDOW funcs -> SELECT -> DISTINCT -> ORDER BY -> LIMIT
```

You **cannot** filter on a window function in `WHERE` (it hasn't been computed). Wrap in a subquery / CTE:

```sql
-- doesn't work:
SELECT * FROM orders WHERE ROW_NUMBER() OVER (...) = 1;
-- works:
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY placed_at DESC) rn
  FROM orders
) t WHERE rn = 1;
```

PG 16 added `QUALIFY`-like ability? — actually no, PG still requires the subquery. (BigQuery/Snowflake have `QUALIFY`.)

## DISTINCT ON — Postgres-only shortcut

```sql
-- latest order per customer
SELECT DISTINCT ON (customer_id) *
FROM orders
ORDER BY customer_id, placed_at DESC;
```

Equivalent to a ROW_NUMBER = 1 trick but often faster because the planner can stream from a matching index.

## Performance notes

- Window functions need a sort unless an index satisfies `PARTITION BY` + `ORDER BY`.
- A composite index `(customer_id, placed_at DESC)` can power a windowed top-N efficiently.
- `work_mem` affects in-memory sort; large windows can spill to disk.

## Interview Questions

**Q1. Difference between RANK, DENSE_RANK, and ROW_NUMBER?**
Ties: `RANK` skips (1,2,2,4); `DENSE_RANK` doesn't (1,2,2,3); `ROW_NUMBER` is arbitrary among ties unless the ORDER BY is total.

**Q2. Compute a 7-day moving average of daily sales.**
```sql
SELECT day, total,
  AVG(total) OVER (ORDER BY day
                   ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM daily_sales;
```

**Q3. Why does `LAST_VALUE(...)` return the current row?**
Because the default frame is `RANGE UNBOUNDED PRECEDING ... CURRENT ROW`. Explicitly extend the frame to `UNBOUNDED FOLLOWING`.

**Q4. Can you reference a window column in WHERE?**
No. Wrap the SELECT in a subquery / CTE and filter outside, because window functions run after WHERE.

**Q5. Top-N per group — three ways?**
`DISTINCT ON`, `ROW_NUMBER OVER (PARTITION BY ...) <= N`, or `LATERAL` subquery. DISTINCT ON for N=1, LATERAL for index-friendly streaming, ROW_NUMBER when you want flexibility.

**Q6. Detect "logged in then bought within 30 minutes".**
```sql
SELECT *,
  LEAD(event_at) OVER (PARTITION BY user_id ORDER BY event_at) - event_at AS gap
FROM events;
```
then filter where event = 'login', next event = 'purchase', and gap < interval '30 min'.

## Common Pitfalls

- Forgetting that `LAST_VALUE` needs an extended frame.
- Trying to filter window output in `WHERE`.
- Re-running the same window subquery instead of using `WINDOW w AS (...)`.
- Sorting without an appropriate index, causing memory pressure.
- Confusing `RANGE` and `ROWS` semantics around tied ordering keys.

## See also

- [Subqueries & CTEs](04-subqueries-and-ctes.md)
- [Indexes](06-indexes.md)
- [Query planner](07-query-planner-and-explain.md)
