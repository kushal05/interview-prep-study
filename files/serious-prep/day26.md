# Day 26 — Reactive, background work, and search

> **Today's goal:** see how apps reach the world (i18n, accessibility), how they do work after the user clicks "Send" (background jobs), how they react to streams of data, and how search works under the hood. Big topics, small examples.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | i18n & Accessibility (a11y) | 15m |
| 2 | Node.js | Background Jobs (BullMQ) | 15m |
| 3 | Spring Boot | WebFlux / Reactive Spring | 15m |
| 4 | MongoDB | Indexing for Aggregation | 10m |
| 5 | Postgres | Materialized Views | 10m |
| 6 | HLD | Search Indexing (inverted index) | 12m |
| 7 | LLD | Cache (LRU + LFU) | 12m |
| 8 | DSA | Math algos (GCD, Sieve) | 15m |
| 9 | Design Pattern | DTO / VO | 8m |
| 10 | DevOps | Service Mesh (Istio) | 10m |

---

## 1. Angular — i18n & Accessibility (a11y)

### Why this exists
The web isn't just English speakers with good vision. **i18n** ("internationalization" — `i` + 18 letters + `n`) lets the same app show in Spanish, French, Hindi. **a11y** ("accessibility" — same naming trick) makes it usable for screen readers and keyboard-only users.

### The idea in plain English
Think of i18n as a **dictionary lookup**: every visible word gets a key, and at build time Angular swaps each key for the right translation. Accessibility is like adding **invisible labels** so a blind user's screen reader can announce what a button does — even though the button only shows an icon.

The two go together because both are about "your app should make sense to everyone, not just you on your laptop."

### Smallest working example
```html
<!-- i18n: mark text for translation -->
<h1 i18n="@@home.title">Welcome</h1>

<!-- a11y: a button with only an icon needs a label -->
<button aria-label="Close dialog" (click)="close()">
  <svg>...</svg>
</button>

<!-- a11y: use semantic tags so screen readers understand structure -->
<nav>...</nav>
<main>...</main>
```

`ng extract-i18n` pulls every `i18n` tag into a translation file. `aria-label` is read aloud by screen readers when no visible text exists.

### Drill (2 min)
Your app has a heart icon button that favorites an item. A blind user hears nothing when they tab to it. What do you add? *(Hint: `aria-label="Add to favorites"`. The icon alone tells the screen reader nothing.)*

**Deep dive (later):** [angular/i18n-and-accessibility.md](../angular/i18n-and-accessibility.md)

---

## 2. Node.js — Background Jobs (BullMQ)

### Why this exists
When a user uploads a photo, you don't want them waiting while you resize it, scan it for nudity, and email a notification — all in one HTTP request. Instead, return "uploaded!" immediately and **process the slow work in the background**. That's a **job queue**.

### The idea in plain English
Imagine a restaurant. The waiter takes your order (fast) and drops a slip into the kitchen (the queue). Cooks pick up slips when they're free. If a dish burns, the slip goes back into the queue and gets retried. **BullMQ** is Node's most popular kitchen — it uses Redis as the slip-board.

Two roles: a **producer** puts jobs in, a **worker** takes jobs out. They can be different processes, even different servers.

### Smallest working example
```javascript
const { Queue, Worker } = require('bullmq');
const connection = { host: 'localhost', port: 6379 };

// Producer (in your API handler)
const emailQueue = new Queue('emails', { connection });
await emailQueue.add('welcome', { to: 'jane@x.com' });

// Worker (a separate process)
new Worker('emails', async (job) => {
  console.log(`Sending to ${job.data.to}`);
  // if this throws, BullMQ retries automatically
}, { connection });
```

The API call returns instantly. The worker handles the slow email send. Failed jobs are retried with exponential backoff.

