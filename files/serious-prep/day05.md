# Day 5 — Forms, streams, aggregations, and queues

> **Today's goal:** see how Angular handles simple forms, how Node streams big data, how Spring magically configures itself, and how messages flow between systems.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Forms: Template-Driven (ngModel) | 15m |
| 2 | Node.js | Streams & Buffers (backpressure idea) | 15m |
| 3 | Spring Boot | Spring Boot Auto-configuration | 15m |
| 4 | MongoDB | Aggregation Pipeline ($match, $group, $project) | 10m |
| 5 | Postgres | Index types (B-tree, GIN, BRIN) | 10m |
| 6 | HLD | Message Queues (Kafka vs RabbitMQ) | 12m |
| 7 | LLD | Schema/Data Modeling (think before you code) | 12m |
| 8 | DSA | Trees (Inorder/Preorder/Postorder Traversal) | 15m |
| 9 | Design Pattern | Prototype | 8m |
| 10 | DevOps | Docker Compose (multi-container apps) | 10m |

---

## 1. Angular — Forms: Template-Driven

### Why this exists
Most apps need forms — login, signup, search. Angular gives you two flavors of forms: **template-driven** (today) and **reactive** (tomorrow). Template-driven is the simpler one: you put your form fields in HTML and Angular wires up the binding automatically.

### The idea in plain English
With template-driven forms, the **template is the source of truth**. You add `ngModel` to each input, give it a `name`, and Angular tracks the value and validity for you. HTML attributes like `required` and `minlength` act as validators.

It's quick to write but awkward when the form gets complex (dynamic fields, async validation, custom errors). For those, use reactive forms.

### Smallest working example
```typescript
import { Component } from '@angular/core';
import { FormsModule, NgForm } from '@angular/forms';

@Component({
    standalone: true,
    imports: [FormsModule],
    template: `
      <form #f="ngForm" (ngSubmit)="submit(f)">
        <input name="email" [(ngModel)]="model.email" required email>
        <input name="password" [(ngModel)]="model.password" required minlength="8">
        <button type="submit" [disabled]="f.invalid">Sign up</button>
      </form>
    `,
})
export class SignupComponent {
    model = { email: '', password: '' };

    submit(f: NgForm) {
        if (f.valid) console.log(this.model);
    }
}
```

`[(ngModel)]` is two-way binding — typing updates `model.email`. `#f="ngForm"` gives the template a reference to the form's state (`f.valid`, `f.dirty`). The `name` attribute is required so Angular can register each control.

### Drill (2 min)
You forget the `name` attribute on an input with `[(ngModel)]`. Why doesn't the form know about that control? *(Hint: the parent form (`ngForm`) builds an internal `FormGroup` keyed by `name`. Without a name, your control isn't registered — `f.value` won't include it.)*

**Deep dive (later):** [angular/forms-template-driven.md](../angular/forms-template-driven.md)

---

## 2. Node.js — Streams & Buffers

### Why this exists
What if you need to process a 10 GB log file? Reading it all into memory (`readFile`) blows up. **Streams** let you process data in small chunks as it arrives, so memory stays bounded.

### The idea in plain English
A stream is water from a tap — you don't fill a bucket and then carry it; you handle each cup as it pours. Node's streams come in four flavors:

- **Readable** — emits chunks (a file being read, an HTTP request body).
- **Writable** — accepts chunks (a file being written, an HTTP response).
- **Duplex** — both (a network socket).
- **Transform** — reads, processes, writes (gzip compression).

**Backpressure** is the heart of stream correctness. If the producer (writing) is faster than the consumer (reading), the buffer between them fills up. The producer needs to pause until the consumer catches up. The `pipeline` helper handles this automatically.

### Smallest working example
```javascript
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';

// Stream a big file through gzip into a compressed output — no whole file in memory
await pipeline(
    createReadStream('big.log'),
    createGzip(),
    createWriteStream('big.log.gz')
);

console.log('Done.');
```

`pipeline` connects each stream's output to the next stream's input, handles backpressure, and propagates errors. Always use `pipeline` over raw `.pipe()` — `pipe()` doesn't forward errors and can leak file descriptors.

