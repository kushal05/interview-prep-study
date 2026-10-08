# Day 29 — Performance, resilience, and streaming

> **Today's goal:** make apps fast (build optimization), portable (twelve-factor), self-protecting (circuit breakers), observable (mongostat), and cost-aware (FinOps). End with Kafka and two hard problems.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Build Optimization | 15m |
| 2 | Node.js | Twelve Factor App | 15m |
| 3 | Spring Boot | Circuit Breaker (Resilience4j) | 15m |
| 4 | MongoDB | Operational Tools (mongostat) | 10m |
| 5 | Postgres | Vacuum in production | 10m |
| 6 | HLD | Streaming Architecture (Kafka) | 12m |
| 7 | LLD | Search Autocomplete | 12m |
| 8 | DSA | Sliding Window Maximum | 15m |
| 9 | Design Pattern | Object Pool | 8m |
| 10 | DevOps | Cost & FinOps | 10m |

---

## 1. Angular — Build Optimization

### Why this exists
Your Angular bundle gets loaded by the user *before they can do anything*. A 2 MB JS bundle on a 3G phone = 4 seconds of staring at a white screen. Build optimization shrinks that bundle aggressively.

### The idea in plain English
Three levers:

- **Tree-shaking** — the bundler drops code you never import. Like Marie Kondo: if it doesn't spark `import`, it goes.
- **Lazy loading** — split the app by route. Only load the `/admin` code when the user actually goes there.
- **Bundle analyzer** — a chart showing what's *in* your bundle. Usually one library (Moment.js!) is eating half of it.

A good first-load bundle target: **under 200 KB gzipped**.

### Smallest working example
```typescript
// Lazy load a route — Angular builds /admin into its own chunk
const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES) }
];
```

```bash
# See what's in the bundle
ng build --configuration production --stats-json
npx webpack-bundle-analyzer dist/your-app/stats.json
```

Common wins: drop `moment.js` (huge — switch to `date-fns` or `Intl`), import RxJS operators individually, use `NgOptimizedImage` for images.

### Drill (2 min)
Why is `import _ from 'lodash'` worse than `import debounce from 'lodash/debounce'`? *(Hint: the first form *can* prevent tree-shaking — the bundler may include all of lodash. The second imports only the debounce function.)*

**Deep dive (later):** [angular/build-and-bundle.md](../angular/build-and-bundle.md)

---

## 2. Node.js — Twelve Factor App

### Why this exists
Apps used to live on one named server with hand-tuned configs. Modern apps live in containers that come and go. The **Twelve Factor App** is **12 rules for cloud-friendly apps** — written by Heroku, followed by everyone.

### The top six you'll be asked about
1. **Codebase** — one repo per app, deployed to many environments.
2. **Dependencies** — declare them explicitly (`package.json`), never rely on system packages.
3. **Config** — *in environment variables*, never in code. Different env, different env vars.
4. **Backing services** — DBs, queues are attached resources, swappable by URL.
5. **Stateless processes** — never store user state on the process. Use Redis / DB.
6. **Disposability** — start fast, shut down cleanly on `SIGTERM`.

### The idea in plain English
Pretend your process can die *any second* and a new one springs up elsewhere. If anything important lives only inside the process (sessions in memory, files in `/tmp`), it's lost. Twelve Factor is the **survival kit**.

### Smallest working example
```javascript
// Config in env (factor 3)
const dbUrl   = process.env.DATABASE_URL;
const port    = process.env.PORT || 3000;

// Stateless (factor 6): no in-memory sessions; store in Redis
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
}));

// Disposability (factor 9): handle SIGTERM
process.on('SIGTERM', async () => {
  await server.close();
  await db.end();
  process.exit(0);
});

// Logs to stdout (factor 11) — let the platform collect them
console.log(JSON.stringify({ level: 'info', msg: 'started' }));
```

