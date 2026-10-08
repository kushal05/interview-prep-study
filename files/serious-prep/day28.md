# Day 28 — Offline, discovery, and time-series

> **Today's goal:** make apps survive bad networks (PWAs), shut down gracefully (SIGTERM), find each other (service discovery), and handle time-shaped data and geography.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | PWA & Service Workers | 15m |
| 2 | Node.js | Health checks & graceful shutdown | 15m |
| 3 | Spring Boot | Service Discovery (Eureka) | 15m |
| 4 | MongoDB | Time Series Collections | 10m |
| 5 | Postgres | PITR backups (WAL archiving) | 10m |
| 6 | HLD | Geo-distributed systems | 12m |
| 7 | LLD | Pub/Sub system | 12m |
| 8 | DSA | Median of Two Sorted Arrays | 15m |
| 9 | Design Pattern | Null Object | 8m |
| 10 | DevOps | SRE practices (SLO/SLI) | 10m |

---

## 1. Angular — PWA & Service Workers

### Why this exists
A **PWA** (Progressive Web App) is **an app-like website that works offline**. On a flaky train Wi-Fi, your normal web app shows a "no internet" dinosaur. A PWA shows cached pages, queues form submits, and syncs when the network returns.

### The idea in plain English
A **Service Worker** is a tiny JavaScript file that sits between your app and the network — like a **smart cache proxy** inside the browser. The browser registers it once, then it intercepts every fetch. If offline, the worker serves cached responses; if online, it can serve cache *and* fetch fresh in the background.

Three things make a PWA: a service worker (offline), a `manifest.json` (icon + name for "Add to Home Screen"), and HTTPS.

### Smallest working example
```bash
# Angular gives you all three for free:
ng add @angular/pwa
```

This generates `ngsw-config.json`:
```json
{
  "index": "/index.html",
  "assetGroups": [{
    "name": "app",
    "installMode": "prefetch",      // download immediately on install
    "resources": { "files": ["/index.html", "/*.css", "/*.js"] }
  }],
  "dataGroups": [{
    "name": "api",
    "urls": ["/api/**"],
    "cacheConfig": { "maxAge": "1h", "maxSize": 100, "strategy": "freshness" }
  }]
}
```

`prefetch` = grab on install. `freshness` = try network first, fall back to cache. `performance` = serve from cache, refresh in background.

### Drill (2 min)
A user opens your app on the subway with no signal. The HTML/JS load — how? *(Hint: the service worker cached them on a previous visit. They never touched the network this time.)*

**Deep dive (later):** [angular/pwa.md](../angular/pwa.md)

---

## 2. Node.js — Health checks & graceful shutdown

### Why this exists
In Kubernetes, your pod can be killed any time (deploys, scaling, node failure). If you exit instantly mid-request, the user gets an error. **Graceful shutdown** = "let me finish what I'm doing, then exit cleanly."

### The idea in plain English
K8s sends your process a `SIGTERM` (polite "please stop"). You have ~30 seconds before it sends `SIGKILL` (forceful kill). In that window: stop accepting new requests, finish in-flight ones, close DB connections, exit.

**Health checks** tell K8s "I'm alive and well":
- **Liveness** — "Am I deadlocked? Restart me if not." Yes/no.
- **Readiness** — "Am I ready to handle traffic? Route around me if not." Yes/no.

Analogy: a restaurant closing. Lock the front door (stop new requests). Serve the people inside (finish in-flight). Wipe tables (close connections). Turn off the lights (exit).

### Smallest working example
```javascript
const server = app.listen(3000);

// Liveness/readiness endpoints
app.get('/healthz', (_, res) => res.send('ok'));         // alive
app.get('/ready', (_, res) =>
  dbReady ? res.send('ok') : res.status(503).send());

// Graceful shutdown
let shuttingDown = false;
process.on('SIGTERM', async () => {
  shuttingDown = true;
  server.close();                          // stop accepting new connections
  await db.end();                          // close DB pool
  process.exit(0);
});

app.use((req, res, next) =>
  shuttingDown ? res.status(503).send('shutting down') : next());
```

