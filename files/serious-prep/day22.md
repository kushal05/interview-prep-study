# Day 22 — Live streams: subscriptions, sockets, change feeds, and distributed storage

> **Today's goal:** clean up RxJS subscriptions, open a two-way socket, expose health metrics, and learn how Postgres can stream every change.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | async pipe & subscription management | 15m |
| 2 | Node.js | WebSockets | 15m |
| 3 | Spring Boot | Spring Boot Actuator | 15m |
| 4 | MongoDB | Validators | 10m |
| 5 | Postgres | Logical decoding (CDC, Debezium) | 10m |
| 6 | HLD | Distributed storage | 12m |
| 7 | LLD | Movie booking (BookMyShow) | 12m |
| 8 | DSA | Segment tree (range sum) | 15m |
| 9 | Design Pattern | Visitor | 8m |
| 10 | DevOps | GCP basics | 10m |

---

## 1. Angular — Async Pipe & Subscription Management

### Why this exists
RxJS observables are like newspaper subscriptions: if you don't cancel them, they keep "delivering" forever — even after the component is destroyed. That's a memory leak. The `async` pipe and `takeUntilDestroyed` make leaks (almost) impossible.

### The idea in plain English
A manual `subscribe()` is a phone line you opened. You must hang up (`unsubscribe`) when done. The `async` pipe is a smart secretary: it picks up the phone when the component mounts and hangs up automatically when the component leaves the page. As a bonus, it triggers change detection on every new value — perfect with `OnPush`.

When you *do* need to subscribe manually (because you have to act in code), pair it with `takeUntilDestroyed(this.destroyRef)` — Angular 16+ unsubscribes on destroy for you.

### Smallest working example
```typescript
// ✅ Best — async pipe, no subscribe at all
@Component({
  template: `<div>{{ user$ | async | json }}</div>`,
})
class UserCard {
  user$ = inject(UserApi).getCurrent();   // returns Observable<User>
}

// ✅ Fine — manual subscribe with auto-cleanup
@Component({ /* ... */ })
class Page {
  private destroyRef = inject(DestroyRef);
  constructor(api: UserApi) {
    api.getCurrent()
       .pipe(takeUntilDestroyed(this.destroyRef))
       .subscribe(u => console.log(u));
  }
}
```

The first version has zero ceremony. The second is needed when the value isn't displayed (e.g., logging, mutation).

### Drill (2 min)
Your component subscribes in `ngOnInit` but never unsubscribes. You navigate away and back many times. What grows? *(Hint: subscriptions accumulate — each destroyed component left a live listener. Memory and CPU climb. Fix with `takeUntilDestroyed` or the async pipe.)*

**Deep dive (later):** [angular/rxjs-and-async.md](../angular/rxjs-and-async.md)

---

## 2. Node.js — WebSockets

### Why this exists
HTTP is request/response: client asks, server answers, done. For chat, live scores, collaborative editing — you need the server to push to the client whenever it wants. **WebSocket** opens a single TCP connection that stays alive both ways.

### The idea in plain English
HTTP = sending postcards. Every message is a fresh envelope with all the headers. WebSocket = a phone call: dial once, talk in both directions until someone hangs up. Lower overhead, lower latency, real-time push.

`ws` is the bare library. `Socket.IO` adds rooms, reconnection, fallbacks (long-polling for old browsers). Pick `ws` for raw control, `Socket.IO` for productivity.

### Smallest working example
```javascript
// server.js
const { WebSocketServer } = require('ws');
const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws) => {
  ws.send('welcome');                     // server → client push
  ws.on('message', (data) => {
    // broadcast to everyone
    for (const client of wss.clients) client.send(`echo: ${data}`);
  });
});
```

```javascript
// browser
const ws = new WebSocket('ws://localhost:8080');
ws.onopen    = () => ws.send('hi');
ws.onmessage = (e) => console.log(e.data);    // "welcome", "echo: hi"
```

One TCP connection, two directions, no polling.

### Drill (2 min)
At 100k concurrent users, a single Node process can't handle all sockets. What scales? *(Hint: run many Node processes behind a load balancer with **sticky sessions** (so a user stays on one node), and use Redis pub/sub to broadcast across nodes.)*

**Deep dive (later):** [nodejs/21-websockets-and-realtime.md](../nodejs/21-websockets-and-realtime.md)

