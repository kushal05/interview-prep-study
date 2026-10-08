# Day 25 — Shipping it: SSR, Docker, schedules, replication, multi-region

> **Today's goal:** make the app SEO-friendly, ship it in a tiny container, schedule background jobs, replicate Postgres, and design for two continents.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | SSR & hydration | 15m |
| 2 | Node.js | Deployment & Docker | 15m |
| 3 | Spring Boot | Spring scheduling (@Scheduled) | 15m |
| 4 | MongoDB | Change streams as pub/sub | 10m |
| 5 | Postgres | Streaming replication | 10m |
| 6 | HLD | Multi-region architecture | 12m |
| 7 | LLD | Job scheduler | 12m |
| 8 | DSA | Bit manipulation (XOR tricks) | 15m |
| 9 | Design Pattern | Service Layer | 8m |
| 10 | DevOps | DevSecOps & secrets | 10m |

---

## 1. Angular — SSR & Hydration

### Why this exists
A single-page Angular app ships an empty `<div>` and asks the browser to download JS to fill it. Search engines and slow phones see... nothing for a long time. **Server-Side Rendering (SSR)** runs Angular on the server, ships pre-rendered HTML, and the browser **hydrates** (attaches event listeners) on top.

### The idea in plain English
- **Client-side rendering (CSR)** = an open kitchen. You sit down, the kitchen prepares your plate while you wait. Slow first bite, fast updates after.
- **Server-side rendering (SSR)** = the restaurant pre-plates your meal. The plate arrives full. Then a waiter quietly adds utensils ("hydration") so you can interact.

Hydration is the trick that makes SSR worth it: instead of throwing away the server HTML and re-rendering, Angular **adopts** the existing DOM and just wires up listeners. No flicker, no double-work.

### Smallest working example
```bash
ng add @angular/ssr             # Angular 17+
```

This adds:
- `server.ts` — a tiny Express server that renders pages on demand.
- `provideClientHydration()` — turns on non-destructive hydration.
- An optional **prerender** list for static routes.

```typescript
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes),
    provideClientHydration(),    // critical — keeps server DOM, attaches listeners
  ],
});
```

```typescript
// guarding browser-only APIs
import { isPlatformBrowser } from '@angular/common';
constructor(@Inject(PLATFORM_ID) private platformId: object) {}
ngOnInit() {
  if (isPlatformBrowser(this.platformId)) {
    window.addEventListener('scroll', ...);   // only in browser
  }
}
```

### Drill (2 min)
You see a "hydration mismatch" warning in the console. What does it mean? *(Hint: the DOM the server rendered differs from what the client would render. Common cause: rendering `new Date()` (different on server and client) or `window.innerWidth`. Render only deterministic content during SSR.)*

**Deep dive (later):** [angular/ssr-and-hydration.md](../angular/ssr-and-hydration.md)

---

## 2. Node.js — Deployment & Docker

### Why this exists
"It worked on my machine" is a deployment crime. **Docker** ships your app **with its environment** — same Node version, same libs, same OS user. The container runs identically on your laptop, CI, and prod.

### The idea in plain English
A Docker image is a frozen filesystem. The `Dockerfile` is the recipe. A **multi-stage build** uses one image to compile/install and a smaller image to run — the final image doesn't carry the build tools.

The second important piece: **graceful shutdown**. When Kubernetes wants to stop your pod, it sends `SIGTERM`. You have ~30 seconds to finish current requests, close DB connections, and exit cleanly. Ignore SIGTERM and you'll drop in-flight requests.

### Smallest working example
```dockerfile
# --- build stage ---
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# --- runtime stage (tiny) ---
FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node                              # non-root!
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

```javascript
// graceful shutdown
const server = app.listen(3000);

function shutdown() {
  console.log('SIGTERM received, draining...');
  server.close(() => {                  // stop accepting new conns
    db.end().then(() => process.exit(0));
  });
  setTimeout(() => process.exit(1), 25_000);  // last resort
}
process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

### Drill (2 min)
Your Docker image is 1.2 GB. The team next door's is 80 MB. What did they do? *(Hint: multi-stage build + Alpine base + only copy `dist` and `node_modules` (production deps only — `npm ci --omit=dev`). Don't `COPY .` the whole repo into the final stage.)*

