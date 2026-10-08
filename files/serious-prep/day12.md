# Day 12 — RxJS, Error Handling, JPQL, and Sliding Windows

> **Today's goal:** see how Angular treats async data as streams, how Node apps handle errors without crashing, how Spring writes JPA queries by hand, and the classic sliding-window technique in DSA.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | RxJS Observables (subscribe & unsubscribe) | 15m |
| 2 | Node.js | Error Handling (sync vs async, central handler) | 15m |
| 3 | Spring Boot | JPQL & Criteria API | 15m |
| 4 | MongoDB | Performance Tuning (explain(), profiler) | 10m |
| 5 | Postgres | JSONB & Arrays | 10m |
| 6 | HLD | URL Shortener Design | 12m |
| 7 | LLD | Chess game design | 12m |
| 8 | DSA | Sliding Window (Longest Substring Without Repeating Characters) | 15m |
| 9 | Design Pattern | Flyweight | 8m |
| 10 | DevOps | GitHub Actions | 10m |

---

## 1. Angular — RxJS Observables (subscribe & unsubscribe)

### Why this exists
A `Promise` resolves once. But many async sources emit many values over time: WebSocket messages, mouse moves, HTTP retries, form input. RxJS calls each of those a **Stream**, and the `Observable` is the type that represents one.

### The idea in plain English
An Observable is a recipe for producing values — nothing happens until you `.subscribe(...)`. When you do, the producer starts (HTTP request fires, interval ticks, socket opens). When you `.unsubscribe()` or the stream completes, the producer **tears down** (timer cleared, socket closed).

Forgetting to unsubscribe = a memory leak. Most Angular components either let `| async` pipe handle subscriptions for them, or store a `Subscription` and unsubscribe in `ngOnDestroy`.

Analogy: an Observable is a newspaper subscription. You sign up (`subscribe`), papers arrive over time, and you cancel (`unsubscribe`) when you're done. The printing press only runs because someone subscribed.

### Smallest working example
```typescript
import { interval, Subscription } from 'rxjs';

@Component({ selector: 'app-clock', template: '{{ tick }}' })
export class ClockComponent implements OnInit, OnDestroy {
  tick = 0;
  private sub?: Subscription;

  ngOnInit() {
    this.sub = interval(1000).subscribe(v => this.tick = v);
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();                  // clean up — no leak
  }
}
```

Even better: in the template, write `{{ tick$ | async }}` — Angular subscribes and unsubscribes for you.

### Drill (2 min)
You subscribe to `interval(1000)` and never unsubscribe. What happens when the component is destroyed? *(Hint: the interval keeps firing forever — the closure holds a reference to the (now-destroyed) component, leaking memory and CPU. Always unsubscribe or use `| async`.)*

**Deep dive (later):** [angular/rxjs-observables.md](../angular/rxjs-observables.md)

---

## 2. Node.js — Error Handling (sync vs async, central handler)

### Why this exists
A Node server is one process serving many users. An unhandled error can crash the whole process — and with it, every in-flight request. Good error handling: catch errors close to the source, normalize them, and respond with a clean status code.

### The idea in plain English
Two error worlds in Node:
- **Sync errors** — `throw new Error(...)` inside synchronous code. Caught by `try/catch`.
- **Async errors** — rejected Promises or `next(err)` in callbacks. `try/catch` doesn't catch them unless you `await`.

In Express, the convention is:
1. In each route, wrap async work in `try/catch` (or use `express-async-handler`).
2. Forward errors with `next(err)`.
3. Have ONE error-handling middleware at the bottom that converts errors to JSON responses with the right status code.

### Smallest working example
```javascript
class AppError extends Error {
  constructor(status, message) { super(message); this.status = status; }
}

app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await db.users.find(req.params.id);
    if (!user) throw new AppError(404, 'User not found');
    res.json(user);
  } catch (err) { next(err); }                    // forward
});

// Central handler — runs on next(err) from anywhere
app.use((err, req, res, next) => {
  const status = err.status || 500;
  res.status(status).json({ error: err.message });
});

// Last-resort: catch unhandled rejections so the process doesn't crash silently
process.on('unhandledRejection', reason => {
  console.error('Unhandled rejection:', reason);
});
```