### Drill (2 min)
Why have *both* liveness and readiness? *(Hint: a pod can be alive (don't restart me!) but temporarily unready (warming up cache — don't route traffic yet). Same signal would either kill you or send bad traffic.)*

**Deep dive (later):** [nodejs/23-deployment-and-docker.md](../nodejs/23-deployment-and-docker.md)

---

## 3. Spring Boot — Service Discovery (Eureka, Consul)

### Why this exists
In a microservices system, the IP of `order-service` changes constantly — pods restart, scale up, move nodes. Hard-coding URLs is hopeless. **Service discovery** is a phonebook that services check in to and look each other up from.

### The idea in plain English
A **discovery server** (Eureka, Consul) is the **office directory**. Each service on startup says "Hi, I'm `order-service`, I'm at 10.0.0.42:8080." When another service wants to call orders, it asks the directory for the current address.

Two flavors:
- **Client-side discovery** — the caller fetches the list and picks one (Eureka model). Caller's load balancer.
- **Server-side discovery** — caller hits a known address (the gateway/k8s service), it picks. Simpler client code.

### Smallest working example
```yaml
# Eureka server (single dependency)
spring:
  application: { name: eureka-server }
eureka:
  client: { register-with-eureka: false, fetch-registry: false }
server: { port: 8761 }
```

```java
// A service registers itself
@SpringBootApplication
@EnableDiscoveryClient
public class OrderApp { ... }

// application.yml
spring: { application: { name: order-service } }
eureka:
  client:
    serviceUrl: { defaultZone: http://localhost:8761/eureka/ }
```

```java
// Another service calls orders by *name*, not URL
@FeignClient(name = "order-service")
interface OrderClient {
    @GetMapping("/orders/{id}") Order findById(@PathVariable String id);
}
```

Feign + Eureka resolves `order-service` to a live instance at call time.

