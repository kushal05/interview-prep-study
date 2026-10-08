# Day 24 — Polish, speed, and streams: animations, caching, gRPC

> **Today's goal:** delight users with animations, shave latency with caching, federate across databases, and learn the protocol that's faster than REST.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Animations | 15m |
| 2 | Node.js | gRPC | 15m |
| 3 | Spring Boot | @Cacheable | 15m |
| 4 | MongoDB | Mongoose middleware | 10m |
| 5 | Postgres | Foreign Data Wrappers | 10m |
| 6 | HLD | Real-time streaming | 12m |
| 7 | LLD | Pub-Sub system | 12m |
| 8 | DSA | String matching (KMP / Rabin-Karp) | 15m |
| 9 | Design Pattern | DAO | 8m |
| 10 | DevOps | Databases in prod | 10m |

---

## 1. Angular — Animations

### Why this exists
A modal that just *appears* feels jarring. The same modal that **fades in over 200ms** feels native. Angular ships an animation API that's declarative — you describe states and transitions; Angular interpolates.

### The idea in plain English
Animations are recipes:
- **Trigger** — a name like `'fadeInOut'` you reference in the template.
- **State** — a snapshot of style (e.g., `void` = "not in DOM yet").
- **Transition** — a path between states with a duration (`':enter'`, `':leave'`, or `'open => closed'`).
- **Animate** — the easing and duration glue.

Think of it as a film director's storyboard: state A → state B over `200ms ease-out`. Angular handles every frame in between.

### Smallest working example
```typescript
import { trigger, transition, style, animate } from '@angular/animations';

@Component({
  selector: 'app-notif',
  animations: [
    trigger('fadeInOut', [
      transition(':enter', [
        style({ opacity: 0, transform: 'translateY(-10px)' }),
        animate('200ms ease-out', style({ opacity: 1, transform: 'translateY(0)' })),
      ]),
      transition(':leave', [
        animate('150ms ease-in', style({ opacity: 0, transform: 'translateY(-10px)' })),
      ]),
    ]),
  ],
  template: `<div *ngIf="show" @fadeInOut class="toast">{{ msg }}</div>`,
})
class NotifComponent { @Input() show = false; @Input() msg = ''; }
```

`provideAnimations()` in `main.ts` is required: `bootstrapApplication(AppComponent, { providers: [provideAnimations()] });`

### Drill (2 min)
Animations look janky on slow phones. What's the simplest mitigation? *(Hint: animate `transform` and `opacity` — they run on the GPU and don't trigger layout. Avoid animating `top`, `left`, `width`, `height` — those cause reflow.)*

**Deep dive (later):** [angular/animations.md](../angular/animations.md)

---

## 2. Node.js — gRPC

### Why this exists
REST + JSON is fine for browsers. For service-to-service traffic, it's wasteful: HTTP/1.1, text JSON, no schema. **gRPC** is HTTP/2 + Protocol Buffers (binary, schema'd). Smaller payloads, lower latency, generated client code.

### The idea in plain English
REST = postcards in plain English. Anyone can read them, but they're verbose. **gRPC = telegrams** — compact, binary, with a strict schema both ends agreed on. You define a `.proto` file (the contract). Tooling generates server stubs and client code for many languages.

When to choose what:
- **REST + JSON** — public APIs, browser clients, debugging by `curl`.
- **gRPC** — internal microservices, mobile clients where bandwidth matters, streaming RPCs.

### Smallest working example
```protobuf
// greet.proto
syntax = "proto3";
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}
message HelloRequest { string name = 1; }
message HelloReply   { string message = 1; }
```

```javascript
// server.js
const grpc = require('@grpc/grpc-js');
const loader = require('@grpc/proto-loader');
const pkg = grpc.loadPackageDefinition(loader.loadSync('greet.proto')).Greeter;

const server = new grpc.Server();
server.addService(pkg.service, {
  SayHello: (call, cb) => cb(null, { message: `Hello, ${call.request.name}` }),
});
server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => server.start());
```

```javascript
// client.js
const client = new pkg('localhost:50051', grpc.credentials.createInsecure());
client.SayHello({ name: 'Kushal' }, (_, res) => console.log(res.message));
// → "Hello, Kushal"
```

