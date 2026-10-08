# Day 17 — State, caches, JWT-in-Spring, and feeds

> **Today's goal:** survey Angular state options, add a Redis cache, secure Spring with JWT, peek at advanced Mongo aggregation, and sketch a newsfeed.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | State Management overview (services vs NgRx vs Signals) | 15m |
| 2 | Node.js | Caching with Redis (cache-aside) | 15m |
| 3 | Spring Boot | JWT Auth in Spring Boot | 15m |
| 4 | MongoDB | Aggregation Advanced ($lookup with pipeline, $facet) | 10m |
| 5 | Postgres | Performance Tuning (pg_stat_statements basics) | 10m |
| 6 | HLD | Newsfeed Design (fan-out on write vs read) | 12m |
| 7 | LLD | Online Banking System | 12m |
| 8 | DSA | DP: LIS / LCS | 15m |
| 9 | Design Pattern | State | 8m |
| 10 | DevOps | Distributed Tracing (OpenTelemetry, spans) | 10m |

---

## 1. Angular — State Management overview

### Why this exists
Components own their own state, but lots of things (current user, theme, cart) are needed by many components. How you share that state is **state management**. There are three common choices in Angular.

### The idea in plain English
Pick by how much state and how much complexity you have:

1. **Service with a Subject/Signal** — simplest. Inject the same service into many components; they read & write its state. Good up to ~20 pieces of shared state.
2. **Signals** — Angular's built-in primitive (Day 16). Reactive, synchronous, minimal boilerplate. Good for "feature-level" state.
3. **NgRx Store** — a Redux-style global store with **actions, reducers, selectors**. Strict, predictable, has dev tools. Worth the boilerplate only when you have lots of shared state, async flows, and a need for time-travel debugging.

**Analogy:** a service is a sticky note on the fridge; Signals are a smart whiteboard that auto-updates other rooms; NgRx is a corporate filing system with check-in/check-out audit logs.

### Smallest working example — service + Signal
```typescript
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class CartService {
  private items = signal<string[]>([]);
  readonly items$ = this.items.asReadonly();   // expose read-only

  add(item: string) { this.items.update(arr => [...arr, item]); }
  clear()           { this.items.set([]); }
}

// Any component:
constructor(public cart: CartService) {}
// template: <p>{{ cart.items$().length }} items</p>
```

This handles 80% of apps. Reach for NgRx only when complexity demands it.

### Drill (2 min)
A small app has a "current user" and a "theme." Should you start with NgRx? *(Hint: no — two pieces of state are fine in a service or signals. NgRx is heavy: actions + reducers + selectors + effects for each feature. Adopt it when your sticky-note system stops scaling.)*

**Deep dive (later):** [angular/state-management-comparison.md](../angular/state-management-comparison.md)

---

## 2. Node.js — Caching with Redis

### Why this exists
Some data is read 1000× more than it's written (user profiles, product catalogs). Hitting the DB every time wastes resources. A **cache** keeps a copy in memory; you check the cache first.

### The idea in plain English
**Cache-aside pattern** (the most common):
1. Read a key from the cache.
2. **Hit** — return the cached value.
3. **Miss** — load from DB, write to cache with a TTL (time-to-live), return.

**Writes:** update the DB, then delete (or update) the cache. Next reader will refill.

**Analogy:** the fridge is your cache, the supermarket is the DB. You check the fridge first. If empty, go to the store and restock. When food expires (TTL), throw it out.

**Redis** is the de-facto in-memory key-value store: sub-millisecond reads, supports strings/hashes/lists/sets/streams.

### Smallest working example
```javascript
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

async function getUser(id) {
  const key = `user:${id}`;

  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);              // HIT

  const user = await db.users.findOne({ _id: id });   // MISS
  await redis.set(key, JSON.stringify(user), 'EX', 300);  // 5 min TTL
  return user;
}

async function updateUser(id, patch) {
  await db.users.updateOne({ _id: id }, { $set: patch });
  await redis.del(`user:${id}`);                      // invalidate
}
```

