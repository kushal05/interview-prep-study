# Day 4 — Transforming data and the cost of fast reads

> **Today's goal:** see how Angular pipes transform values in templates, how Node ships a useful standard library, how Spring "cross-cuts" concerns with AOP, and the real cost behind indexes and caches.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Pipes (built-in + a custom one) | 15m |
| 2 | Node.js | Core Modules (fs, path, os) | 15m |
| 3 | Spring Boot | Spring AOP (logging cross-cutting concerns) | 15m |
| 4 | MongoDB | Indexes basics & the ESR rule | 10m |
| 5 | Postgres | Window Functions (ROW_NUMBER, LAG) | 10m |
| 6 | HLD | Caching strategies (cache-aside, TTL) | 12m |
| 7 | LLD | API Design (REST verbs, status codes, idempotency) | 12m |
| 8 | DSA | Linked Lists (Reverse a Linked List) | 15m |
| 9 | Design Pattern | Builder | 8m |
| 10 | DevOps | Docker Basics (Dockerfile + image vs container) | 10m |

---

## 1. Angular — Pipes

### Why this exists
You have a date in your component (`new Date()`) but you want to show it as "May 20, 2026" in HTML. You don't want to write formatting code in the class. **Pipes** transform values right inside the template.

### The idea in plain English
A pipe is a small function you apply in a template with the `|` symbol — borrowed from Unix shells. `{{ value | pipeName }}` runs `value` through `pipeName` and shows the result.

Angular ships built-ins: `date`, `currency`, `uppercase`, `lowercase`, `json`, `async`. You can also write your own with `@Pipe`.

Pipes are **pure by default** — they only re-run when their input changes. That makes them cheap (no recomputing every change-detection cycle).

### Smallest working example
```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'truncate', standalone: true })
export class TruncatePipe implements PipeTransform {
    transform(value: string, limit = 50): string {
        if (!value) return '';
        return value.length > limit ? value.slice(0, limit) + '…' : value;
    }
}
```

```html
<!-- Built-ins -->
<p>{{ today | date:'mediumDate' }}</p>           <!-- May 20, 2026 -->
<p>{{ price | currency:'INR' }}</p>              <!-- ₹1,234.50 -->
<p>{{ name  | uppercase }}</p>                   <!-- KUSHAL -->

<!-- Our custom one -->
<p>{{ article.body | truncate:200 }}</p>
```

`transform(value, ...args)` is the contract. The first argument is what's on the left of the `|`; extra args come after the colons.

### Drill (2 min)
Why is `{{ items | filter:'active' }}` (a custom filter pipe) usually better than `{{ filterActive(items) }}` (a method call) in templates? *(Hint: methods get called on every change-detection cycle (many times per second). A pure pipe only re-runs when its inputs change — Angular caches the previous result.)*

**Deep dive (later):** [angular/pipes.md](../angular/pipes.md)

---

## 2. Node.js — Core Modules (fs, path, os)

### Why this exists
Before reaching for `npm install some-library`, check if Node already has what you need. Node's **core modules** (the ones prefixed with `node:`) handle files, paths, and OS info — no install needed.

### The idea in plain English
- **`node:fs`** — read and write files. Use the **promise** version (`node:fs/promises`) so you can `await` and don't block the event loop.
- **`node:path`** — join paths the right way. Never concatenate with `'/'` yourself — `path.join` works on Windows and POSIX.
- **`node:os`** — OS info: CPU count, free memory, temp directory.

The recurring rule: do anything slow asynchronously. Synchronous calls like `readFileSync` are fine at startup, but never in a request handler.

### Smallest working example
```javascript
import { readFile, writeFile } from 'node:fs/promises';
import path from 'node:path';
import os from 'node:os';

// File I/O — promise version (async, non-blocking)
const data = await readFile('config.json', 'utf8');
await writeFile('out.txt', 'hello');

// Path — cross-platform
const fullPath = path.join('users', 'kushal', 'docs');
console.log(fullPath);          // users/kushal/docs (POSIX) or users\kushal\docs (Windows)

// OS info
console.log(os.cpus().length);  // number of logical CPUs
console.log(os.tmpdir());       // OS temp directory
```