### Drill (2 min)
You write `const data = await readFile('huge.log', 'utf8');` to process a 10 GB file. What happens? *(Hint: `readFile` loads the entire file into memory at once. On a 10 GB file, you'll run out of memory (or hit V8's ~512 MB string limit). Use a stream and process chunk by chunk.)*

**Deep dive (later):** [nodejs/08-streams-and-buffers.md](../nodejs/08-streams-and-buffers.md)

---

## 3. Spring Boot — Auto-configuration

### Why this exists
You add `spring-boot-starter-web` to your project. Magically, you get an embedded Tomcat, JSON support, sensible error handling, and dozens of beans. You didn't write any config. How? **Auto-configuration.**

### The idea in plain English
Spring Boot has many `@Configuration` classes baked into its starter libraries. Each one is marked with **conditions** — "only activate me if class X is on the classpath" or "only if the user hasn't defined their own bean of this type."

At startup, Spring Boot scans for these configs and applies the ones whose conditions match. That's the magic: opinionated defaults that you can override by simply providing your own bean.

### Smallest working example
```java
// Simplified version of what Spring Boot does for DataSource
@Configuration
@ConditionalOnClass(DataSource.class)              // only if DataSource is on the classpath
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                       // only if user hasn't defined one
    @ConfigurationProperties("spring.datasource")  // bind YAML to the bean
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

If you want to override:
```java
@Bean
public DataSource dataSource() {
    // Your own config — Spring Boot's @ConditionalOnMissingBean steps aside
    return new HikariDataSource(myConfig);
}
```

The pattern is: Boot provides a sensible default; you override by defining your own bean.

### Drill (2 min)
You add `spring-boot-starter-data-jpa` but no `DataSource` config. The app starts with an in-memory H2 database. Why? *(Hint: the JPA starter pulls in H2 as a default. Boot's auto-config sees no user `DataSource` bean and creates one pointing at the embedded H2. Add real DB config (or remove H2 from dependencies) to swap it.)*

**Deep dive (later):** [spring-detailed/PART-14-spring-boot-internals.md](../spring-detailed/PART-14-spring-boot-internals.md)

---

## 4. MongoDB — Aggregation Pipeline

### Why this exists
`find()` is great for "fetch matching documents." But what if you want to *group, sum, sort, join*? That's the **aggregation pipeline** — Mongo's analytical query engine.

### The idea in plain English
Documents flow through a series of **stages**, each one transforming the stream. Think of it as the Unix pipe (`grep | sort | uniq`) for Mongo data.

The three most common stages:
- **`$match`** — filter docs. Same syntax as `find()`. Put it first to leverage indexes.
- **`$group`** — group by a key and compute aggregates (`$sum`, `$avg`, `$max`).
- **`$project`** — pick or rename fields, or compute new ones.

Other useful stages: `$sort`, `$limit`, `$lookup` (left-outer-join with another collection), `$unwind` (explode an array into one doc per element).

### Smallest working example
```javascript
db.orders.aggregate([
  // Stage 1: only completed orders this year
  { $match: { status: "completed", year: 2026 } },

  // Stage 2: group by user, sum totals, count orders
  { $group: {
      _id: "$user_id",
      revenue: { $sum: "$total" },
      orderCount: { $sum: 1 }
  } },

  // Stage 3: top 5 by revenue
  { $sort:  { revenue: -1 } },
  { $limit: 5 }
]);
```

The result is the top 5 users by revenue with their order counts — produced in one query.

### Drill (2 min)
Why does putting `$match` *first* matter so much for performance? *(Hint: `$match` early can use indexes and reduces the working set for later stages. A `$match` placed *after* `$group` operates on already-aggregated rows in memory — no index, more work upstream.)*

**Deep dive (later):** [mongodb/05-aggregation-pipeline.md](../mongodb/05-aggregation-pipeline.md)

---

## 5. Postgres — Index types (B-tree, GIN, BRIN)

### Why this exists
Yesterday: indexes exist. Today: Postgres has *multiple* index types and picking the right one matters more than people realize. The default works most of the time, but specialized data shapes have specialized indexes.

### The idea in plain English
- **B-tree** (default) — balanced tree. Use for equality, ranges (`<`, `BETWEEN`), and sort. 95% of indexes are B-trees.
- **GIN** (Generalized Inverted Index) — for "composite" values where each row has many lookup keys. Use for `jsonb` queries, arrays, and full-text search.
- **BRIN** (Block Range Index) — tiny summary index. Use on huge tables where data is naturally ordered (a timestamp column with append-only inserts).

A simple way to choose: ask "what's in the column?"
- Plain values, queried with `=` or `<`? B-tree.
- JSON, arrays, or text searched with keywords? GIN.
- Billions of rows ordered by time? BRIN.

### Smallest working example
```sql
-- B-tree (the default — `USING BTREE` is implicit)
CREATE INDEX idx_users_email ON users (email);