### Drill (2 min)
You store uploaded files on the local disk. Why is that a problem under twelve-factor? *(Hint: a new pod starts on a different node — no file. Use S3, GCS, or a shared volume.)*

**Deep dive (later):** [nodejs/23-deployment-and-docker.md](../nodejs/23-deployment-and-docker.md)

---

## 3. Spring Boot — Circuit Breaker (Resilience4j)

### Why this exists
When a downstream service is slow or broken, your service starts piling up retries — threads block, memory grows, you take *yourself* down too. A **circuit breaker** stops calling a broken service until it recovers.

### The idea in plain English
A circuit breaker is **an electrical fuse for service calls**. After too many failures it **opens** — all calls fail instantly without trying the network. After a cooldown it goes **half-open**, lets one call through to check, and either closes (back to normal) or opens again.

Three states: **Closed** (everything fine), **Open** (failing fast), **Half-open** (testing the waters).

### Smallest working example
```java
@Service
public class PaymentClient {
    private final RestTemplate rest;
    public PaymentClient(RestTemplate r) { this.rest = r; }

    @CircuitBreaker(name = "payment", fallbackMethod = "queued")
    public PaymentResp charge(Charge c) {
        return rest.postForObject("/charge", c, PaymentResp.class);
    }

    // called when circuit is open or call fails
    public PaymentResp queued(Charge c, Throwable t) {
        queue.add(c);
        return new PaymentResp("QUEUED");
    }
}
```

```yaml
resilience4j.circuitbreaker:
  instances:
    payment:
      failureRateThreshold: 50           # 50% failure rate trips the breaker
      slidingWindowSize: 10               # over last 10 calls
      waitDurationInOpenState: 30s
```

Companion patterns from Resilience4j: **Retry**, **Bulkhead** (limit concurrent calls), **RateLimiter**, **TimeLimiter**.

### Drill (2 min)
Why is "fail fast" better than "retry forever" when a downstream is dead? *(Hint: retries pile up — your threads wait, queues fill, memory rises. Failing fast frees your service to keep serving other things.)*

