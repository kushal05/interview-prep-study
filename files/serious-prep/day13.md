# Day 13 — Flattening, REST, Caches & Search

> **Today's goal:** master the four RxJS flattening operators, lock in REST API conventions, understand Hibernate's two-level cache, and sketch a rate limiter.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | RxJS Operators (mergeMap, switchMap, concatMap, exhaustMap) | 15m |
| 2 | Node.js | REST API Design (resources, verbs, status codes) | 15m |
| 3 | Spring Boot | Hibernate Caching (L1 vs L2) | 15m |
| 4 | MongoDB | Atlas & Cloud (M0 free tier, network access) | 10m |
| 5 | Postgres | Full-Text Search (tsvector, tsquery) | 10m |
| 6 | HLD | Rate Limiter Design (token bucket, leaky bucket) | 12m |
| 7 | LLD | Snake & Ladder game | 12m |
| 8 | DSA | Two Pointers (Container With Most Water) | 15m |
| 9 | Design Pattern | Strategy | 8m |
| 10 | DevOps | Jenkins Basics (declarative pipeline) | 10m |

---

## 1. Angular — RxJS Operators (mergeMap, switchMap, concatMap, exhaustMap)

### Why this exists
Often an outer stream emits something and you want to start an **inner** stream for each emission — like "for each search keystroke, fire an HTTP request." The four flattening operators differ only in how they handle **overlap**: what to do when a new outer value arrives while an inner is still running.

### The idea in plain English
| Operator | Behavior | Real use |
|---|---|---|
| `mergeMap` | Start every inner in parallel; merge their outputs | Independent side-effects, order doesn't matter |
| `switchMap` | Cancel the in-flight inner, start a new one (latest wins) | Type-ahead search, route param changes |
| `concatMap` | Queue inners, run one at a time, preserve order | Sequential POSTs, animations that mustn't overlap |
| `exhaustMap` | Ignore new outer emissions while one inner is still alive | Submit button — ignore double-clicks |

Mnemonic: **m**erge = all, **s**witch = swap, **c**oncat = queue, **e**xhaust = drop.

### Smallest working example
```typescript
// Type-ahead: cancel stale HTTP requests
search$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.api.search(q))             // previous request cancelled
);

// Idempotent upload queue: one at a time, in order
upload$ = this.queue$.pipe(
  concatMap(file => this.api.upload(file))
);

// Save button: ignore extra clicks while saving
save$ = this.click$.pipe(
  exhaustMap(() => this.api.save(this.form.value))
);
```

Picking the right one is mostly about: **do you want all of them? the latest? in order? or only the first?**

### Drill (2 min)
A user clicks a "save" button 5 times in 2 seconds. Which operator prevents 5 duplicate POSTs while the first one is still in flight? *(Hint: `exhaustMap` — it ignores new outer emissions until the current inner completes.)*

**Deep dive (later):** [angular/rxjs-operators-transformation.md](../angular/rxjs-operators-transformation.md)

---

## 2. Node.js — REST API Design (resources, verbs, status codes)

### Why this exists
REST gives your API a predictable shape so clients can use it without reading 50 pages of docs. Same conventions in every codebase = lower cognitive load forever.

### The idea in plain English
Think in **resources**, not actions. A resource is a *thing* — `/users`, `/orders/42`. Then use HTTP **verbs** for what you do to it:
- `GET` — read (idempotent, safe).
- `POST` — create (not idempotent).
- `PUT` — replace whole resource (idempotent).
- `PATCH` — partial update.
- `DELETE` — remove (idempotent).

**Status codes** in three families:
- **2xx success** — `200 OK`, `201 Created`, `204 No Content`.
- **4xx client error** — `400 Bad Request`, `401 Unauthorized` (not logged in), `403 Forbidden` (logged in, not allowed), `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`.
- **5xx server error** — `500` (unknown), `502/503/504` (upstream / unavailable / timeout).

