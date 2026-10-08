# Day 30 — Mock Interview Day

> **Today's goal:** Practice answering, not learning. Speak each answer out loud or in writing. You've covered enough — now it's about retrieval.
>
> **Total time:** ~2h15m. (No 5-min breaks — these are timed mocks.)

| # | Pillar | Time | What you'll do |
|---|--------|------|----------------|
| 1 | Angular | 15m | 5 rapid-fire questions |
| 2 | Node.js | 15m | 5 rapid-fire questions |
| 3 | Spring Boot | 15m | 5 rapid-fire questions |
| 4 | MongoDB | 10m | 4 rapid-fire questions |
| 5 | Postgres | 10m | 4 rapid-fire questions |
| 6 | HLD | 15m | Design ONE system (TinyURL OR Twitter feed) |
| 7 | LLD | 15m | Design ONE system (Parking Lot OR Splitwise) |
| 8 | DSA | 15m | Solve 2 medium problems |
| 9 | Design Pattern | 8m | Name + intent + one example for 5 patterns |
| 10 | DevOps | 10m | 5 rapid-fire questions |

How to use this page: cover the answers with your hand. Read the question. Speak your answer aloud. *Then* peek at the model answer. Mark each Q with ✓ (nailed it), △ (close), or ✗ (need to revisit).

---

## 1. Angular — Mock Interview

### Q1. What's the difference between a component, a directive, and a pipe?
A **component** is a directive *with a template* — it produces DOM. A **directive** modifies an existing element (`*ngIf`, custom directives that attach behavior). A **pipe** transforms a value for display (`{{ price | currency }}`). All three are declared in a module/standalone import.

### Q2. Signals vs RxJS — when to use which?
**Signals** for synchronous state inside a component (counters, toggles, derived values). They integrate with change detection automatically. **RxJS** for async streams — HTTP, WebSocket, debounced search, anything time-based. Convert one to the other with `toSignal()` / `toObservable()`.

### Q3. What is change detection and what does OnPush change?
Change detection is Angular checking each component to see if a binding changed, then updating the DOM. **Default** strategy checks all components on every event. **OnPush** only checks a component when its `@Input`s change *by reference*, an event fires inside it, or you call `markForCheck()`. Result: fewer checks, faster app.

### Q4. How do you prevent XSS in Angular?
Angular escapes by default for `{{ }}` and `[innerHTML]`. The dangerous escape hatch is `DomSanitizer.bypassSecurityTrustHtml()` — only use after server-side sanitization. Pair with a **CSP** header (`Content-Security-Policy: default-src 'self'`) so the browser blocks any script that *does* slip through.

### Q5. How do you lazy-load a route and why does it matter?
Use `loadComponent` or `loadChildren` with dynamic `import()`. The route's code becomes a separate JS chunk, downloaded only when the user navigates there. Smaller initial bundle = faster Time To Interactive. Combine with route preloading strategies for the best UX.

---

## 2. Node.js — Mock Interview

### Q1. What is the event loop and why is it single-threaded?
Node uses one main thread to run JS, with a queue-based loop processing callbacks. I/O is delegated to libuv's worker pool; when it finishes, results land on the queue. Single-threaded means no concurrency bugs in your code — but a CPU-heavy task blocks everything. Offload such work to **worker_threads** or a job queue.

### Q2. Difference between `process.nextTick`, microtasks (Promises), and `setTimeout`?
`process.nextTick` runs *before* any other I/O in the same tick — highest priority. **Microtasks** (Promise `.then`) run after each macrotask, draining fully. **Macrotasks** (`setTimeout`, `setImmediate`, I/O) run one per loop iteration. Order: sync code → nextTick → microtasks → timers/I/O.

### Q3. How do streams help with large files?
Streams process data in chunks instead of loading the whole file into RAM. `fs.createReadStream(path).pipe(res)` reads and sends a 10 GB file with constant memory. Without streams, `fs.readFileSync` would OOM.

### Q4. How do you scale Node beyond one core?
Use the built-in **cluster** module or **PM2** to fork worker processes (one per CPU core); the master distributes connections. Behind that, run multiple containers behind a load balancer. For CPU-heavy parts of a single request, use **worker_threads**.

### Q5. How do you handle graceful shutdown?
Listen for `SIGTERM`. Stop accepting new connections (`server.close()`), let in-flight requests finish, close DB pools and queue workers, then `process.exit(0)`. Set a fallback timer (~25 s) to force-exit if anything hangs.

---

## 3. Spring Boot — Mock Interview