**Deep dive (later):** [spring-next/08-spring-boot-advanced.md — Performance Tuning](../spring-next/08-spring-boot-advanced.md#8-performance-tuning)

---

## 4. MongoDB — Operational Tools

### Why this exists
At 3 AM, "the database is slow" — what do you do? Mongo ships tools to peek inside a running server without writing a query. Three names to know: `mongostat`, `mongotop`, the **database profiler**.

### The tools in plain English
- **mongostat** — like Linux `top` for Mongo. Live counters: inserts/sec, queries/sec, page faults, lock %. Spot a sudden spike.
- **mongotop** — which *collections* are eating I/O time. Useful when you don't know where the load comes from.
- **Profiler** — captures slow queries to a system collection. Like a "speed camera" for Mongo.

### Smallest working examples
```bash
# Live counters every 2 seconds
mongostat --port 27017 2

# Which collections are hot?
mongotop --port 27017 2

# Inside mongosh — enable the profiler
db.setProfilingLevel(1, { slowms: 100 });    // log queries > 100ms

# Read what it caught
db.system.profile.find().sort({ ts: -1 }).limit(5);
```

Profiler **level 1** = slow queries only. Level 2 = everything (don't run in prod — fills disk).

### Drill (2 min)
mongostat shows `qr|qw: 200|50` — read queue 200, write queue 50. What's the read-side problem? *(Hint: a queue means clients are *waiting* for the database. 200 reads queued = severe contention. Likely a missing index or a long-running write holding a lock.)*

**Deep dive (later):** [mongodb/12-performance-tuning.md](../mongodb/12-performance-tuning.md)

---

## 5. Postgres — Vacuum in production

### Why this exists
Postgres uses **MVCC** — when you `UPDATE` or `DELETE`, the old row isn't removed, just marked obsolete. Over time those **dead tuples** accumulate, tables bloat, queries slow. **VACUUM** reclaims them.

### The idea in plain English
Imagine your office throws away no paper — it just stacks "outdated" on top. Eventually you can't find anything. **VACUUM** is the cleaning crew: it removes outdated rows so the live data stays compact.

**Autovacuum** runs this automatically. The problem: defaults are conservative. On a high-update table, autovacuum can't keep up — dead tuples grow, indexes bloat, queries slow.

### Tuning knobs
```sql
-- Inspect bloat on a table
SELECT relname,
       n_live_tup,                       -- live rows
       n_dead_tup,                       -- dead rows (bad if huge)
       last_autovacuum
FROM   pg_stat_user_tables
WHERE  relname = 'orders';

-- Per-table autovacuum tuning (more aggressive on hot tables)
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.05,  -- vacuum when 5% dead (default 20%)
  autovacuum_vacuum_cost_limit  = 2000    -- let it work harder per round
);

-- Manual vacuum (safe to run; doesn't lock writes)
VACUUM (ANALYZE) orders;
```

The companion **ANALYZE** updates query planner stats. After bulk inserts always run `VACUUM ANALYZE`.

### Drill (2 min)
A table has `n_dead_tup = 5,000,000`. Queries got slow. Why does vacuuming help? *(Hint: every scan walks past dead rows. Removing them shrinks pages, fits more in cache, and lets indexes point at fewer entries. Faster.)*

**Deep dive (later):** [postgres/17-vacuum-and-bloat.md](../postgres/17-vacuum-and-bloat.md)

---

## 6. HLD — Streaming Architecture (Kafka)

### Why this exists
A pub/sub message broker plus durable storage plus replay = **Kafka**. It's the backbone of modern event-driven systems: every change in one service becomes an event other services react to.

### The idea in plain English
Three concepts to memorize:

- **Topic** — a named log of events ("orders", "page_views").
- **Partition** — each topic is split into ordered logs for parallelism. Messages with the same **key** always go to the same partition (so ordering per-key is preserved).
- **Consumer group** — a team of consumers that *split* a topic's partitions. Each partition is read by *one* consumer in the group at a time. Scale a consumer = add more partitions.

Kafka **stores** every event for days/weeks. A new service can come online and **replay** history — no other broker does this so well.

### A picture
```
Topic: orders  (3 partitions)

Partition 0:  [A1][A2][A3][A4]
Partition 1:  [B1][B2][B3]
Partition 2:  [C1][C2][C3][C4][C5]

Consumer Group "shipping":
  Consumer-1 reads Partition 0
  Consumer-2 reads Partition 1
  Consumer-3 reads Partition 2

Consumer Group "analytics":
  Consumer-X reads ALL partitions   (different group → independent progress)
```

Same data, two groups, independent progress. That's why Kafka is everywhere.

### Drill (2 min)
Why does ordering *only* work per-key, not per-topic? *(Hint: parallelism. Messages from different partitions are consumed in parallel by different workers — there's no way to guarantee total ordering without giving up parallelism.)*

**Deep dive (later):** [system-design/high-level-design/10-real-world-system-designs.md](../system-design/high-level-design/10-real-world-system-designs.md)

---

## 7. LLD — Search Autocomplete (Trie + ranking + cache)

### Why this exists
"As-you-type" search suggestions need to respond in **< 50 ms**. Hitting a database per keystroke is hopeless. The classic design: an in-memory **Trie** plus ranked suggestions plus aggressive caching.

### The idea in plain English
A **Trie** (prefix tree) is a tree where each path from the root spells a word. Searching "ja" → walk root → 'j' → 'a' → now collect every word in that subtree.

But a trie alone returns all matches alphabetically. Real autocomplete needs **ranking** — popularity, click-through rate, personalization. Store top-N suggestions *at each node* so retrieval is O(prefix_length).

```
Trie excerpt for ["java", "javascript", "jakarta", "jam"]:

      root
       │
       j
       │
       a
      ╱ │ ╲
     k  m  v
     │     │
     a     a
     │     │
     r     ★ (java — popularity 100)
     │     │
     t     s
     │     │
     a     ...
     │
     ★ (jakarta — popularity 30)
```

### Smallest working example
```java
class TrieNode {
    Map<Character, TrieNode> children = new HashMap<>();
    PriorityQueue<Suggestion> topK = new PriorityQueue<>(...);   // pre-computed
}

class Autocomplete {
    private final TrieNode root = new TrieNode();

    void add(String word, int popularity) {
        TrieNode cur = root;
        for (char c : word.toCharArray()) {
            cur = cur.children.computeIfAbsent(c, k -> new TrieNode());
            updateTopK(cur, word, popularity);
        }
    }

    List<String> suggest(String prefix) {
        TrieNode cur = root;
        for (char c : prefix.toCharArray()) {
            cur = cur.children.get(c);
            if (cur == null) return List.of();
        }
        return cur.topK.stream().map(s -> s.word).toList();
    }
}
```

Add an **LRU cache** on top: `Map<prefix, List<suggestion>>`. Hot prefixes ("re", "th") never touch the trie.

### Drill (2 min)
Why pre-compute top-K *at each node* instead of walking the subtree at query time? *(Hint: walking the subtree is slow if many words share the prefix. Pre-computed top-K is O(1) — pay at insert time, win at every query.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Sliding Window Maximum

### The problem
Given an array `nums` and window size `k`, return the maximum of every window of size k as the window slides one step at a time.

```
nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3
Output: [3, 3, 5, 5, 6, 7]
```

### The naive
For each window, scan k items → O(n·k). For n = 10⁵ and k = 1000, that's 10⁸ — too slow.

### The trick: monotonic deque
Maintain a **deque** (double-ended queue) of *indices*. Keep it **decreasing** by value — the front always holds the max of the current window.

For each new index `i`:
1. Drop indices from the **back** while `nums[back] <= nums[i]` (they can never be max again).
2. Push `i` at the back.
3. Drop the **front** if it's outside the window (`front <= i - k`).
4. Once `i >= k - 1`, record `nums[front]` as this window's max.

```java
public int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> dq = new ArrayDeque<>();   // holds indices
    int n = nums.length;
    int[] out = new int[n - k + 1];

    for (int i = 0; i < n; i++) {
        // 1. drop smaller from back
        while (!dq.isEmpty() && nums[dq.peekLast()] <= nums[i]) dq.pollLast();
        // 2. push current
        dq.offerLast(i);
        // 3. drop out-of-window from front
        if (dq.peekFirst() <= i - k) dq.pollFirst();
        // 4. record
        if (i >= k - 1) out[i - k + 1] = nums[dq.peekFirst()];
    }
    return out;
}
```

**Why O(n)** : each index enters and leaves the deque at most once.

### Drill (3 min)
Trace `[1, 3, -1]` with k=3. After i=0: dq=[0]. After i=1: drop 0 (1≤3) → dq=[1]. After i=2: -1 is smaller, keep → dq=[1, 2]. i ≥ k-1, record `nums[1] = 3`. *(Output starts [3, ...] — correct.)*

**Deep dive (later):** [dsa/patterns.md](../dsa/patterns.md)

---

## 9. Design Pattern — Object Pool

### Intent
Reuse expensive-to-create objects (DB connections, threads, sockets) from a fixed-size pool instead of creating new ones every time.

### When you'd use it
- JDBC connection pools (HikariCP) — opening a DB connection costs ~50ms.
- Thread pools — spinning up an OS thread is expensive.
- Game engines — bullets, particles spawned thousands of times per second.

### Smallest working example
```java
class ConnectionPool {
    private final BlockingQueue<Connection> pool;

    public ConnectionPool(int size, Supplier<Connection> factory) {
        pool = new ArrayBlockingQueue<>(size);
        for (int i = 0; i < size; i++) pool.offer(factory.get());
    }

    public Connection acquire() throws InterruptedException {
        return pool.take();              // blocks if empty
    }

    public void release(Connection c) {
        pool.offer(c);                   // back into pool
    }
}

ConnectionPool db = new ConnectionPool(10, () -> DriverManager.getConnection("..."));
Connection c = db.acquire();
try { /* use c */ } finally { db.release(c); }
```

Real-world pools also: validate connections, expire stale ones, size up/down dynamically.

### Drill (1 min)
What goes wrong if you forget `release()`? *(Hint: the pool runs dry. The next `acquire()` blocks forever. This is the classic "connection leak" production bug — always release in `finally` or use try-with-resources.)*

**Deep dive (later):** [design-patterns/common/creational/object-pool.md](../design-patterns/common/creational/object-pool.md)

---

## 10. DevOps — Cost & FinOps

### Why this exists
Cloud bills surprise people. **FinOps** is the discipline of treating cost as a feature — measuring it, owning it, optimizing it. A 30% bill reduction with no service impact is a normal first quarter.

### The five biggest levers
1. **Right-sizing** — your `t3.xlarge` runs at 8% CPU. Switch to `t3.small`. Massive savings.
2. **Reserved / Savings Plans** — commit to 1- or 3-year usage for a 30–60% discount over on-demand.
3. **Spot instances** — AWS sells unused capacity 70–90% off, but can take it back with 2 min notice. Great for batch / stateless workers.
4. **Storage tiering** — old S3 objects → Glacier (cheap, slow). Postgres archive → S3.
5. **K8s requests/limits** — wrong `requests` reserves too much node capacity. Wrong `limits` lets pods get OOM-killed. Use VPA / metrics-server to tune.

### A real example
```yaml
# Pod with sensible requests/limits
resources:
  requests:                # what K8s reserves for this pod
    cpu: 100m              # 0.1 of a core
    memory: 128Mi
  limits:                  # the cap
    cpu: 500m              # 0.5 of a core
    memory: 256Mi
```

If `requests` is 4Gi but you actually use 200Mi, you're wasting 95% of your node capacity. Tools: `kubectl top pods`, **Goldilocks**, **VerticalPodAutoscaler**.

### Drill (2 min)
You run a nightly batch job for 4 hours. On-demand or spot instance? *(Hint: spot. Even if interrupted, you restart from a checkpoint. 70% cheaper. Don't use spot for the *database*.)*

**Deep dive (later):** [devops/15-cloud-aws-basics.md](../devops/15-cloud-aws-basics.md)

---

## End-of-day checklist

- [ ] Angular: I can name three ways to shrink a bundle (lazy loading, tree-shaking, replacing big libs)
- [ ] Node.js: I can list the first 4 twelve-factor rules
- [ ] Spring: I can name the three states of a circuit breaker
- [ ] MongoDB: I know what `mongostat`, `mongotop`, and the profiler each show
- [ ] Postgres: I can explain why dead tuples slow queries and what VACUUM does
- [ ] HLD: I can explain partitions, consumer groups, and replay in Kafka
- [ ] LLD: I can sketch a Trie node holding top-K suggestions
- [ ] DSA: I can trace one window of Sliding Window Maximum
- [ ] DP: I can explain why a pool needs `release()` in `finally`
- [ ] DevOps: I can name two FinOps levers (right-sizing, spot, reserved, tiering)

**If you remember just one thing today:** every "make it fast / make it cheap" decision is **measure first, then act**. Bundle analyzer, mongostat, `pg_stat_user_tables`, `kubectl top` — measurement comes before optimization.

**Tomorrow:** Day 30 — the mock interview. Rapid-fire questions across all ten pillars, one HLD, one LLD, two DSA, and an honest self-assessment.