The custom `AppError` carries the HTTP status, so the handler doesn't need to guess.

### Drill (2 min)
A route does `await db.find(id)` outside a `try/catch`. The promise rejects. What happens? *(Hint: Express 5 forwards it to the error handler automatically. Express 4 needs explicit try/catch (or `express-async-handler`) — otherwise the request hangs.)*

**Deep dive (later):** [nodejs/12-error-handling.md](../nodejs/12-error-handling.md)

---

## 3. Spring Boot — JPQL & Criteria API

### Why this exists
Derived queries (`findByEmail`) are great until you need a JOIN or a filter that depends on which params the caller passed. **JPQL (Java Persistence Query Language)** lets you write SQL-like queries in terms of entities, not tables. **Criteria API** builds queries programmatically — useful when the query shape depends on runtime conditions.

### The idea in plain English
JPQL looks like SQL but operates on **entity classes and fields**, not tables and columns. `SELECT u FROM User u WHERE u.email = :email` — note `User` (the class) and `u.email` (the field). JPA translates this to the right SQL for your database.

Two ways to use it:
- **`@Query` on a repository method** — fixed JPQL string, parameters via `:name`.
- **`@Query` with `JOIN FETCH`** — load related entities in one query (avoids the N+1 problem and `LazyInitializationException`).

Criteria API is verbose but type-safe and dynamic — pick it when filters are optional and built at runtime.

### Smallest working example
```java
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> byEmail(@Param("email") String email);

    // JOIN FETCH loads orders in the same query — no lazy exception
    @Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
    Optional<User> withOrders(@Param("id") Long id);

    // Pageable + sort, parametrized
    @Query("SELECT u FROM User u WHERE u.role = :role")
    Page<User> byRole(@Param("role") String role, Pageable page);
}
```

`u` is a "JPQL alias" — like `FROM users AS u` in SQL.

### Drill (2 min)
A repo loads 1 user; their orders are `LAZY`. The service iterates each order's items in a loop and runs 50 extra SELECTs. What's this called and how do you fix it? *(Hint: the N+1 problem. Fix with a `JOIN FETCH` query that pulls users + orders + items in one SQL.)*