**Deep dive (later):** [nodejs/23-deployment-and-docker.md](../nodejs/23-deployment-and-docker.md)

---

## 3. Spring Boot — Spring Scheduling

### Why this exists
"Send a reminder email every morning at 9am." "Refresh the cache every 5 minutes." You don't want to spin up a cron daemon — Spring has `@Scheduled`.

### The idea in plain English
Annotate a method with `@Scheduled`, and Spring runs it on a timer. Three variants:
- `fixedRate = 60000` — run every minute, no matter how long the previous run took.
- `fixedDelay = 60000` — wait one minute *after the previous run finishes*.
- `cron = "0 0 9 * * *"` — cron expression (sec, min, hour, day, month, day-of-week).

Enable with `@EnableScheduling` on a `@Configuration` class.

### Smallest working example
```java
@SpringBootApplication
@EnableScheduling
class App { /* ... */ }

@Component
class Reminders {
    @Scheduled(cron = "0 0 9 * * *", zone = "Asia/Kolkata")
    void dailyAt9am() {
        // send daily summary emails
    }

    @Scheduled(fixedDelay = 60_000)
    void refreshCache() {
        // runs every 60s, waits if a run takes longer
    }
}
```

By default everything runs on **one thread** — long jobs delay others. Configure a pool:

```java
@Configuration
@EnableScheduling
class SchedulerConfig implements SchedulingConfigurer {
    public void configureTasks(ScheduledTaskRegistrar reg) {
        reg.setScheduler(Executors.newScheduledThreadPool(5));
    }
}
```

### Drill (2 min)
Your app runs on 3 replicas. `@Scheduled` runs the daily job. How many emails go out? *(Hint: three times — one per pod. Fix with a distributed lock (ShedLock with Redis/Mongo/Postgres) so only one pod runs the job.)*