`EX 300` is the TTL in seconds — even if you forget to invalidate, the cache won't be stale forever.

### Drill (2 min)
Why delete instead of "update" the cache on write? *(Hint: delete is simpler and safe. Updating means you race with concurrent updates — you might write stale data. Next read will refill from the DB with the freshest version.)*

**Deep dive (later):** [nodejs/17-caching-with-redis.md](../nodejs/17-caching-with-redis.md)

---

## 3. Spring Boot — JWT Auth in Spring Boot

### Why this exists
Day 14 covered JWT mechanics in Node. The Spring side is similar but lives inside the filter chain you saw on Day 16.

### The idea in plain English
You add a **custom filter** before the standard auth filter. It:
1. Reads `Authorization: Bearer <token>` from the incoming request.
2. Verifies the signature and expiry.
3. If valid, builds an `Authentication` object (user + roles) and puts it in the `SecurityContext`.
4. Lets the request continue. Downstream code sees the user via `SecurityContextHolder.getContext().getAuthentication()`.

The DB is **not** queried per request — that's the whole point of JWT. (You can still revoke by maintaining a small blacklist of token IDs if needed.)

### Smallest working example
```java
@Component
public class JwtFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String header = req.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            try {
                Claims claims = Jwts.parser()
                    .setSigningKey(SECRET).parseClaimsJws(header.substring(7)).getBody();
                var auth = new UsernamePasswordAuthenticationToken(
                    claims.getSubject(), null, List.of(new SimpleGrantedAuthority("ROLE_USER")));
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (JwtException ignored) { /* let downstream return 401 */ }
        }
        chain.doFilter(req, res);
    }
}

// In your SecurityConfig:
http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
```

Now every authenticated request flows through this filter; controllers see the user via `@AuthenticationPrincipal` or `SecurityContextHolder`.

### Drill (2 min)
Why `OncePerRequestFilter`? *(Hint: Spring may dispatch internally (forward, include) and re-trigger filters. `OncePerRequestFilter` guarantees the filter body runs at most once per HTTP request — avoiding double-decoding the token.)*

**Deep dive (later):** [spring-next/06-spring-security.md](../spring-next/06-spring-security.md)

---

## 4. MongoDB — Aggregation Advanced

### Why this exists
The aggregation pipeline can do far more than `find` + `match` + `group`. Two power tools you'll meet often: **`$lookup` with pipeline** (smarter joins) and **`$facet`** (multiple pipelines from one input).

### The idea in plain English
- **`$lookup` with pipeline** — a join, but you can filter/project on the joined collection *inside* the lookup. Avoids loading full documents you'll throw away.
- **`$facet`** — runs several sub-pipelines on the same input documents and returns all their results in one query. Perfect for dashboards (counts + top-10 + histogram in one round-trip).

**Analogy:** `$lookup` with pipeline is "fetch only the books the user borrowed, not the whole catalog." `$facet` is splitting a stream of customers into multiple counters at once (gender, age bucket, country) without re-running the pipeline.

### Smallest working example
```javascript
// $lookup with pipeline — join orders with only RECENT line items per order
db.orders.aggregate([
  { $match: { status: "PAID" } },
  {
    $lookup: {
      from: "lineItems",
      let: { orderId: "$_id" },
      pipeline: [
        { $match: { $expr: { $eq: ["$orderId", "$$orderId"] } } },
        { $match: { createdAt: { $gte: ISODate("2026-05-01") } } },
        { $project: { sku: 1, qty: 1 } }
      ],
      as: "recentItems"
    }
  }
]);

// $facet — one query, three aggregations
db.users.aggregate([
  {
    $facet: {
      total:      [ { $count: "n" } ],
      byCountry:  [ { $group: { _id: "$country", n: { $sum: 1 } } } ],
      ageBuckets: [ { $bucket: { groupBy: "$age", boundaries: [0,18,30,45,60,200] } } ]
    }
  }
]);
```

One DB round-trip, three answers. Beats running three separate queries.

