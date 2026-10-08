# Day 23 — Directives, micro-services, and big data

> **Today's goal:** write your own DOM behavior in Angular, learn the four patterns every microservice talks about, listen to Kafka in Spring, and shape a billion-event pipeline.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Custom directives | 15m |
| 2 | Node.js | Microservices patterns | 15m |
| 3 | Spring Boot | Kafka with Spring Boot | 15m |
| 4 | MongoDB | Aggregation optimization | 10m |
| 5 | Postgres | Extensions (pg_trgm, uuid-ossp) | 10m |
| 6 | HLD | Big data pipeline | 12m |
| 7 | LLD | Ride-sharing (Uber/Lyft) | 12m |
| 8 | DSA | Union Find (connected components) | 15m |
| 9 | Design Pattern | Repository | 8m |
| 10 | DevOps | Azure basics | 10m |

---

## 1. Angular — Custom Directives

### Why this exists
Components are big — they own their template. Sometimes you want to **add behavior** to an existing element: a tooltip on hover, an autofocus on load, a click outside listener. That's a **directive** — a class that attaches to an existing DOM element.

### The idea in plain English
A directive is a "personality trait" you stick on an element. A component is a "person with body and clothes." `<button>` is a button — but `<button appConfetti>` is a button that throws confetti on click. The button's HTML is unchanged; the directive added behavior.

Two helpers do the heavy lifting:
- `@HostBinding` — set a property/class/style on the host element.
- `@HostListener` — listen to an event on the host element.

### Smallest working example
```typescript
import { Directive, ElementRef, HostBinding, HostListener } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  standalone: true,
})
export class HighlightDirective {
  @HostBinding('style.background') bg = '';

  @HostListener('mouseenter') onEnter() { this.bg = 'yellow'; }
  @HostListener('mouseleave') onLeave() { this.bg = ''; }
}

// usage: <p appHighlight>Hover me</p>
```

`HostBinding` syncs `bg` to the element's inline style. `HostListener` wires DOM events to methods. No template, no children — pure behavior.

### Drill (2 min)
Difference between `*ngIf` and `appHighlight`? *(Hint: `*ngIf` is a **structural** directive — it adds/removes DOM nodes (the `*` is sugar for `<ng-template>`). `appHighlight` is an **attribute** directive — same DOM, different behavior.)*

**Deep dive (later):** [angular/custom-directives.md](../angular/custom-directives.md)

---

## 2. Node.js — Microservices Patterns

### Why this exists
A monolith becomes hard to deploy as teams grow: one team's bug blocks everyone's release. **Microservices** split the app into independently deployed services. That creates new problems — how do they find each other? How does the outside world talk to many services? Patterns solve these.

### The four patterns to know
1. **API Gateway** — one front door for clients. Routes to internal services, handles auth, rate limits, retries. Hides internal structure.
2. **Service discovery** — services aren't at fixed IPs. A registry (Consul, Eureka, K8s DNS) tells `OrderService` how to reach `PaymentService` today.
3. **Circuit breaker** — when `PaymentService` is down, stop hammering it. Fail fast for 30 seconds, then probe. (Library: `opossum` in Node, Resilience4j in Java.)
4. **Saga (vs two-phase commit)** — distributed transactions. Instead of locking many DBs, chain compensating actions: "if step 3 fails, undo step 2, then undo step 1."

### The big picture
```
[Client] ─▶ [API Gateway] ─▶ [Auth] ─┐
                                      ├─▶ Orders ──▶ Payments ──▶ Email
                                      │     │
                                      │     └─▶ Inventory
                                      └─▶ Catalog
              service discovery: K8s DNS / Consul
              circuit breaker on every outgoing call
```

### Tiny circuit breaker in Node
```javascript
const CircuitBreaker = require('opossum');

async function callPayment(orderId) {
  const res = await fetch(`http://payments/charge/${orderId}`);
  if (!res.ok) throw new Error('payment failed');
  return res.json();
}

const breaker = new CircuitBreaker(callPayment, {
  timeout: 3000,                // each call max 3s
  errorThresholdPercentage: 50, // open after 50% failures
  resetTimeout: 30_000,         // try again after 30s
});

