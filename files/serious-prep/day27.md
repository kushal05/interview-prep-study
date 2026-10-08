# Day 27 — Security, observability, and distributed systems

> **Today's goal:** learn how apps defend themselves (XSS, mTLS), how they tell you what's happening (structured logs), and how multiple services talk in production (Spring Cloud, sharding, CDNs). Big topics, small bites.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Security (XSS, CSP) | 15m |
| 2 | Node.js | Logging & Monitoring (pino) | 15m |
| 3 | Spring Boot | Spring Cloud (Config, Gateway) | 15m |
| 4 | MongoDB | Sharded Cluster Operations | 10m |
| 5 | Postgres | Query Tuning (EXPLAIN ANALYZE) | 10m |
| 6 | HLD | CDN & Edge | 12m |
| 7 | LLD | Splitwise — advanced settlements | 12m |
| 8 | DSA | Top K Frequent Elements | 15m |
| 9 | Design Pattern | Specification | 8m |
| 10 | DevOps | Advanced K8s (Operators, CRDs) | 10m |

---

## 1. Angular — Security (XSS & CSP)

### Why this exists
**XSS** (Cross-Site Scripting) is when an attacker tricks your app into running their JavaScript in a user's browser — stealing cookies, hijacking sessions. Angular blocks most of it automatically, but you can override the protection and shoot yourself in the foot.

### The idea in plain English
Angular treats every value going into the DOM as **untrusted by default** and escapes it. The danger zone: when you actively *bypass* that protection (`bypassSecurityTrustHtml`) or build URLs from user input.

A **CSP** (Content Security Policy) is a second wall: a browser header that says "only load scripts from my domain — even if attacker code slips through, the browser refuses to run it."

Think of it like a bank: Angular escaping is the cashier checking ID, CSP is the locked vault behind them.

### Smallest working example
```typescript
// SAFE — Angular escapes special chars in {{ }} and [innerHTML]
template: `<div>{{ comment }}</div>`             // <script> shows as text

// DANGEROUS — only do this if 'html' is sanitized server-side
constructor(private sanitizer: DomSanitizer) {}
trusted(html: string): SafeHtml {
  return this.sanitizer.bypassSecurityTrustHtml(html);
}

// CSP header (set by your server)
// Content-Security-Policy: default-src 'self'; script-src 'self'
```

The CSP above means: only load scripts from the same origin. Inline `<script>` tags or `eval()` are blocked.

### Drill (2 min)
A user types `<script>alert(1)</script>` into a comment box. You render it with `{{ comment }}`. Does the alert fire? *(Hint: no — Angular escapes it to text. The alert would only fire if you used `bypassSecurityTrustHtml` on it.)*

**Deep dive (later):** [angular/angular-security.md](../angular/angular-security.md)

---

## 2. Node.js — Logging & Monitoring (pino / winston)

### Why this exists
`console.log` is fine on your laptop. In production with 50 servers and millions of requests, you need logs that machines can read, search, and correlate. That means **structured logs** — JSON, not human prose.

### The idea in plain English
A structured log is **a row in a spreadsheet**, not a sentence in a diary. Each entry has the same fields: timestamp, level, message, request ID. Tools like Datadog, Loki, Elastic eat JSON and let you query: "show me all `level=error` logs for `userId=42` in the last hour."

**Correlation IDs** are the magic glue: stamp every incoming request with a UUID, pass it to every service it touches, log it everywhere. Now you can trace one user's journey across 10 services.

### Smallest working example
```javascript
const pino = require('pino');
const logger = pino({ level: 'info' });

logger.info({ userId: 42, route: '/orders' }, 'order placed');
// {"level":30,"time":1716...,"userId":42,"route":"/orders","msg":"order placed"}

// Correlation ID middleware (Express)
app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] || crypto.randomUUID();
  req.log = logger.child({ reqId: req.id });
  next();
});

app.get('/orders', (req, res) => {
  req.log.info('fetching orders');   // every log carries reqId automatically
  res.json([]);
});
```

`pino` is fast (asynchronous). `winston` is more featureful (transports, formats). Both produce JSON.

### Drill (2 min)
Why log `{ userId: 42 }` as a structured field instead of `"user 42 placed an order"`? *(Hint: a query like "all logs for user 42" needs a structured field. Parsing free text in production at scale is slow and fragile.)*

**Deep dive (later):** [nodejs/19-security.md](../nodejs/19-security.md)

---

## 3. Spring Boot — Spring Cloud Microservices

### Why this exists
When you split a monolith into 20 microservices, new problems appear: where do they get their config? How does a request reach the right service? How do they discover each other? **Spring Cloud** is a set of libraries that solve these — built on top of Spring Boot.

### The idea in plain English
Two pieces matter on day 1:

- **Config Server** — a central place where all services read their settings (DB URLs, secrets, feature flags). Instead of each service having its own `application.yml`, they fetch on startup.
- **API Gateway** — the single front door. All client requests hit the gateway, which routes to the right backend service. Handles auth, rate limiting, logging in one place.

Think of it as an **office building**: the gateway is the receptionist (routes visitors), the config server is HR (everyone fetches policies from one place).

### Smallest working example
```yaml
# Config Server: serves files from a Git repo
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/me/configs

# A client service fetches its config on startup
spring:
  config:
    import: optional:configserver:http://localhost:8888

# Gateway: route /api/orders/** to order-service
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://order-service
          predicates: [ Path=/api/orders/** ]
```

`lb://` means "look it up in service discovery" (Day 28 covers Eureka).

### Drill (2 min)
Why is putting auth in the gateway better than putting it in each service? *(Hint: one place to update, one place to audit, and individual services can trust requests that reached them. Avoids 20 copies of the same auth code.)*

**Deep dive (later):** [spring-next/06-spring-security.md](../spring-next/06-spring-security.md) · [Micrometer Tracing — spring-next/08](../spring-next/08-spring-boot-advanced.md#2-spring-cloud-overview)

---

## 4. MongoDB — Sharded Cluster Operations

### Why this exists
When data outgrows one server, MongoDB splits it across many — **sharding**. But sharding adds moving parts: who routes the query? Who decides where data lives? What happens when one shard gets too hot?

### The idea in plain English
Imagine a library too big for one building. You split books across 5 buildings by author's last name (the **shard key**). To find a book:

- A **router** (mongos) knows which building holds which authors.
- A **balancer** notices when one building gets overloaded and moves chunks of books to lighter buildings.
- **Jumbo chunks** are oversized boxes that can't be moved (bad shard key choice — too many books by one author).

Picking the right shard key is the most important decision. Once data is sharded, it's painful to change.

### Smallest working example
```javascript
// Enable sharding on a database
sh.enableSharding("shop");

// Shard a collection by a hashed key (even distribution)
sh.shardCollection("shop.orders", { customerId: "hashed" });

// Check the balancer state
sh.getBalancerState();

// See chunk distribution
db.orders.getShardDistribution();
```

`mongos` is the only thing your app talks to — it figures out which shard(s) to hit.

### Drill (2 min)
You shard `orders` by `customerId`. A query filters by `productId`. What happens? *(Hint: a **scatter-gather** — mongos queries every shard and merges results. Slow. Always filter by the shard key for fast queries.)*

**Deep dive (later):** [mongodb/10-sharding.md](../mongodb/10-sharding.md)

---

## 5. Postgres — Query Tuning with EXPLAIN ANALYZE

### Why this exists
"My query is slow" is the most common database complaint. Postgres has a built-in answer: ask it what it's actually doing. `EXPLAIN ANALYZE` runs your query and prints the plan with real timings.

### The idea in plain English
A query plan is a **recipe** — Postgres telling you "I'll grab rows this way, then join them that way, then sort." `EXPLAIN ANALYZE` runs the recipe and shows where time was spent.

Two things to look for:
- **Seq Scan** on a big table = missing index (usually).
- **Big difference between estimated and actual rows** = stale stats. Run `ANALYZE` to refresh.

Think of yourself as a detective reading the plan tree from the inside out: leaves run first, then their parents.

### Smallest working example
```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) AS orders
FROM   users u
JOIN   orders o ON o.user_id = u.id
WHERE  u.country = 'IN'
GROUP  BY u.name;

-- Sample output (read inside-out):
-- HashAggregate  (cost=... actual time=42.1 rows=1200)
--   ->  Hash Join  (actual time=12.0..38.5)
--         ->  Seq Scan on orders (actual time=0.0..15.0 rows=50000)   ← suspicious
--         ->  Hash  ...
--               ->  Index Scan on users_country_idx (rows=1200)
```

The `Seq Scan on orders` is the bottleneck. Add `CREATE INDEX ON orders(user_id);` and rerun.

### Drill (2 min)
What's the difference between `EXPLAIN` and `EXPLAIN ANALYZE`? *(Hint: `EXPLAIN` shows the *planned* path without running. `ANALYZE` actually executes the query and reports real timings. Don't run `ANALYZE` on a destructive query!)*

**Deep dive (later):** [postgres/07-query-planner-and-explain.md](../postgres/07-query-planner-and-explain.md)

---

## 6. HLD — CDN & Edge

### Why this exists
A user in Mumbai loading your site from a server in Virginia waits ~250 ms just for the network round trip — before your code does anything. A **CDN** (Content Delivery Network) keeps copies of your static files near every major city, so the user reads from a server 5 ms away.