### Q1. What does Spring Boot do that Spring alone doesn't?
**Autoconfiguration** — looks at the classpath, configures sensible defaults (DataSource, web server, JSON). **Starter dependencies** — one BOM gives you a coherent set. **Embedded server** — runs as `java -jar`. **Production-ready endpoints** — Actuator for health, metrics, info. You can override anything; Spring Boot just removes boilerplate.

### Q2. How does `@Transactional` actually work?
Spring wraps the bean in a **proxy**. When you call an `@Transactional` method *from another bean*, the proxy starts a transaction, runs the method, commits or rolls back. Two gotchas: calling another `@Transactional` method on the *same* instance (`this.foo()`) bypasses the proxy → no new transaction. And it only rolls back on unchecked exceptions by default.

### Q3. What's the difference between `@Component`, `@Service`, `@Repository`, `@Controller`?
All four are stereotypes — they tell Spring "make me a bean." `@Repository` adds exception translation for persistence errors. `@Controller` is for MVC routing (paired with `@RequestMapping`); `@RestController` = `@Controller` + `@ResponseBody`. `@Service` is purely semantic — marks business logic. Use the right one for the layer to keep code readable.

### Q4. What is a circuit breaker and when do you reach for one?
A circuit breaker (Resilience4j) stops calling a failing downstream after N failures. States: closed (normal) → open (fail fast) → half-open (probe). Reach for it when calling external services that *can* be slow or down; pair with timeouts, retries, and a fallback. Prevents one bad dependency from taking down your service.

### Q5. How does Spring Security authenticate a request?
A chain of **filters** runs per request. `UsernamePasswordAuthenticationFilter` handles form login; a JWT filter reads the `Authorization` header and validates the token. On success it sets an `Authentication` in the `SecurityContext`. Then `FilterSecurityInterceptor` checks if that authority is allowed at the matched URL. Anything fails → 401/403.

---

## 4. MongoDB — Mock Interview

### Q1. When would you embed vs. reference?
**Embed** when the child data is small, owned by the parent, and read together (a user's addresses, an order's line items). **Reference** when the child is large, shared, or queried independently (a `Product` referenced by many `Order`s; an `Author` referenced by many `Article`s). Embedding favors read locality; referencing avoids duplication.

### Q2. What's the ESR rule for compound indexes?
**E**quality first, **S**ort second, **R**ange last. If your query is `{ status: 'paid' } sort by createdAt desc limit 10`, the right index is `{ status: 1, createdAt: -1 }`. Status (equality) gets the prefix, createdAt (sort) follows. Always verify with `.explain('executionStats')`.

### Q3. How do transactions work in MongoDB?
Multi-document ACID transactions on replica sets / sharded clusters since 4.0/4.2. `session.startTransaction()`, run ops, `commitTransaction()` or `abortTransaction()`. Default read concern is `snapshot`. They're slower than single-document ops — embed-when-possible avoids needing them.

### Q4. What does `mongos` do in a sharded cluster?
`mongos` is the query router. It reads the shard key from each query, consults the cluster metadata (held by config servers), and routes to the right shard(s). For queries *without* the shard key it does a scatter-gather across all shards — expensive. Pick shard keys with high cardinality and queries that include them.

---

## 5. Postgres — Mock Interview

### Q1. What's the difference between `INNER`, `LEFT`, `RIGHT`, `FULL` JOIN?
`INNER` returns only rows with matches on both sides. `LEFT` returns all left rows + matching right (NULLs where no match). `RIGHT` is the mirror. `FULL OUTER` returns everything from both with NULLs filling gaps. 90% of the time you want `INNER` or `LEFT`.

### Q2. What is MVCC and how does it affect isolation levels?
**Multi-Version Concurrency Control** — instead of locking, each transaction sees a snapshot of data as it was at start. New writes create new row versions (old ones stay until VACUUM). This means readers never block writers. Isolation levels (`READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`) differ in which snapshot you see.

### Q3. How do you read an `EXPLAIN ANALYZE` plan?
Read **inside-out** (leaves first). Look for: **Seq Scan** on large tables (missing index?), big delta between **estimated and actual** rows (stale `ANALYZE`?), `Sort` nodes spilling to disk, `Nested Loop` with a high outer row count. Fix the worst leaf first, re-run, repeat.

### Q4. When do you use a partial index?
When a query filters on a small subset of rows. `CREATE INDEX ON orders(user_id) WHERE status = 'open'` — index only open orders, not the 99% completed. Smaller index, faster writes, faster reads for the targeted query.

---

## 6. HLD — Design Twitter Feed (15 minutes)

> Pick ONE: **TinyURL** *or* **Twitter feed**. Write or speak through the design. Below is a model walkthrough for Twitter feed — adapt the structure for whichever you pick.