breaker.fallback(() => ({ status: 'queued_for_retry' }));

await breaker.fire('order-123');
```

### Drill (2 min)
Your `OrderService` makes 3 HTTP calls per request: Catalog, Payment, Email. Catalog is slow today and adds 800ms. Why is this a problem and what's the fix? *(Hint: sequential calls compound latency. Parallelize independent calls (`Promise.all`), add timeouts, and protect each with a circuit breaker.)*

**Deep dive (later):** [nodejs/22-microservices-and-grpc.md](../nodejs/22-microservices-and-grpc.md)

---

## 3. Spring Boot — Kafka with Spring Boot

### Why this exists
Kafka is the de-facto durable message log. Producers append messages to topics; consumers read at their own pace. Spring Boot makes both ends one annotation each.

### The idea in plain English
A Kafka topic is a **never-ending log file**. Producers append to the end. Consumers read forward, remembering their position (the *offset*). Multiple consumers can read the same topic without bothering each other. Messages persist for days — replay is free.

Spring's `KafkaTemplate` sends; `@KafkaListener` receives. Configuration goes in `application.yml`.

### Smallest working example
```yaml
spring:
  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      key-deserializer:   org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties.spring.json.trusted.packages: "com.example.events"
```

```java
@Service
class OrderEventPublisher {
    private final KafkaTemplate<String, OrderPlaced> template;
    OrderEventPublisher(KafkaTemplate<String, OrderPlaced> template) { this.template = template; }
    void publish(OrderPlaced e) {
        template.send("orders", e.orderId(), e);
    }
}

@Service
class OrderEventConsumer {
    @KafkaListener(topics = "orders", groupId = "order-service")
    void onOrder(OrderPlaced event) {
        // process the event
    }
}
```

The `groupId` is key: every consumer in the same group splits the partitions. Different groups each read the full stream.

### Drill (2 min)
Your consumer processes a message but crashes before committing the offset. What happens after restart? *(Hint: Kafka re-delivers from the last committed offset → the message is processed again. Your handler must be **idempotent** (safe to run twice).)*

**Deep dive (later):** [spring-next/08-spring-boot-advanced.md § Apache Kafka with Spring Boot](../spring-next/08-spring-boot-advanced.md#apache-kafka-with-spring-boot)

---

## 4. MongoDB — Aggregation Optimization

### Why this exists
`aggregate([...])` is Mongo's pipeline language: stage 1 → stage 2 → stage 3. Stage order matters massively for performance. The same logic can run in 30ms or 30s.

### The big rule
**Filter early. Sort late. Project last.** Get rid of documents and fields as soon as possible so later stages do less work.

| Stage | When to put it | Why |
|---|---|---|
| `$match` | First, if possible | Uses indexes; cuts data volume early |
| `$project` | After all filters | Reduce field width before expensive ops |
| `$sort` | After `$match` | Smaller set to sort |
| `$lookup` | After `$match` on the local collection | Avoid joining rows you'll throw away |
| `$limit` | After `$sort` | Sort then take top N |

### Smallest working example
```javascript
// ❌ SLOW: lookup runs over all orders, then we filter
db.orders.aggregate([
  { $lookup: { from: 'users', localField: 'userId', foreignField: '_id', as: 'user' } },
  { $match: { status: 'shipped' } },
  { $project: { total: 1, 'user.name': 1 } }
]);