### Drill (2 min)
You're building an admin dashboard with "total users", "users by country", and "signups by month." How many queries? *(Hint: one, using `$facet`. Each metric is a sub-pipeline. One scan of the collection feeds all three.)*

**Deep dive (later):** [mongodb/05-aggregation-pipeline.md](../mongodb/05-aggregation-pipeline.md)

---

## 5. Postgres — Performance Tuning

### Why this exists
A slow query is rarely "the database is slow" — it's usually a missing index, a bad query plan, or a sequential scan you didn't expect. `pg_stat_statements` is your stethoscope.

### The idea in plain English
`pg_stat_statements` is an extension that records every SQL statement and how long it took, summed across runs. You can ask: "what query is consuming the most total time?" The answer is almost always the right place to optimize.

**Analogy:** instead of profiling every endpoint by hand, you ask the database for a leaderboard: "show me the queries eating my CPU."

Companion tool: **`EXPLAIN (ANALYZE)`** on a slow query shows you exactly which join is slow and whether it used an index.

### Smallest working example
```sql
-- One-time: enable
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
-- (also add to shared_preload_libraries in postgresql.conf and restart)

-- Top 10 by total time
SELECT
  substring(query, 1, 80) AS short_query,
  calls,
  total_exec_time::int    AS total_ms,
  mean_exec_time::int     AS mean_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Then EXPLAIN the worst offender
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;
```

If you see `Seq Scan` on a big table where you filter by `customer_id` — add an index on `customer_id` and you've just fixed it.

### Drill (2 min)
`EXPLAIN ANALYZE` shows `Seq Scan on orders (rows=2,000,000)`. Why is that often bad? *(Hint: full table scan on 2M rows means reading every block from disk. An index on the filter column turns this into an index scan reading dozens of blocks. Order-of-magnitude faster.)*

**Deep dive (later):** [postgres/18-performance-tuning.md](../postgres/18-performance-tuning.md)

---

## 6. High-Level Design — Newsfeed

### Why this exists
Twitter, Instagram, LinkedIn — all show "a stream of posts from people you follow." Two opposite strategies. Both are valid; the choice depends on your read/write ratio.

### The idea in plain English
**Fan-out on write** (push)
- When a user posts, write the post into **every follower's inbox** in advance.
- Reading a feed is just `SELECT * FROM inbox WHERE user_id = me ORDER BY ts DESC`.
- Pros: super fast reads.
- Cons: a celebrity with 10M followers writes 10M rows on every post. Expensive at the tail.

**Fan-out on read** (pull)
- Don't pre-build inboxes. When a user opens the feed, fetch the latest posts from everyone they follow and merge.
- Pros: cheap writes; no work for users who never log in.
- Cons: slow reads, especially if you follow thousands.

**The real-world answer: hybrid.** Push for normal users, pull for "celebrities" with huge follower counts.

### A minimal sketch
```
Normal user posts:
  Post Service → for each follower → push to Inbox cache (Redis sorted set)
  Reading feed = read sorted set, paginate.

Celebrity posts (>100k followers):
  Post Service → store the post in Posts table
  Reading feed = read your follower inbox AND merge in any celebrity posts you follow
                 since your last visit (pull on read).
```

### Drill (3 min)
A user has 100 followers and posts once a day. You read your feed 50× a day. Push or pull? *(Hint: push (fan-out on write). One write produces 100 small writes; you save 50 expensive reads. Writes are rare and small; reads are frequent and big.)*

**Deep dive (later):** [system-design/high-level-design/15-newsfeed.md](../system-design/high-level-design/15-newsfeed.md)

---

## 7. LLD — Online Banking System

### Why this exists
Banking is the classic "money must never be wrong" LLD. Tests transactions, audit logs, concurrency.

### Core entities
- **Customer** — id, KYC info
- **Account** — id, type (CHECKING/SAVINGS), balance, owner, status
- **Transaction** — debit or credit on one account
- **Transfer** — paired transactions (debit one account, credit another), atomic
- **Ledger** — append-only log of all transactions for audit