### Drill (2 min)
Your worker crashes mid-job. What happens to that job? *(Hint: BullMQ marks it as "stalled" and reassigns it to another worker. The job isn't lost because it lives in Redis, not in worker memory.)*

**Deep dive (later):** [nodejs/20-performance-and-clustering.md](../nodejs/20-performance-and-clustering.md)

---

## 3. Spring Boot — WebFlux / Reactive Spring

### Why this exists
Classic Spring MVC gives each HTTP request its own thread. With 10,000 concurrent requests you need 10,000 threads — expensive. **WebFlux** uses a small thread pool and an event loop (like Node.js) so a few threads can juggle thousands of requests by switching when one is waiting on I/O.

### The idea in plain English
Think of **Mono** as "a future single value" and **Flux** as "a future stream of values." Instead of "give me the user now," you say "here's what to do *when* the user arrives." The framework schedules everything for you.

Analogy: ordering food via app vs. standing at the counter. The counter (MVC) ties up your time. The app (WebFlux) pings you when ready — you do other things meanwhile.

### Smallest working example
```java
@RestController
public class UserController {
    private final UserRepository repo;
    public UserController(UserRepository r) { this.repo = r; }

    @GetMapping("/users/{id}")
    public Mono<User> getUser(@PathVariable String id) {
        return repo.findById(id);        // returns Mono<User> — no blocking
    }

    @GetMapping("/users")
    public Flux<User> all() {
        return repo.findAll();            // a stream of users
    }
}
```

Notice: no `User` returned directly. `Mono`/`Flux` describe **what will happen**. Spring subscribes to them and writes the response when data arrives.

### Drill (2 min)
Why is calling `.block()` on a `Mono` inside a WebFlux handler a mistake? *(Hint: it freezes the event-loop thread waiting — defeating the entire point. Chain operators like `.map()` and `.flatMap()` instead.)*

**Deep dive (later):** [spring-next/08-spring-boot-advanced.md — Async Processing](../spring-next/08-spring-boot-advanced.md#5-async-processing)

---

## 4. MongoDB — Indexing for aggregation

### Why this exists
Aggregation pipelines (`$match`, `$group`, `$sort`) can scan millions of documents. Without the right index, a "top 10 orders this month" query takes seconds instead of milliseconds.

### The idea in plain English
A **compound index** is like a phone book sorted by last name, then first name. If your query filters by `status = 'paid'` and then sorts by `createdAt DESC`, an index on `{ status: 1, createdAt: -1 }` lets MongoDB jump straight to the right rows in the right order — **no scan, no in-memory sort**.

Rule of thumb (the ESR rule): **Equality** fields first, **Sort** fields next, **Range** fields last.

### Smallest working example
```javascript
// The query
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $sort:  { createdAt: -1 } },
  { $limit: 10 }
]);

// The index that supports it
db.orders.createIndex({ status: 1, createdAt: -1 });

// Verify it's used
db.orders.aggregate([...]).explain("executionStats");
// Look for "IXSCAN" not "COLLSCAN"
```

`IXSCAN` = index scan (fast). `COLLSCAN` = full collection scan (slow). If you see `COLLSCAN`, your index is missing or unused.

### Drill (2 min)
You also start filtering by `customerId`. Where does it go in the compound index? *(Hint: it's another equality filter, so put it before the sort: `{ customerId: 1, status: 1, createdAt: -1 }`.)*

**Deep dive (later):** [mongodb/12-performance-tuning.md](../mongodb/12-performance-tuning.md)

---

## 5. Postgres — Materialized views

### Why this exists
Some queries are expensive (joins across 5 tables, aggregations over millions of rows) but the answer doesn't need to be live-to-the-second. A **materialized view** stores the *result* of a query like a snapshot table — instant reads, refresh on a schedule.

### The idea in plain English
A regular **view** is a saved query — it runs every time you read it. A **materialized view** is a saved *answer* — fast to read, but stale until you refresh. Think of it as cached SQL.

Use it for dashboards, reports, anything where "data from 5 minutes ago" is fine.

### Smallest working example
```sql
-- Create
CREATE MATERIALIZED VIEW daily_sales AS
SELECT date_trunc('day', created_at) AS day,
       SUM(amount) AS total
FROM   orders
GROUP  BY 1;

-- Read (fast — no joins, no aggregation at query time)
SELECT * FROM daily_sales ORDER BY day DESC LIMIT 30;

-- Refresh (rebuild the snapshot)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;
```

`CONCURRENTLY` keeps the view readable while refreshing (requires a unique index on the view).

### Drill (2 min)
Why not just put this on a hot dashboard with a regular query? *(Hint: aggregating millions of rows on every page load is slow and burns CPU. A materialized view does the work once per refresh window.)*

**Deep dive (later):** [postgres/18-performance-tuning.md](../postgres/18-performance-tuning.md)

---

## 6. HLD — Search Indexing (inverted index)

### Why this exists
"Find every document containing the word 'pizza'" — searching every document one by one is hopeless at scale. Search engines flip the data upside-down so they can answer in milliseconds. That flipped structure is the **inverted index**.

### The idea in plain English
A book has a table of contents (chapter → page). Search engines need the **opposite** — a word → list of pages it appears on. That's why it's called *inverted*. ElasticSearch, Lucene, Solr — all built on this idea.

To search "cheap pizza," look up "cheap" → docs {3, 9, 17}. Look up "pizza" → docs {3, 8, 17}. Intersect → {3, 17}. Done. No document scan needed.

### Smallest working example
```
Documents:
  doc1: "I love pizza"
  doc2: "Pizza is the best"
  doc3: "I love sushi"

Inverted index (after lowercase + tokenize):
  i      → [doc1, doc3]
  love   → [doc1, doc3]
  pizza  → [doc1, doc2]
  is     → [doc2]
  the    → [doc2]
  best   → [doc2]
  sushi  → [doc3]

Query "love pizza" → intersect [doc1, doc3] ∩ [doc1, doc2] = [doc1]
```

ElasticSearch shards this index across nodes for scale, ranks results with **TF-IDF** or **BM25**, and refreshes near-real-time.

### Drill (2 min)
Why is searching "the" useless and slow? *(Hint: "the" appears in almost every doc — the posting list is huge and brings no relevance. Engines skip such **stop words** during indexing.)*

**Deep dive (later):** [system-design/high-level-design/10-real-world-system-designs.md](../system-design/high-level-design/10-real-world-system-designs.md)

---

## 7. LLD — Cache (LRU + LFU)

### Why this exists
RAM is small. Disk and network are slow. A **cache** keeps the hottest items in RAM. But when it fills up, *which item do you kick out?* Two famous rules: **LRU** (Least Recently Used) and **LFU** (Least Frequently Used).

### The idea in plain English
- **LRU** — kick out whatever you haven't touched in the longest time. Bias toward **recent** activity.
- **LFU** — kick out whatever you've touched the *fewest* times. Bias toward **popular** items, even if popularity was last week.

Use LRU when access patterns shift fast (web sessions). Use LFU when some items are eternally hot (a celebrity's profile pic). LRU is the default in most caches because it's cheaper to implement.

### Smallest working example (LRU)
```java
// JDK gives you an LRU cache for free: LinkedHashMap with accessOrder=true
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int cap;
    public LRUCache(int cap) {
        super(cap, 0.75f, true);   // true = order by access, not insertion
        this.cap = cap;
    }
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> e) {
        return size() > cap;        // evict oldest when full
    }
}

LRUCache<String, String> c = new LRUCache<>(3);
c.put("a", "1"); c.put("b", "2"); c.put("c", "3");
c.get("a");                          // touches "a" — now "b" is oldest
c.put("d", "4");                     // evicts "b"
```

### Drill (2 min)
Your cache holds product pages. One product is being scraped by a bot 1000× a minute; real users hit other products. LRU or LFU? *(Hint: LRU. Under LFU the bot's product would dominate forever. LRU lets human traffic patterns win out.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Math algos: GCD & Sieve

### Why this exists
Two classics every interviewer expects you to know: finding the **greatest common divisor** and finding all **primes up to N**. They show up in number theory questions and as helpers in bigger problems.

### GCD — Euclid's algorithm
The trick: `gcd(a, b) = gcd(b, a % b)`. Stop when `b = 0`.

```java
int gcd(int a, int b) {
    while (b != 0) {
        int t = b;
        b = a % b;
        a = t;
    }
    return a;
}
// gcd(12, 18) = 6
```

Why it works: any number dividing both `a` and `b` also divides `a % b`. We shrink the problem each step. Time: O(log min(a, b)).

### Sieve of Eratosthenes — all primes up to N
Start with all numbers marked "prime." Walk up; for each prime found, **cross out its multiples**.

```java
boolean[] sieve(int n) {
    boolean[] isPrime = new boolean[n + 1];
    Arrays.fill(isPrime, true);
    isPrime[0] = isPrime[1] = false;
    for (int i = 2; i * i <= n; i++) {
        if (isPrime[i]) {
            for (int j = i * i; j <= n; j += i) isPrime[j] = false;
        }
    }
    return isPrime;
}
```

Time: **O(n log log n)** — basically linear in practice. Space: O(n).

### Drill (3 min)
Why do we start crossing out from `i * i` and not `2 * i`? *(Hint: smaller multiples like `2i`, `3i`, ..., `(i-1) * i` were already crossed out when we processed those smaller primes.)*

**Deep dive (later):** [dsa/patterns.md](../dsa/patterns.md) · [dsa/CRAM.md](../dsa/CRAM.md)

---

## 9. Design Pattern — DTO / VO

### Intent
A **DTO** (Data Transfer Object) carries data across boundaries — API ↔ client, service ↔ service. A **VO** (Value Object) represents a meaningful value, identified by what it *contains*, not by an ID.

### The difference in plain English
- **DTO** — "I'm a courier package. Stuff data in me, ship me over the wire." Mutable or immutable, no behavior.
- **VO** — "I *am* this value. Two VOs with the same fields are equal." Immutable, often has small helpers.

A `UserDTO` shaped for your JSON API and a `Money` value object (amount + currency) are different beasts.

### Smallest working example
```java
// DTO — flat, serializable, just data
public record UserDTO(Long id, String email, String displayName) {}

// VO — value identity, immutable, has behavior
public record Money(BigDecimal amount, String currency) {
    public Money plus(Money other) {
        if (!currency.equals(other.currency))
            throw new IllegalArgumentException("currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }
}

Money a = new Money(new BigDecimal("10"), "INR");
Money b = new Money(new BigDecimal("10"), "INR");
a.equals(b);  // true — same value
```

Java `record` makes both effortless: auto `equals`, `hashCode`, `toString`.

### Drill (1 min)
Why don't we just send our JPA `@Entity` to the client directly? *(Hint: entities carry DB plumbing — lazy proxies, cascade rules, internal fields. A DTO gives the client exactly the shape it needs, nothing more.)*

**Deep dive (later):** [design-patterns/common/structural/dto-vo.md](../design-patterns/common/structural/dto-vo.md)

---

## 10. DevOps — Service Mesh (Istio)

### Why this exists
In microservices, every service needs TLS, retries, timeouts, metrics, tracing. Writing this in every service (and every language) is painful. A **service mesh** moves that plumbing *out of your code* and into a sidecar proxy next to every service.

### The idea in plain English
A service mesh is **a highway with built-in traffic monitoring**. Every car (request) goes through an entry/exit booth (sidecar proxy) that handles encryption, counts cars, and reroutes around accidents — *without the driver writing any code*.

In Istio, the sidecar is **Envoy**. Your app talks to localhost; Envoy handles the rest: mTLS to other services, retries, circuit breaking, metrics.

### Smallest working example
```yaml
# Inject sidecar by labeling the namespace
kubectl label namespace default istio-injection=enabled

# Apply a traffic-shift rule: send 10% of traffic to v2
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: ["reviews"]
  http:
  - route:
    - destination: { host: reviews, subset: v1 }
      weight: 90
    - destination: { host: reviews, subset: v2 }
      weight: 10
```

You didn't change a line of app code. Canary deploys, A/B tests, mTLS between services — all from config.

### Drill (2 min)
What is **mTLS** and why is "mutual" the important word? *(Hint: mutual TLS means both sides prove their identity with certificates. Regular TLS only proves the server's identity — mTLS also verifies the *client*, which stops a rogue pod from impersonating a real one.)*

**Deep dive (later):** [devops/25-service-mesh-istio.md](../devops/25-service-mesh-istio.md)

---

## End-of-day checklist

- [ ] Angular: I can mark text for translation with `i18n` and add `aria-label` to an icon-only button
- [ ] Node.js: I can explain why background jobs exist and name the two roles (producer, worker)
- [ ] Spring: I can read a controller that returns `Mono<User>` and `Flux<User>` without panic
- [ ] MongoDB: I can recite the ESR rule for compound indexes (Equality, Sort, Range)
- [ ] Postgres: I can write `CREATE MATERIALIZED VIEW` and `REFRESH MATERIALIZED VIEW CONCURRENTLY`
- [ ] HLD: I can sketch an inverted index for 3 tiny documents
- [ ] LLD: I can explain LRU vs LFU in one sentence each
- [ ] DSA: I can write Euclid's GCD and explain why the Sieve starts at `i*i`
- [ ] DP: I can say one difference between a DTO and a VO
- [ ] DevOps: I can explain what a sidecar proxy is and what mTLS verifies

**If you remember just one thing today:** modern apps push slow or cross-cutting work *out of the request path* — into background queues (BullMQ), into materialized views, into sidecars (Istio). The HTTP handler should do as little as possible.

**Tomorrow:** security and observability — XSS hardening, structured logging, Spring Cloud microservices, sharded operations, query tuning, CDNs, Top K, Specification pattern, K8s Operators.