// ✅ FAST: filter first → lookup runs on a small set
db.orders.aggregate([
  { $match: { status: 'shipped' } },                                  // uses index on status
  { $lookup: { from: 'users', localField: 'userId', foreignField: '_id', as: 'user' } },
  { $project: { total: 1, 'user.name': 1 } }
]);
```

Use `explain('executionStats')` to see what's actually happening.

### Drill (2 min)
Your pipeline has `$sort` then `$match`. How would you rewrite it? *(Hint: swap — `$match` first to shrink the input, then `$sort` on the smaller set. The result is the same; the cost drops.)*

**Deep dive (later):** [mongodb/15-aggregation-tuning.md](../mongodb/15-aggregation-tuning.md)

---

## 5. Postgres — Extensions

### Why this exists
Postgres has a plugin system. Without changing core, you add new types, functions, even indexes. Two extensions you'll meet again and again: **`uuid-ossp`** (generate UUIDs) and **`pg_trgm`** (fuzzy text search).

### The idea in plain English
An extension is a bundle of SQL + C code that adds features. `CREATE EXTENSION foo;` installs it for a database. They live in shared libraries shipped with Postgres or available from the community.

- **`uuid-ossp`** — generate UUID values. Useful for primary keys that don't expose row counts to clients.
- **`pg_trgm`** — splits strings into trigrams (three-letter substrings) and indexes them. Lets `ILIKE '%searh%'` use an index instead of scanning every row, and adds similarity (`%`) operators for fuzzy matching.

### Smallest working example
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- UUIDs as primary keys
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  buyer_name TEXT
);

-- Fuzzy index on buyer_name
CREATE INDEX idx_invoices_buyer_trgm ON invoices USING gin (buyer_name gin_trgm_ops);

-- Fuzzy queries (typo-tolerant)
SELECT id, buyer_name, similarity(buyer_name, 'Acmme Corp') AS score
FROM invoices
WHERE buyer_name % 'Acmme Corp'   -- % = "is similar to"
ORDER BY score DESC LIMIT 10;
```

Without `pg_trgm`, `WHERE buyer_name ILIKE '%acmme%'` scans the whole table. With the gin trigram index, it uses the index.

### Drill (2 min)
Should every column be a UUID instead of a `BIGSERIAL`? *(Hint: no. UUIDs are 16 bytes vs 8 for bigint, and random UUIDs hurt B-tree write locality. Use UUIDs when you need globally unique IDs across services or to avoid exposing counts; otherwise `BIGSERIAL` is cheaper.)*

**Deep dive (later):** [postgres/16-extensions.md](../postgres/16-extensions.md)

---

## 6. High-Level Design — Big Data Pipeline

### Why this exists
"Track every page view from millions of users and let analysts query it." A single Postgres can't keep up. The canonical answer: a **streaming pipeline** — ingest, process, store.

### The big picture
```
  [ App / SDK ]
       │   events
       ▼
   [ Kafka ]   ←─ durable, replayable buffer (days of history)
       │
   ┌───┴───┐
   ▼       ▼
[ Flink ]  [ Spark batch ]
  (stream)   (hourly)
   │           │
   ▼           ▼
[ ClickHouse / BigQuery / Snowflake ]   ←─ analyst's query engine
       ▲
       │
     Looker / Metabase
```

### The five things to mention
1. **Kafka** decouples producers from consumers. App keeps writing even if the warehouse is down.
2. **Partition by key.** Events for the same user go to the same partition → ordering is preserved per user.
3. **Stream processor (Flink / Kafka Streams)** for low-latency metrics (sessionization, anomaly detection). **Batch (Spark)** for heavy daily aggregations.
4. **Warehouse vs OLTP.** Postgres = transactions. BigQuery / Snowflake / ClickHouse = columnar, built for `SELECT count(*) FROM events WHERE ...` over billions of rows.
5. **Schema registry.** Avro/Protobuf schemas centrally stored so producers and consumers agree on field types.

### Drill (2 min)
Why partition by user_id instead of random? *(Hint: events for the same user land on the same partition, so a consumer sees them in order. Random partitioning breaks per-user ordering — bad for sessionization.)*

**Deep dive (later):** [system-design/high-level-design/23-big-data-pipeline.md](../system-design/high-level-design/23-big-data-pipeline.md)

---

## 7. Low-Level Design — Ride Sharing (Uber)

### The classes
- `User` (rider) and `Driver`.
- `Trip` — rider, driver, pickup, drop-off, status, fare.
- `Location` (lat, lon). Drivers report this every few seconds.
- `Matcher` — finds nearby drivers given a request.

### The hard parts
1. **Find nearest drivers fast.** Iterating through all drivers is O(N) per request. Use a **geo-index**: Redis `GEOADD`/`GEOSEARCH`, or H3/S2 cells. A query "drivers within 2 km of (lat, lon)" returns in milliseconds.