**Deep dive (later):** [spring-next/08-spring-boot-advanced.md — Scheduling](../spring-next/08-spring-boot-advanced.md#6-scheduling)

---

## 4. MongoDB — Pub/Sub via Change Streams

### Why this exists
Mongo isn't a message broker, but **change streams** turn it into one. Subscribe to a collection and get real-time `insert/update/delete` events — no polling. Great for "watch this collection" features.

### The idea in plain English
A change stream is a tailable feed of the oplog (Mongo's internal change log used for replication). Open one with `db.collection.watch()`; Mongo pushes events as they happen, with a resume token you can use to pick up where you left off after a restart.

Requires a replica set or sharded cluster (not standalone) because change streams ride on the oplog.

### Smallest working example
```javascript
const stream = db.collection('orders').watch();

stream.on('change', (evt) => {
  // evt.operationType: 'insert' | 'update' | 'delete' | 'replace'
  // evt.fullDocument: the new doc (for inserts/updates)
  // evt._id: the resume token
  console.log('changed:', evt.documentKey, evt.operationType);
});

// Filter at the source — Mongo only pushes matching events
const filtered = db.collection('orders').watch([
  { $match: { operationType: 'insert', 'fullDocument.amount': { $gte: 1000 } } }
]);
```

Use cases:
- Trigger emails on new orders.
- Sync to a search index (Elastic, Atlas Search).
- Live-update a dashboard via WebSocket.

### Drill (2 min)
Your service consumes a change stream. It crashes for 10 minutes. Will it lose events? *(Hint: not if you saved the **resume token** on each event. On restart, pass it to `watch({ resumeAfter: token })` to pick up from there. Works as long as the oplog window covers your downtime.)*

**Deep dive (later):** [mongodb/17-change-streams-and-pubsub.md](../mongodb/17-change-streams-and-pubsub.md)

---

## 5. Postgres — Streaming Replication

### Why this exists
One Postgres = one machine = one failure. **Streaming replication** keeps one or more standby servers in lock-step with the primary. If the primary dies, you promote a standby. Standbys can also serve read traffic.

### The idea in plain English
Every write on the primary is appended to the **Write-Ahead Log (WAL)**. The standby connects and streams that WAL in near-real time, replaying it on its own copy. Two modes:
- **Asynchronous** (default) — fast, primary doesn't wait for standby. Tiny risk of data loss on failover.
- **Synchronous** — primary commits only after standby has the WAL. No loss, but slower (limited by network RTT).

A **hot standby** can serve reads while replicating; useful for read-heavy workloads.

### Smallest working example
```ini
# primary postgresql.conf
wal_level = replica
max_wal_senders = 10
synchronous_standby_names = ''     # async by default
```

```bash
# On standby: base backup from primary
pg_basebackup -h primary.host -D /var/lib/postgresql/data -U replicator -P -R -X stream

# That writes standby.signal and primary_conninfo into postgresql.auto.conf
# Start standby:
pg_ctl -D /var/lib/postgresql/data start
```

```sql
-- On primary: who's connected
SELECT application_name, state, sent_lsn, replay_lsn FROM pg_stat_replication;
```

### Failover
1. Primary dies.
2. Promote standby: `pg_ctl promote -D /var/lib/postgresql/data` (or your manager — Patroni, repmgr).
3. Apps reconnect via DNS or a connection-proxy update.

### Drill (2 min)
You have one primary and one async standby. Primary crashes 100ms after a commit, before the WAL reaches the standby. What happens? *(Hint: that last commit is lost when you promote the standby. To prevent it, use synchronous replication for critical writes — at the cost of latency.)*

**Deep dive (later):** [postgres/18-streaming-replication.md](../postgres/18-streaming-replication.md)

---

## 6. High-Level Design — Multi-region Architecture

### Why this exists
Two reasons to go multi-region: **resilience** (a whole region can die — see AWS outages) and **latency** (Indian users shouldn't wait for US servers). The choice is between **active-active** (both regions serve writes) and **active-passive** (only one is primary).

### The idea in plain English
- **Active-passive** — one region serves; the other is a warm standby with replicated data. On disaster, flip DNS. Easier to build (one source of truth) but slower recovery (RTO minutes, sometimes hours) and the standby region's capacity sits idle.
- **Active-active** — both regions serve writes. You need to either partition users by region (US users → US region) or solve **conflict resolution** (last-write-wins, CRDTs). Lower latency, higher complexity.

A simple rule:
- Read-heavy, low-write-conflict apps → active-active with regional partitioning.
- Strong-consistency apps (banks) → active-passive, primary in one region, standbys elsewhere.

### The big picture (active-passive)
```
       Route 53 (DNS health checks)
        /                   \
       ▼                     ▼
   Region A (active)     Region B (passive)
   - app servers         - app servers (warm)
   - primary DB ────▶ async replication ────▶ replica DB
   - cache
```

On failover, R53 swaps the active record to Region B, you promote the DB replica.

### Five things to mention
1. **Data residency** — GDPR, India's DPDP, etc. EU user data may have to stay in EU. Partitioning by region helps.
2. **Latency-based routing** — DNS sends users to the nearest region.
3. **Cross-region replication cost** — bandwidth between regions isn't free.
4. **RTO vs RPO** — RTO = how long to recover; RPO = how much data can you lose. Choose based on business need.
5. **Backups in another region** — even passive regions need their own backups. A bug that deletes data replicates too.

### Drill (2 min)
Your app uses active-active with last-write-wins. Two users update the same record at the same time in different regions. What can go wrong? *(Hint: one update silently overwrites the other. For collaborative editing or finance, last-write-wins isn't acceptable. Use vector clocks, CRDTs, or a single-writer partition per record.)*

**Deep dive (later):** [system-design/high-level-design/25-multi-region.md](../system-design/high-level-design/25-multi-region.md)

---

## 7. Low-Level Design — Job Scheduler

### Why this exists
"Send this email in 24 hours." "Retry this webhook with exponential backoff." "Generate reports every Sunday at 3am." You need a job scheduler.

### The classes
- `Job` — id, type, payload, `runAt`, status, attempts, priority.
- `JobQueue` — stores jobs ordered by `runAt`.
- `Worker` — polls the queue, runs jobs, updates status.
- `Scheduler` — accepts new jobs, persists them.

### The data structure
A **min-heap (priority queue) keyed by `runAt`** is the classic answer for delayed jobs. The earliest job is at the top. For persistence, an indexed table works fine:

```sql
CREATE TABLE jobs (
  id          BIGSERIAL PRIMARY KEY,
  payload     JSONB,
  run_at      TIMESTAMPTZ NOT NULL,
  priority    INT DEFAULT 0,
  status      TEXT DEFAULT 'pending',  -- pending/running/done/failed
  attempts    INT DEFAULT 0,
  locked_until TIMESTAMPTZ
);
CREATE INDEX ON jobs (run_at) WHERE status = 'pending';
```

Workers claim jobs atomically:
```sql
UPDATE jobs SET status='running', locked_until=now() + interval '1 minute', attempts=attempts+1
WHERE id = (
  SELECT id FROM jobs
  WHERE status='pending' AND run_at <= now()
  ORDER BY priority DESC, run_at ASC
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
RETURNING *;
```

`SKIP LOCKED` lets multiple workers race without blocking each other.

### Retry & backoff
```java
if (failed) {
    long delay = (long) Math.pow(2, attempts) * 1000;   // 1s, 2s, 4s, 8s, ...
    job.runAt = now().plusMillis(delay);
    job.status = attempts < maxAttempts ? "pending" : "dead";
}
```

Send dead jobs to a **DLQ** (dead-letter queue) for manual review.

### Drill (2 min)
A worker grabbed a job, then crashed. The job is stuck in `running`. How do other workers pick it up? *(Hint: every job has a `locked_until`. Workers also consider rows where `status='running' AND locked_until < now()` as available. The crashed worker's lock expires; another worker takes over.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Bit Manipulation (XOR Tricks)

### Why this exists
A few interview problems are trivial with XOR and confusing without it. Worth recognizing.

### The XOR cheat sheet
- `x ^ 0 = x`
- `x ^ x = 0`
- XOR is commutative and associative.

That means: if every number appears twice except one, **XOR everything** → the duplicates cancel, the unique one remains.

### Single Number — the classic
```javascript
// nums has every element twice except one. Find it. O(n) time, O(1) space.
function singleNumber(nums) {
  let acc = 0;
  for (const n of nums) acc ^= n;
  return acc;
}

singleNumber([4, 1, 2, 1, 2]);   // 4
```

### Other handy bit tricks
```javascript
// Is n a power of two?  Powers of two have a single bit set.
const isPowerOfTwo = n => n > 0 && (n & (n - 1)) === 0;

// Count set bits (Brian Kernighan)
function popCount(n) {
  let c = 0; while (n) { n &= n - 1; c++; } return c;
}

// Swap two numbers without a temp (cute trick, rarely useful)
let a = 5, b = 9;
a ^= b; b ^= a; a ^= b;   // a=9, b=5
```

### Drill (2 min)
A list has numbers 1..n exactly once, except one number is missing. Find it. *(Hint: XOR all `1..n` together; XOR all list elements; XOR those two results. Pairs cancel, leaving the missing number. O(n) time, O(1) space.)*

**Deep dive (later):** [dsa/ds-js/bit-manipulation.md](../dsa/ds-js/bit-manipulation.md)

---

## 9. Design Pattern — Service Layer

### Intent
**Put business rules in their own layer**, between controllers (HTTP) and repositories (DB). Controllers stay thin; services know the business.

### When you'd use it
- Almost any backend with non-trivial logic. Spring projects almost always have `@Service` classes.
- When the same business rule is needed in multiple controllers (web, batch, message handler).

### The layering picture
```
[ Controller (HTTP) ]   — parse request, return response
        │
        ▼
[ Service (business) ]  — rules, transactions, orchestration
        │
        ▼
[ Repository (data) ]   — talk to DB
```

### Smallest working example
```java
@RestController
@RequiredArgsConstructor
class OrderController {
    private final OrderService orders;
    @PostMapping("/orders")
    OrderDto create(@RequestBody NewOrder req) {
        return orders.placeOrder(req.userId(), req.items());
    }
}

@Service
@RequiredArgsConstructor
class OrderService {
    private final InventoryRepository inv;
    private final OrderRepository ord;
    private final EventBus bus;

    @Transactional
    OrderDto placeOrder(long userId, List<Item> items) {
        for (var i : items) inv.reserve(i.sku(), i.qty());
        var order = ord.save(new Order(userId, items));
        bus.publish(new OrderPlaced(order.id()));
        return OrderDto.from(order);
    }
}
```

The controller is dumb HTTP plumbing. The service owns the transaction and the rule "reserve inventory before saving the order."

### Drill (1 min)
Why is `@Transactional` on the **service** method, not the controller? *(Hint: the transaction boundary belongs to the business operation. Controllers shouldn't know about transactions; services do. Also, transactional behavior across services in the same call requires careful annotation placement.)*

**Deep dive (later):** [design-patterns/java/java-specific/service-layer.md](../design-patterns/java/java-specific/service-layer.md)

---

## 10. DevOps — DevSecOps & Secrets

### Why this exists
Security can't be a checklist at the end. **DevSecOps** = security is part of the pipeline from day one. The most common production sins are leaked secrets, unscanned containers, and unknown dependencies.

### The five musts
1. **Secrets out of code.** Use Vault, AWS Secrets Manager, GCP Secret Manager, Kubernetes Secrets (sealed). Never commit `.env` files. CI should fail if a secret is detected (`git-secrets`, `truffleHog`).

2. **KMS (Key Management Service)** for encryption keys. Apps never see the raw key — they ask KMS to encrypt/decrypt. Rotation is automatic.

3. **Image scanning.** Every Docker image goes through `trivy` or `grype` in CI. Block builds with critical CVEs.

4. **SBOM (Software Bill of Materials).** Generate a list of every library in your image (`syft`, `cyclonedx`). When a new CVE lands (Log4Shell), you instantly know which apps are affected.

5. **Dependency scanning.** `npm audit`, `pip audit`, `dependabot`, `snyk` — run on PRs. Auto-PR security patches.

### Kubernetes Secrets vs ConfigMaps — analogy
- **ConfigMap** = a whiteboard. Plaintext config (`LOG_LEVEL=info`). Anyone with cluster read can see it.
- **Secret** = a locked drawer. (Slightly) protected at rest in etcd, can be encrypted with a KMS key.

For real secrets, integrate with Vault or your cloud's secret manager via CSI driver — Secrets in K8s alone are base64, not encryption.

### Smallest working example
```bash
# Scan a Docker image
trivy image myapp:latest --severity HIGH,CRITICAL --exit-code 1

# Generate an SBOM
syft myapp:latest -o cyclonedx-json > sbom.json

# Pull a secret in code (AWS)
aws secretsmanager get-secret-value --secret-id prod/db-pass
```

```yaml
# K8s — fetch a secret from Vault via CSI driver (no plaintext in YAML)
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata: { name: db-secrets }
spec:
  provider: vault
  parameters:
    objects: |
      - objectName: db-password
        secretPath: secret/data/myapp
        secretKey: password
```

### Drill (2 min)
A developer committed `AWS_SECRET_ACCESS_KEY` to a public GitHub repo for 30 seconds before deleting the commit. What do you do? *(Hint: assume it's compromised — bots scan GitHub in seconds. Immediately rotate the key in AWS, audit CloudTrail for any usage, force-push history removal (`git filter-repo` or `bfg`), and add a pre-commit hook to block secrets going forward.)*

**Deep dive (later):** [devops/21-security-devsecops.md](../devops/21-security-devsecops.md)

---

## End-of-day checklist

- [ ] Angular: I can explain hydration as "adopt the server DOM, don't rebuild"
- [ ] Node: I wrote a multi-stage Dockerfile and a SIGTERM handler
- [ ] Spring: I know how to schedule a cron job and why only one pod should run it
- [ ] MongoDB: I can describe a change stream and a resume token
- [ ] Postgres: I know the difference between async and sync replication
- [ ] HLD: I can name two trade-offs of active-active vs active-passive
- [ ] LLD: I sketched a job scheduler with `SKIP LOCKED`
- [ ] DSA: I solved Single Number with one line of XOR
- [ ] DP: I can place `@Transactional` on the right layer
- [ ] DevOps: I can list five DevSecOps practices

**If you remember just one thing today:** every shipping concern — SSR, Docker, scheduling, replication, multi-region, secrets — comes down to **what happens when one thing fails**? Plan for it before it does, and the system stays up.

**Tomorrow:** you've completed the 25-day plan. Spend a day picking three weakest pillars, re-running their drills, and writing a mini-app that touches all four (Angular UI → Node API → Spring service → Postgres + Mongo).