**Deep dive (later):** [spring-next/05-spring-data-jpa.md § 6. JPQL and Native Queries](../spring-next/05-spring-data-jpa.md#6-jpql-and-native-queries) (see its Specifications subsection for Criteria-API-based dynamic queries)

---

## 4. MongoDB — Performance Tuning (explain(), profiler)

### Why this exists
A slow query is often not the database's fault — it's a missing index or a query that scans the whole collection. Mongo gives you `explain()` to see exactly how it ran a query and the **profiler** to log slow ops.

### The idea in plain English
- `explain('executionStats')` — runs the query and shows the plan: which index was used (or none — a *COLLSCAN*), how many docs examined, how long it took.
- **Database profiler** — set `db.setProfilingLevel(1, { slowms: 100 })` to log every operation slower than 100ms into `system.profile`.
- Aim: every common query reads only as many documents as it returns (or close to it). If `docsExamined >> nReturned`, you're missing an index.

### Smallest working example
```javascript
// Check the plan
db.orders.find({ userId: 'u123', status: 'open' }).explain('executionStats');
// Look at: 'winningPlan.stage' (IXSCAN good, COLLSCAN bad),
//          'executionStats.totalDocsExamined',
//          'executionStats.executionTimeMillis'

// Add an index to fix a COLLSCAN
db.orders.createIndex({ userId: 1, status: 1 });

// Turn on profiler for the session
db.setProfilingLevel(1, { slowms: 100 });
db.system.profile.find({ millis: { $gt: 100 } }).sort({ ts: -1 }).limit(5);
```

Compound index field order matters — put equality fields first, then range/sort fields (the **ESR rule**: Equality, Sort, Range).

### Drill (2 min)
`explain()` shows `totalDocsExamined: 1,000,000` and `nReturned: 5`. Healthy? *(Hint: very unhealthy. Mongo scanned a million docs to return 5. Add an index that matches the filter; ratio should approach 1:1.)*

**Deep dive (later):** [mongodb/12-performance-tuning.md](../mongodb/12-performance-tuning.md)

---

## 5. Postgres — JSONB & Arrays

### Why this exists
Sometimes you need flexible, schema-less fields inside an otherwise relational database — feature flags, user preferences, document content. Postgres's **JSONB** column stores binary JSON and lets you query into it with index support. Plus, arrays are first-class column types.

### The idea in plain English
- `JSONB` — stores JSON in a binary form. Query operators:
  - `->` returns JSON: `data->'name'`
  - `->>` returns text: `data->>'name'`
  - `@>` "contains" — `data @> '{"role":"admin"}'` true if `data` has all those keys/values
  - `?` "has key" — `data ? 'email'`
- **Array columns** — `text[]`, `int[]`. Use `ANY(arr)` to query.
- A **GIN index** on JSONB or arrays makes containment queries fast.

### Smallest working example
```sql
CREATE TABLE users (
  id    BIGSERIAL PRIMARY KEY,
  email TEXT,
  tags  TEXT[],                          -- ['vip','beta']
  data  JSONB                            -- { role: 'admin', prefs: {...} }
);

INSERT INTO users (email, tags, data) VALUES
  ('a@x.com', ARRAY['vip','beta'], '{"role":"admin"}'),
  ('b@x.com', ARRAY['user'],       '{"role":"user"}');

-- Query JSONB
SELECT email FROM users WHERE data @> '{"role":"admin"}';

-- Query array
SELECT email FROM users WHERE 'vip' = ANY(tags);

-- Indexes for both
CREATE INDEX users_data_gin ON users USING gin (data);
CREATE INDEX users_tags_gin ON users USING gin (tags);
```

JSONB is great for sparse / nested data, but heavy queries are still faster on real columns. Promote a JSON key to a column when you query it often.

### Drill (2 min)
You query `WHERE data->>'role' = 'admin'` and it's slow. Why doesn't your GIN index help? *(Hint: GIN on JSONB accelerates containment (`@>`), not text equality (`->>`). Either query with `data @> '{"role":"admin"}'`, or create an expression index: `CREATE INDEX ... ON users ((data->>'role'))`.)*

**Deep dive (later):** [postgres/13-jsonb-and-arrays.md](../postgres/13-jsonb-and-arrays.md)

---

## 6. HLD — URL Shortener Design

### Why this exists
A "design a URL shortener" question (bit.ly, tinyurl) packs every important HLD topic into 30 minutes: API, key generation, storage choice, scale, caching.

### The idea in plain English
Two endpoints:
1. `POST /shorten` body `{ url }` → returns `{ short }`.
2. `GET /:short` → redirects to the original URL (HTTP 301 or 302).

Three big design choices:
- **How to make the short code?**
  - **Counter + base62** — auto-increment a global counter (1, 2, 3...), encode as base62 (`1`, `2`, ... `B`, ..., `Z2k9`). 7 chars = 62^7 ≈ 3.5 trillion codes. Predictable; needs a distributed ID generator (Snowflake-style).
  - **Hash the URL** — first 7 chars of base62(MD5(url)). Risk of collisions; need to retry or detect.
  - **Pre-generate keys** in batches and hand them out — simple and fast.
- **DB?** Key-value (`short → url`) is the perfect fit. Redis (cache) + a durable store (Postgres / DynamoDB / Cassandra).
- **Caching?** Yes — redirects are read-heavy with a long tail. Cache hot URLs in Redis with TTL.

### A picture
```
POST /shorten ──► generate short code (counter+base62 or hash)
                 INSERT (short, long_url) into Postgres
                 SET   short → long_url in Redis
                 return short

GET /:short ──►  GET short from Redis  (cache hit → redirect)
                 else SELECT from Postgres + populate cache
                 issue 301/302
```

### Drill (2 min)
What status code for the redirect: 301 or 302? *(Hint: 301 (permanent) lets browsers and search engines cache the redirect — fewer hits to your server, but you can't change the destination later. 302 (temporary) keeps you in control. Use 302 unless you're sure mappings never change.)*

**Deep dive (later):** [system-design/high-level-design/10-real-world-system-designs.md](../system-design/high-level-design/10-real-world-system-designs.md)

---

## 7. LLD — Chess game design

### Why this exists
Chess LLD tests polymorphism beautifully — every piece type has the same interface (`canMove(from, to, board)`) but different rules.

### The idea in plain English
Core classes:
- **Board** — 8×8 grid of `Square`s; each square holds a `Piece` or null.
- **Piece** — abstract; subclasses `King`, `Queen`, `Rook`, `Bishop`, `Knight`, `Pawn`. Each implements `canMove(from, to, board)`.
- **Move** — `from`, `to`, optional `capturedPiece`, special flags (castling, en passant, promotion).
- **Game** — manages turns, runs moves through validation, checks for check/checkmate.

Why subclasses? Each piece's move rule is different. A `Bishop` moves diagonally; a `Knight` jumps in an L. Polymorphism keeps the rule next to the piece.

### Smallest working example (skeleton)
```java
abstract class Piece {
    final Color color;
    Piece(Color c) { this.color = c; }
    abstract boolean canMove(Position from, Position to, Board board);
}

class Knight extends Piece {
    Knight(Color c) { super(c); }
    public boolean canMove(Position f, Position t, Board b) {
        int dr = Math.abs(f.r - t.r), dc = Math.abs(f.c - t.c);
        return (dr == 2 && dc == 1) || (dr == 1 && dc == 2);
    }
}

class Bishop extends Piece {
    Bishop(Color c) { super(c); }
    public boolean canMove(Position f, Position t, Board b) {
        int dr = Math.abs(f.r - t.r), dc = Math.abs(f.c - t.c);
        if (dr != dc) return false;
        return b.isPathClear(f, t);                // helper on Board
    }
}
```

The `Game` only calls `piece.canMove(...)`; it doesn't care what kind of piece it is.

### Drill (2 min)
Where should "is this move putting MY king in check?" live — `Piece.canMove`, `Board`, or `Game`? *(Hint: `Game` (or a `MoveValidator` it owns). `Piece.canMove` should only know how *that piece* moves; check-detection requires looking at the whole board after the hypothetical move.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Sliding Window: Longest Substring Without Repeating Characters

### The problem
Given a string `s`, return the length of the longest substring with no repeated characters.

```
Input:  "abcabcbb"
Output: 3   // "abc"

Input:  "bbbbb"
Output: 1   // "b"
```

### The idea
The naive way: check every substring — O(n²) or worse. The **sliding window** way: keep a window `[left, right]` that always has unique characters.

- Expand `right` one step at a time, adding the new char.
- If the new char is already in the window, shrink `left` until it isn't.
- Track the max window size seen.

Each pointer moves at most n times → **O(n)** total.

### Smallest working example
```javascript
function lengthOfLongestSubstring(s) {
  const seen = new Set();
  let left = 0, best = 0;

  for (let right = 0; right < s.length; right++) {
    while (seen.has(s[right])) {
      seen.delete(s[left]);
      left++;
    }
    seen.add(s[right]);
    best = Math.max(best, right - left + 1);
  }
  return best;
}

lengthOfLongestSubstring("abcabcbb");   // 3
```

**Time:** O(n). **Space:** O(min(n, alphabet)).

### Drill (3 min)
Trace the window for `"abba"`. *(You should see: right=0 'a' → set={a}, best=1; right=1 'b' → set={a,b}, best=2; right=2 'b' → duplicate, shrink: delete 'a', delete 'b', left=2; set={b}, best=2; right=3 'a' → set={b,a}, best=2.)*

**Deep dive (later):** [dsa/ds-js/sliding-window.md](../dsa/ds-js/sliding-window.md) · [dsa/ds-java/sliding-window.md](../dsa/ds-java/sliding-window.md)

---

## 9. Design Pattern — Flyweight

### Intent
**Share lots of fine-grained objects to save memory.** When millions of objects are nearly identical, store the shared part once and just reference it.

### When you'd use it
- Text rendering — every glyph "A" on screen shares one Glyph object; only position differs.
- Game characters — 10,000 trees share one Tree mesh + texture; only x/y vary.
- Chess pieces — there are only 12 *kinds* (6 types × 2 colors), but 32 piece instances on the board.

### Smallest working example
```java
// Shared (intrinsic) state
class TreeType {
    final String name; final Color color; final String texture;
    TreeType(String n, Color c, String t) { name=n; color=c; texture=t; }
}

class TreeTypeFactory {
    private static final Map<String, TreeType> POOL = new HashMap<>();
    public static TreeType get(String name, Color color, String tex) {
        String key = name + color + tex;
        return POOL.computeIfAbsent(key, k -> new TreeType(name, color, tex));
    }
}

// Per-instance (extrinsic) state — kept by the caller, not by the flyweight
class Tree {
    final int x, y;
    final TreeType type;
    Tree(int x, int y, TreeType type) { this.x = x; this.y = y; this.type = type; }
}
```

10,000 trees, 5 types → 5 `TreeType` objects in memory, not 10,000.

### Drill (1 min)
Why does the `Tree` instance hold position but not texture? *(Hint: position is **extrinsic** — different per tree. Texture is **intrinsic** — shared. The whole point is to push extrinsic data out and reuse the intrinsic.)*

**Deep dive (later):** [design-patterns/common/structural/flyweight.md](../design-patterns/common/structural/flyweight.md)

---

## 10. DevOps — GitHub Actions

### Why this exists
GitHub Actions is GitHub's built-in CI/CD. You commit a YAML file in `.github/workflows/`, and GitHub runs it on every push, PR, or schedule. No separate Jenkins/CircleCI to set up.

### The idea in plain English
A **workflow** is the YAML file. It has **jobs**, each runs on a **runner** (a Linux/Mac/Windows VM). Each job has **steps** — either shell commands or pre-built **actions** (`actions/checkout`, `actions/setup-node`). Steps share a workspace within a job; different jobs run in separate VMs (use **artifacts** to pass files between them).

Analogy: GitHub Actions is "when *event* happens, do *these things*." `on: push` is the trigger; the steps are the chores.

### Smallest working example
```yaml
# .github/workflows/ci.yml
name: CI
on:
  push: { branches: [main] }
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm test
      - run: npm run build

  deploy:
    needs: test                            # only if test passes
    if: github.ref == 'refs/heads/main'    # only on main
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}    # from repo Secrets
```

Secrets are stored in GitHub repo settings — they're injected as env vars, never logged.

### Drill (2 min)
You want your tests to run on Node 18, 20, and 22 in parallel. What feature do you reach for? *(Hint: a **matrix** — `strategy: matrix: { node: [18, 20, 22] }` plus `with: { node-version: ${{ matrix.node }} }`. GitHub spawns 3 jobs in parallel.)*

**Deep dive (later):** [devops/11-github-actions.md](../devops/11-github-actions.md)

---

## End-of-day checklist

- [ ] Angular: I can subscribe to an Observable and explain why I must unsubscribe
- [ ] Node.js: I can write a custom `AppError` class and one central error handler
- [ ] Spring: I can write a `@Query` with JPQL and explain `JOIN FETCH`
- [ ] MongoDB: I can read `explain('executionStats')` output and spot a COLLSCAN
- [ ] Postgres: I know `@>`, `->>`, and when GIN helps vs doesn't
- [ ] HLD: I can sketch a URL shortener with caching and pick a short-code strategy
- [ ] LLD: I can explain why every chess piece needs its own `canMove` implementation
- [ ] DSA: I solved "longest substring without repeating characters" with O(n)
- [ ] DP: I can describe Flyweight with the tree-in-a-game example
- [ ] DevOps: I can write a small GitHub Actions workflow with secrets and a deploy gate

**If you remember just one thing today:** **streams beat single values.** RxJS Observables, JSONB GIN indexes, sliding windows, GitHub Actions matrix jobs — all win by treating data as a flow you can transform.

**Tomorrow:** RxJS flattening operators, REST API design, Hibernate caching, full-text search, rate limiters, and the Strategy pattern.
