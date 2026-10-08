# Day 16 — Reactive UIs, secure APIs, smart indexes

> **Today's goal:** meet Angular Signals, plug Node into a database safely, peek at Spring Security's filter chain, and learn why Postgres needs to "vacuum."
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Signals (v17+: signal, computed, effect) | 15m |
| 2 | Node.js | Database Integration (Mongoose, Prisma basics) | 15m |
| 3 | Spring Boot | Spring Security basics (filter chain) | 15m |
| 4 | MongoDB | Change Streams & Time Series collections | 10m |
| 5 | Postgres | Vacuum & Bloat (why vacuum exists) | 10m |
| 6 | HLD | Search Engine Design (inverted index, ranking) | 12m |
| 7 | LLD | Snake Game | 12m |
| 8 | DSA | Dynamic Programming intro (Fibonacci, memoization) | 15m |
| 9 | Design Pattern | Iterator | 8m |
| 10 | DevOps | Monitoring & Observability (Prometheus + Grafana, USE/RED) | 10m |

---

## 1. Angular — Signals

### Why this exists
RxJS is powerful but heavy for "simple state" (a counter, a toggle). Signals (added in Angular 17) give you a tiny, synchronous, fine-grained reactive primitive. Less ceremony, faster change detection.

### The idea in plain English
A **signal** is a smart variable that tells the UI when to redraw. You read it with `mySignal()` (call it like a function). When you write to it, Angular knows exactly which views depend on it and re-renders only those.

Three primitives:
- **`signal(value)`** — a reactive cell holding a value
- **`computed(() => ...)`** — derived value that auto-updates when its inputs change
- **`effect(() => ...)`** — side effect that runs when its inputs change (e.g., log, save to localStorage)

**Analogy:** a spreadsheet. Cells hold values (`signal`). Other cells have formulas that depend on them (`computed`). When you change A1, B1's formula recalculates automatically — and your chart redraws (`effect`).

### Smallest working example
```typescript
import { Component, signal, computed, effect } from '@angular/core';

@Component({
  standalone: true,
  template: `
    <button (click)="count.set(count() + 1)">+</button>
    <p>count: {{ count() }}, double: {{ double() }}</p>
  `,
})
export class CounterComponent {
  count  = signal(0);
  double = computed(() => this.count() * 2);

  constructor() {
    effect(() => console.log('count is now', this.count()));
  }
}
```

`double()` recomputes only when `count()` changes. The `effect` logs only on changes — no manual `subscribe`/`unsubscribe`.

### Drill (2 min)
Why call `this.count()` (with parentheses) instead of `this.count`? *(Hint: a signal is a function. Calling it both reads the current value and registers a dependency so the framework knows this code depends on the signal.)*

**Deep dive (later):** [angular/signals.md](../angular/signals.md)

---

## 2. Node.js — Database Integration

### Why this exists
You can write raw SQL or raw Mongo queries everywhere, but it gets tangled fast. Drivers and ORMs give you typed queries, connection pooling, and migrations.

### The idea in plain English
Two popular tools to know:
- **Mongoose** (for MongoDB) — define a **Schema** for documents, get a **Model** with typed CRUD and validation built-in.
- **Prisma** (for SQL DBs: Postgres, MySQL) — define your schema in `schema.prisma`, run `prisma generate`, get a fully typed client.

**Analogy:** raw driver = writing SQL by hand each time. ORM = a librarian who knows your shelves and hands you exactly the right book when you describe it.

### Smallest working example
```javascript
// --- Mongoose ---
const mongoose = require('mongoose');
const User = mongoose.model('User', new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  age:   { type: Number, min: 18 },
}));
await mongoose.connect(process.env.MONGO_URL);
await User.create({ email: 'k@x.com', age: 25 });

// --- Prisma ---
// In schema.prisma:
//   model User { id Int @id @default(autoincrement()) email String @unique age Int }
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();
await prisma.user.create({ data: { email: 'k@x.com', age: 25 } });
```

Both are async, both pool connections, both validate types. Pick by which DB you use.

### Drill (2 min)
Why is it bad to call `mongoose.connect()` inside every request handler? *(Hint: it would open a new connection per request and leak. Connect once at startup; the driver pools and reuses connections across requests.)*

**Deep dive (later):** [nodejs/16-database-integration.md](../nodejs/16-database-integration.md)

---

## 3. Spring Boot — Spring Security basics

### Why this exists
You don't want to write auth from scratch. Spring Security handles login, password hashing, session management, CSRF, CORS, and a dozen attacks you haven't heard of yet — all by configuring a **filter chain**.

### The idea in plain English
When a request hits your Spring app, it doesn't go straight to your controller. It walks through a **chain of filters**, each one a step at airport security:

```
Request → [check token] → [check authorization] → [check CSRF] → ... → Controller
```

Each filter can let the request through, modify it, or short-circuit with `401/403`. You configure which filters apply to which paths.

**Analogy:** airport security lanes. ID check first, then bag X-ray, then boarding pass scan. Fail any check → you don't board.

### Smallest working example
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())   // simple username/password
            .csrf(csrf -> csrf.disable());          // disable for stateless APIs
        return http.build();
    }
}
```

- `/public/**` is open
- `/admin/**` needs the `ADMIN` role
- Everything else needs a logged-in user

### Drill (2 min)
You see `403 Forbidden` for `/admin/users` even though you logged in successfully. Likely cause? *(Hint: you're authenticated but not in the `ADMIN` role. `authenticated()` passes; `hasRole("ADMIN")` fails → 403, not 401.)*

**Deep dive (later):** [spring-next/06-spring-security.md](../spring-next/06-spring-security.md)

---

## 4. MongoDB — Change Streams & Time Series

### Why this exists
Two modern features that solve real problems: **change streams** let you react to writes in real time without polling; **time series collections** store IoT/metrics data far more efficiently.

### The idea in plain English
- **Change stream** — Mongo's "tail the oplog" feature with a clean API. You subscribe to a collection and get a stream of insert/update/delete events. Great for cache invalidation, search indexing, fan-out to other services.
- **Time series collections** — a new collection type optimized for `{timestamp, value, metadata}` records. Internally Mongo packs them tightly and creates clustered indexes by time. Up to 70% disk savings.

**Analogy:** change stream is a subscription to the front desk's intercom — you hear about every move in the building. Time series is a specialized box for stamps — it stacks them better than a generic drawer.

### Smallest working example
```javascript
// Change stream — listen for new orders
const stream = db.orders.watch([{ $match: { operationType: 'insert' } }]);
stream.on('change', (event) => {
  console.log('new order:', event.fullDocument);
  // forward to search indexer, cache invalidator, etc.
});

// Time series collection
db.createCollection('sensorReadings', {
  timeseries: {
    timeField: 'ts',
    metaField: 'sensorId',
    granularity: 'minutes'
  }
});
db.sensorReadings.insertOne({ ts: new Date(), sensorId: 'A1', value: 23.4 });
```

### Drill (2 min)
You need to push a row to Elasticsearch every time an order is inserted. Polling vs change stream? *(Hint: change stream — push, not pull. No wasted polls, sub-second latency, and Mongo handles resume tokens for replay after a crash.)*

**Deep dive (later):** [mongodb/15-change-streams-and-time-series.md](../mongodb/15-change-streams-and-time-series.md)

---

## 5. Postgres — Vacuum & Bloat

### Why this exists
Postgres uses **MVCC** — Multi-Version Concurrency Control. When you UPDATE a row, Postgres doesn't overwrite it; it writes a new version and marks the old one as dead. Without cleanup, dead rows pile up. That's **bloat**. `VACUUM` is the cleanup.

### The idea in plain English
Imagine a notebook where you never erase — when you "edit" a line, you cross it out and write a new line below. After months, your notebook is half crossed-out garbage. Vacuum is the eraser pass that frees space (and resets a counter that prevents wraparound bugs).

Two flavors:
- **Autovacuum** — Postgres runs this automatically. Reuses dead row space; doesn't return space to OS.
- **`VACUUM FULL`** — rewrites the whole table; returns space to OS. Locks the table fully. Use rarely.

### Smallest working example
```sql
-- Cheap, online — reuses dead space inside table files
VACUUM users;

-- Also update statistics for the planner
VACUUM (ANALYZE) users;

-- Heavy: rewrites and locks the whole table
VACUUM FULL users;

-- See bloat per table
SELECT relname, n_dead_tup, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 10;
```

### Drill (2 min)
A table has 1M live rows and 9M dead rows. Queries got slow. Did you forget to vacuum? *(Hint: probably yes, or autovacuum is tuned too conservatively. `n_dead_tup` >> `n_live_tup` means the planner is reading mostly garbage. Run `VACUUM (ANALYZE)` and check autovacuum settings.)*

**Deep dive (later):** [postgres/17-vacuum-and-bloat.md](../postgres/17-vacuum-and-bloat.md)

---

## 6. High-Level Design — Search Engine

### Why this exists
Search-by-keyword (Google, Elasticsearch) cannot use a SQL `LIKE '%foo%'` — that's a full table scan. Search engines use an **inverted index** to make this fast.

### The idea in plain English
A normal index goes "row → values." An **inverted index** goes "word → list of documents that contain it." That's the back-of-book index in a textbook.

**The pipeline:**
1. **Tokenize** — break "The Quick brown fox" into `[the, quick, brown, fox]`.
2. **Normalize** — lowercase, strip punctuation, optionally stem (`running → run`).
3. **Index** — for each token, append the document id.
4. **Query** — tokenize the query the same way, look up the postings lists, intersect.
5. **Rank** — score matching documents (TF-IDF, BM25, recency, popularity) and return top N.

**Analogy:** the index at the back of a book lets you jump straight to "monad → pages 42, 87, 153" instead of scanning all 400 pages.

### A minimal sketch
```
"the quick brown fox"  →  tokenize  →  [the, quick, brown, fox]

inverted index after indexing two docs:
  the   → [1, 2]
  quick → [1]
  brown → [1, 2]
  fox   → [1]
  lazy  → [2]
  dog   → [2]

Query "brown fox":
  postings(brown) ∩ postings(fox) = {1}
  rank → doc 1
```

In real systems: sharding by document id, replicas for read scale, async indexing pipeline so writes don't block.

### Drill (3 min)
Searching for "brown FOX" matches doc 1 ("the quick brown fox") in our index. Why? *(Hint: both indexing and querying normalize to lowercase. The same transform on both sides makes case-insensitive matching automatic.)*

**Deep dive (later):** [system-design/high-level-design/14-search-engine.md](../system-design/high-level-design/14-search-engine.md)

---

## 7. LLD — Snake Game

### Why this exists
Snake is the perfect "tick-based simulation" LLD — clean state machine, simple rules, easy to extend (powerups, walls, multiplayer).

### Core entities
- **Cell** — coordinate (x, y) on the grid
- **Snake** — ordered list of `Cell`s; head moves forward, tail follows (unless food eaten)
- **Food** — a random `Cell` not occupied by the snake
- **Board** — width/height, knows what's where
- **Game** — has a state (`RUNNING`, `OVER`), tick loop, direction input

### A clean class sketch
```java
enum Direction { UP, DOWN, LEFT, RIGHT }

class Snake {
    Deque<Cell> body = new ArrayDeque<>();
    Direction dir = Direction.RIGHT;

    Cell nextHead() {
        Cell h = body.peekFirst();
        return switch (dir) {
            case UP    -> new Cell(h.x, h.y - 1);
            case DOWN  -> new Cell(h.x, h.y + 1);
            case LEFT  -> new Cell(h.x - 1, h.y);
            case RIGHT -> new Cell(h.x + 1, h.y);
        };
    }
}

class Game {
    void tick() {
        Cell next = snake.nextHead();
        if (board.outOfBounds(next) || snake.body.contains(next)) { state = OVER; return; }
        snake.body.addFirst(next);
        if (next.equals(food.cell)) food.respawn(board, snake); else snake.body.pollLast();
    }
}
```

`Deque` is perfect — `addFirst` for head growth, `pollLast` for tail trim. `contains` on a `Deque` is O(n); for huge snakes, also keep a `HashSet<Cell>` of occupied cells.

### Drill (3 min)
The snake is moving RIGHT and the user presses LEFT. What should happen? *(Hint: ignore it. A 180° turn would make the snake collide with itself on the next tick. Most implementations reject opposite-direction inputs.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Dynamic Programming intro

### The problem
Compute the nth Fibonacci number. `fib(n) = fib(n-1) + fib(n-2)`.

### The naive recursion
```javascript
function fib(n) {
  if (n < 2) return n;
  return fib(n - 1) + fib(n - 2);
}
```

This re-computes the same subproblems thousands of times. `fib(40)` takes seconds. Time: **O(2^n)** — exponential.

### The fix: memoization (top-down DP)
Cache each result the first time. Next time you need it, return the cached answer.

```javascript
function fib(n, memo = new Map()) {
  if (n < 2) return n;
  if (memo.has(n)) return memo.get(n);
  const result = fib(n - 1, memo) + fib(n - 2, memo);
  memo.set(n, result);
  return result;
}
```

**Time:** O(n). **Space:** O(n).

### The bottom-up version (tabulation)
Build up from `fib(0)` and `fib(1)` to `fib(n)`. Same idea, no recursion.

```javascript
function fib(n) {
  if (n < 2) return n;
  let a = 0, b = 1;
  for (let i = 2; i <= n; i++) [a, b] = [b, a + b];
  return b;
}
// O(n) time, O(1) space
```

### Why DP works (the big idea)
Two ingredients:
1. **Overlapping subproblems** — the same sub-question shows up many times (fib(3) is computed many times in the naive version).
2. **Optimal substructure** — the answer to the big problem is built from answers to smaller problems.

Memoize subproblems → trade memory for time.

### Drill (3 min)
For `fib(5)` with memoization, how many times do we compute `fib(2)`? *(Hint: exactly once. Every later call hits the cache. The naive version computes it 3+ times.)*

**Deep dive (later):** [dsa/ds-js/dynamic-programming.md](../dsa/ds-js/dynamic-programming.md) · [dsa/ds-java/dynamic-programming.md](../dsa/ds-java/dynamic-programming.md)

---

## 9. Design Pattern — Iterator

### Intent
**Give a uniform way to walk through a collection** without exposing how the collection is stored (array, tree, linked list, paged result set).

### When you'd use it
- `for (var x : collection)` in Java — that's an Iterator under the hood
- Database cursors — fetch rows one at a time without loading the whole result
- Lazy sequences — infinite streams (e.g., a stream of prime numbers)

### The simplest version (Java)
```java
class Range implements Iterable<Integer> {
    private final int start, end;
    Range(int start, int end) { this.start = start; this.end = end; }

    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            int i = start;
            public boolean hasNext() { return i < end; }
            public Integer next()    { return i++; }
        };
    }
}

// Use it — same syntax as any collection
for (int n : new Range(1, 5)) System.out.println(n);   // 1 2 3 4
```

The caller doesn't know `Range` doesn't actually store the numbers — it generates them on demand. That's the win: **the consumer doesn't depend on the storage.**

### Drill (1 min)
You're processing a query that returns 10 million rows. Should you load them all or use an iterator/cursor? *(Hint: iterator/cursor — loads one row at a time, so memory stays flat. Loading 10M rows at once could OOM your service.)*

**Deep dive (later):** [design-patterns/common/behavioral/iterator.md](../design-patterns/common/behavioral/iterator.md)

---

## 10. DevOps — Monitoring & Observability

### Why this exists
"Is it working?" is too vague. Monitoring gives you numbers; observability lets you ask new questions of those numbers. You can't fix what you can't see.

### The idea in plain English
Two acronyms to know — both are checklists for "what should I measure?":

**RED — for request-driven services (APIs, microservices)**
- **R**ate: requests per second
- **E**rrors: % of requests failing
- **D**uration: latency distribution (p50, p95, p99)

**USE — for resources (CPU, disk, queue)**
- **U**tilization: % busy
- **S**aturation: how much work is queued / waiting
- **E**rrors: hardware/system errors

**The standard stack:**
- **Prometheus** scrapes metrics from your app (`/metrics` endpoint exposes counters and gauges)
- **Grafana** dashboards and alerts on those metrics
- Apps export metrics via a client library (`prom-client` for Node, Micrometer for Spring)

**Analogy:** a car has a speedometer (rate), fuel gauge (utilization), and check-engine light (errors). Without them you'd be driving blind.

### Smallest working example (Node + prom-client)
```javascript
const promClient = require('prom-client');

const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'status'],
});

app.use((req, res, next) => {
  res.on('finish', () =>
    httpRequestsTotal.inc({ method: req.method, status: res.statusCode })
  );
  next();
});

app.get('/metrics', async (_, res) => {
  res.type(promClient.register.contentType);
  res.end(await promClient.register.metrics());
});
```

Prometheus scrapes `/metrics` every 15s. Grafana plots `rate(http_requests_total[1m])` for live RPS.

### Drill (2 min)
Your error rate is 0.5% but customer complaints are rolling in. What metric did you forget? *(Hint: duration / latency. p99 latency could have doubled while error rate stayed low — slow but successful responses still anger users. Always track RED, not just E.)*

**Deep dive (later):** [devops/18-monitoring-and-observability.md](../devops/18-monitoring-and-observability.md)

---

## End-of-day checklist

- [ ] Angular: I can explain `signal`, `computed`, `effect` and why you call them as functions
- [ ] Node.js: I can pick Mongoose vs Prisma based on the DB
- [ ] Spring: I can sketch a `SecurityFilterChain` with role-based path matchers
- [ ] MongoDB: I can describe a change stream and when to use a time series collection
- [ ] Postgres: I can explain what vacuum cleans up and why
- [ ] HLD: I can explain "inverted index" with the back-of-book analogy
- [ ] LLD: I can model Snake with a `Deque` and a tick loop
- [ ] DSA: I converted naive Fibonacci into memoized DP
- [ ] DP: I can give one real-world example of Iterator (cursors, streams)
- [ ] DevOps: I know RED vs USE and which is for what

**If you remember just one thing today:** every layer wins by **caching** work — Signals cache derivations, ORMs cache connections, vacuum cleans cache misses, DP caches subproblems, Prometheus caches time series.

**Tomorrow:** state management options, Redis caching, JWT in Spring, advanced aggregation, performance tuning, newsfeeds, banking, LIS/LCS, State pattern, distributed tracing.
