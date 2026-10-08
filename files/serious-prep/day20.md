# Day 20 — Going faster: performance, testing, and shopping carts

> **Today's goal:** make Angular skip wasted work, watch Node use CPU, learn the two main Spring test styles, and sketch the e-commerce stack everyone interviews on.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | OnPush performance | 15m |
| 2 | Node.js | Performance & profiling | 15m |
| 3 | Spring Boot | Spring Testing (@SpringBootTest, @WebMvcTest) | 15m |
| 4 | MongoDB | Atlas Search | 10m |
| 5 | Postgres | Connection pooling (PgBouncer) | 10m |
| 6 | HLD | E-commerce design | 12m |
| 7 | LLD | Notification system | 12m |
| 8 | DSA | Topological Sort (Course Schedule) | 15m |
| 9 | Design Pattern | Mediator | 8m |
| 10 | DevOps | Pulumi vs CloudFormation | 10m |

---

## 1. Angular — OnPush Performance

### Why this exists
By default, Angular re-checks **every** component after **every** event — a click, a timer, a fetch. For a small app that's fine. For a dashboard with 500 rows, it makes typing feel laggy. `OnPush` says "skip me unless something I care about changed."

### The idea in plain English
Default Angular = a teacher who re-grades every student's paper whenever any one student raises a hand. `OnPush` = "I'll only re-grade a student if their paper actually changed." A "change" is decided by **reference equality** — same object in memory or different object? `{a: 1}` and `{a: 1}` look identical but are two different objects, so OnPush sees a change. Mutating an existing object (`obj.a = 2`) keeps the same reference, so OnPush sees no change and skips the re-render.

That's the OnPush deal: **give me new objects, not modified ones**.

`trackBy` is the related trick for `*ngFor`: it tells Angular "this list item is the same one as before, don't destroy and recreate the DOM."

### Smallest working example
```typescript
@Component({
  selector: 'app-row',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `<div>{{ user.name }}</div>`,
})
export class RowComponent {
  @Input() user!: { id: number; name: string };
}

// Parent — the WRONG way (mutation, same reference)
this.user.name = 'New name';        // OnPush row does NOT update

// Parent — the RIGHT way (new reference)
this.user = { ...this.user, name: 'New name' };  // OnPush row updates
```

The spread `{ ...this.user, name: 'New name' }` creates a brand-new object, so Angular sees a different reference and re-renders.

### Drill (2 min)
You have `*ngFor="let p of products"`. Adding `trackBy: trackById` makes it noticeably faster. Why? *(Hint: without trackBy, Angular destroys and recreates every `<li>` whenever the array reference changes. With trackBy, it reuses DOM nodes whose `id` is unchanged.)*

**Deep dive (later):** [angular/performance-optimization.md](../angular/performance-optimization.md) · [angular/change-detection.md](../angular/change-detection.md)

---

## 2. Node.js — Performance & Profiling

### Why this exists
"It feels slow" isn't a bug report. You need numbers — which function ate the CPU, which call took 800ms. Node ships with tools for exactly that, and `clinic.js` is a popular wrapper that makes the output readable.

### The idea in plain English
A profiler is like a security camera on your code. It records what was running every few milliseconds. After a minute it tells you, "Function `parseInvoice` was on screen 42% of the time." That's your hotspot — the place worth optimizing. Don't guess. Don't optimize what isn't slow.

Two flavors:
- **CPU profile** — where time was spent computing.
- **Heap snapshot** — where memory is being held (useful for leaks).

Node's built-in `--prof` produces a raw V8 log. `clinic doctor` runs your app, watches it, and produces a friendly HTML report with charts.

### Smallest working example
```bash
# Built-in profiler
node --prof app.js
# After running, V8 leaves an isolate-XYZ.log file
node --prof-process isolate-XYZ.log > flat.txt

# Friendlier (npm i -g clinic autocannon)
clinic doctor -- node app.js
# Open http://localhost:3000 a few times, then Ctrl+C
# clinic opens a browser with CPU, memory, and event-loop graphs

# Generate load while profiling
autocannon -d 30 -c 50 http://localhost:3000
```