-- GIN for a jsonb column with mixed keys
CREATE INDEX idx_users_data_gin ON users USING GIN (data);
-- Now: SELECT ... WHERE data @> '{"role":"admin"}' uses the index

-- BRIN for a huge time-ordered events table
CREATE INDEX idx_events_time_brin ON events USING BRIN (created_at);
-- A B-tree on this column might be 30 GB; BRIN might be 1 MB
```

### Drill (2 min)
You have a `jsonb` column and run `WHERE data->>'role' = 'admin'`. Why is GIN useful here, and what would B-tree miss? *(Hint: a regular B-tree indexes only specific column values. GIN indexes every key/value pair *inside* the JSON, so any `data->>'X' = Y` query can use it. B-tree only works if you index the specific expression `((data->>'role'))`.)*

**Deep dive (later):** [postgres/06-indexes.md](../postgres/06-indexes.md)

---

## 6. High-Level Design — Message Queues (Kafka vs RabbitMQ)

### Why this exists
You don't always want services calling each other directly. If service A talks to B synchronously and B is slow, A is slow too. A **message queue** decouples them: A drops a message into the queue, B picks it up when ready. The two big choices are **Kafka** and **RabbitMQ**.

### The idea in plain English
- **Kafka** — a distributed *log*. Messages are written to disk in append-only files (partitions). Consumers read by offset. Messages stay around for days or weeks; you can replay them.
- **RabbitMQ** — a traditional *message broker*. Messages are pushed to consumers and acknowledged. Once acknowledged, they're gone.

**When to pick which:**

| Need | Pick |
|---|---|
| Replay events for analytics or recovery | Kafka |
| Microservice work queue with retries | RabbitMQ |
| Massive throughput (millions/sec) | Kafka |
| Per-message routing (topic patterns) | RabbitMQ |
| Don't want to operate a broker | Cloud equivalent (SQS, Pub/Sub) |

Kafka is great when you treat events as facts that may be useful later. RabbitMQ is great when each message is a task that's done once.

### A picture
```
Producer ──message──> [ Queue / Topic ] ──message──> Consumer(s)

Kafka:        partitioned log on disk, consumers track their offset
RabbitMQ:     queue in memory, consumer acks each message
```

### Drill (2 min)
You want all events about user 42 to be processed *in order*. How does Kafka give you that? *(Hint: use `user_id` as the partition key. Kafka hashes the key; same key always goes to the same partition; messages within a partition are strictly ordered. Different users may parallelize across partitions.)*

**Deep dive (later):** [system-design/high-level-design/05-message-queues-and-streaming.md](../system-design/high-level-design/05-message-queues-and-streaming.md)

---

## 7. Low-Level Design — Schema/Data Modeling

### Why this exists
Software is rewritten; data lasts. A bad schema haunts every query, every migration, every report for years. Spend time on it *before* you write the first INSERT.

### The idea in plain English
The basics of relational modeling:

- **One-to-many** — the foreign key lives on the "many" side. (`books.author_id` references `authors.id`.)
- **Many-to-many** — use a **join table** with two foreign keys. (`books_tags` with `book_id` and `tag_id`.)
- **Surrogate keys** (auto-incrementing IDs) are usually better than **natural keys** (email, SSN) — they're stable when business rules change.

For documents (Mongo), the big question is **embed vs reference**:
- **Embed** when data is read together and the child is small and bounded (an order's line items).
- **Reference** when the child is large, queried independently, or unbounded (a user's event log).

A practical habit: every table benefits from `created_at`, `updated_at`, and a soft-delete flag (`deleted_at NULL`) so you can audit and recover.

### Smallest working example
```sql
-- One-to-many
CREATE TABLE authors (id BIGINT PRIMARY KEY, name TEXT);
CREATE TABLE books (
    id BIGINT PRIMARY KEY,
    author_id BIGINT REFERENCES authors(id),
    title TEXT,
    created_at TIMESTAMPTZ DEFAULT now(),
    deleted_at TIMESTAMPTZ NULL
);

