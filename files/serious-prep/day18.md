# Day 18 — Store, tests, OAuth2, and durable systems

> **Today's goal:** see how NgRx works in 15 minutes, write a real test for an API, understand OAuth2/OIDC at a high level, and design a distributed cache.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | NgRx Store (actions, reducers, selectors) | 15m |
| 2 | Node.js | Testing (Jest + Supertest) | 15m |
| 3 | Spring Boot | OAuth2 / OIDC with Spring Security | 15m |
| 4 | MongoDB | Geospatial Queries (2dsphere, $near) | 10m |
| 5 | Postgres | Backup & PITR (pg_basebackup, WAL archiving) | 10m |
| 6 | HLD | Distributed Cache Design (consistent hashing) | 12m |
| 7 | LLD | Logger system (multi-level, async) | 12m |
| 8 | DSA | DP: 0/1 Knapsack | 15m |
| 9 | Design Pattern | Template Method | 8m |
| 10 | DevOps | Incident Response & SLOs (SLI/SLO/SLA, error budgets) | 10m |

---

## 1. Angular — NgRx Store

### Why this exists
For complex apps with lots of shared state, lots of async, and a need to debug "what changed when," NgRx (Redux for Angular) gives you a single source of truth and time-travel dev tools.

### The idea in plain English
NgRx is built on three pieces — all just plain objects/functions:

- **Action** — a description of an event ("user added item to cart")
- **Reducer** — a pure function `(state, action) => newState`. Given the old state and an action, returns the new state. No mutation.
- **Selector** — a function that reads from state. Components subscribe to selectors, not raw state — selectors are memoized (recompute only when their inputs change).

**Analogy:** a bank ledger. Every change is a logged transaction (action). Recomputing the balance is the reducer. A specialized report (selector) reads the ledger and gives you "balance of account X" without touching the ledger.

### Smallest working example
```typescript
// actions.ts
export const addItem = createAction('[Cart] Add', props<{ item: string }>());

// reducer.ts
export const cartReducer = createReducer(
  { items: [] as string[] },
  on(addItem, (state, { item }) => ({ items: [...state.items, item] }))
);

// selectors.ts
export const selectCart = (s: AppState) => s.cart;
export const selectCount = createSelector(selectCart, c => c.items.length);

// component.ts
constructor(private store: Store<AppState>) {
  this.count$ = store.select(selectCount);
}
addToCart() { this.store.dispatch(addItem({ item: 'banana' })); }
```

Three rules:
1. **State is immutable** — never mutate `state.items.push(...)`. Always return a new object.
2. **Reducers are pure** — no API calls, no `Date.now()`, no randomness.
3. **Side effects live in `@ngrx/effects`** (we cover that on Day 19).