### The idea in plain English
A CDN is **a local copy of your website near every city**. Cloudflare, Akamai, CloudFront all run thousands of **edge** servers (POPs — Points of Presence). When a user requests `/logo.png`, they hit the nearest POP. If the file is cached → instant. If not → the POP fetches it from your origin, caches it, then serves.

**Edge functions** take this further: small bits of JavaScript that run *at the edge* (Cloudflare Workers, Vercel Edge). Things like A/B testing, geo-redirects, auth checks happen 5 ms from the user.

### Cache control basics
```http
GET /assets/logo.png
Response:
  Cache-Control: public, max-age=31536000, immutable
  ETag: "abc123"
```

`max-age=31536000` = cache for 1 year. `immutable` = "this URL never changes." Bundlers add a hash to filenames (`logo.abc123.png`) so a new version gets a new URL — no cache invalidation needed.

### Cache invalidation
Two strategies:
- **Purge** — tell the CDN to drop a URL (Cloudflare API call). Slow, eventual.
- **Versioned URLs** — change the URL when content changes. Instant, recommended.

### Drill (2 min)
You change your logo but users still see the old one. Why? *(Hint: the CDN cached `logo.png` for a year. Either purge it via the CDN's API, or — better — rename the file to `logo-v2.png` so the URL itself changes.)*

**Deep dive (later):** [system-design/high-level-design/07-storage-and-cdn.md](../system-design/high-level-design/07-storage-and-cdn.md)

---

## 7. LLD — Splitwise advanced (settlements, multi-currency)

### Why this exists
Basic Splitwise tracks "Alice owes Bob $10." Real life is messier: 5 friends, overlapping debts, two currencies. The goal of **settlement** is to reduce a tangled web into the fewest possible transfers.

### The idea in plain English
After many expenses, you have a **net balance** per person: +X means owed, −X means owes. The minimum-transactions problem: with these balances, what's the smallest set of transfers that zeros everyone out?

Greedy works well: take the person owed the most and the person owing the most, settle between them, repeat.

For **multi-currency**, store the *original* currency on each expense and convert to a common one (the group's base currency) using exchange rates at the time of the expense — never the time of the settlement, or history shifts under you.

### Smallest working example
```java
class Settlement {
    // Greedy: match max creditor with max debtor
    public List<Transfer> settle(Map<User, BigDecimal> balances) {
        PriorityQueue<Entry> creditors = ...;   // max-heap by amount
        PriorityQueue<Entry> debtors  = ...;    // max-heap by abs(amount)
        List<Transfer> out = new ArrayList<>();
        while (!creditors.isEmpty() && !debtors.isEmpty()) {
            var c = creditors.poll();
            var d = debtors.poll();
            BigDecimal amount = c.amount.min(d.amount.abs());
            out.add(new Transfer(d.user, c.user, amount));
            // re-add residue
            if (c.amount.compareTo(amount) > 0) creditors.add(new Entry(c.user, c.amount.subtract(amount)));
            if (d.amount.abs().compareTo(amount) > 0) debtors.add(new Entry(d.user, d.amount.add(amount)));
        }
        return out;
    }
}
```

For a group of N people, you get **at most N−1** transfers (often fewer).

### Drill (2 min)
Why store the original currency *and* the converted amount on each expense? *(Hint: rates change daily. If you reconvert old expenses with today's rate, balances drift. The historical rate is part of the truth of that expense.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Top K Frequent Elements

### The problem
Given an array `nums` and integer `k`, return the `k` most frequent elements.

```
Input:  nums = [1,1,1,2,2,3], k = 2
Output: [1, 2]
```

### The idea
Two steps:
1. **Count frequencies** with a hash map.
2. **Pick the top k** from the counts.

The naive "sort all counts" is O(n log n). Two faster ways:

### Approach 1: Min-heap of size k — O(n log k)
Keep a min-heap of size k. Push items in. If size exceeds k, pop the smallest. End → heap holds top k.

```java
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> count = new HashMap<>();
    for (int n : nums) count.merge(n, 1, Integer::sum);

    PriorityQueue<int[]> heap = new PriorityQueue<>((a, b) -> a[1] - b[1]);  // min-heap by freq
    for (var e : count.entrySet()) {
        heap.offer(new int[]{ e.getKey(), e.getValue() });
        if (heap.size() > k) heap.poll();
    }

    int[] res = new int[k];
    for (int i = k - 1; i >= 0; i--) res[i] = heap.poll()[0];
    return res;
}
```

### Approach 2: Bucket sort — O(n)
Frequencies can't exceed `n`. Make `n+1` buckets indexed by frequency. Put numbers in their bucket. Walk from the top bucket down, collecting `k` items.

Use heap when k << n. Use buckets when k is comparable to n.

### Drill (3 min)
Why a **min-heap** when we want the *top* k? *(Hint: a min-heap of size k always has the smallest of the "top so far" at its root. If a new item is bigger than the root, we kick out the root and add the new one. The k biggest survive.)*

**Deep dive (later):** [dsa/patterns.md](../dsa/patterns.md)

---

## 9. Design Pattern — Specification

### Intent
Encapsulate a business rule (a "specification") into a reusable object you can **combine** with `and`, `or`, `not`. Replaces sprawling `if` chains.

### When you'd use it
- A search filter UI: combine multiple criteria dynamically.
- Eligibility rules: "is over 18 AND has verified email AND is not banned."
- Domain rules you want to unit-test in isolation.

### Smallest working example
```java
interface Spec<T> {
    boolean isSatisfiedBy(T t);
    default Spec<T> and(Spec<T> other) { return t -> isSatisfiedBy(t) && other.isSatisfiedBy(t); }
    default Spec<T> or (Spec<T> other) { return t -> isSatisfiedBy(t) || other.isSatisfiedBy(t); }
    default Spec<T> negate()           { return t -> !isSatisfiedBy(t); }
}

Spec<User> isAdult       = u -> u.age() >= 18;
Spec<User> emailVerified = u -> u.emailVerified();
Spec<User> banned        = u -> u.isBanned();

Spec<User> canPost = isAdult.and(emailVerified).and(banned.negate());

users.stream().filter(canPost::isSatisfiedBy).toList();
```

Each rule is a tiny class. They compose. They test cleanly. Spring Data JPA has a built-in `Specification<T>` for queries.

### Drill (1 min)
Why not just write `if (user.age() >= 18 && user.emailVerified() && !user.isBanned())`? *(Hint: that works for one rule. The moment you need *combinations* — admin sees one rule, frontend sees another — copy-paste begins. Specifications let you build rules from parts.)*

**Deep dive (later):** [design-patterns/common/behavioral/specification.md](../design-patterns/common/behavioral/specification.md)

---

## 10. DevOps — Advanced K8s (Operators & CRDs)

### Why this exists
Kubernetes (K8s) knows about Pods, Deployments, Services out of the box. But what if you want K8s to know about *your* thing — say, a Postgres cluster? You teach it with a **Custom Resource Definition (CRD)** and write an **Operator** to manage instances.

### The idea in plain English
A K8s **Operator** is **a robot DBA that watches a custom resource and fixes it**. You file a "ticket" (a custom resource: "I want a Postgres cluster with 3 replicas"). The operator watches for tickets, creates the pods, monitors them, runs backups, handles failover — all automatically.

Pattern: **Observe → Diff → Act**. Operator reads desired state, sees actual state, takes actions to make them match. This loop runs forever.

### Smallest working example
```yaml
# 1. Define a new resource type (CRD)
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: { name: postgresclusters.db.example.com }
spec:
  group: db.example.com
  names: { kind: PostgresCluster, plural: postgresclusters }
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas: { type: integer }

# 2. Create a custom resource — the operator picks it up
apiVersion: db.example.com/v1
kind: PostgresCluster
metadata: { name: prod-db }
spec:
  replicas: 3
```

Real-world operators: **CloudNativePG** (Postgres), **Strimzi** (Kafka), **cert-manager** (TLS certs).

### Drill (2 min)
What does an Operator do that a plain `Deployment` can't? *(Hint: a Deployment just runs N copies of a pod. An operator handles *domain knowledge* — replica failover order, schema migrations, backup scheduling — things only a DBA would know.)*

**Deep dive (later):** [devops/06-kubernetes-fundamentals.md](../devops/06-kubernetes-fundamentals.md)

---

## End-of-day checklist

- [ ] Angular: I can explain why `{{ comment }}` is safe but `bypassSecurityTrustHtml` is dangerous
- [ ] Node.js: I can name two reasons structured logs beat `console.log` in production
- [ ] Spring: I can describe what Spring Cloud Config Server and API Gateway each do
- [ ] MongoDB: I can explain what `mongos` does and why a bad shard key causes scatter-gather
- [ ] Postgres: I know what `Seq Scan` in an `EXPLAIN ANALYZE` output usually means
- [ ] HLD: I can explain why versioned URLs are better than CDN purges
- [ ] LLD: I can describe the greedy approach to minimizing settlement transfers
- [ ] DSA: I can implement Top K with a min-heap and explain why min, not max
- [ ] DP: I can compose two `Spec<T>` rules with `.and()`
- [ ] DevOps: I can explain "an Operator is a robot DBA that watches a CRD"

**If you remember just one thing today:** every production concern — security, logging, routing, scaling — has a *pattern* and a *library*. You don't reinvent these. You learn the names so you can reach for the right tool.

**Tomorrow:** offline-first apps, service discovery, time-series data, point-in-time recovery, geo-distribution, pub/sub, hard DSA, SRE practices.