-- Many-to-many (a book has many tags, a tag tags many books)
CREATE TABLE tags (id BIGINT PRIMARY KEY, name TEXT UNIQUE);
CREATE TABLE books_tags (
    book_id BIGINT REFERENCES books(id),
    tag_id  BIGINT REFERENCES tags(id),
    PRIMARY KEY (book_id, tag_id)
);
```

The join table has its own primary key (the pair of foreign keys), and `ON DELETE CASCADE` can be added so removing a book also removes its tag links.

### Drill (2 min)
You're modeling a blog with posts and comments. Each post can have thousands of comments. Embed or reference? *(Hint: reference. A Mongo doc has a 16 MB cap; thousands of comments could blow that. More importantly, you'd want to paginate, edit, and delete comments independently — that's far easier in a separate collection with a `post_id` field.)*

**Deep dive (later):** [system-design/low-level-design/06-schema-and-data-modeling.md](../system-design/low-level-design/06-schema-and-data-modeling.md)

---

## 8. DSA — Trees (Inorder/Preorder/Postorder Traversal)

### The problem
A binary tree has nodes, each with a value, a left child, and a right child. There are three classic ways to visit every node — they differ in *when* the parent is visited relative to its children.

```
        1
       / \
      2   3
     / \
    4   5
```

- **Preorder** (Parent → Left → Right): `1, 2, 4, 5, 3`
- **Inorder** (Left → Parent → Right): `4, 2, 5, 1, 3`
- **Postorder** (Left → Right → Parent): `4, 5, 2, 3, 1`

### The idea
For each traversal, you write a recursive function with three steps in different orders. The position of "visit the parent" is what changes.

**Why each is useful:**
- **Preorder**: copying or serializing a tree (parent first so the reader can rebuild).
- **Inorder**: on a BST (binary search tree), inorder gives values in **sorted order**.
- **Postorder**: deleting a tree (free children before parent), evaluating expression trees.

### Smallest working example
```javascript
class TreeNode {
    constructor(val) { this.val = val; this.left = null; this.right = null; }
}

function inorder(root, out = []) {
    if (root === null) return out;
    inorder(root.left, out);
    out.push(root.val);           // visit between left and right
    inorder(root.right, out);
    return out;
}

function preorder(root, out = []) {
    if (root === null) return out;
    out.push(root.val);           // visit FIRST
    preorder(root.left, out);
    preorder(root.right, out);
    return out;
}
```

The only difference is where `out.push(root.val)` sits. The recursion structure is identical.

**Time:** O(n). **Space:** O(h) for the recursion stack (h = tree height).

### Drill (2 min)
On a BST, inorder produces sorted values. Why? *(Hint: a BST has the invariant `left < parent < right` at every node. Inorder visits left subtree, then parent, then right subtree — so by induction every left value comes before the parent, every right value after. The whole traversal is sorted.)*

**Deep dive (later):** [dsa/ds-js/trees-graphs.md](../dsa/ds-js/trees-graphs.md) · [dsa/ds-java/trees-graphs.md](../dsa/ds-java/trees-graphs.md)

---

## 9. Design Pattern — Prototype

### Intent
**Create new objects by cloning an existing one** instead of building from scratch. Useful when object creation is expensive (heavy initialization, DB lookups, etc.) and you want a near-copy with just a few changes.

### When you'd use it
- The object has lots of pre-configured state and you want a slightly different one.
- Construction is significantly more expensive than copying.
- You're working in a prototype-based language like JavaScript, where cloning is natural.

### Smallest working example
```javascript
const reportTemplate = {
    header: 'Quarterly Report',
    sections: ['Revenue', 'Expenses', 'Forecast'],
    metadata: { version: 1, classification: 'internal' },
};