2. **Dispatch.** Send the ride request to the top 5 drivers. First to accept wins. Use a short timeout (15s) per driver, then move on.

3. **Trip state machine.**
   ```
   REQUESTED → ACCEPTED → ARRIVED → IN_PROGRESS → COMPLETED
                  │
                  └─→ CANCELLED
   ```
   Each transition is a row update + an event published to Kafka.

4. **Surge pricing.** Per region (e.g., H3 cell), compute `demand / supply`. Multiply base fare. Done with a stream job over Kafka.

### Smallest skeleton
```java
class MatchingService {
    Trip request(User rider, Location pickup, Location drop) {
        var nearby = locationIndex.search(pickup, 2_000);   // 2 km
        var top    = nearby.stream().sorted(byEtaTo(pickup)).limit(5).toList();
        return dispatcher.offerToDrivers(rider, top, pickup, drop);
    }
}
```

### Drill (2 min)
What happens to a trip when the driver's app crashes mid-ride? *(Hint: locations stop updating. Server detects no heartbeat for N seconds, flags the trip for support, lets rider end it from their side. Don't auto-cancel — they may just be in a tunnel.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Union Find (Connected Components)

### Why this exists
"How many friend groups are there?" "Are A and B in the same network?" These are graph problems where you only care about connectivity, not paths. **Union-Find** (also called **DSU** — Disjoint Set Union) answers them in nearly O(1).

### The idea in plain English
Picture islands. Each node starts as its own island. `union(a, b)` builds a bridge — they're now the same island. `find(a)` returns the island's ID. Two nodes are connected iff `find(a) == find(b)`.

Two optimizations make it near-constant:
- **Path compression** — when you `find(a)`, point every node on the path directly to the root.
- **Union by rank/size** — when merging, hang the smaller tree under the larger root.

### Smallest working example
```javascript
class DSU {
  constructor(n) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.rank = new Array(n).fill(0);
  }
  find(x) {
    if (this.parent[x] !== x) this.parent[x] = this.find(this.parent[x]);
    return this.parent[x];
  }
  union(a, b) {
    const ra = this.find(a), rb = this.find(b);
    if (ra === rb) return false;       // already same set
    if (this.rank[ra] < this.rank[rb]) this.parent[ra] = rb;
    else if (this.rank[ra] > this.rank[rb]) this.parent[rb] = ra;
    else { this.parent[rb] = ra; this.rank[ra]++; }
    return true;
  }
}

// Number of connected components
function countComponents(n, edges) {
  const dsu = new DSU(n);
  let components = n;
  for (const [a, b] of edges) if (dsu.union(a, b)) components--;
  return components;
}
```

### Drill (3 min)
`n = 5`, `edges = [[0,1],[1,2],[3,4]]`. How many components? *(Hint: 0-1-2 merge into one, 3-4 merge into another, 5th node is alone. Answer: 3. Wait — n=5 means nodes 0..4, so 0-1-2 and 3-4 → 2 components.)*

**Deep dive (later):** [dsa/ds-js/graphs.md](../dsa/ds-js/graphs.md) · [dsa/ds-java/graphs.md](../dsa/ds-java/graphs.md)

---

## 9. Design Pattern — Repository

### Intent
**Put all data access for one entity behind one interface.** The rest of the app doesn't know if data lives in Postgres, Mongo, an HTTP API, or memory.

### When you'd use it
- Almost any backend with a database. The repo hides SQL/Mongo queries.
- When you want to swap the storage layer (in-memory for tests, real DB for prod).
- When you want testable services — pass a mock repo, no DB needed.

### Smallest working example
```java
interface UserRepository {
    Optional<User> findById(long id);
    User save(User u);
    List<User> findByEmail(String email);
}

class JpaUserRepository implements UserRepository {
    private final EntityManager em;
    JpaUserRepository(EntityManager em) { this.em = em; }

    public Optional<User> findById(long id) {
        return Optional.ofNullable(em.find(User.class, id));
    }
    public User save(User u) { em.persist(u); return u; }
    public List<User> findByEmail(String email) {
        return em.createQuery("SELECT u FROM User u WHERE u.email = :e", User.class)
                 .setParameter("e", email).getResultList();
    }
}

class InMemoryUserRepository implements UserRepository {
    private final Map<Long, User> store = new HashMap<>();
    public Optional<User> findById(long id) { return Optional.ofNullable(store.get(id)); }
    public User save(User u) { store.put(u.id(), u); return u; }
    public List<User> findByEmail(String email) {
        return store.values().stream().filter(u -> u.email().equals(email)).toList();
    }
}
```