`clinic doctor` will tell you if the bottleneck is CPU, I/O, garbage collection, or the event loop being blocked.

### Drill (2 min)
Your `/report` endpoint takes 4s. CPU profile shows `JSON.parse` at 70%. What's the next step? *(Hint: are you parsing the same big JSON repeatedly? Cache the parsed object. Or — is it really JSON you need? A binary format like protobuf or CBOR may be much smaller.)*

**Deep dive (later):** [nodejs/20-performance-and-clustering.md](../nodejs/20-performance-and-clustering.md)

---

## 3. Spring Boot — Spring Testing

### Why this exists
A Spring app has many moving parts: controllers, services, repositories, a database. Testing every layer end-to-end is slow. Spring offers two main test styles: **`@SpringBootTest`** (start the whole app) and **`@WebMvcTest`** (only the web layer). Use the lightest one that proves what you want to prove.

### The idea in plain English
Imagine testing a car. You don't drive it across the country to know the brakes work — you put it on a dyno (test rig) that loads just the brake system. `@WebMvcTest` is the dyno: it loads only your controller and a fake (`MockMvc`) HTTP layer. Fast (under a second). `@SpringBootTest` is the full road test — it boots the entire `ApplicationContext`, including the database. Slower, but it's the only way to catch wiring problems between layers.

Rule of thumb: write **many** `@WebMvcTest`s, **few** `@SpringBootTest`s.

### Smallest working example
```java
// Fast — only the controller, repo is mocked
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mvc;
    @MockBean UserRepository repo;        // fake — we control its behavior

    @Test
    void getById_returns200() throws Exception {
        when(repo.findById(1L)).thenReturn(Optional.of(new User(1L, "Jane")));

        mvc.perform(get("/users/1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.name").value("Jane"));
    }
}
```

`MockMvc` lets you call the controller without a real HTTP server. `@MockBean` replaces the real `UserRepository` with a mock you can program.

### Drill (2 min)
Your `@WebMvcTest` for `OrderController` fails with "No qualifying bean of type EmailService." Why? *(Hint: `@WebMvcTest` only loads the web layer. Services aren't auto-included. Add `@MockBean EmailService emailService;` — or move the test to `@SpringBootTest` if you really need the full app.)*

**Deep dive (later):** [spring-next/07-spring-boot-testing.md](../spring-next/07-spring-boot-testing.md)

---

## 4. MongoDB — Atlas Search

### Why this exists
A `find({ title: /angular/i })` regex query is fine for 1,000 documents and miserable for 10 million — it scans every document. **Atlas Search** is Lucene (the search engine behind Elasticsearch) built into MongoDB Atlas. It maintains an inverted index so text queries return in milliseconds.

### The idea in plain English
A regular index is like a contents page (lookup by exact key). A **search index** is like a book's back-of-book index: "the word *angular* appears on pages 12, 88, 301." That structure (an *inverted index*) is what makes full-text search fast. Atlas Search lets you create one with a few lines of config — no separate Elasticsearch cluster to run.

It supports fuzzy matching (typos), synonyms, autocomplete, and ranking by relevance.

### Smallest working example
```javascript
// 1. In Atlas UI: create a search index named "default" on the `articles` collection.
// 2. Then query it:

db.articles.aggregate([
  {
    $search: {
      index: "default",
      text: {
        query: "angular onpush",
        path: ["title", "body"],
        fuzzy: { maxEdits: 1 }      // tolerates 1 typo (e.g., "anglar")
      }
    }
  },
  { $limit: 10 },
  { $project: { title: 1, score: { $meta: "searchScore" } } }
]);
```

`$search` must be the **first** stage in the pipeline. `score` ranks hits by relevance.

### Drill (2 min)
Users complain that searching for "iphone" doesn't find "iPhone 15" written with a capital P. What feature solves this? *(Hint: the default analyzer lowercases everything, so it should match. If it doesn't, you may have used `keyword` analyzer (exact match). Switch to the default `lucene.standard` analyzer.)*