// Shallow clone with override
const q1 = { ...reportTemplate, header: 'Q1 Report' };

// Deep clone (Node 17+, modern browsers)
const q2 = structuredClone(reportTemplate);
q2.header = 'Q2 Report';
q2.metadata.version = 2;       // doesn't affect the template
```

**Shallow vs deep clone:** spread (`{...obj}`) copies only top-level fields — nested objects are still shared by reference. `structuredClone` walks the whole structure.

In Java, the pattern usually uses a copy constructor or `Object.clone()`. In modern Java, records and copy methods (`withTitle(...)`) cover most cases.

### Drill (1 min)
What's the bug here? `const copy = { ...user }; copy.address.city = "Pune";` Does it affect the original `user`? *(Hint: yes. Spread is shallow — `copy.address` is the same object as `user.address`. Changing `copy.address.city` changes both. Use `structuredClone(user)` for a deep copy.)*

**Deep dive (later):** [design-patterns/common/creational/prototype.md](../design-patterns/common/creational/prototype.md)

---

## 10. DevOps — Docker Compose

### Why this exists
A real app is rarely one container. It's web + database + cache + maybe a queue. Running each with `docker run` is fiddly. **Docker Compose** describes the whole setup in one YAML file and brings everything up with `docker compose up`.

### The idea in plain English
A Compose file declares **services** (containers), **networks** (so they can talk to each other), and **volumes** (so DB data survives restarts). Each service gets a hostname matching its service name — your app connects to `postgres://db:5432`, not an IP.

`depends_on` controls start order. But just because the DB container started doesn't mean it's ready to accept connections — use a **healthcheck** and `condition: service_healthy` for real readiness.

### Smallest working example
```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      DATABASE_URL: postgres://db:5432/mydb
    depends_on:
      db: { condition: service_healthy }

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      retries: 5

volumes:
  pgdata:
```

```bash
docker compose up -d        # start everything in background
docker compose logs -f app  # tail logs
docker compose down         # stop and remove containers (volumes persist)
```

The `pgdata` named volume keeps the DB's data between restarts. Without it, `docker compose down` would wipe everything.

### Drill (2 min)
What's the difference between plain `depends_on: [db]` and `depends_on: { db: { condition: service_healthy } }`? *(Hint: plain `depends_on` only enforces start order — the app starts after the DB container starts, but the DB may not be ready to accept connections yet. Adding `condition: service_healthy` (with a `healthcheck` on the DB) makes Compose wait until the DB reports healthy.)*

**Deep dive (later):** [devops/05-docker-compose.md](../devops/05-docker-compose.md)

---

## End-of-day checklist

- [ ] Angular: I built a small `[(ngModel)]` form
- [ ] Node.js: I can explain backpressure using the "tap and bucket" idea
- [ ] Spring: I can describe what `@ConditionalOnMissingBean` lets users do
- [ ] MongoDB: I wrote a 3-stage aggregation pipeline
- [ ] Postgres: I can match B-tree / GIN / BRIN to a use case each
- [ ] HLD: I can pick Kafka vs RabbitMQ for one scenario
- [ ] LLD: I can model a many-to-many relationship with a join table
- [ ] DSA: I traced an inorder traversal of a small tree by hand
- [ ] DP: I know the difference between shallow and deep clone
- [ ] DevOps: I wrote a 2-service `compose.yml`

**If you remember just one thing today:** the aggregation pipeline (Mongo) and the stream pipeline (Node) are the same idea at different scales — data flowing through ordered stages, each transformation cheap to add or remove.

**Tomorrow:** reactive forms, EventEmitter, REST controllers, schema design patterns, the query planner, load balancers, concurrency, BST validation, Adapter pattern, and networking fundamentals.