Spring Data takes it further — define an interface, Spring generates the implementation at startup.

### Drill (1 min)
What's the difference between Repository and DAO? *(Hint: they overlap. Classic distinction — DAO is closer to a table; Repository is closer to a *collection* of domain objects and may join multiple tables. In practice, many codebases use the words interchangeably.)*

**Deep dive (later):** [design-patterns/java/java-specific/repository.md](../design-patterns/java/java-specific/repository.md)

---

## 10. DevOps — Azure Basics

### Why this exists
Many enterprises use Azure because they already use Microsoft (Office 365, AD). Knowing the four core services covers most interviews and onboarding to a Microsoft shop.

### The four to know
| Azure | What it is | AWS equivalent |
|---|---|---|
| **VM** (Virtual Machines) | virtual servers | EC2 |
| **Blob Storage** | object storage | S3 |
| **App Service** | managed web apps (PaaS) | Elastic Beanstalk |
| **AKS** (Azure Kubernetes Service) | managed K8s | EKS |

### Two Azure-isms
1. **Resource Groups.** Resources live in groups for easy lifecycle and cost management. `az group delete --name dev-rg` nukes everything in the group.
2. **Entra ID (formerly Azure AD).** Identity provider tightly integrated with corporate AD. Used for SSO and managed identities (Azure's version of IAM roles).

### Smallest working example
```bash
# Login
az login

# Resource group
az group create --name dev-rg --location eastus

# A VM
az vm create --resource-group dev-rg --name web1 \
  --image Ubuntu2204 --admin-username azureuser \
  --generate-ssh-keys

# A storage account + container
az storage account create --name myappstg123 --resource-group dev-rg --sku Standard_LRS
az storage container create --name uploads --account-name myappstg123

# App Service (Linux, Node 20)
az appservice plan create --name myplan --resource-group dev-rg --is-linux --sku B1
az webapp create --name my-app-1234 --plan myplan --resource-group dev-rg \
  --runtime "NODE:20-lts"
```

### Identity / IAM model
- **Managed identities** — Azure's version of "no keys in code." Attach to a resource (VM, App Service); apps inside use them to call Azure APIs.
- **RBAC roles** — `Reader`, `Contributor`, `Owner`, or fine-grained ones — assigned at subscription/resource-group/resource scope.

### Drill (2 min)
Your App Service needs to read from a Blob container. How do you avoid embedding a connection string? *(Hint: enable a managed identity on the App Service, give it `Storage Blob Data Reader` on the storage account, use `DefaultAzureCredential` in code. No secrets.)*

**Deep dive (later):** [devops/17-cloud-azure-basics.md](../devops/17-cloud-azure-basics.md)

---

## End-of-day checklist

- [ ] Angular: I can build a directive with `@HostListener` and `@HostBinding`
- [ ] Node: I can name the four microservice patterns and one tool for each
- [ ] Spring: I can produce and consume Kafka messages with one annotation each
- [ ] MongoDB: I know to put `$match` before `$lookup`
- [ ] Postgres: I know what `pg_trgm` solves
- [ ] HLD: I can sketch a Kafka → stream processor → warehouse pipeline
- [ ] LLD: I can explain how Uber dispatches a ride
- [ ] DSA: I implemented Union-Find with path compression
- [ ] DP: I can explain Repository in one sentence
- [ ] DevOps: I can map three Azure services to AWS

**If you remember just one thing today:** most "scale" patterns boil down to two ideas — **decouple** (Kafka, gateway, repository) and **shard** (partition by key, geo-index). Both let you grow one part of the system without dragging the rest.

**Tomorrow:** animations, gRPC, Spring caching, Mongoose hooks, FDW, real-time streaming, pub-sub LLD, KMP, DAO pattern, and databases in production.