A 200-byte JSON might become 30 bytes in protobuf. Multiply by millions of RPCs.

### Drill (2 min)
You want server → client streaming (push a stock price every second). REST or gRPC? *(Hint: gRPC. It supports four call types — unary, server-streaming, client-streaming, bidirectional. REST would need WebSocket or SSE on top.)*

**Deep dive (later):** [nodejs/22-microservices-and-grpc.md](../nodejs/22-microservices-and-grpc.md)

---

## 3. Spring Boot — @Cacheable

### Why this exists
The same `findById(42)` runs a million times a day. Computing or fetching it every time wastes CPU and DB. A **cache** stores the result the first time and returns it from memory thereafter. Spring's `@Cacheable` makes a method cacheable with one annotation.

### The idea in plain English
A cache is a sticky note: "I asked this before; here's the answer." `@Cacheable("users")` says: "For each unique argument list, run this method once and reuse the answer." Behind the scenes Spring wraps the method in a proxy that checks the cache before calling.

- `@Cacheable` — read with caching.
- `@CachePut` — write through (update the cache).
- `@CacheEvict` — remove an entry (after a delete).

The cache "store" can be in-memory (Caffeine), Redis, or anything that implements `CacheManager`.

### Smallest working example
```java
@Service
class UserService {
    private final UserRepository repo;
    UserService(UserRepository repo) { this.repo = repo; }

    @Cacheable("users")                // key = id by default
    public User findById(long id) {
        return repo.findById(id).orElseThrow();
    }

    @CacheEvict(value = "users", key = "#user.id")
    public void update(User user) {
        repo.save(user);
    }
}

@SpringBootApplication
@EnableCaching                          // turn the magic on
class App { /* ... */ }
```

Without `@EnableCaching`, the annotation does nothing.