Always `'utf8'` when reading text files; otherwise you get a Buffer (raw bytes).

### Drill (2 min)
Why is `readFileSync` discouraged inside an Express route handler? *(Hint: Node has one thread for your JS. `readFileSync` blocks that thread until disk responds, freezing every other request in flight. The async version doesn't.)*

**Deep dive (later):** [nodejs/07-core-modules.md](../nodejs/07-core-modules.md)

---

## 3. Spring Boot — Spring AOP

### Why this exists
Imagine adding logging to every service method, or wrapping every method in a try/catch for metrics. Editing 100 methods is painful. **AOP (Aspect-Oriented Programming)** lets you define that behavior *once* and have Spring apply it to many methods automatically.

### The idea in plain English
AOP is like a security guard standing at every door of a building. You don't put a guard inside each room (each method). You define one guard (an **aspect**) and a rule for which doors they watch (a **pointcut**). Whenever someone walks through a matching door, the guard's behavior (the **advice**) runs.

Behind the scenes, Spring wraps your bean in a **proxy** — a stand-in object that looks identical. Calls go through the proxy, which fires the advice and then calls the real method.

You've already used AOP without knowing it: `@Transactional`, `@Async`, `@Cacheable`. They all use proxies.

### Smallest working example
```java
@Aspect
@Component
public class TimingAspect {
    private static final Logger log = LoggerFactory.getLogger(TimingAspect.class);

    // Pointcut: any public method in com.example.service
    @Around("execution(public * com.example.service..*(..))")
    public Object timeIt(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();                  // call the real method
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            log.info("{} took {}ms", pjp.getSignature().toShortString(), ms);
        }
    }
}
```

Every method in `com.example.service` is now timed and logged — without editing those methods.

### Drill (2 min)
You add `@Transactional` to a method, but it doesn't seem to start a transaction. The method is `private`. Why? *(Hint: AOP works via a proxy. Proxies can only intercept *public* calls that go through them; private calls happen on `this` and bypass the proxy. Make it public.)*

**Deep dive (later):** [spring-detailed/PART-16-transactions-proxies-aop.md](../spring-detailed/PART-16-transactions-proxies-aop.md)

---

## 4. MongoDB — Indexes basics & the ESR rule

### Why this exists
Without an index, Mongo has to look at *every document* in a collection to find matches — called a **collection scan** (COLLSCAN). On a 10-million-document collection, that's slow. An index is a side data structure (usually a B-tree) that makes lookups go from O(n) to O(log n).

### The idea in plain English
Think of a book's index at the back. Without it, finding every mention of "MongoDB" means flipping through every page. With it, you jump straight to the right pages.

A **compound index** indexes multiple fields together — useful when you query by multiple things. The order of fields matters.

The **ESR rule** is how you order fields in a compound index:
1. **E**quality fields first (matched with `$eq` or `$in`).
2. **S**ort fields next.
3. **R**ange fields (`$gt`, `$lt`) last.

Why? Equality narrows the search most aggressively, then sort uses the index's natural order, then range scans the rest.

### Smallest working example
```javascript
// Query
db.orders.find({ user_id: 42, status: "shipped" })
         .sort({ placed_at: -1 })
         .limit(10);

// Index following ESR — equality, equality, sort
db.orders.createIndex({ user_id: 1, status: 1, placed_at: -1 });

// Check what the planner picked
db.orders.find({ user_id: 42 }).explain("executionStats");
// Look for: "IXSCAN" (good) vs "COLLSCAN" (bad)
```

**Cost of indexes:** every insert/update has to update every relevant index. A collection with 10 indexes pays roughly 10× the write cost.

### Drill (2 min)
You have an index on `{ user_id: 1, status: 1 }`. Will a query on just `{ status: "shipped" }` use it? *(Hint: no. Compound indexes work only on a leading prefix — the first field, the first two, etc. A query starting with `status` alone won't use this index.)*

**Deep dive (later):** [mongodb/04-indexes.md](../mongodb/04-indexes.md)

---

## 5. Postgres — Window Functions

### Why this exists
Sometimes you want to compute a value for each row that depends on *other rows* — like ranking, running totals, or comparing to the previous row. `GROUP BY` collapses rows; **window functions** keep them.

### The idea in plain English
A window function runs alongside `SELECT` and computes a value per row, looking at a "window" of related rows.

The shape: `FUNCTION() OVER (PARTITION BY group ORDER BY sort)`. `PARTITION BY` divides rows into groups; `ORDER BY` orders within each group.

Three you should recognize:
- `ROW_NUMBER()` — 1, 2, 3, 4 per row, no ties.
- `LAG(col)` — value from the previous row.
- `LEAD(col)` — value from the next row.

### Smallest working example
```sql
-- Rank orders per user, highest total first
SELECT user_id, id, total,
       ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total DESC) AS rn
FROM orders;

-- user_id | id  | total | rn
-- 1       | 102 | 200   | 1
-- 1       | 101 | 100   | 2
-- 2       | 103 | 150   | 1

-- Day-over-day revenue using LAG
SELECT day, revenue,
       LAG(revenue) OVER (ORDER BY day) AS prev_day,
       revenue - LAG(revenue) OVER (ORDER BY day) AS delta
FROM daily_revenue;
```

The first query keeps every order row but adds a per-user ranking. With `GROUP BY`, you'd only see one row per user.

### Drill (2 min)
You want "top 3 highest-paid employees per department." Why does this need a window function rather than a simple `GROUP BY`? *(Hint: `GROUP BY` collapses every department to one row. You need to keep all employees but tag each with its rank — that's `ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC)`. Then `WHERE rn <= 3`.)*

**Deep dive (later):** [postgres/05-window-functions.md](../postgres/05-window-functions.md)

---

## 6. High-Level Design — Caching strategies

### Why this exists
Hitting your database 10,000 times a second is expensive. If the same data is read often, store the answer in a fast, in-memory **cache** (like Redis) and check there first. Caching is the single most common performance lever in real systems.

### The idea in plain English
A cache is a scratchpad on your desk. Looking something up on the desk is faster than walking to the filing cabinet. But the desk has limited space (so things get evicted), and the desk copy can go stale if the filing cabinet changes.

The most common pattern is **cache-aside** (lazy loading):
1. App asks the cache. If hit, done.
2. If miss, app asks the DB, stores the answer in the cache, returns it.

A **TTL (time-to-live)** is "how long until this cache entry expires." Short TTL = fresh but more DB hits; long TTL = fewer DB hits but more stale data.

### Smallest working example
```javascript
async function getUser(id) {
    const cached = await redis.get(`user:${id}`);
    if (cached) return JSON.parse(cached);

    const user = await db.users.findById(id);
    if (user) {
        await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 300);  // 5 min TTL
    }
    return user;
}
```

When you update the DB user, also delete the cache key (`redis.del('user:42')`) — otherwise readers see the old value until TTL.

### A picture of the flow
```
   Client → App → check cache (Redis) → hit? return immediately
                                      → miss? fetch from DB
                                              → write to cache
                                              → return
```

### Drill (2 min)
What's the trade-off when you set TTL to 1 hour instead of 1 minute? *(Hint: fewer DB hits (better perf, lower cost), but stale data can linger up to an hour. The answer depends on how tolerant your use case is of staleness — fine for a product catalog, terrible for a user's account balance.)*

**Deep dive (later):** [system-design/high-level-design/04-caching.md](../system-design/high-level-design/04-caching.md)

---

## 7. Low-Level Design — API Design (REST)

### Why this exists
A web API is the contract between your service and its clients. Done well, it's predictable, easy to retry, and easy to change. Done badly, every client integration is bespoke and brittle. **REST** is the most common style.

### The idea in plain English
REST is just: **resources + HTTP verbs + status codes**. A resource is a thing (`/users`, `/orders/42`). Verbs describe the action on it.

| Verb | What it does | Safe? | Idempotent? |
|---|---|---|---|
| GET | read | yes | yes |
| POST | create / non-idempotent action | no | no |
| PUT | replace entire resource | no | yes |
| PATCH | partial update | no | usually |
| DELETE | remove | no | yes |

**Idempotent** = doing it N times has the same effect as doing it once. Crucial for retries: clients (or load balancers) will retry on network errors.

URLs use **nouns**, not verbs: `GET /users/123`, not `POST /getUserById`.

### A clean URL pattern
```
GET    /users            list users
POST   /users            create one (returns 201, with Location header)
GET    /users/{id}       fetch one
PUT    /users/{id}       replace entire user
PATCH  /users/{id}       partial update
DELETE /users/{id}       remove
```

A few status codes worth knowing:
- `200` OK · `201` Created · `204` No Content (good for DELETE)
- `400` Bad Request · `401` Unauthorized · `403` Forbidden · `404` Not Found · `409` Conflict
- `500` Server Error · `503` Service Unavailable

### Drill (2 min)
Your endpoint sends an SMS. The client retries on a timeout. Without protection, you might send 5 SMSes. How do you prevent that? *(Hint: pass an `Idempotency-Key` header — a unique ID per attempt. The server records the key with the response and replays the same response for duplicates instead of re-sending the SMS.)*

**Deep dive (later):** [system-design/low-level-design/05-api-design-and-rest.md](../system-design/low-level-design/05-api-design-and-rest.md)

---

## 8. DSA — Linked Lists (Reverse a Linked List)

### The problem
Given the head of a singly-linked list, reverse it. Return the new head.

Input: `1 → 2 → 3 → 4 → 5 → null`
Output: `5 → 4 → 3 → 2 → 1 → null`

### The idea
You have three pointers: `prev`, `curr`, `next`. Walk through the list, and for each node:
1. Save the next node (so you don't lose the rest of the list).
2. Flip the current node's pointer to point backwards.
3. Move `prev` and `curr` one step forward.

It's like reversing a chain of cars — for each car, unhitch the one behind, hitch it to the one in front of you, then move on.

### Smallest working example
```javascript
class ListNode {
    constructor(val) { this.val = val; this.next = null; }
}

function reverseList(head) {
    let prev = null;
    let curr = head;
    while (curr !== null) {
        const next = curr.next;   // 1. save the next node
        curr.next = prev;          // 2. flip this node's pointer
        prev = curr;               // 3a. advance prev
        curr = next;               //  3b. advance curr
    }
    return prev;                   // prev is the new head
}
```

**Walk-through with `1 → 2 → 3`:**
```
start:  prev=null, curr=1
iter1:  next=2, 1.next=null, prev=1, curr=2     → list: null←1   2→3
iter2:  next=3, 2.next=1,    prev=2, curr=3     →       null←1←2   3
iter3:  next=null, 3.next=2, prev=3, curr=null  →       null←1←2←3
return prev=3
```

**Time:** O(n). **Space:** O(1) — just three pointer variables.

### Drill (2 min)
Why do we need to save `next` *before* flipping `curr.next`? *(Hint: once you set `curr.next = prev`, the original `curr.next` is gone. If you didn't save it first, you'd have no way to walk forward.)*

**Deep dive (later):** [dsa/ds-js/linked-lists.md](../dsa/ds-js/linked-lists.md) · [dsa/ds-java/linked-lists.md](../dsa/ds-java/linked-lists.md)

---

## 9. Design Pattern — Builder

### Intent
**Build a complex object step by step**, using a fluent API instead of a giant constructor with 10 arguments.

### When you'd use it
- A class has many fields, especially optional ones.
- You want immutability but constructor parameters are getting unwieldy.

### Smallest working example
```java
public final class Pizza {
    private final int size;
    private final boolean cheese;
    private final boolean pepperoni;
    private final boolean olives;

    private Pizza(Builder b) {
        this.size = b.size;
        this.cheese = b.cheese;
        this.pepperoni = b.pepperoni;
        this.olives = b.olives;
    }

    public static class Builder {
        private final int size;
        private boolean cheese, pepperoni, olives;

        public Builder(int size) { this.size = size; }

        public Builder cheese()    { this.cheese = true;    return this; }
        public Builder pepperoni() { this.pepperoni = true; return this; }
        public Builder olives()    { this.olives = true;    return this; }

        public Pizza build() { return new Pizza(this); }
    }
}

// Usage
Pizza p = new Pizza.Builder(12).cheese().pepperoni().olives().build();
```

Each method returns `this`, so calls chain. The call site reads like named arguments.

You've seen this pattern in `StringBuilder`, `HttpClient.newBuilder()...build()`, and SQL query builders.

### Drill (1 min)
Why is the constructor `private`? *(Hint: so callers must go through the `Builder`. That gives the Builder a chance to validate before constructing (e.g., reject pizza size < 6), and it keeps the surface area clean.)*

**Deep dive (later):** [design-patterns/common/creational/builder.md](../design-patterns/common/creational/builder.md)

---

## 10. DevOps — Docker Basics

### Why this exists
"It works on my machine" is the oldest bug in software. **Docker** packages your app + its dependencies + the right OS bits into an **image**, and any computer with Docker can run it identically.

### The idea in plain English
- **Image** = a frozen recipe. Layered filesystem + metadata. Immutable.
- **Container** = a running instance of an image. Like a process started from that recipe.

Same image, many containers — like running the same `.exe` many times.

A **Dockerfile** is the recipe to build the image. Each instruction (`FROM`, `COPY`, `RUN`) creates a **layer**. Docker caches layers — if an instruction's inputs haven't changed, the layer is reused. That's why you order Dockerfiles "least-changing first" (dependencies before source code).

### Smallest working example
```dockerfile
# Base image — always pin a version, never use :latest
FROM node:20-alpine

WORKDIR /app

# Copy dependency manifest FIRST so the npm install layer is cached
# until package.json changes
COPY package*.json ./
RUN npm ci --omit=dev

# Now copy source code (changes more often)
COPY src ./src

EXPOSE 3000
CMD ["node", "src/index.js"]
```

```bash
docker build -t my-app:1.0 .       # build image
docker run -p 3000:3000 my-app:1.0 # run container, map host:container ports
```

The `-p 3000:3000` maps your host's port 3000 to the container's port 3000.

### Drill (2 min)
Why do we copy `package.json` *before* `src`, not together? *(Hint: layer caching. Source code changes every commit, but dependencies change rarely. If we copy them together, every code change re-runs `npm ci`. Copying `package.json` first means `npm ci` only re-runs when dependencies actually change.)*

**Deep dive (later):** [devops/04-docker-basics.md](../devops/04-docker-basics.md)

---

## End-of-day checklist

- [ ] Angular: I wrote a one-method custom pipe
- [ ] Node.js: I can name three `node:` core modules and what they do
- [ ] Spring: I can explain why `@Transactional` doesn't work on private methods
- [ ] MongoDB: I can recite ESR (Equality, Sort, Range)
- [ ] Postgres: I wrote one `ROW_NUMBER() OVER (PARTITION BY ...)` query
- [ ] HLD: I can describe the cache-aside flow in 3 steps
- [ ] LLD: I know which HTTP verbs are idempotent and which aren't
- [ ] DSA: I reversed a linked list with three pointers
- [ ] DP: I built one `Pizza` using a Builder
- [ ] DevOps: I know the difference between an image and a container

**If you remember just one thing today:** every speed-up in this stack — a pipe, an index, a cache — buys you fast reads by paying extra at write time. Measure both sides before adding one.

**Tomorrow:** template-driven forms, streams, Spring auto-configuration, aggregation pipelines, Postgres index types, message queues, schema modeling, tree traversals, Prototype pattern, and Docker Compose.