### Drill (2 min)
What happens to in-flight calls when an instance dies? *(Hint: Eureka removes it after a heartbeat timeout (~30s). Until then, callers might pick a dead instance — that's why you also need retries and timeouts in the client.)*

**Deep dive (later):** [spring-next/08-spring-boot-advanced.md — Spring Cloud Overview](../spring-next/08-spring-boot-advanced.md#2-spring-cloud-overview)

---

## 4. MongoDB — Time Series Collections

### Why this exists
IoT sensors, stock prices, server metrics — data that arrives with timestamps and is mostly inserted (rarely updated). Storing this in a normal collection wastes space and is slow to query by time range. MongoDB 5.0+ has **time series collections** built for this shape.

### The idea in plain English
A time-series collection groups documents that share a **metaField** (e.g. `sensorId`) into **buckets** by time. Instead of one document per reading, MongoDB internally packs many readings together — way smaller on disk, way faster to scan by time range.

You write documents the normal way. The bucketing is invisible.

### Smallest working example
```javascript
db.createCollection("temps", {
  timeseries: {
    timeField: "ts",          // required — must be a date field
    metaField: "sensorId",    // groups data by sensor for bucketing
    granularity: "seconds"    // hint: how often points arrive
  }
});

// Insert (same as a normal collection)
db.temps.insertOne({ ts: new Date(), sensorId: "S1", value: 22.4 });

// Query — range scans use bucketing internally
db.temps.find({
  sensorId: "S1",
  ts: { $gte: ISODate("2026-05-20T00:00Z") }
});
```

Set `granularity` to `seconds` / `minutes` / `hours` to match your insertion rate. Wrong granularity = bigger buckets = slower scans.

### Drill (2 min)
Why is a time-series collection a bad fit for user profiles? *(Hint: profiles get *updated*, not just inserted. Time-series collections are append-only and optimize that pattern — random updates are slow.)*

**Deep dive (later):** [mongodb/15-change-streams-and-time-series.md](../mongodb/15-change-streams-and-time-series.md)

---

## 5. Postgres — PITR backups (WAL archiving)

### Why this exists
"Restore from last night's backup" loses up to 24 hours of data. **Point-in-Time Recovery (PITR)** lets you restore to *any* moment — e.g., "30 seconds before the dev ran the bad DELETE." It's the gold standard for production databases.

### The idea in plain English
Postgres writes every change to a **WAL** (Write-Ahead Log) before touching tables — it's the play-by-play of the database. PITR is:

1. A **base backup** (a snapshot of the database files).
2. **All WAL files** archived from that point onward.

To recover: restore the base backup, then replay the WAL up to your chosen timestamp. Like a video game save with all subsequent actions recorded — you can rewind to any frame.

### Smallest working example
```ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /archive/%f'    # or push to S3
```

```bash
# Take a base backup
pg_basebackup -D /backup/base -Fp -X stream

# Disaster strikes at 14:32:00. Recover to 14:31:30:
# 1. Restore base backup files
# 2. Create recovery.signal in $PGDATA, then in postgresql.conf:
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-05-20 14:31:30'

# 3. Start Postgres — it replays WAL until the target, then stops.
```

Tools that automate this: **pgBackRest**, **WAL-G**, **Barman**.

### Drill (2 min)
You only do nightly base backups. Could you recover to 2pm yesterday? *(Hint: yes, as long as you archived WAL continuously. Replay WAL from last night's backup forward to 2pm.)*

**Deep dive (later):** [postgres/19-backup-and-recovery.md](../postgres/19-backup-and-recovery.md)

---

## 6. HLD — Geo-distributed systems

### Why this exists
A single-region app fails when that region fails (AWS us-east-1 has *taken down the internet* multiple times). **Geo-distribution** means running in multiple regions and routing users to the closest healthy one.

### The idea in plain English
Three building blocks:

- **Geo-DNS** — DNS that returns different IPs based on the user's location. A user in Tokyo gets the Asia-Pacific IP, a user in Berlin gets the EU IP.
- **Regional failover** — if APAC goes down, geo-DNS redirects Tokyo users to the EU region.
- **CRDTs** (Conflict-Free Replicated Data Types) — special data structures (counters, sets) where concurrent writes in two regions merge without conflicts. Useful when you can't have a single primary.

The hard problem: **data**. Writing to two regions risks conflicts. You either pin writes to one region (simple but slow for far users) or use CRDTs / multi-master (complex but global writes).

### Patterns
| Strategy | Read | Write | Notes |
|---|---|---|---|
| Active-passive | local | one region (primary) | failover swaps primary |
| Active-active w/ partitioning | local | local for owned data | by user region usually |
| Multi-master + CRDTs | local | local | needs CRDT-friendly data |

### Drill (2 min)
Why is a globally-strong-consistent SQL DB across regions hard? *(Hint: a write must wait for acknowledgement from a distant region — adds 100+ms latency. Most apps trade strict consistency for speed.)*

**Deep dive (later):** [system-design/high-level-design/10-real-world-system-designs.md](../system-design/high-level-design/10-real-world-system-designs.md)

---

## 7. LLD — Pub/Sub system

### Why this exists
"When an order is placed, send an email, update inventory, notify shipping" — you don't want the order service to know about all three. **Pub/Sub** (publish/subscribe) lets the order service shout "order placed!" and any number of services listen — no direct calls.

### The idea in plain English
A **topic** is a named channel. Publishers post messages to it. Subscribers register interest. The broker (Kafka, RabbitMQ, Pub/Sub, SNS+SQS) routes copies of each message to every subscriber.

Key terms:
- **Topic** — the channel ("orders").
- **Subscription** — a subscriber's queue of unread messages.
- **Ordering** — do messages arrive in the order published? Usually only per-partition/key.
- **Delivery semantics** — at-most-once (may lose), at-least-once (may duplicate — design for idempotency), exactly-once (expensive).

### Smallest working example
```java
interface MessageBroker {
    void publish(String topic, Message msg);
    void subscribe(String topic, String groupId, Consumer<Message> handler);
}

class InMemoryBroker implements MessageBroker {
    private Map<String, Map<String, BlockingQueue<Message>>> topics = new ConcurrentHashMap<>();

    public void publish(String topic, Message msg) {
        topics.getOrDefault(topic, Map.of())
              .values().forEach(q -> q.offer(msg));      // fan out to every subscriber
    }

    public void subscribe(String topic, String groupId, Consumer<Message> handler) {
        var queue = topics
            .computeIfAbsent(topic, t -> new ConcurrentHashMap<>())
            .computeIfAbsent(groupId, g -> new LinkedBlockingQueue<>());
        new Thread(() -> { while (true) handler.accept(queue.take()); }).start();
    }
}
```

In Kafka: each `groupId` is a **consumer group** — work is split across consumers in a group, and each group gets every message.

### Drill (2 min)
"At-least-once" means messages may be delivered twice. How do you survive that? *(Hint: make your consumer **idempotent** — e.g., use the message's UUID as the DB key. A repeat insert collides and is safely ignored.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Median of Two Sorted Arrays

### The problem
Given two sorted arrays `nums1` and `nums2`, return the median (middle value) of the combined sorted array. Required time: **O(log(min(m, n)))**.

```
Input:  nums1 = [1, 3], nums2 = [2]
Output: 2.0

Input:  nums1 = [1, 2], nums2 = [3, 4]
Output: 2.5   // average of 2 and 3
```

### The idea
The naive merge is O(m+n) — too slow for the bar. The trick: **binary search for the partition**.

Imagine slicing both arrays at some point. Left side has `(m+n+1)/2` elements (combined), right side has the rest. The slice is correct when `max(left) <= min(right)` across both arrays. Binary search the smaller array for that slice.

### Code (the partition trick)
```java
public double findMedianSortedArrays(int[] A, int[] B) {
    if (A.length > B.length) return findMedianSortedArrays(B, A);
    int m = A.length, n = B.length, half = (m + n + 1) / 2;
    int lo = 0, hi = m;

    while (lo <= hi) {
        int i = (lo + hi) / 2;        // partition in A
        int j = half - i;             // partition in B
        int aLeft  = i == 0 ? Integer.MIN_VALUE : A[i - 1];
        int aRight = i == m ? Integer.MAX_VALUE : A[i];
        int bLeft  = j == 0 ? Integer.MIN_VALUE : B[j - 1];
        int bRight = j == n ? Integer.MAX_VALUE : B[j];

        if (aLeft <= bRight && bLeft <= aRight) {
            if ((m + n) % 2 == 1) return Math.max(aLeft, bLeft);
            return (Math.max(aLeft, bLeft) + Math.min(aRight, bRight)) / 2.0;
        } else if (aLeft > bRight) hi = i - 1;
        else                       lo = i + 1;
    }
    throw new IllegalArgumentException();
}
```

**Why O(log)** : we binary-search positions in the smaller array.

### Drill (3 min)
Trace `A=[1,3]`, `B=[2]`. First iteration: `i = 1, j = 1`. `aLeft=1, aRight=3, bLeft=2, bRight=∞`. Is `aLeft <= bRight && bLeft <= aRight`? *(Yes: 1 ≤ ∞ and 2 ≤ 3. Odd total → return `max(1, 2) = 2`.)*

**Deep dive (later):** [dsa/patterns.md](../dsa/patterns.md) · [dsa/CRAM.md](../dsa/CRAM.md)

---

## 9. Design Pattern — Null Object

### Intent
Instead of returning `null` and forcing every caller to write `if (x != null)`, return a **harmless object** that does nothing (or sensible defaults). Removes nullchecks.

### When you'd use it
- A `Logger.NULL` that silently discards messages — for tests or disabled logging.
- A `User.GUEST` returned when no one is logged in (still has `getName()`, just returns "Guest").
- A `Cache.NONE` that always misses.

### Smallest working example
```java
interface Logger {
    void log(String msg);
    Logger NULL = msg -> {};       // does nothing, never null
}

class OrderService {
    private final Logger logger;
    public OrderService(Logger logger) { this.logger = logger; }
    // no null checks anywhere — log just no-ops if NULL was passed
    void place() { logger.log("placing"); ... }
}

new OrderService(Logger.NULL).place();          // safe
new OrderService(new ConsoleLogger()).place();  // logs to console
```

The callers don't change. The `null` check vanishes.

### Drill (1 min)
What's the trade-off? *(Hint: callers can no longer distinguish "no logger" from "no-op logger" — sometimes you *want* to know nothing was configured. Null Object hides absence by design.)*

**Deep dive (later):** [design-patterns/common/behavioral/null-object.md](../design-patterns/common/behavioral/null-object.md)

---

## 10. DevOps — SRE practices (SLO/SLI, error budgets)

### Why this exists
"Is the system OK?" needs a real answer, not vibes. **SRE** (Site Reliability Engineering) gives you a vocabulary: **SLI** (what you measure), **SLO** (the target you promise), **error budget** (how much you're allowed to fail before stopping releases).

### The idea in plain English
- **SLI** (Service Level Indicator) — a *number* you measure. E.g., "the fraction of requests under 300ms."
- **SLO** (Service Level Objective) — a *target* for an SLI. "99.9% of requests under 300ms over 30 days." Think of SLO as **"we promise 99.9% uptime"** in writing.
- **SLA** — the *contract* (legal/contractual SLO, usually with refunds attached).
- **Error budget** — `100% − SLO` = how much failure you're allowed. 99.9% SLO ⇒ 0.1% budget ⇒ ~43 minutes of downtime per month.

The brilliant part: when you've **burned the budget**, you stop shipping risky features and focus on reliability. When the budget is healthy, you can take more risk.

**Chaos engineering** is intentionally breaking things in production (kill a pod, drop a region) to find weaknesses before they find you. Tools: Chaos Monkey, Gremlin.

### Drill (2 min)
Your SLO is 99.5% successful requests over 30 days. Today is the 20th, you've had 0.4% failures so far. Should you push a risky feature? *(Hint: budget is 0.5%, you've used 0.4%. Only 0.1% left for 10 days. Hold the risky feature; focus on stability.)*

**Deep dive (later):** [devops/20-incident-response.md](../devops/20-incident-response.md)

---

## End-of-day checklist

- [ ] Angular: I can explain why a service worker lets a PWA load offline
- [ ] Node.js: I can name two things to do on `SIGTERM` before exiting
- [ ] Spring: I can describe how Eureka helps services find each other
- [ ] MongoDB: I can list the three required time-series options (`timeField`, `metaField`, `granularity`)
- [ ] Postgres: I can explain why PITR needs both a base backup *and* WAL archives
- [ ] HLD: I can describe what geo-DNS does in one sentence
- [ ] LLD: I can define topic, subscription, and "at-least-once" delivery
- [ ] DSA: I understand why binary searching the partition gives O(log) median
- [ ] DP: I can explain why a Null Object removes null checks from callers
- [ ] DevOps: I can compute an error budget from an SLO (100% − SLO)

**If you remember just one thing today:** real production systems plan for **failure**, not success. Service workers cache for offline. SIGTERM handlers drain gracefully. PITR replays after disaster. SLOs accept that some failure is normal.

**Tomorrow:** performance and resilience — bundle optimization, twelve-factor apps, circuit breakers, Mongo ops tools, vacuum tuning, Kafka streams, autocomplete, sliding-window DSA, object pools, FinOps.