**Deep dive (later):** [mongodb/12-atlas-search.md](../mongodb/12-atlas-search.md)

---

## 5. Postgres — Connection Pooling

### Why this exists
Each Postgres connection costs ~10MB of memory and a backend process. A web app that opens a fresh connection per request will exhaust the server at modest traffic. A **pool** keeps a small set of connections open and hands them out.

### The idea in plain English
A connection pool is a **taxi stand**. Instead of every passenger hailing a fresh cab from far away (slow), a few cabs wait at the stand and get reused. When a passenger is done, the cab returns. Fewer cabs serve more passengers.

Your app library (HikariCP for Java, `pg` for Node) already has a pool inside the app. But when you scale to many app instances, the database sees `instances × pool_size` connections. **PgBouncer** sits *between* your apps and Postgres and shares one big pool across all apps.

### Smallest working example
```ini
# pgbouncer.ini
[databases]
mydb = host=db.internal port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction          # release connection back when txn ends
max_client_conn = 5000           # apps can connect this many
default_pool_size = 25           # actual Postgres connections
```

Apps now connect to `pgbouncer:6432`. Postgres sees only 25 real connections even though 5,000 apps think they're talking directly.

`pool_mode = transaction` is the most common: a Postgres connection is reused as soon as the app commits its transaction.