### Smallest working example
```javascript
const app = require('express')();
app.use(express.json());

app.get('/users',          (req, res) => res.json(users));
app.get('/users/:id',      (req, res) => {
  const u = users.find(x => x.id === req.params.id);
  if (!u) return res.status(404).json({ error: 'not found' });
  res.json(u);
});
app.post('/users',         (req, res) => {
  const u = create(req.body);
  res.status(201).location(`/users/${u.id}`).json(u);
});
app.put('/users/:id',      (req, res) => res.json(replace(req.params.id, req.body)));
app.patch('/users/:id',    (req, res) => res.json(patch(req.params.id, req.body)));
app.delete('/users/:id',   (req, res) => { remove(req.params.id); res.status(204).end(); });
```

Notice `201 Created` returns a `Location` header pointing to the new resource — a small REST courtesy.

### Drill (2 min)
A client retries the same POST due to a network blip and you create two orders. What's the fix? *(Hint: idempotency keys — the client sends a unique header (`Idempotency-Key: abc-123`); the server stores the result of the first request and returns the same response for any retry with the same key.)*

**Deep dive (later):** [nodejs/13-rest-api-design.md](../nodejs/13-rest-api-design.md)

---

## 3. Spring Boot — Hibernate Caching (L1 vs L2)

### Why this exists
The database is the slowest piece of most apps. Caching saves hits to the DB. Hibernate (the JPA implementation) ships with two levels of caching baked in.

### The idea in plain English
- **L1 cache** — built in, per-session. Inside one `EntityManager`, fetching the same entity twice hits the cache the second time. Always on.
- **L2 cache** — optional, shared across sessions and threads (process-wide). Needs a provider like EhCache or Caffeine and `@Cacheable` on the entity. Survives across requests.

Analogy: L1 is your **desk drawer** — instantly reachable, but only yours and only for now. L2 is the **bookshelf** in the shared office — slower than the drawer, but anyone in the team can reach it.

Cache invalidation is the hard part. Stale data is worse than no cache. Use L2 carefully: read-heavy, rarely-changing entities (e.g., country lists).

### Smallest working example
```java
// 1. Enable L2 in application.properties
//    spring.jpa.properties.hibernate.cache.use_second_level_cache=true
//    spring.jpa.properties.hibernate.cache.region.factory_class=jcache

// 2. Mark the entity cacheable
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country {
    @Id private String code;
    private String name;
}

// L1 demo — same session, second find is a cache hit
public void demo() {
    User a = repo.findById(1L).get();    // SQL fires
    User b = repo.findById(1L).get();    // cache hit, no SQL
    // a == b
}
```