### 1. Clarify (1 min)
- Read-heavy or write-heavy? *(Read-heavy: ~1000:1.)*
- How many DAUs? *(Say 200M, 2B posts/day.)*
- Feed must show in < 200 ms p99.
- Strong consistency? *(No — eventual is fine.)*

### 2. Functional & non-functional requirements
- **Functional:** post a tweet, follow/unfollow, view home timeline (tweets by people you follow), view user profile.
- **Non-functional:** low latency on feed read, high availability, can tolerate seconds of feed staleness.

### 3. APIs
```
POST   /tweets           { content }
GET    /feed?cursor=...  → list of tweets
POST   /follow/{userId}
```

### 4. Capacity (back of envelope)
- 2B tweets/day → ~23K writes/sec, peak ~70K.
- 200M DAU × 50 feed views/day → 10B reads/day → ~115K reads/sec.
- Tweet = ~300 bytes → 600 GB/day, 220 TB/year.

### 5. Data model
- `tweets(id, userId, content, createdAt)` — partitioned by `userId`.
- `follows(followerId, followeeId)` — wide row per user.
- `feed(userId, tweetId, createdAt)` — precomputed per user.

### 6. The core trade-off: fan-out
- **Fan-out on write (push):** when user A tweets, write the tweet into the feed of every follower. Fast read, expensive write — explodes for celebrities (millions of followers per tweet).
- **Fan-out on read (pull):** at feed-view time, query recent tweets from everyone you follow and merge. Slow read.
- **Hybrid (what Twitter uses):** push for normal users, pull for celebrities. At feed view, merge precomputed feed + live celebrity tweets.

### 7. High-level architecture
```
Client
  │
  ▼
API Gateway ── Auth ── Rate limit
  │
  ├── Write path → Tweet Service → Cassandra (tweets)
  │                       │
  │                       └─ Kafka ─► Fan-out workers ─► Redis (per-user feed)
  │
  └── Read path → Feed Service ── Redis (precomputed feed) ── Cassandra (cold)
                       │
                       └── Celebrity Service (pull live tweets from celebs you follow)
```

### 8. Key decisions
- **Redis** for the hot per-user feed (most recent ~800 tweets). LRU evicts inactive users.
- **Cassandra** for full tweet history (write-heavy, wide-row friendly).
- **Kafka** decouples write from fan-out — bursts get absorbed in the queue.
- **CDN** for media (images, video) — never serve from origin.

### 9. Bottlenecks & solutions
- **Celebrity fan-out:** hybrid push/pull as above.
- **Hot tweet (viral):** cache aggressively; CDN for any media.
- **Feed pagination:** cursor-based (use last `createdAt + tweetId`), not offset.

### 10. Wrap-up
Mention what you'd *not* build day 1: search (later, with ElasticSearch), recommendations (offline ML), trending topics (streaming pipeline).

---

## 7. LLD — Design a Parking Lot (15 minutes)

> Pick ONE: **Parking Lot** *or* **Splitwise**. Below is a model for Parking Lot — focus on classes, relationships, key methods.

### 1. Clarify (1 min)
- Multiple lots? *(One lot with multiple levels.)*
- Vehicle types? *(Car, Motorcycle, Truck — each fits in a specific spot size.)*
- Payment? *(Hourly. Card or cash.)*
- Reserved spots? *(Out of scope today.)*

### 2. Core classes
```
ParkingLot
  └── Level (1..N)
        └── ParkingSpot (1..N)   spotType: SMALL | MEDIUM | LARGE
              └── parkedVehicle: Vehicle?

Vehicle (abstract)         type: CAR | MOTORCYCLE | TRUCK
  ├── Car
  ├── Motorcycle
  └── Truck

Ticket               id, vehicle, spot, entryTime, exitTime?, amount?
ParkingService       park(vehicle), unpark(ticketId)
PricingStrategy      compute(entryTime, exitTime, spotType) → BigDecimal
PaymentProcessor     pay(ticket, method) → Receipt
```

### 3. Relationships
- `ParkingLot` *has-many* `Level`. `Level` *has-many* `ParkingSpot`.
- `Vehicle` is the abstract parent; subclasses report their required spot size.
- `Ticket` references a `Vehicle` and a `ParkingSpot`. `PricingStrategy` is pluggable (hourly, flat rate, peak surge).

