# Stored Procedures, Functions & Triggers

> TL;DR: PG supports **functions** (return values, can't manage transactions) and **procedures** (PG 11+; can `COMMIT`/`ROLLBACK`). Default language is PL/pgSQL; you can also use SQL, PL/Python, PL/V8. Triggers fire `BEFORE`/`AFTER` `INSERT`/`UPDATE`/`DELETE`/`TRUNCATE`, at row or statement level. Event triggers fire on DDL.

## Functions vs procedures

| | Function | Procedure |
|---|----------|-----------|
| Returns | Value(s) | No return (OUT parameters allowed since PG 14) |
| Invoke | `SELECT fn(...)` | `CALL proc(...)` |
| Can `COMMIT`/`ROLLBACK` inside | No | Yes (PG 11+) |
| Used in expressions | Yes | No |
| Common use | Helper logic, derived columns, triggers | Long-running jobs, batch ops |

## SQL functions (simple, fast)

```sql
CREATE OR REPLACE FUNCTION add(a int, b int)
RETURNS int
LANGUAGE sql
IMMUTABLE PARALLEL SAFE
AS $$
  SELECT a + b;
$$;

SELECT add(2,3);
```

The planner can inline simple SQL functions — they're as efficient as the underlying query.

## PL/pgSQL — the workhorse

```sql
CREATE OR REPLACE FUNCTION transfer(p_from bigint, p_to bigint, p_amount numeric)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
  v_balance numeric;
BEGIN
  IF p_amount <= 0 THEN
    RAISE EXCEPTION 'amount must be positive';
  END IF;

  SELECT balance INTO v_balance
  FROM accounts WHERE id = p_from FOR UPDATE;

  IF v_balance < p_amount THEN
    RAISE EXCEPTION 'insufficient funds';
  END IF;

  UPDATE accounts SET balance = balance - p_amount WHERE id = p_from;
  UPDATE accounts SET balance = balance + p_amount WHERE id = p_to;
END;
$$;

SELECT transfer(1, 2, 100);
```

### Control flow

```sql
IF cond THEN ... ELSIF cond THEN ... ELSE ... END IF;

WHILE cond LOOP
  ...
END LOOP;

FOR r IN SELECT id, name FROM users LOOP
  RAISE NOTICE '% %', r.id, r.name;
END LOOP;

FOR i IN 1..10 LOOP ... END LOOP;
```

### Exceptions

```sql
BEGIN
  ...
EXCEPTION
  WHEN unique_violation THEN
    RAISE NOTICE 'duplicate ignored';
  WHEN OTHERS THEN
    RAISE;             -- re-raise
END;
```

Wrapping in a `BEGIN ... EXCEPTION` block creates an implicit subtransaction (savepoint) — expensive in tight loops.

## Returning sets

```sql
CREATE FUNCTION top_customers(p_n int)
RETURNS TABLE(id bigint, total numeric)
LANGUAGE sql STABLE AS $$
  SELECT customer_id, SUM(amount)
  FROM orders GROUP BY customer_id
  ORDER BY 2 DESC LIMIT p_n;
$$;

SELECT * FROM top_customers(10);
```

## Volatility — IMMUTABLE / STABLE / VOLATILE

Tells the planner whether to call the function:

| Class | Promise | Examples |
|-------|---------|----------|
| IMMUTABLE | Same input -> same output, no DB reads | `lower()`, math |
| STABLE | Same input within a single statement | `now()`, lookups |
| VOLATILE (default) | Can do anything, including write | `random()`, INSERT |

This affects index usability — `WHERE created_at < now()` works because `now()` is stable. An `IMMUTABLE` expression index can index a function call.

## SECURITY DEFINER vs INVOKER

```sql
CREATE FUNCTION admin_op() RETURNS void
LANGUAGE plpgsql SECURITY DEFINER
SET search_path = pg_catalog, public
AS $$ ... $$;
```

`SECURITY DEFINER` runs with the **owner's** privileges (like sudo). Always pin `search_path` to avoid attacker-shadowed functions, and revoke `EXECUTE` from `PUBLIC` if you've granted it elsewhere.

## Procedures (PG 11+)

```sql
CREATE PROCEDURE archive_old_orders()
LANGUAGE plpgsql AS $$
DECLARE
  c CURSOR FOR SELECT id FROM orders WHERE placed_at < now() - interval '5 years';
BEGIN
  FOR row IN c LOOP
    DELETE FROM orders WHERE id = row.id;
    IF (c.rownumber % 1000) = 0 THEN
      COMMIT;     -- batches
    END IF;
  END LOOP;
END;
$$;

CALL archive_old_orders();
```

Cannot be called inside an existing transaction (no nested commit).

## Triggers

A trigger ties a function to a table event.

### Row-level audit trigger

```sql
CREATE TABLE orders_audit (
  id bigserial PRIMARY KEY,
  order_id bigint,
  changed_by text,
  changed_at timestamptz DEFAULT now(),
  before_data jsonb,
  after_data jsonb
);

CREATE OR REPLACE FUNCTION trg_orders_audit() RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  INSERT INTO orders_audit(order_id, changed_by, before_data, after_data)
  VALUES (COALESCE(NEW.id, OLD.id),
          current_user,
          to_jsonb(OLD),
          to_jsonb(NEW));
  RETURN NEW;
END;
$$;

CREATE TRIGGER orders_audit_trg
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION trg_orders_audit();
```

### BEFORE vs AFTER

- **BEFORE**: can modify `NEW` (e.g. fill `updated_at`), or `RETURN NULL` to skip the row entirely.
- **AFTER**: row is already written; useful for cascading inserts, audit, notifications.

### ROW vs STATEMENT

- `FOR EACH ROW` — once per affected row; access `OLD` / `NEW`.
- `FOR EACH STATEMENT` — once per statement; in PG 10+ can use **transition tables**:

```sql
CREATE TRIGGER orders_change_summary
AFTER UPDATE ON orders
REFERENCING OLD TABLE AS old NEW TABLE AS new
FOR EACH STATEMENT EXECUTE FUNCTION summarize_changes();
```

### Order of execution

If multiple triggers fire on the same event, they run **alphabetically by name**. Some teams prefix with numbers (`10_audit`, `20_notify`).

### When NOT to use triggers

- Complex business logic — hides behavior, hard to test/version.
- High-throughput inserts — triggers add per-row cost.
- Notifications that *must* succeed externally — use outbox table + cron, not trigger + NOTIFY.

## Event triggers — fire on DDL

```sql
CREATE OR REPLACE FUNCTION block_drops() RETURNS event_trigger AS $$
BEGIN
  IF current_user <> 'admin' THEN
    RAISE EXCEPTION 'no DROP for you';
  END IF;
END;
$$ LANGUAGE plpgsql;

CREATE EVENT TRIGGER guard_drops
ON sql_drop EXECUTE FUNCTION block_drops();
```

Useful for schema governance.

## NOTIFY / LISTEN

```sql
LISTEN orders_inserted;
-- somewhere else:
NOTIFY orders_inserted, 'order 42';
```

Asynchronous pub/sub inside one PG instance. Combine with triggers for cache invalidation. Payload limited to 8000 bytes.

## Interview Questions

**Q1. Function vs procedure?**
Functions return a value and can't `COMMIT` inside; called via SELECT. Procedures (PG 11+) can manage transactions and are invoked with CALL.

**Q2. What does `IMMUTABLE` mean and why does it matter?**
The function returns the same output for the same input forever. The planner can pre-evaluate constants, cache, and create indexes on expressions calling it.

**Q3. BEFORE vs AFTER trigger — when to use each?**
BEFORE for mutating NEW (filling defaults, validation that may cancel the row by `RETURN NULL`). AFTER for actions that need the row to exist (audit, cascading writes, NOTIFY).

**Q4. ROW vs STATEMENT trigger?**
ROW fires once per row affected and can access OLD/NEW; STATEMENT fires once per SQL statement. STATEMENT with transition tables is efficient for bulk operations.

**Q5. Risks of SECURITY DEFINER?**
Privilege escalation if the function executes user-controlled SQL or trusts mutable `search_path`. Always pin `search_path` and audit input handling.

**Q6. When are triggers a bad idea?**
For business logic — they hide behavior, complicate testing, and slow hot writes. Prefer application code or explicit jobs.

## Common Pitfalls

- `BEGIN EXCEPTION ... END` in tight loops — every block is a subtransaction.
- Forgetting to `RETURN NEW` in BEFORE trigger -> silently drops the row.
- Recursive triggers — A INSERT fires trigger that INSERTs into A again.
- Triggers performing slow external work synchronously (HTTP, network) -> all writes slow.
- `SECURITY DEFINER` without `SET search_path` -> hijack risk.
- Procedures that COMMIT mid-loop -> outer transaction expectations break.

## See also

- [JSONB & arrays](13-jsonb-and-arrays.md) for `to_jsonb` patterns
- [Spring data layer / JPA listeners](../spring-detailed/PART-08-databases-and-sql.md)
- [Locking](11-locking.md)