---

## 3. Spring Boot — Spring Boot Actuator

### Why this exists
Once your app is in production, you need answers: is it alive? Is it ready to take traffic? What's the memory? What are the slow endpoints? Adding all that by hand is busywork. **Actuator** is a Spring module that exposes `/actuator/*` endpoints with health, metrics, env, info, threads — for free.

### The idea in plain English
Actuator is your app's **dashboard wired in from day one**. Kubernetes asks `/actuator/health` to decide if the pod is healthy. Prometheus scrapes `/actuator/prometheus` for graphs. You don't write any of that — you add the dependency and configure which endpoints to expose.

Three endpoints matter most:
- `/actuator/health` — `UP` or `DOWN`. Used by load balancers and K8s liveness probes.
- `/actuator/metrics` — counters, timers, gauges (HTTP latency, DB connections, JVM memory).
- `/actuator/info` — your app's build version, commit hash.

### Smallest working example
```xml
<!-- pom.xml -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
  endpoint.health.show-details: when_authorized
  health.probes.enabled: true     # adds /health/liveness, /health/readiness
```

Now `GET /actuator/health` returns `{ "status": "UP" }`. Add a Prometheus dep and `/actuator/prometheus` exposes thousands of metrics.

### Drill (2 min)
Your K8s pod is restarting in a loop because the readiness probe fails. What endpoint should you point the probe at? *(Hint: `/actuator/health/readiness` — it returns `UP` only once Spring is fully started and all `ReadinessIndicator`s are healthy. `/health/liveness` is for "is the JVM alive at all.")*