### 4. Key method signatures
```java
class ParkingService {
    Ticket park(Vehicle v) {
        ParkingSpot spot = findFirstFreeSpot(v.requiredSize());
        if (spot == null) throw new LotFullException();
        spot.assign(v);
        return ticketRepo.save(new Ticket(v, spot, Instant.now()));
    }

    Receipt unpark(String ticketId, PaymentMethod method) {
        Ticket t = ticketRepo.find(ticketId);
        t.setExitTime(Instant.now());
        BigDecimal amount = pricing.compute(t);
        t.getSpot().release();
        return payment.pay(t, method);
    }
}
```

### 5. Design patterns in play
- **Strategy** for `PricingStrategy` and `PaymentProcessor` — swap algorithms without touching `ParkingService`.
- **Factory** for `Vehicle` instances from API payloads.
- **State** for `ParkingSpot` (`FREE`, `OCCUPIED`, `OUT_OF_SERVICE`).
- **Singleton** for `ParkingLot` (one per deployment).

### 6. Concurrency
Two cars enter at once — both could claim the last spot. Solutions: lock per-level + atomic CAS on the spot, or push events to a queue and have a single worker assign. In a distributed setting, use a Redis lock or DB transaction with `SELECT ... FOR UPDATE`.

### 7. Extensions you'd mention
- Reserved spots (electric, disabled, premium).
- Sensors / IoT for live spot status.
- Multi-lot system with central availability service.
- Surge pricing during peak hours.

---

## 8. DSA — Two Medium Problems (15 minutes)

> Solve both. Speak your approach *before* coding. Brute force first, then optimize.

### Problem 1: Group Anagrams
Given an array of strings, group anagrams together.
```
Input:  ["eat","tea","tan","ate","nat","bat"]
Output: [["bat"],["nat","tan"],["ate","eat","tea"]]
```

**Idea:** anagrams share the same *sorted* characters. Use that as the map key.

```java
public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();
    for (String s : strs) {
        char[] c = s.toCharArray();
        Arrays.sort(c);
        map.computeIfAbsent(new String(c), k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(map.values());
}
```
**Time:** O(n · k log k) where k = average string length. **Space:** O(n · k).

A faster O(n · k) variant uses a 26-int frequency array as the key.

### Problem 2: Longest Substring Without Repeating Characters
Given a string, find the length of the longest substring with no repeating characters.
```
Input:  "abcabcbb"
Output: 3       // "abc"
```

**Idea:** sliding window. Expand right; if a duplicate appears, shrink left past the previous occurrence.

```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastIdx = new HashMap<>();
    int best = 0, left = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastIdx.containsKey(c) && lastIdx.get(c) >= left) {
            left = lastIdx.get(c) + 1;     // jump past the duplicate
        }
        lastIdx.put(c, right);
        best = Math.max(best, right - left + 1);
    }
    return best;
}
```
**Time:** O(n). **Space:** O(min(n, alphabet)).

---

## 9. Design Patterns Cram (8 minutes)

> Name + intent + one real example. Cover 5 patterns out loud.

| Pattern | Intent | One example |
|---|---|---|
| **Singleton** | One instance, global access | Logger, Spring beans (default scope) |
| **Factory** | Object creation without `new` everywhere | `Calendar.getInstance()`, `List.of()` |
| **Builder** | Construct complex objects step-by-step | `StringBuilder`, Lombok `@Builder`, OkHttp `Request.Builder` |
| **Strategy** | Swap algorithms behind a common interface | `Comparator`, payment processor switch |
| **Observer** | Subscribers notified on subject change | DOM events, RxJS, Spring `ApplicationEventPublisher` |
| **Decorator** | Add behavior without subclassing | `BufferedReader(new FileReader(...))`, Express middleware |
| **Adapter** | Make one interface look like another | `Arrays.asList(...)`, JDBC drivers |
| **Template Method** | Skeleton in parent, fill steps in child | `HttpServlet.service` → `doGet`, Spring `JdbcTemplate` |
| **Command** | Wrap a request as an object | Undo/redo, job queues (BullMQ jobs) |
| **Specification** | Combinable business rules | Spring Data `Specification<T>` |

Self-check: name **three** GoF categories. *(Creational, Structural, Behavioral.)*

---

## 10. DevOps — Mock Interview

### Q1. SIGTERM vs SIGKILL?
`SIGTERM` (15) is a polite stop request — the app can catch it, finish work, close connections, exit cleanly. `SIGKILL` (9) is the kernel killing the process instantly — uncatchable, work in flight is lost. Always handle SIGTERM in production code.

### Q2. Difference between liveness, readiness, and startup probes?
**Liveness** — "am I alive? restart me if not." **Readiness** — "am I ready for traffic? route around me if not." **Startup** — "I'm slow to boot; pause the other probes until I say I've started." Use all three for a slow-starting app with warm-up.