### Drill (2 min)
Your app uses `SET search_path` once per session, then runs many queries. Will `transaction` pool mode work? *(Hint: no. In transaction mode, the next transaction may land on a different backend connection that doesn't have `search_path` set. Use `session` pool mode, or move the `SET` into each transaction.)*

**Deep dive (later):** [postgres/13-connection-pooling.md](../postgres/13-connection-pooling.md)

---

## 6. High-Level Design — E-commerce

### Why this exists
"Design Amazon" is the most common system-design prompt. You won't actually build Amazon, but you'll show how you think about catalog, cart, checkout, inventory, and order processing.

### The big picture
```
[ Browser / App ]
       │
       ▼
[  API Gateway  ] ── auth ── rate limit
       │
   ┌───┼─────────────┬─────────────┬──────────────┐
   ▼   ▼             ▼             ▼              ▼
Catalog  Cart      Checkout    Order Service   Payment
Service  Service   Service     (state machine) Service
   │      │           │            │              │
   ▼      ▼           ▼            ▼              ▼
ES/Mongo  Redis     Postgres    Postgres       Stripe
(search)  (fast)    (txn)       + Kafka        API
```

### Five things to mention in an interview
1. **Catalog** is read-heavy → cache aggressively (Redis), use a search index (Elastic/Atlas Search).
2. **Cart** is per-user and bursty → Redis with TTL is enough. No need for strong consistency.
3. **Checkout** is the heart of correctness → use a **transaction** to reserve inventory and create the order in one shot.
4. **Inventory** races are the classic bug → "reserve" stock when the cart enters checkout (with TTL), "commit" on payment success, "release" on failure or timeout.
5. **Eventual consistency for downstream** → emit an `OrderPlaced` event to Kafka. Email, analytics, recommendations, shipping all consume it. They don't have to be up for checkout to work.

### Drill (2 min)
Two users both buy "the last red sneaker, size 9" at the same millisecond. How do you guarantee only one succeeds? *(Hint: a row lock or atomic decrement on the inventory row inside the checkout transaction: `UPDATE inventory SET qty = qty - 1 WHERE sku = ? AND qty > 0` — only one transaction matches `qty > 0`.)*

**Deep dive (later):** [system-design/high-level-design/20-ecommerce.md](../system-design/high-level-design/20-ecommerce.md)

---

## 7. Low-Level Design — Notification System

### Why this exists
Almost every app needs to send messages — email, SMS, push, Slack. A good design lets you add a new channel without rewriting the rest. This is the **Strategy** pattern in disguise.

### The class skeleton
```java
interface NotificationChannel {
    void send(User user, Message msg);
}

class EmailChannel implements NotificationChannel { /* SES */ }
class SmsChannel   implements NotificationChannel { /* Twilio */ }
class PushChannel  implements NotificationChannel { /* FCM */ }

class NotificationService {
    private final Map<ChannelType, NotificationChannel> channels;

    public NotificationService(List<NotificationChannel> all) {
        // Spring injects every implementation it finds
        this.channels = all.stream().collect(toMap(
            c -> c.type(), c -> c));
    }

    public void notify(User user, Message msg, ChannelType type) {
        channels.get(type).send(user, msg);
    }
}
```

Adding WhatsApp tomorrow = a new class that implements `NotificationChannel`. Zero changes to `NotificationService`.

### Three real-world details
1. **Retries** — third-party APIs fail. Wrap `send` in a retry with exponential backoff, then push to a dead-letter queue.
2. **Templates** — body text varies per user. Store templates in DB; render with placeholders (`Hi {{name}}, your order {{id}}...`).
3. **User preferences** — `user.notificationPrefs.email = false` should suppress the channel.

### Drill (2 min)
You need to send the same message via email **and** SMS. How do you avoid sending twice if email succeeds but SMS times out and is retried? *(Hint: store a unique `(user_id, message_id, channel)` row when sent. Before sending, check if the row exists. This is **idempotency**.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Topological Sort (Course Schedule)

### The problem
You have `n` courses. Some courses have prerequisites (`[1, 0]` means "take 0 before 1"). Return `true` if it's possible to finish all courses, or list a valid order.

```
n = 4, prerequisites = [[1,0],[2,1],[3,2]]
Order: 0 → 1 → 2 → 3
```

If there's a **cycle** (you need A for B and B for A), it's impossible.

### The idea
Build a directed graph. A **topological order** is a linear order where every edge points forward. The standard algorithm (Kahn's) uses in-degree:

1. Compute in-degree for every node (how many arrows point at it).
2. Push all nodes with in-degree 0 into a queue.
3. Pop a node, add to result, decrement in-degree of its neighbors. If a neighbor's in-degree becomes 0, push it.
4. If you popped all `n` nodes, you have a valid order. Otherwise there's a cycle.

```javascript
function canFinish(n, prerequisites) {
  const graph = Array.from({ length: n }, () => []);
  const inDeg = new Array(n).fill(0);
  for (const [a, b] of prerequisites) {
    graph[b].push(a);            // edge b → a
    inDeg[a]++;
  }
  const q = [];
  for (let i = 0; i < n; i++) if (inDeg[i] === 0) q.push(i);
  let visited = 0;
  while (q.length) {
    const node = q.shift();
    visited++;
    for (const next of graph[node]) {
      if (--inDeg[next] === 0) q.push(next);
    }
  }
  return visited === n;
}
```

Time: O(V + E). Space: O(V + E).

### Drill (3 min)
Trace `canFinish(2, [[1,0],[0,1]])` by hand. *(Hint: in-degrees start as `[1, 1]`. The queue is empty. `visited` stays 0. Returns `false` — there's a cycle.)*

**Deep dive (later):** [dsa/ds-js/graphs.md](../dsa/ds-js/graphs.md) · [dsa/ds-java/graphs.md](../dsa/ds-java/graphs.md)

---

## 9. Design Pattern — Mediator

### Intent
**Let objects talk through a middleman instead of directly.** Reduces the spaghetti of N classes all knowing about each other.

### When you'd use it
- A chat room where users send messages — users talk to the *room*, not to each other.
- A form where field A enables field B which disables field C — fields talk to a *form controller*, not each other.
- An air-traffic control tower — planes talk to the tower, not plane-to-plane.

### Smallest working example
```java
interface ChatRoom {
    void send(String from, String msg);
    void register(User u);
}

class ChatRoomImpl implements ChatRoom {
    private final List<User> users = new ArrayList<>();
    public void register(User u) { users.add(u); }
    public void send(String from, String msg) {
        for (User u : users) if (!u.name.equals(from)) u.receive(from, msg);
    }
}

class User {
    final String name; final ChatRoom room;
    User(String name, ChatRoom room) { this.name = name; this.room = room; room.register(this); }
    void send(String msg)       { room.send(name, msg); }
    void receive(String from, String msg) { System.out.println(name + " got: " + msg); }
}
```

Each `User` knows only the room, not other users. Add a 100th user → no existing user changes.

### Drill (1 min)
What's the risk if the mediator grows too big? *(Hint: it becomes a god object — every change touches it. Split by feature, or graduate to an event bus.)*

**Deep dive (later):** [design-patterns/common/behavioral/mediator.md](../design-patterns/common/behavioral/mediator.md)

---

## 10. DevOps — Pulumi vs CloudFormation

### Why this exists
**Infrastructure as Code (IaC)** = declare your servers, databases, and networks in files, not click-by-click in a console. The two big questions: which tool, and what language?

### The idea in plain English
- **CloudFormation** — AWS-native, declarative YAML/JSON. You describe the desired state; AWS figures out the steps. Free, deeply integrated, but YAML for complex stacks gets painful (no loops, no real conditionals, awkward refs).
- **Pulumi** — you write real code in TypeScript, Python, Go, or C#. Loops, functions, packages, tests — everything you already know. Multi-cloud (AWS, GCP, Azure). Free for individuals, paid for teams.
- **Terraform** (worth knowing) — HCL, its own DSL, multi-cloud, huge community. Sits between the two in feel.

When to pick what:
| Need | Pick |
|---|---|
| AWS-only, want zero extra tools | CloudFormation |
| Want a real language with loops/tests | Pulumi |
| Multi-cloud with biggest community | Terraform |

### Smallest working example
```typescript
// Pulumi — TypeScript, real loops
import * as aws from "@pulumi/aws";

const buckets = ["uploads", "logs", "backups"].map(name =>
  new aws.s3.Bucket(name, { versioning: { enabled: true } })
);

export const bucketNames = buckets.map(b => b.id);
```

```yaml
# CloudFormation — pure YAML, repetition
Resources:
  Uploads: { Type: AWS::S3::Bucket, Properties: { VersioningConfiguration: { Status: Enabled } } }
  Logs:    { Type: AWS::S3::Bucket, Properties: { VersioningConfiguration: { Status: Enabled } } }
  Backups: { Type: AWS::S3::Bucket, Properties: { VersioningConfiguration: { Status: Enabled } } }
```

The Pulumi version scales naturally; the YAML grows linearly with every bucket.

### Drill (2 min)
Your team is comfortable with TypeScript and writes unit tests for everything. AWS-only. Which IaC tool fits best? *(Hint: Pulumi — same language, same testing tools. CloudFormation would force them into YAML.)*

**Deep dive (later):** [devops/14-iac-pulumi-vs-cloudformation.md](../devops/14-iac-pulumi-vs-cloudformation.md)

---

## End-of-day checklist

- [ ] Angular: I can explain why mutating an object breaks OnPush
- [ ] Node: I know what a CPU profile shows and one tool that produces one
- [ ] Spring: I can pick `@WebMvcTest` vs `@SpringBootTest` for a given case
- [ ] MongoDB: I can describe Atlas Search as an inverted index inside Mongo
- [ ] Postgres: I can explain why a pool is a "taxi stand"
- [ ] HLD: I named five components of an e-commerce backend
- [ ] LLD: I sketched a notification system with a `NotificationChannel` interface
- [ ] DSA: I solved Course Schedule with Kahn's algorithm
- [ ] DP: I can name one good use case for the Mediator pattern
- [ ] DevOps: I can pick Pulumi vs CloudFormation given a constraint

**If you remember just one thing today:** every "make it faster" technique is the same pattern — measure, find the hotspot, give up unnecessary work. OnPush skips re-renders, pooling skips connection setup, search indexes skip full scans.

**Tomorrow:** code splitting (lazy routes), Node clustering, Spring profiles, Mongo migrations, Postgres roles, payment systems, hotels, tries, Memento, and AWS basics.