**Deep dive (later):** [spring-next/03-spring-boot-fundamentals.md § 7. Actuator](../spring-next/03-spring-boot-fundamentals.md#7-actuator) (metrics/tracing: [08 § 2. Spring Cloud Overview](../spring-next/08-spring-boot-advanced.md#2-spring-cloud-overview))

---

## 4. MongoDB — Validators

### Why this exists
"Schema-less" lets you put `email: "x"` or `email: 42` in the same collection. That's freedom — and a footgun. **Validators** add server-side checks so bad documents are rejected at insert time. With Mongoose (Node ORM), you also get them at the application layer.

### The idea in plain English
A validator is a contract. "Every user document must have an `email` that's a string and matches an email regex; `age` is a number between 0 and 150." Mongo will reject inserts/updates that violate the rules. Mongoose adds shortcuts: `required`, `min`, `max`, `match`, `enum`, custom functions.

There are two layers:
- **Mongoose schema validation** — runs in your Node app. Fast feedback, easy to write.
- **Mongo `$jsonSchema` validator** — runs in the database. Still works if someone bypasses your app.

### Smallest working example
```javascript
// Mongoose
const userSchema = new mongoose.Schema({
  email: { type: String, required: true, match: /^\S+@\S+\.\S+$/ },
  age:   { type: Number, min: 0, max: 150 },
  role:  { type: String, enum: ['user', 'admin'], default: 'user' },
});

const User = mongoose.model('User', userSchema);
await User.create({ email: 'not-an-email' });
// → ValidationError: email does not match /^\S+@\S+\.\S+$/
```

```javascript
// Server-side, no Mongoose required
db.runCommand({
  collMod: 'users',
  validator: { $jsonSchema: {
    bsonType: 'object',
    required: ['email'],
    properties: {
      email: { bsonType: 'string', pattern: '^\\S+@\\S+\\.\\S+$' },
      age:   { bsonType: 'int', minimum: 0, maximum: 150 }
    }
  } }
});
```

### Drill (2 min)
You add a `required: true` rule on `phone` to an existing collection. Old documents have no phone. What happens? *(Hint: existing documents stay (validation runs on **insert/update** by default). Your next update on an old doc may fail. Use `validationLevel: 'moderate'` to apply only to docs already valid, or backfill.)*

**Deep dive (later):** [mongodb/14-validators-and-schema.md](../mongodb/14-validators-and-schema.md)

---

## 5. Postgres — Logical Decoding (Change Data Capture)

### Why this exists
"When a row in Postgres changes, send an event to Kafka." Sounds simple. Doing it without race conditions or polling is hard. **Logical decoding** is Postgres reading its own write-ahead log (WAL) and turning each `INSERT/UPDATE/DELETE` into a stream you can subscribe to. **Debezium** is the popular tool that does this for you.

### The idea in plain English
Every Postgres change is written to a journal called the WAL (Write-Ahead Log) before the row is updated on disk. Logical decoding **reads the journal as events**: "row 42 in users changed `email` from X to Y at time T." Debezium connects to Postgres as a replica, consumes that stream, and publishes the events to Kafka — that's **Change Data Capture (CDC)**.

Why is this great? You decouple writers from consumers. Your app just writes to Postgres. Analytics, search indexing, audit logs all consume the change stream. No dual writes, no missed events.

### Smallest working example
```sql
-- 1. enable logical replication (postgresql.conf)
-- wal_level = logical

-- 2. create a publication (what tables to stream)
CREATE PUBLICATION orders_pub FOR TABLE orders, customers;

-- 3. create a replication slot
SELECT pg_create_logical_replication_slot('debezium_slot', 'pgoutput');
```

```yaml
# Debezium connector (simplified)
name: pg-connector
config:
  connector.class: io.debezium.connector.postgresql.PostgresConnector
  database.hostname: db
  publication.name: orders_pub
  slot.name: debezium_slot
  topic.prefix: pg
```

A row update on `orders` now becomes a Kafka message on topic `pg.public.orders` containing the old and new row.

### Drill (2 min)
You created a replication slot for Debezium but never start the consumer. A week later disk is full. Why? *(Hint: an active slot **prevents WAL cleanup** until the consumer catches up. No consumer = WAL accumulates forever. Always monitor slot lag and drop unused slots.)*

**Deep dive (later):** [postgres/15-logical-decoding-and-cdc.md](../postgres/15-logical-decoding-and-cdc.md)

---

## 6. High-Level Design — Distributed Storage

### Why this exists
A single machine can't hold your data, can't survive disk failure, and can't serve users in another continent quickly. **Distributed storage** splits and copies data across many machines. The two pillars: **partitioning** (where does a row live?) and **replication** (how many copies?).

### The idea in plain English
- **Partitioning (sharding)** — split data by a key. E.g., users with `id % 16 == 3` live on shard 3. Solves "data too big for one node."
- **Replication** — each shard has copies on multiple nodes. If one dies, another takes over. Solves "node can die."
- **Replication factor (RF)** — usually 3. Three copies means you can lose two nodes and still serve.
- **Consistent hashing** — instead of `id % N` (which reshards everything when N changes), hash the key onto a ring; each node owns a slice. Adding a node moves only `1/N` of the data.

### The big picture
```
   client → coordinator → [shard 3] → primary  ──┐
                                                 │ replicate
                                          ├──▶ replica A
                                          └──▶ replica B
```

Write to primary, replicate to A and B. **Quorum** writes succeed if W out of RF nodes ack; quorum reads need R nodes; `R + W > RF` gives strong consistency for read-after-write.

### Drill (2 min)
RF = 3. You set `W = 1` (one ack to succeed). What happens if the primary acks then dies before replicating? *(Hint: the write is lost. With `W = 2` (or quorum = 2), you need at least one replica before acking — survives a primary crash.)*

**Deep dive (later):** [system-design/high-level-design/22-distributed-storage.md](../system-design/high-level-design/22-distributed-storage.md)

---

## 7. Low-Level Design — Movie Booking (BookMyShow)

### The classes
- `Movie`, `Cinema`, `Hall`, `Show` (movie + hall + time).
- `Seat` (hall, row, number, type).
- `ShowSeat` (per show, status: `AVAILABLE / LOCKED / BOOKED`).
- `Booking`, `User`, `Payment`.

### The hard parts
1. **Hold seats during checkout.** A user clicks 3 seats and starts paying. Until they pay, those seats are unavailable to others but not yet "booked." Lock them with a **TTL** (e.g., 5 min). If they pay → `BOOKED`. If they don't → expire back to `AVAILABLE`.

2. **Concurrency.** Two users grab seat B7 at the same time. Use a Redis lock or a DB unique constraint:
   ```sql
   UPDATE show_seats SET status='LOCKED', locked_by=:u, locked_until=now()+'5 minutes'
     WHERE show_id=:s AND seat_id=:t
     AND (status='AVAILABLE' OR locked_until < now());
   -- rows updated == 0 → someone else got it
   ```

3. **Payment + booking atomically.** Wrap "mark seats BOOKED + insert Booking" in one transaction. If payment fails, release locks.

### Smallest skeleton
```java
record Lock(long userId, long showId, Set<Long> seatIds, Instant until) {}

class BookingService {
    Lock hold(long userId, long showId, Set<Long> seatIds) {
        if (!seatRepo.tryLock(showId, seatIds, userId, Duration.ofMinutes(5)))
            throw new SeatsTakenException();
        return new Lock(userId, showId, seatIds, Instant.now().plus(Duration.ofMinutes(5)));
    }

    @Transactional
    Booking confirm(Lock lock, Payment payment) {
        if (!seatRepo.stillLockedBy(lock.userId(), lock.showId(), lock.seatIds()))
            throw new LockExpiredException();
        seatRepo.markBooked(lock.showId(), lock.seatIds());
        return bookingRepo.save(new Booking(...));
    }
}
```

### Drill (2 min)
A user holds seats and closes the tab without paying. How are the seats released? *(Hint: a background job (or query filter) treats any lock with `locked_until < now()` as effectively `AVAILABLE`. Cheap and correct.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Segment Tree (Range Sum)

### Why this exists
"Given an array, answer many `sum(l, r)` queries while values change." A loop is O(n) per query. A **segment tree** answers each query in O(log n) and supports updates in O(log n).

### The idea in plain English
Imagine pre-summing chunks. The leaves are the original values. Each internal node stores the sum of its children. Querying `sum(l, r)` walks down, taking whole subtrees inside `[l, r]` and only going deeper at the edges.

```
              [0..7]=36
           /            \
       [0..3]=10      [4..7]=26
       /     \        /     \
     [0..1] [2..3]  [4..5] [6..7]
     ...
```

Updating index 5 walks one root-to-leaf path — `log n` nodes change.

### Smallest working example
```javascript
class SegTree {
  constructor(arr) {
    this.n = arr.length;
    this.t = new Array(4 * this.n).fill(0);
    this._build(arr, 1, 0, this.n - 1);
  }
  _build(a, node, l, r) {
    if (l === r) { this.t[node] = a[l]; return; }
    const m = (l + r) >> 1;
    this._build(a, node*2, l, m);
    this._build(a, node*2+1, m+1, r);
    this.t[node] = this.t[node*2] + this.t[node*2+1];
  }
  query(ql, qr, node=1, l=0, r=this.n-1) {
    if (qr < l || r < ql) return 0;          // outside
    if (ql <= l && r <= qr) return this.t[node]; // fully inside
    const m = (l + r) >> 1;
    return this.query(ql, qr, node*2, l, m)
         + this.query(ql, qr, node*2+1, m+1, r);
  }
  update(i, v, node=1, l=0, r=this.n-1) {
    if (l === r) { this.t[node] = v; return; }
    const m = (l + r) >> 1;
    if (i <= m) this.update(i, v, node*2, l, m);
    else        this.update(i, v, node*2+1, m+1, r);
    this.t[node] = this.t[node*2] + this.t[node*2+1];
  }
}
```

### Drill (2 min)
Why size the array `4 * n` instead of `2 * n`? *(Hint: a segment tree on an n that's not a power of 2 can be deeper than `2n`. `4n` is the safe upper bound.)*

**Deep dive (later):** [dsa/ds-js/trees-advanced.md](../dsa/ds-js/trees-advanced.md)

---

## 9. Design Pattern — Visitor

### Intent
**Add new operations to a class hierarchy without modifying the classes.** Especially useful for tree/AST traversal.

### When you'd use it
- A compiler walking an Abstract Syntax Tree to do (1) type checking, (2) optimization, (3) code generation. Each is a separate visitor.
- Computing total size of files/folders, then again for "permissions report."
- Exporting a document to HTML, then to PDF — two visitors over the same tree.

### Smallest working example
```java
interface Shape { <T> T accept(Visitor<T> v); }
class Circle    implements Shape { double r; public <T> T accept(Visitor<T> v) { return v.visit(this); } }
class Square    implements Shape { double s; public <T> T accept(Visitor<T> v) { return v.visit(this); } }

interface Visitor<T> { T visit(Circle c); T visit(Square s); }

class AreaVisitor implements Visitor<Double> {
    public Double visit(Circle c) { return Math.PI * c.r * c.r; }
    public Double visit(Square s) { return s.s * s.s; }
}

class JsonVisitor implements Visitor<String> {
    public String visit(Circle c) { return "{\"type\":\"circle\",\"r\":" + c.r + "}"; }
    public String visit(Square s) { return "{\"type\":\"square\",\"s\":" + s.s + "}"; }
}
```

Adding `Triangle` is awkward — every visitor must add `visit(Triangle)`. Adding a new operation (e.g., `PerimeterVisitor`) is easy — no shape changes.

### Drill (1 min)
When is Visitor a **bad** fit? *(Hint: when the shape hierarchy changes often (you'll edit every visitor) but operations are stable. Use polymorphic methods on shapes instead.)*

**Deep dive (later):** [design-patterns/common/behavioral/visitor.md](../design-patterns/common/behavioral/visitor.md)

---

## 10. DevOps — GCP Basics

### Why this exists
GCP is the third-largest cloud after AWS and Azure. The names are different but the ideas map cleanly. Knowing five GCP services makes you cloud-fluent.

### The five to know
| GCP | What it is | AWS equivalent |
|---|---|---|
| **Compute Engine** | virtual machines | EC2 |
| **Cloud Storage** | buckets of files | S3 |
| **Cloud SQL** | managed Postgres/MySQL | RDS |
| **GKE** (Google Kubernetes Engine) | managed K8s | EKS |
| **Cloud Functions** | serverless functions | Lambda |

### Two GCP-isms worth knowing
1. **Projects.** Everything in GCP lives in a *project*. Billing, IAM, resources all scope to a project. You can have many projects (per-team, per-env).
2. **gcloud CLI.** Authenticate once: `gcloud auth login`. Set project: `gcloud config set project my-proj`. All commands run against that project.

### Smallest working example
```bash
# Spin up a VM
gcloud compute instances create web-1 \
  --machine-type=e2-medium \
  --zone=asia-south1-a \
  --image-family=debian-12 \
  --image-project=debian-cloud

# Upload to a bucket
gsutil mb -l asia-south1 gs://my-uploads
gsutil cp report.pdf gs://my-uploads/

# Create a Postgres instance
gcloud sql instances create app-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=asia-south1
```

### IAM model
- **Members** — users, service accounts, groups.
- **Roles** — bundles of permissions (`roles/storage.objectViewer`).
- **Bindings** — grant a role to a member on a resource.

```bash
gcloud projects add-iam-policy-binding my-proj \
  --member='serviceAccount:web@my-proj.iam.gserviceaccount.com' \
  --role='roles/storage.objectViewer'
```

### Drill (2 min)
Your Cloud Run service needs to read from a Cloud Storage bucket. How do you give it access? *(Hint: assign a **service account** to the Cloud Run service, and grant that service account `roles/storage.objectViewer` on the bucket. No keys checked into code.)*

**Deep dive (later):** [devops/16-cloud-gcp-basics.md](../devops/16-cloud-gcp-basics.md)

---

## End-of-day checklist

- [ ] Angular: I know why `async` pipe is preferred over manual `subscribe`
- [ ] Node: I can explain "phone call vs postcard" for WebSocket vs HTTP
- [ ] Spring: I can name what `/actuator/health` is used for
- [ ] MongoDB: I added a Mongoose `required` + `match` rule
- [ ] Postgres: I can describe what a logical replication slot does
- [ ] HLD: I can explain `R + W > RF`
- [ ] LLD: I sketched seat locking with a TTL
- [ ] DSA: I can answer range-sum in O(log n) with a segment tree
- [ ] DP: I named two use cases for the Visitor pattern
- [ ] DevOps: I can map three GCP services to AWS equivalents

**If you remember just one thing today:** "real-time" and "always correct" both come from the same idea — keep an event stream of changes. WebSockets stream events to clients; CDC streams DB changes to consumers; replication streams writes to replicas.

**Tomorrow:** custom directives, microservices, Spring + Kafka, aggregation tuning, Postgres extensions, big-data pipelines, ride-sharing LLD, union-find, Repository pattern, and Azure basics.