### Drill (2 min)
You added `@Cacheable("users")`. Stale data appears after a manual DB update. Why? *(Hint: the cache doesn't know the DB changed underneath. Either invalidate on writes (`@CacheEvict`), set a TTL on the cache, or use Redis pub/sub to broadcast invalidations.)*

**Deep dive (later):** [spring-next/08-spring-boot-advanced.md — Caching](../spring-next/08-spring-boot-advanced.md#4-caching)

---

## 4. MongoDB — Mongoose Middleware

### Why this exists
You want to **hash a password before saving** or **log every delete**. Putting that logic in every service is messy. Mongoose **middleware** (also called *hooks*) runs before or after model operations — one definition, every call.

### The idea in plain English
A hook is a webhook for your model. Mongoose calls your function around `save`, `validate`, `findOneAndUpdate`, etc. Two kinds:
- **`pre`** — runs **before** the operation.
- **`post`** — runs **after**.

Common uses: hashing passwords, timestamps, soft delete, search-index updates, audit logs.

### Smallest working example
```javascript
const bcrypt = require('bcrypt');

const userSchema = new mongoose.Schema({
  email: String,
  password: String,
});

userSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.post('save', function (doc) {
  console.log(`saved user ${doc._id}`);
});

const User = mongoose.model('User', userSchema);

await User.create({ email: 'k@x.com', password: 'plaintext' });
// stored as: { email: 'k@x.com', password: '$2b$10$....' }
```

`this` inside `pre('save')` is the document being saved.

### Drill (2 min)
You added a `pre('save')` hook to hash passwords. Users created with `findOneAndUpdate` don't get hashed. Why? *(Hint: `findOneAndUpdate` doesn't trigger `save` hooks. Add a `pre('findOneAndUpdate')` hook too — `this` there is the query, not the doc.)*

**Deep dive (later):** [mongodb/16-mongoose-middleware.md](../mongodb/16-mongoose-middleware.md)

---

## 5. Postgres — Foreign Data Wrappers (FDW)

### Why this exists
You have one Postgres for orders and another for analytics. Or data lives in MySQL, CSV files, or even another database engine. **Foreign Data Wrappers** let one Postgres **query another database as if it were local**.

### The idea in plain English
A foreign table looks like a regular table but is actually a "view" pointing to a remote source. Run `SELECT * FROM remote_orders` and Postgres connects to the other server, fetches, and returns. The `postgres_fdw` extension connects to another Postgres; other FDWs talk to MySQL, MongoDB, CSV, even Excel.

Useful for migrations ("read old DB while writing new"), federated queries, reporting.

### Smallest working example
```sql
CREATE EXTENSION postgres_fdw;

CREATE SERVER analytics_db FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'analytics.internal', dbname 'analytics', port '5432');

CREATE USER MAPPING FOR app_user SERVER analytics_db
  OPTIONS (user 'reader', password 'secret');

-- Make the remote schema available
IMPORT FOREIGN SCHEMA public LIMIT TO (events)
  FROM SERVER analytics_db INTO public;

-- Now query it like a local table
SELECT user_id, count(*)
FROM events                          -- actually on analytics_db
WHERE created_at > now() - interval '1 day'
GROUP BY user_id;
```

The query is sent to the remote server; Postgres pulls just the result.

### Drill (2 min)
A FDW query is 100× slower than expected. What's the most common reason? *(Hint: predicates aren't being pushed down to the remote server. Check `EXPLAIN` — if the foreign scan pulls all rows then filters locally, you're bottlenecked by network. Add indexes on the remote table, use `fetch_size`, ensure the `WHERE` clause is push-down-compatible.)*

**Deep dive (later):** [postgres/17-foreign-data-wrappers.md](../postgres/17-foreign-data-wrappers.md)

---

## 6. High-Level Design — Real-time Streaming

### Why this exists
"Show users a live leaderboard." "Detect credit-card fraud within 50ms." Batch jobs (run once an hour) are too slow. You need **streaming** — process events as they arrive.

### The idea in plain English
A streaming system is a conveyor belt. Events enter on one end (Kafka), get transformed in real time (Flink / Kafka Streams / Spark Structured Streaming), and exit somewhere useful (DB, alert, dashboard).

Three guarantees you'll meet:
- **At-most-once** — fast, may drop messages on failure.
- **At-least-once** — never drops, may duplicate.
- **Exactly-once** — never drops, never duplicates. Hardest to implement. Kafka + Flink can do it end-to-end with idempotent producers and transactional commits.

### The big picture
```
   producers ──▶ Kafka (durable log)
                   │
                   ▼
              [ Flink job ]   ← stateful, windowed
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
   alerts DB   leaderboard   data lake
```

### Five interview points
1. **Stateful streams.** Sessionization, joins, windowed aggregations need state — stored in RocksDB on each worker.
2. **Watermarks** handle out-of-order events. "I'll wait 5 min for late events before closing this window."
3. **Backpressure.** When the sink is slow, the system slows producers, not crashes.
4. **Exactly-once needs cooperation** — Kafka transactions on input + idempotent or transactional sink.
5. **Replay.** A bug in your job? Reset the consumer group's offset to a past time and reprocess.

### Drill (2 min)
You want a 5-minute rolling average of click counts per user. What concept does the stream processor use? *(Hint: a **tumbling** or **sliding window**. Tumbling = non-overlapping 5-min buckets; sliding = a 5-min window that advances every minute.)*

**Deep dive (later):** [system-design/high-level-design/24-real-time-streaming.md](../system-design/high-level-design/24-real-time-streaming.md)

---

## 7. Low-Level Design — Pub-Sub System

### Why this exists
You want services to talk **without knowing about each other**. `OrderService` publishes `OrderPlaced`. `EmailService` and `AnalyticsService` subscribe. Add `ShippingService` tomorrow — `OrderService` doesn't change. That's **publish-subscribe**.

### The classes
- `Topic` — a named channel (`orders`, `payments`).
- `Subscriber` — has an `onMessage(msg)` callback.
- `Broker` — keeps topics and subscriber lists; routes messages.

### Smallest skeleton
```java
class Broker {
    private final Map<String, List<Subscriber>> subs = new ConcurrentHashMap<>();

    void subscribe(String topic, Subscriber s) {
        subs.computeIfAbsent(topic, k -> new CopyOnWriteArrayList<>()).add(s);
    }
    void publish(String topic, Object msg) {
        var list = subs.getOrDefault(topic, List.of());
        for (var s : list) s.onMessage(msg);   // fan-out
    }
}

interface Subscriber { void onMessage(Object msg); }
```

### The hard parts
1. **Async delivery.** Don't block the publisher. Run subscribers on a thread pool or send via a queue.
2. **Durability.** What if the broker crashes mid-broadcast? Persist messages (this is what Kafka/RabbitMQ give you).
3. **Backpressure.** A slow subscriber shouldn't drown the broker. Bounded queues, drop policies, or per-subscriber lag metrics.
4. **At-least-once vs exactly-once.** Same trade-off as streaming. Default is at-least-once; subscribers must be idempotent.
5. **Filtering / topics vs queues.** Pub-sub = every subscriber gets every message (topic). Work queue = only one consumer gets each message.

### Drill (2 min)
You add `analytics.onMessage` that's slow. The order checkout becomes slow. What's wrong? *(Hint: synchronous publish in the same thread. Move subscriber calls to a queue/thread pool so the publisher returns immediately.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — String Matching (KMP / Rabin-Karp)

### Why this exists
"Find `needle` inside `haystack`." Naively checking every position is `O(n × m)`. For long strings or patterns, that's slow. **KMP** and **Rabin-Karp** do it in O(n + m) average.

### The big idea behind each
- **KMP (Knuth-Morris-Pratt)** — precompute a "failure table" for the pattern so when a mismatch happens, you skip ahead without re-checking what you already know matches. Worst-case O(n + m).
- **Rabin-Karp** — slide a window of size m over haystack, compute a *rolling hash* in O(1) per step, and compare hashes. Same hash → check character by character. Average O(n + m). Great for **multiple-pattern** search.

### Naive vs KMP — intuition
```
haystack: ABCABCABD
pattern : ABCABD

Mismatch at index 5 (D vs C). Naive jumps back to start.
KMP knows "ABCAB" was matched. Suffix "AB" of the matched
part is also a prefix of the pattern. So it shifts by 3
without rechecking "AB".
```

### Smallest Rabin-Karp
```javascript
function rabinKarp(haystack, needle) {
  const n = haystack.length, m = needle.length;
  if (m > n) return -1;
  const BASE = 256, MOD = 1_000_000_007n;

  let hHash = 0n, nHash = 0n, pow = 1n;
  for (let i = 0; i < m; i++) {
    hHash = (hHash * BigInt(BASE) + BigInt(haystack.charCodeAt(i))) % MOD;
    nHash = (nHash * BigInt(BASE) + BigInt(needle.charCodeAt(i)))   % MOD;
    if (i < m - 1) pow = (pow * BigInt(BASE)) % MOD;
  }
  for (let i = 0; i <= n - m; i++) {
    if (hHash === nHash && haystack.slice(i, i + m) === needle) return i;
    if (i < n - m) {
      hHash = (hHash - BigInt(haystack.charCodeAt(i)) * pow % MOD + MOD) % MOD;
      hHash = (hHash * BigInt(BASE) + BigInt(haystack.charCodeAt(i + m))) % MOD;
    }
  }
  return -1;
}
```

Rolling hash: remove the leftmost char, multiply by base, add the new right char.

### Drill (2 min)
Why compare strings character-by-character even after hashes match? *(Hint: hash collisions. Two different strings can have the same hash. Hash match is a "probably equal"; confirm with a real compare.)*

**Deep dive (later):** [dsa/ds-js/strings.md](../dsa/ds-js/strings.md)

---

## 9. Design Pattern — DAO (Data Access Object)

### Intent
**Wrap a single table or document collection behind one class.** The DAO knows the SQL/Mongo syntax; the rest of the app doesn't.

### When you'd use it
- Layered backends. Service calls DAO; DAO calls JDBC/JPA/Mongo.
- Anytime you want to keep raw query syntax (SQL strings, joins) out of business code.

### Smallest working example
```java
class UserDao {
    private final JdbcTemplate jdbc;
    UserDao(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    public Optional<User> findById(long id) {
        return jdbc.query(
            "SELECT id, email, name FROM users WHERE id = ?",
            (rs, i) -> new User(rs.getLong("id"), rs.getString("email"), rs.getString("name")),
            id
        ).stream().findFirst();
    }

    public long insert(String email, String name) {
        var key = new GeneratedKeyHolder();
        jdbc.update(con -> {
            var ps = con.prepareStatement(
                "INSERT INTO users(email, name) VALUES (?, ?)",
                Statement.RETURN_GENERATED_KEYS);
            ps.setString(1, email); ps.setString(2, name);
            return ps;
        }, key);
        return key.getKey().longValue();
    }
}
```

### DAO vs Repository
- **DAO** = "talks to a table." Thin wrapper.
- **Repository** = "talks to a *collection of domain objects*." May aggregate across tables.
- In day-to-day code, the line is blurry. Many teams just call them all `XxxRepository`.

### Drill (1 min)
Why hide SQL inside a DAO instead of inline in the service? *(Hint: tests, swapping storage, and reusability. Also, SQL changes don't ripple through business code.)*

**Deep dive (later):** [design-patterns/java/java-specific/dao.md](../design-patterns/java/java-specific/dao.md)

---

## 10. DevOps — Databases in Prod

### Why this exists
A database in production has different needs than your laptop. Five things you must handle: **backups, replication, pooling, slow queries, deployments**.

### The five must-haves
1. **Backups & restore.**
   - Take logical (`pg_dump`) AND physical (snapshot) backups.
   - **Test restore monthly.** Untested backups don't exist.
   - Keep multiple ages: daily, weekly, monthly.

2. **Replication.** Stream WAL to a hot standby. If primary dies, promote standby. RDS / Cloud SQL do this for you with multi-AZ.

3. **Connection pooling.** PgBouncer (Postgres) or ProxySQL (MySQL). Without it, app restarts overwhelm the DB with connection storms.

4. **Slow query monitoring.** Enable `pg_stat_statements` (Postgres) or slow query log (MySQL). Look at the top 10 queries by total time daily.

5. **Schema deployments.** Migrations must be:
   - **Backward compatible** — old app must work against new schema (deploy in two steps).
   - **Online** — `CREATE INDEX CONCURRENTLY` to avoid table locks.
   - **Reversible** — every migration has a tested `down`.

### Smallest concrete example
```bash
# Daily backup
pg_dump -Fc -d app > backup-$(date +%F).dump

# Restore test (on staging!)
pg_restore -d app_restored backup-2026-05-19.dump

# Top slow queries
psql -c "SELECT query, calls, mean_exec_time
         FROM pg_stat_statements
         ORDER BY mean_exec_time DESC LIMIT 10;"

# Online index
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

### Drill (2 min)
Your boss asks "if the DB dies right now, how much data do we lose?" What's the answer if your backup is "daily at 2am"? *(Hint: up to 24 hours. To do better, set up **continuous WAL archiving** (point-in-time recovery, PITR). Then RPO can be seconds.)*

**Deep dive (later):** [devops/24-databases-in-prod.md](../devops/24-databases-in-prod.md)

---

## End-of-day checklist

- [ ] Angular: I can write a `trigger` with `:enter` and `:leave`
- [ ] Node: I can describe one case where gRPC beats REST
- [ ] Spring: I can list the three Spring cache annotations
- [ ] MongoDB: I added a `pre('save')` hook and know which methods it skips
- [ ] Postgres: I know what a FDW does
- [ ] HLD: I can name three streaming delivery guarantees
- [ ] LLD: I sketched a pub-sub broker with a topic → subscribers map
- [ ] DSA: I can hash a rolling window for Rabin-Karp
- [ ] DP: I know the DAO-vs-Repository distinction (loosely)
- [ ] DevOps: I can list the five must-haves for prod databases

**If you remember just one thing today:** **performance comes from doing less work**. Cache (don't recompute), gRPC (don't send fat JSON), animations on the GPU (don't reflow), aggregation `$match` early (don't carry junk through stages), indexes for queries (don't scan).

**Tomorrow:** SSR + hydration, Docker for Node, Spring scheduling, Mongo change streams, Postgres replication, multi-region, job scheduler LLD, bit tricks, Service Layer pattern, and DevSecOps.