### Drill (2 min)
Why do reducers have to be pure? *(Hint: predictability and time-travel. Given the same state + action, you always get the same result. That's what lets the dev tools "rewind" your app and replay actions.)*

**Deep dive (later):** [angular/state-ngrx.md](../angular/state-ngrx.md)

---

## 2. Node.js — Testing

### Why this exists
Untested code is "hope-driven development." Tests give you the confidence to refactor and ship. Two libraries cover 80% of Node API testing: **Jest** (test runner + assertions) and **Supertest** (HTTP requests against your Express app, no real server).

### The idea in plain English
- **Jest** — `describe`, `it`, `expect`. Watches files, runs tests in parallel, gives you mocks.
- **Supertest** — wraps your Express app and lets you call it like an HTTP client without binding to a port.

**Three levels of test:**
- **Unit** — a single function. Fast, no I/O. (Most of your tests.)
- **Integration** — multiple modules together, maybe an in-memory DB.
- **End-to-end (e2e)** — real HTTP request, real DB. Slowest. Few of these.

### Smallest working example
```javascript
// app.js
const express = require('express');
const app = express();
app.get('/health', (_, res) => res.json({ ok: true }));
module.exports = app;

// app.test.js
const request = require('supertest');
const app = require('./app');

describe('GET /health', () => {
  it('returns 200 with { ok: true }', async () => {
    const res = await request(app).get('/health');
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ ok: true });
  });
});
```

Run with `npx jest`. Supertest spins up the app in-process, sends the request, and gives you the response. No server needed.

### Drill (2 min)
Why is mocking the database in unit tests good but mocking everything in integration tests bad? *(Hint: unit tests want isolation — mock the DB to test logic in microseconds. Integration tests want realism — mocking everything just tests your mocks, not your wiring.)*

**Deep dive (later):** [nodejs/18-testing.md](../nodejs/18-testing.md)

---

## 3. Spring Boot — OAuth2 / OIDC with Spring Security

### Why this exists
"Sign in with Google" is OAuth2 + OIDC. You delegate authentication to a trusted **identity provider (IdP)** (Google, Okta, Auth0, Keycloak). You never see the user's password.

### The idea in plain English
**OAuth2** is about **authorization** — getting an access token that lets you call APIs on a user's behalf.
**OIDC (OpenID Connect)** is a thin layer on top of OAuth2 that adds an **ID token** identifying the user — that's how "sign in with Google" works.

**The flow (Authorization Code with PKCE):**
1. User clicks "Login with Google" → app redirects to Google's auth URL.
2. User signs in at Google. Google redirects back with an auth code.
3. Your server exchanges the code for an **ID token** (who they are) and an **access token** (what they can call).
4. Your server creates a local session or issues its own JWT.

**Analogy:** the IdP is the airport check-in counter. They verify your ID and hand you a boarding pass (token). The gate (your API) checks the boarding pass, not your passport.

### Smallest working setup (Spring Boot)
```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, email, profile
```

```java
@Configuration
@EnableWebSecurity
class SecurityConfig {
    @Bean
    SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a.anyRequest().authenticated())
            .oauth2Login(Customizer.withDefaults());     // does all the redirects for you
        return http.build();
    }
}
```

That's it. Hitting any protected endpoint redirects to Google, comes back logged in. Spring auto-discovers Google's endpoints via OIDC.

### Drill (2 min)
What's the difference between an ID token and an access token? *(Hint: ID token says "who the user is" (consumed by your client to greet "Hi, Jane"). Access token says "what you can do on their behalf" (sent to APIs as a Bearer token). Same flow, two purposes.)*

**Deep dive (later):** [spring-next/06-spring-security.md](../spring-next/06-spring-security.md)

---

## 4. MongoDB — Geospatial Queries

### Why this exists
"Find restaurants within 1 km of me" is too painful with regular indexes. Mongo's **2dsphere** index understands the curvature of the Earth and answers proximity queries in milliseconds.

### The idea in plain English
Store coordinates as **GeoJSON** objects: `{ type: "Point", coordinates: [lng, lat] }`. **Important:** longitude first, latitude second. (It feels backward; you'll forget it once and never again.)

Build a 2dsphere index on the field and query with operators like:
- **`$near`** — sort by distance from a point
- **`$geoWithin`** — within a polygon (e.g., a city boundary)
- **`$geoIntersects`** — features that touch or cross a shape

### Smallest working example
```javascript
db.places.createIndex({ location: "2dsphere" });

db.places.insertMany([
  { name: "Cafe A", location: { type: "Point", coordinates: [77.6, 12.97] } },
  { name: "Cafe B", location: { type: "Point", coordinates: [77.7, 12.95] } },
]);

// Find places within 2 km of [77.62, 12.97]
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [77.62, 12.97] },
      $maxDistance: 2000      // meters
    }
  }
});
```

Results come sorted by distance, ascending. Add `$maxDistance` to bound the search.

### Drill (2 min)
You see `[lat, lng]` in a tutorial. Is it right? *(Hint: no — GeoJSON is `[lng, lat]`. If you pass `[lat, lng]` your "nearest restaurant" will be in the wrong hemisphere.)*

**Deep dive (later):** [mongodb/16-geospatial-queries.md](../mongodb/16-geospatial-queries.md)

---

## 5. Postgres — Backup & PITR

### Why this exists
`pg_dump` is fine for small DBs, but it snapshots only the moment you ran it. For real production, you want **point-in-time recovery (PITR)**: "restore to 2 minutes before the bad DELETE."

### The idea in plain English
Two pieces:
1. **Base backup** — a full physical copy of the data directory, taken once a day (or week). Use `pg_basebackup`.
2. **WAL archiving** — every transaction log file is shipped to backup storage as it fills.

To restore: pick a base backup, replay WAL on top of it up to the second before the disaster.

**Analogy:** the base backup is a photo of your house at midnight. WAL archives are timestamped diary entries of every change since. Combine them to reconstruct your house at 09:42:13 on any day.

### Smallest working example
```ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/wal/%f'        # ship each WAL file
```

```bash
# Take a base backup
pg_basebackup -D /backup/base -Ft -z -P

# Restore later — recovery.conf (or postgresql.auto.conf in PG12+)
restore_command = 'cp /backup/wal/%f %p'
recovery_target_time = '2026-05-20 09:42:00 IST'
```

Start Postgres in recovery mode; it replays WAL up to your target time, then opens for connections.

### Drill (2 min)
A bug deleted important rows at 10:00. Your last `pg_dump` was at 02:00. Without WAL archiving, what's the best you can do? *(Hint: restore to 02:00 — you lose 8 hours of legitimate writes. PITR with WAL would restore to 09:59:59 and lose only the bad delete.)*

**Deep dive (later):** [postgres/19-backup-and-recovery.md](../postgres/19-backup-and-recovery.md)

---

## 6. High-Level Design — Distributed Cache

### Why this exists
One Redis node is fast but limited in memory. For terabyte-scale caches (Memcached, Redis Cluster), you spread the data across many nodes. The trick: how do you know which node holds which key?

### The idea in plain English
Naive answer: `node = hash(key) % N`. Works until you add or remove a node — then **every key remaps** and your hit ratio collapses to 0%.

**Consistent hashing** fixes this. Imagine a clock with positions 0–360°.
- Each node is placed at some position (`hash(node_id) mod 360`).
- Each key is placed at some position (`hash(key) mod 360`).
- A key is owned by the **next node clockwise** from its position.

When you add a node, only the keys between the new node and its clockwise neighbor move. ~1/N of the keys, not 100%.

**Virtual nodes:** to balance load, each physical node owns 100+ positions on the ring. Random placement → smoother distribution.

### A minimal sketch
```
Ring (360 positions):

  0° ──────── 90° ──────── 180° ──────── 270° ──────── 360°
       N1            N2              N3            N1 (wraps)

Key "user:42" hashes to 100° → owned by N2.
Add N4 at 250° → only keys in (180°, 250°] move from N3 to N4.
```

In code, the ring is a sorted structure (e.g., a `TreeMap` in Java) and lookups are O(log N).

### Drill (3 min)
A 4-node cache uses naive `hash(key) % 4`. You add a 5th node. What % of keys move? *(Hint: roughly 80% — `mod 4` and `mod 5` rarely agree. With consistent hashing it's ~20% (one node's share). That's the whole point.)*

**Deep dive (later):** [system-design/high-level-design/16-distributed-cache.md](../system-design/high-level-design/16-distributed-cache.md)

---

## 7. LLD — Logger System

### Why this exists
A homegrown logger is the perfect mini-project for showing layered design, async I/O, and the Chain of Responsibility / Decorator vibe.

### Core entities
- **LogLevel** — `DEBUG < INFO < WARN < ERROR < FATAL`
- **LogRecord** — `{ timestamp, level, message, context }`
- **Appender (Handler)** — destination: console, file, syslog, HTTP
- **Filter** — decides whether to forward a record (e.g., by level or by package name)
- **Logger** — facade for app code; routes records to one or more appenders

### A clean class sketch
```java
enum LogLevel { DEBUG, INFO, WARN, ERROR, FATAL }

interface Appender { void append(LogRecord r); }

class ConsoleAppender implements Appender {
    public void append(LogRecord r) { System.out.println(r); }
}

class AsyncFileAppender implements Appender {
    private final BlockingQueue<LogRecord> q = new LinkedBlockingQueue<>(10_000);

    public AsyncFileAppender() {
        new Thread(this::drain, "log-writer").start();
    }
    public void append(LogRecord r) { q.offer(r); }   // never blocks the caller
    private void drain() {
        while (true) {
            try (var w = new FileWriter("app.log", true)) {
                w.write(q.take().toString() + "\n");
            } catch (Exception e) { /* swallow */ }
        }
    }
}

class Logger {
    private LogLevel min = LogLevel.INFO;
    private List<Appender> appenders = new ArrayList<>();

    public void log(LogLevel lvl, String msg) {
        if (lvl.ordinal() < min.ordinal()) return;    // filter
        var r = new LogRecord(Instant.now(), lvl, msg, Map.of());
        for (var a : appenders) a.append(r);
    }
}
```

The **async file appender** is the key idea: the calling thread never waits for disk I/O. The queue absorbs bursts.

### Drill (3 min)
Why is `q.offer(...)` better than `q.put(...)` here? *(Hint: `offer` returns false if the queue is full instead of blocking. Better to drop a log line than to block a hot request thread. Tune queue size + add a "dropped" counter.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — DP: 0/1 Knapsack

### The problem
Given `n` items, each with a weight `w[i]` and value `v[i]`, and a knapsack capacity `W`, choose a subset of items (each taken **at most once** — that's the "0/1") to maximize total value without exceeding `W`.

```
items: (w=2, v=3), (w=3, v=4), (w=4, v=5)
capacity W = 5
best: pick items 1 and 2 → weight 5, value 7
```

### The 2D DP
Let `dp[i][c]` = max value using the first `i` items with capacity `c`.
- Don't take item i: `dp[i][c] = dp[i-1][c]`
- Take item i (if it fits): `dp[i][c] = max(dp[i-1][c], dp[i-1][c - w[i]] + v[i])`

```javascript
function knapsack(weights, values, W) {
  const n = weights.length;
  const dp = Array.from({ length: n + 1 }, () => new Array(W + 1).fill(0));
  for (let i = 1; i <= n; i++) {
    for (let c = 0; c <= W; c++) {
      dp[i][c] = dp[i-1][c];                                       // skip
      if (c >= weights[i-1]) {
        dp[i][c] = Math.max(dp[i][c],
                            dp[i-1][c - weights[i-1]] + values[i-1]); // take
      }
    }
  }
  return dp[n][W];
}

knapsack([2, 3, 4], [3, 4, 5], 5);  // → 7
```

**Time:** O(n × W). **Space:** O(n × W) — and can be compressed to O(W) using one row (iterate `c` downward to avoid using an updated value).

### Why "0/1" matters
If you could take an item multiple times, the recurrence changes (look up "unbounded knapsack"). The "each item once" constraint is why we use `dp[i-1]` not `dp[i]` on the take branch — we can't take it again in the same step.

### Drill (3 min)
For the sample (`weights=[2,3,4]`, `values=[3,4,5]`, `W=5`), what is `dp[2][5]`? *(Hint: 7 — items 1 and 2, weight 2+3=5, value 3+4=7. Adding item 3 (weight 4) can't beat that within capacity 5.)*

**Deep dive (later):** [dsa/ds-js/dynamic-programming.md](../dsa/ds-js/dynamic-programming.md)

---

## 9. Design Pattern — Template Method

### Intent
**Define the skeleton of an algorithm in a base class, let subclasses fill in specific steps.**

### When you'd use it
- Test frameworks (`setUp` → `runTest` → `tearDown`) — JUnit is built on this
- HTTP servlets (parent handles request lifecycle, subclasses override `doGet`/`doPost`)
- ETL pipelines with a fixed order but variable steps (extract → transform → load)

### The simplest version (Java)
```java
abstract class ReportGenerator {
    // The template method — final, can't be overridden
    public final void generate() {
        var data = fetchData();
        var transformed = transform(data);
        write(transformed);
    }

    protected abstract List<Row> fetchData();
    protected List<Row> transform(List<Row> rows) { return rows; }   // optional hook
    protected abstract void write(List<Row> rows);
}

class CsvReport extends ReportGenerator {
    protected List<Row> fetchData() { /* SQL query */ }
    protected void write(List<Row> rows) { /* write CSV file */ }
}

class JsonReport extends ReportGenerator {
    protected List<Row> fetchData() { /* SQL query */ }
    protected void write(List<Row> rows) { /* write JSON file */ }
}
```

`generate()` enforces the order. Subclasses fill in what changes. No duplicated lifecycle code.

### Drill (1 min)
Why is `generate()` declared `final`? *(Hint: so subclasses can't override the skeleton and break the order. That's the whole point — the template is fixed; only the steps are open.)*

**Deep dive (later):** [design-patterns/common/behavioral/template-method.md](../design-patterns/common/behavioral/template-method.md)

---

## 10. DevOps — Incident Response & SLOs

### Why this exists
"It's up" is too binary. Real systems are sometimes slow, sometimes flaky. SLOs give you a precise contract with yourself: "fast enough, often enough, or I owe my users an apology."

### The idea in plain English
Three acronyms in order:
- **SLI** — Service Level **Indicator**. A measurable thing: "% of requests with p99 < 300ms."
- **SLO** — Service Level **Objective**. A target for the SLI: "99.9% of requests with p99 < 300ms over 30 days."
- **SLA** — Service Level **Agreement**. The contractual promise to customers, usually with penalties. SLA targets are lower than SLOs (give yourself room).

**Error budget:** if your SLO is 99.9%, you have 0.1% room for things to be bad. Over 30 days that's ~43 minutes. Spend it on planned risky changes; freeze deploys when you've burned through it.

**Analogy:** SLO is "I promise to be 99.9% on time." Error budget is the 43-minute "late pass" you get to use however you want. If a bad deploy uses 30 of those minutes, you only have 13 left.

**Incident playbook (memorize this loop):**
1. **Detect** — pager fires (alert on an SLI breach, not on every CPU spike)
2. **Triage** — assign an incident commander, declare severity
3. **Mitigate** — restore service first, even with a hack (rollback, scale up)
4. **Resolve** — fix the actual cause
5. **Postmortem** — write a blameless writeup: timeline, root cause, action items

### Smallest working example (an SLO and its alert)
```
SLI: availability = good_requests / total_requests
SLO: 99.9% over a 30-day window

Burn-rate alert (Prometheus):
  Alert when, over the last 1h, the error rate is so high that
  if it continued for 30 days you'd exhaust your budget in 6 hours.
```

That alert only fires when you're burning the budget meaningfully fast — not on every spike.

### Drill (2 min)
Your team is "permanently on fire." Every alert is paged. What's likely wrong with the SLO? *(Hint: it's too strict, or you alert on raw thresholds instead of error budgets. Move to budget-burn-rate alerts so only meaningful trouble pages you.)*

**Deep dive (later):** [devops/20-incident-response.md](../devops/20-incident-response.md)

---

## End-of-day checklist

- [ ] Angular: I can describe action → reducer → selector → component
- [ ] Node.js: I wrote a Supertest test for one endpoint
- [ ] Spring: I can name the difference between ID token and access token
- [ ] MongoDB: I created a 2dsphere index and ran a `$near` query
- [ ] Postgres: I can explain why WAL archiving is needed for PITR
- [ ] HLD: I can explain why consistent hashing avoids the 100%-remap problem
- [ ] LLD: I can sketch an async file appender with a bounded queue
- [ ] DSA: I wrote the 0/1 knapsack DP from scratch
- [ ] DP: I can explain why the template method should be `final`
- [ ] DevOps: I know what an SLI, SLO, SLA, and error budget are

**If you remember just one thing today:** every robust system has an **escape valve** — error budgets, bounded queues, virtual nodes, immutable state. Build them in before you need them.

**Tomorrow:** NgRx Effects, security hardening, method-level auth, index strategies, read replicas, sharding, vending machines, greedy, Chain of Responsibility, Terraform.