### Drill (2 min)
You enable L2 cache on `User`. Another microservice updates the `users` table directly in Postgres. What goes wrong? *(Hint: stale data — Hibernate doesn't see the external change and serves the old cached entity. L2 cache only works safely when **all** writes go through Hibernate.)*

**Deep dive (later):** [spring-detailed/PART-11-orm-internals.md § 11.9 The second-level cache](../spring-detailed/PART-11-orm-internals.md#119-the-second-level-cache-l2)

---

## 4. MongoDB — Atlas & Cloud (M0 free tier)

### Why this exists
Running a production Mongo cluster yourself means handling replication, backups, upgrades, monitoring, and TLS. **MongoDB Atlas** is the official cloud-managed Mongo — it does all that for you on AWS / GCP / Azure. The free **M0** tier is enough to ship a real demo project.

### The idea in plain English
You create a project in Atlas, spin up a cluster (M0 free, M10+ paid), and Atlas gives you a connection string. Three things you must configure before the connection works:
1. **A database user** with a password.
2. **Network access** — IP allowlist. Add your laptop's IP, or `0.0.0.0/0` for "from anywhere" (dev only, never prod).
3. **The driver connection string** in your app.

M0 specs: 512 MB storage, shared CPU/RAM, ~100 connections. Fine for demos and learning. Backups are automatic on M10+.

### Smallest working example
```javascript
// .env (DO NOT COMMIT)
// MONGO_URI=mongodb+srv://app-user:s3cret@cluster0.abcde.mongodb.net/myapp

const { MongoClient } = require('mongodb');
const client = new MongoClient(process.env.MONGO_URI);

await client.connect();
const db = client.db('myapp');
await db.collection('users').insertOne({ name: 'Jane' });
```

`mongodb+srv://` triggers DNS-based replica-set discovery — the driver finds all nodes automatically.

### Drill (2 min)
You can connect from your laptop but not from your deployed Vercel app. What's likely missing? *(Hint: Atlas network access — add Vercel's IPs (or `0.0.0.0/0` for early testing) to the allowlist. The database user/password are separate from network access.)*

**Deep dive (later):** [mongodb/13-security.md](../mongodb/13-security.md)

---

## 5. Postgres — Full-Text Search (tsvector, tsquery)

### Why this exists
`WHERE name LIKE '%search%'` is fine for tiny tables but slow on millions of rows and can't handle "words in any order" or stemming ("run" matches "running"). Postgres has built-in full-text search via two special types.

### The idea in plain English
- `tsvector` — a normalized list of "lexemes" (stemmed words) extracted from text. `'The cats are running'` → `'cat':2 'run':4`.
- `tsquery` — a search expression. `'cat & run'` matches docs containing both.
- The `@@` operator tests a vector against a query.
- A **GIN index** on the tsvector column makes it fast.

You typically store a precomputed `tsvector` column (kept in sync via a trigger or generated column) so queries don't recompute.

### Smallest working example
```sql
ALTER TABLE articles ADD COLUMN search_vec tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title,'')),    'A') ||
    setweight(to_tsvector('english', coalesce(body, '')),    'B')
  ) STORED;

CREATE INDEX idx_articles_fts ON articles USING gin (search_vec);

-- Search; rank by relevance
SELECT id, title,
       ts_rank(search_vec, plainto_tsquery('english', 'kafka streams')) AS rank
  FROM articles
 WHERE search_vec @@ plainto_tsquery('english', 'kafka streams')
 ORDER BY rank DESC
 LIMIT 10;
```

`setweight(..., 'A')` boosts matches in the title over the body. `plainto_tsquery` accepts user input safely.

### Drill (2 min)
A user searches `"streaming"` but your articles contain `"streams"`. Will they match? *(Hint: yes, if you used the `english` config — both stem to `stream`. That's why FTS feels smarter than `LIKE`.)*

**Deep dive (later):** [postgres/14-full-text-search.md](../postgres/14-full-text-search.md)

---

## 6. HLD — Rate Limiter Design (token bucket, leaky bucket)

### Why this exists
Without rate limiting, one buggy client can hammer your API and starve everyone else. Rate limiters cap how often a client (by IP, user ID, API key) can hit you.

### The idea in plain English
Two classic algorithms:
- **Token Bucket** — each client has a bucket holding up to N tokens; tokens refill at a fixed rate (say, 10/sec). Each request consumes one token. Empty bucket → reject (or wait). Allows **bursts** up to bucket size.
- **Leaky Bucket** — requests are queued in a bucket with a hole at the bottom; the bucket drains at a fixed rate. Smooths out bursts to a steady stream. Overflow → reject.

Plus simpler **fixed window** (count requests per minute — but has a boundary spike problem) and **sliding window** (more accurate, slightly more state).

Where to store state? **Redis** is the standard choice — atomic ops (`INCR`, `EXPIRE`), low latency, distributed across all your API servers.

### Smallest working example
```javascript
// Token bucket using Redis (pseudocode)
async function allow(userId) {
  const key = `rate:${userId}`;
  const now = Date.now();

  // Atomic-ish refill + consume via a Lua script in production;
  // simplified here.
  const [tokens, lastRefill] = await redis.hmget(key, 'tokens', 'ts');
  let t = tokens != null ? +tokens : BUCKET_SIZE;
  const elapsed = (now - (+lastRefill || now)) / 1000;
  t = Math.min(BUCKET_SIZE, t + elapsed * REFILL_PER_SEC);

  if (t < 1) return false;                                  // reject
  await redis.hmset(key, 'tokens', t - 1, 'ts', now);
  await redis.expire(key, 3600);
  return true;
}
```

Production: do the refill+consume in a Lua script so it's atomic.

### Drill (2 min)
Your API allows 100 req/min. A client makes 100 requests in the first second of each minute. With a fixed-window limiter, what's the worst case? *(Hint: 100 requests at second 59 of one window, then 100 at second 1 of the next — 200 in 2 seconds. Sliding window or token bucket avoids this boundary spike.)*

**Deep dive (later):** [system-design/high-level-design/10-real-world-system-designs.md](../system-design/high-level-design/10-real-world-system-designs.md)

---

## 7. LLD — Snake & Ladder game

### Why this exists
A small, complete game with dice, multiple players, and "events on landing" — great practice for event/listener thinking and clean OOP.

### The idea in plain English
Classes:
- **Board** — size (100), map of `position → jump target` (snakes pull you down, ladders push you up).
- **Player** — name and current position.
- **Dice** — `roll()` returns 1–6.
- **Game** — round-robin: roll, move, apply jumps, check win.

The cleanest design treats snakes and ladders identically as **Jump(from, to)** entries on the board. The board doesn't care if it's a snake or ladder; it just teleports the player.

### Smallest working example (skeleton)
```java
class Board {
    final int size = 100;
    final Map<Integer, Integer> jumps = new HashMap<>();
    void addJump(int from, int to) { jumps.put(from, to); }
    int resolve(int pos) { return jumps.getOrDefault(pos, pos); }
}

class Game {
    final Board board; final Queue<Player> players; final Dice dice = new Dice();

    public Player play() {
        while (true) {
            Player p = players.poll();
            int roll = dice.roll();
            int next = p.position + roll;
            if (next > board.size) {                  // overshoot: skip turn
                players.offer(p); continue;
            }
            p.position = board.resolve(next);         // applies snake or ladder
            if (p.position == board.size) return p;   // winner
            players.offer(p);
        }
    }
}
```

The "overshoot stays put" rule, and "resolve" treating snakes and ladders as one Map, keeps the loop clean.

### Drill (2 min)
You're asked to add a "miss-a-turn" trap on square 17. Where does the new logic live? *(Hint: add a new mechanism, e.g., a `Map<Integer, Effect>` of square-to-effect on the Board, and apply the effect after `resolve(...)`. Don't sprinkle `if (square == 17)` checks in `Game`.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Two Pointers: Container With Most Water

### The problem
Given heights `[1, 8, 6, 2, 5, 4, 8, 3, 7]`, treat each as a vertical line. Pick two lines to form a container (water sits between them). The container's area = `min(h[i], h[j]) * (j - i)`. Return the maximum area.

```
Input:  [1,8,6,2,5,4,8,3,7]
Output: 49      // lines at index 1 (h=8) and 8 (h=7) → min(8,7)*(8-1) = 49
```

### The idea
Naive: check all pairs → O(n²). Smart: **two pointers** at the ends, move inward.

- Start with `left = 0, right = n-1`. Compute area.
- The width can only **decrease**. To possibly get a bigger area, we need a **taller** line.
- Move the pointer at the **shorter** line inward — moving the taller one can never help (shorter still caps the height, width shrinks).
- Stop when the pointers meet.

Each pointer moves at most n times → **O(n)**.

### Smallest working example
```javascript
function maxArea(h) {
  let left = 0, right = h.length - 1, best = 0;
  while (left < right) {
    const area = Math.min(h[left], h[right]) * (right - left);
    best = Math.max(best, area);
    if (h[left] < h[right]) left++;
    else                    right--;
  }
  return best;
}

maxArea([1,8,6,2,5,4,8,3,7]);   // 49
```

**Time:** O(n). **Space:** O(1).

### Drill (3 min)
Why is moving the **shorter** pointer always at least as good as moving the taller one? *(Hint: the height is capped by the shorter side. Moving the taller one reduces width but keeps the same cap or worse — area can't grow. Moving the shorter one *might* find a new taller line.)*

**Deep dive (later):** [dsa/ds-js/two-pointers.md](../dsa/ds-js/two-pointers.md) · [dsa/ds-java/two-pointers.md](../dsa/ds-java/two-pointers.md)

---

## 9. Design Pattern — Strategy

### Intent
**Make an algorithm pluggable.** Define a family of algorithms behind one interface; let the client pick which one at runtime.

### When you'd use it
- Payment: card, UPI, PayPal — same `pay(amount)` interface, different implementations.
- Sorting: pick `QuickSort`, `MergeSort`, `BubbleSort` based on input size.
- Compression: gzip, brotli, zstd — pick by client capability.

Analogy: a pen body with swappable tips (fine, bold, gel). Same pen, different tips, plug in whichever you need.

### Smallest working example
```java
interface PaymentStrategy {
    void pay(int cents);
}

class CardPayment implements PaymentStrategy {
    public void pay(int c) { System.out.println("Card: " + c); }
}
class UpiPayment implements PaymentStrategy {
    public void pay(int c) { System.out.println("UPI: " + c); }
}

class Checkout {
    private PaymentStrategy strategy;
    public void setStrategy(PaymentStrategy s) { this.strategy = s; }
    public void checkout(int amount) { strategy.pay(amount); }
}

// Usage — pick at runtime
Checkout c = new Checkout();
c.setStrategy(new UpiPayment());
c.checkout(99_00);
```

Adding `PayPalPayment` tomorrow doesn't change `Checkout` — that's the Open/Closed Principle in action.

### Drill (1 min)
How is Strategy different from polymorphism in general? *(Hint: Strategy is a **specific use** of polymorphism — the swap happens at runtime, often via setter or constructor injection. Plain polymorphism just means "many implementations of one interface" — Strategy adds the intent of choosing dynamically.)*

**Deep dive (later):** [design-patterns/common/behavioral/strategy.md](../design-patterns/common/behavioral/strategy.md)

---

## 10. DevOps — Jenkins Basics (declarative pipeline)

### Why this exists
Jenkins is the original CI/CD server — open source, self-hosted, runs anywhere. Huge install base in enterprises. You'll see it in interviews and at most companies older than 5 years.

### The idea in plain English
You write a **Jenkinsfile** (Groovy-flavored YAML, basically) committed to your repo. Jenkins reads it on each push. The **declarative pipeline** is the modern, structured syntax — `pipeline { agent ... stages { stage('build') { ... } } }`.

Key concepts:
- **Agent** — where to run (any worker, a specific label, a Docker image).
- **Stages** — logical phases (Build, Test, Deploy). Each contains **steps**.
- **Post** — hooks for `success`, `failure`, `always` — send Slack, archive logs.
- **Credentials** — stored in Jenkins, injected via `withCredentials` block.

### Smallest working example
```groovy
// Jenkinsfile
pipeline {
  agent { docker { image 'node:20-alpine' } }

  stages {
    stage('Install') { steps { sh 'npm ci' } }
    stage('Test')    { steps { sh 'npm test' } }
    stage('Build')   { steps { sh 'npm run build' } }

    stage('Deploy') {
      when { branch 'main' }
      steps {
        withCredentials([string(credentialsId: 'deploy-key', variable: 'KEY')]) {
          sh './deploy.sh'
        }
      }
    }
  }

  post {
    failure { mail to: 'team@x.com', subject: 'Build failed', body: "${env.BUILD_URL}" }
  }
}
```

The Docker agent ensures every build runs in the same Node 20 image — no "works on my Jenkins box" issues.

### Drill (2 min)
GitHub Actions vs Jenkins for a brand-new GitHub project? *(Hint: GitHub Actions — it's already integrated, free for public repos, zero infra. Jenkins is the right choice when you need self-hosted control, an enormous plugin catalog, or you're already running it across the org.)*

**Deep dive (later):** [devops/12-jenkins-basics.md](../devops/12-jenkins-basics.md)

---

## End-of-day checklist

- [ ] Angular: I can pick mergeMap / switchMap / concatMap / exhaustMap given a scenario
- [ ] Node.js: I can match REST verbs to status codes (e.g., POST returns 201)
- [ ] Spring: I can explain L1 vs L2 caches and why L2 needs care
- [ ] MongoDB: I can connect to Atlas and know the 3 things to configure
- [ ] Postgres: I can write a `to_tsvector` + `@@` query and explain stemming
- [ ] HLD: I can sketch a token-bucket rate limiter on Redis
- [ ] LLD: I can model snakes and ladders as one `Jump` map
- [ ] DSA: I solved Container With Most Water with two pointers
- [ ] DP: I can describe Strategy with the swappable-pen-tip analogy
- [ ] DevOps: I can read a Jenkinsfile and identify agent, stages, post

**If you remember just one thing today:** **bound everything.** Bound how often clients can hit you (rate limit). Bound how much you keep in cache (TTL). Bound how many concurrent in-flight ops (exhaustMap, queues). Unbounded systems eventually break.

**Tomorrow:** time to consolidate — revisit weak spots, do a timed mock interview, and pick one project to deepen.