### A clean class sketch
```java
enum TxType { CREDIT, DEBIT }

class Transaction {
    UUID id;
    Account account;
    TxType type;
    BigDecimal amount;
    Instant at;
    String referenceId;   // links the two halves of a transfer
}

class TransferService {
    @Transactional
    public void transfer(Account from, Account to, BigDecimal amount) {
        if (from.balance.compareTo(amount) < 0) throw new InsufficientFundsException();
        String ref = UUID.randomUUID().toString();
        ledger.append(new Transaction(UUID.randomUUID(), from, TxType.DEBIT,  amount, Instant.now(), ref));
        ledger.append(new Transaction(UUID.randomUUID(), to,   TxType.CREDIT, amount, Instant.now(), ref));
        from.balance = from.balance.subtract(amount);
        to.balance   = to.balance.add(amount);
    }
}
```

Two rules: **BigDecimal**, never `double`. **Append-only ledger** — you never mutate a transaction; corrections are new transactions.

### Drill (3 min)
Why store both DEBIT and CREDIT rows instead of just storing the balance change once? *(Hint: double-entry accounting. Every transaction has two sides — money came from somewhere and went somewhere. The ledger lets you audit either side and reconstruct any balance at any moment in time.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — DP: LIS / LCS

### Two classic DP problems

**LIS — Longest Increasing Subsequence:** given `[10, 9, 2, 5, 3, 7, 101, 18]`, the longest *increasing* subsequence (not contiguous) is `[2, 3, 7, 101]` → length 4.

**LCS — Longest Common Subsequence:** given two strings `"abcde"` and `"ace"`, the longest *subsequence* present in both is `"ace"` → length 3.

### LIS — the O(n²) DP solution
Let `dp[i]` = length of the longest increasing subsequence ending at index `i`. For each `i`, look at all `j < i`: if `nums[j] < nums[i]`, then `dp[i]` could be `dp[j] + 1`.

```javascript
function lengthOfLIS(nums) {
  const dp = new Array(nums.length).fill(1);
  for (let i = 1; i < nums.length; i++) {
    for (let j = 0; j < i; j++) {
      if (nums[j] < nums[i]) dp[i] = Math.max(dp[i], dp[j] + 1);
    }
  }
  return Math.max(...dp);
}
```

### LCS — the classic 2D DP
Let `dp[i][j]` = LCS length of `a[0..i)` and `b[0..j)`.
- If `a[i-1] === b[j-1]`: `dp[i][j] = dp[i-1][j-1] + 1` (match — extend the diagonal)
- Else: `dp[i][j] = max(dp[i-1][j], dp[i][j-1])` (skip a char from one side)

```javascript
function lcs(a, b) {
  const dp = Array.from({ length: a.length + 1 }, () => new Array(b.length + 1).fill(0));
  for (let i = 1; i <= a.length; i++) {
    for (let j = 1; j <= b.length; j++) {
      dp[i][j] = a[i-1] === b[j-1]
        ? dp[i-1][j-1] + 1
        : Math.max(dp[i-1][j], dp[i][j-1]);
    }
  }
  return dp[a.length][b.length];
}
```

### The common pattern
Both define `dp[i]` (or `dp[i][j]`) as "the answer up to position i" and build the answer step by step from earlier answers. That's the DP recipe.

### Drill (3 min)
For `lcs("abc", "ac")`, what's `dp[3][2]`? *(Hint: 2 — the LCS is `"ac"`. Trace by hand: matches at a (i=1,j=1) and c (i=3,j=2), with a skip in between.)*

**Deep dive (later):** [dsa/ds-js/dynamic-programming.md](../dsa/ds-js/dynamic-programming.md)

---

## 9. Design Pattern — State

### Intent
**Let an object behave differently as its internal state changes**, by delegating to a state object. Replaces giant `if/else` or `switch` chains.

### When you'd use it
- A document workflow (draft → review → published → archived) with different allowed actions per state
- A TCP connection (closed, listening, established) with different methods valid per state
- A vending machine (idle, hasCoin, dispensing)

### The simplest version (Java)
```java
interface OrderState {
    void pay(Order o);
    void ship(Order o);
}

class CreatedState implements OrderState {
    public void pay(Order o)  { o.setState(new PaidState()); }
    public void ship(Order o) { throw new IllegalStateException("pay first"); }
}
class PaidState implements OrderState {
    public void pay(Order o)  { throw new IllegalStateException("already paid"); }
    public void ship(Order o) { o.setState(new ShippedState()); }
}
class ShippedState implements OrderState { /* both no-ops or errors */ }

class Order {
    private OrderState state = new CreatedState();
    public void setState(OrderState s) { this.state = s; }
    public void pay()  { state.pay(this); }
    public void ship() { state.ship(this); }
}
```

The `Order` doesn't have a giant `if (status == ...)`. Each state knows its own rules. Adding a new state means adding one class — no edits to existing ones.

### Drill (1 min)
This pattern lets you **add new states without touching old code**. Which SOLID principle is that? *(Hint: Open/Closed Principle — open for extension, closed for modification.)*

**Deep dive (later):** [design-patterns/common/behavioral/state.md](../design-patterns/common/behavioral/state.md)

---

## 10. DevOps — Distributed Tracing

### Why this exists
In microservices, one request hops through 5 services. If it's slow, **which one** is slow? Logs alone won't tell you. Tracing connects the dots.

### The idea in plain English
A **trace** is the journey of one request through your system. It's made of **spans** — each span is one unit of work (an HTTP call, a DB query) with a start and end time. Spans nest: the outer span (the API request) contains inner spans (DB query, cache check, downstream service call).

Two key ideas:
- **Trace ID** — one ID for the whole journey, propagated via HTTP headers (`traceparent`) from service to service.
- **Span ID** — unique per span; spans know their parent.

**Analogy:** a trace is a flight itinerary with multiple legs. Each leg (span) is a flight with departure/arrival times. The whole itinerary has a booking ID (trace ID). When the trip runs late, you see exactly which leg was delayed.

**OpenTelemetry (OTel)** is the open-source standard — vendor-neutral SDK that produces spans you can ship to Jaeger, Tempo, Honeycomb, Datadog, etc.

### Smallest working example (Node + OTel)
```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

new NodeSDK({
  instrumentations: [getNodeAutoInstrumentations()],
}).start();

// That's it — Express, HTTP, Redis, MongoDB calls become spans automatically.
// In a downstream service, the traceparent header is propagated for free.
```

In Jaeger UI you now see a flame-graph: "/api/order took 800ms, of which 650ms was waiting on the inventory service which spent 600ms on a DB query." You found the slow spot in 5 seconds.

### Drill (2 min)
You have 3 services. A user's request is slow. Without tracing, what do you do? *(Hint: tail logs of all three, hope the timestamps line up, hope you can correlate by user id. With tracing, one trace ID gives you the whole story sorted in a flame graph.)*

**Deep dive (later):** [devops/21-distributed-tracing.md](../devops/21-distributed-tracing.md)

---

## End-of-day checklist

- [ ] Angular: I can pick between service+signal vs NgRx for a given scenario
- [ ] Node.js: I implemented cache-aside (read-check-fill, write-invalidate)
- [ ] Spring: I can sketch a JWT filter and where to plug it in the chain
- [ ] MongoDB: I can use `$facet` to combine multiple aggregations in one query
- [ ] Postgres: I can query `pg_stat_statements` to find slow queries
- [ ] HLD: I can compare fan-out on write vs read for a newsfeed
- [ ] LLD: I can model accounts and transfers with a ledger
- [ ] DSA: I can write the LIS and LCS DP recurrences from memory
- [ ] DP: I can explain when to use the State pattern vs an enum switch
- [ ] DevOps: I know what a trace, span, and trace ID are

**If you remember just one thing today:** the right answer is almost always **"pre-compute the work that gets read most"** — that's caches, fan-out on write, DP tables, and selectors. Optimize for the dominant access pattern.

**Tomorrow:** NgRx Store deep dive, Jest tests, OAuth2/OIDC, geospatial queries, PITR, distributed cache, logger LLD, knapsack, Template Method, SLOs.