### Q3. What is a service mesh and what problem does it solve?
A service mesh (Istio, Linkerd) injects a sidecar proxy beside every pod. The proxy handles mTLS, retries, timeouts, traffic shifting, and emits metrics — all without code changes. Solves the "every service reimplements the same plumbing" problem across languages.

### Q4. Explain blue-green vs canary deployment.
**Blue-green:** two identical environments. Switch all traffic from blue to green at once. Rollback = flip back. Fast, but new version is exposed to 100% immediately. **Canary:** route a small fraction (1%, 5%, 25%) to the new version, watch metrics, ramp if healthy. Safer, slower.

### Q5. What is an SLO and what does an "error budget" mean?
**SLO** — the reliability target you commit to, e.g., 99.9% successful requests over 30 days. **Error budget** = `100% − SLO` = how much you're allowed to fail. 99.9% SLO ⇒ 0.1% ⇒ ~43 minutes/month. Burn budget too fast → freeze risky releases, focus on reliability.

---

## Self-Assessment Scorecard

Rate yourself 1-5 on each pillar. Be honest — this is for *you*.

| Pillar | Score (1-5) | Criteria |
|---|---|---|
| Angular | __ | 5 = can build a Reactive Form with custom validator from scratch; 1 = recognize the name |
| Node.js | __ | 5 = can explain the event loop + write a streaming server; 1 = recognize the name |
| Spring Boot | __ | 5 = can wire IoC, JPA, transactions, security blindfolded; 1 = recognize the name |
| MongoDB | __ | 5 = can model embed/reference + write a 3-stage aggregation; 1 = recognize the name |
| Postgres | __ | 5 = can read EXPLAIN ANALYZE + tune a query; 1 = recognize the name |
| HLD | __ | 5 = can design a major system (Twitter, Uber) end-to-end in 45 min; 1 = recognize the name |
| LLD | __ | 5 = can class-diagram a parking lot under interview pressure; 1 = recognize the name |
| DSA | __ | 5 = can solve LeetCode medium in 25 min; 1 = recognize the name |
| Design Pattern | __ | 5 = name 10 patterns + an example each; 1 = recognize the name |
| DevOps | __ | 5 = can ship a service on K8s with CI/CD, observability, SLOs; 1 = recognize the name |
| **Total** | **__ / 50** | |

### Score interpretation

- **40-50** → You're interview-ready. Schedule a real mock with another engineer or a service this week.
- **30-39** → Solid foundation. Pick the 2 lowest pillars and spend a focused week deepening each.
- **20-29** → Repeat the plan from Day 1 with focus on **coding along** — don't just read. Type each example.
- **< 20** → Don't worry. Broaden exposure first; see [Where to go next](#where-to-go-next).

---

## Where to go next

- **Real mock interviews:** Pramp (free, peer-to-peer), interviewing.io (anonymous, real engineers), friends in industry. One real mock teaches you more than three days of solo prep.
- **System design practice:** pick one design every week — Alex Xu's *System Design Interview vol 1 & 2*. Sketch it, time-box it, then read his version.
- **DSA cadence:** 1 LeetCode medium per day for 2 months. Mix patterns — don't grind only one. Use [dsa/patterns.md](../dsa/patterns.md) as your map.
- **Build something real:** apply 3+ pillars in a single side project (Angular front-end + Spring/Node back-end + Postgres + Redis + deployed on K8s). Hands-on consolidation beats any course.
- **Read source code:** Spring Boot's source, RxJS operators, Express internals. The first commit you understand changes how you think.
- **For deep-dives in this repo:** see [README.md](README.md) for the full file index.

---

## Final notes

**What 30 days gets you:** the vocabulary. You can hear "circuit breaker" or "ESR rule" or "fan-out on write" and not flinch. You can ask follow-up questions in an interview instead of nodding politely.

**What 30 days does not get you:** mastery. That takes years per pillar. Don't expect to leave today able to design Twitter from memory in 20 minutes — that's the goal at month 6.

**Pick your battles.** If you're interviewing for a backend role, go deep on Spring (or Node), Postgres, and HLD. If frontend, go deep on Angular, RxJS, and design patterns. Generalists need everything; specialists need three pillars deep + the rest at recognition level.

**One last drill:** without looking at the table of contents, name the 10 pillars in this plan. *(Angular, Node.js, Spring Boot, MongoDB, Postgres, HLD, LLD, DSA, Design Patterns, DevOps.)* If you got all 10, you have the map. The territory comes with practice.

Good luck. Now close this file and go solve a problem.
