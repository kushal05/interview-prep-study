# Day 7 — DI, async, exceptions, and the cloud-native bits

> **Today's goal:** see how Angular's DI hierarchy works, master the Promise combinators, handle errors in Spring properly, and meet Kubernetes' main objects.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Dependency Injection (providers, hierarchy) | 15m |
| 2 | Node.js | Promises & async/await | 15m |
| 3 | Spring Boot | Exception Handling (@ControllerAdvice) | 15m |
| 4 | MongoDB | Embedded vs Referenced (data modeling tradeoffs) | 10m |
| 5 | Postgres | Transactions & ACID (BEGIN/COMMIT/ROLLBACK) | 10m |
| 6 | HLD | Storage & CDN (object storage, edge caching) | 12m |
| 7 | LLD | Parking Lot system (classic LLD) | 12m |
| 8 | DSA | Heaps / Priority Queue (Kth Largest Element) | 15m |
| 9 | Design Pattern | Decorator | 8m |
| 10 | DevOps | Kubernetes Fundamentals | 10m |

---

## 1. Angular — Dependency Injection

### Why this exists
Day 1 you saw that Angular hands services to components automatically. Today: how does it pick *which* instance? The answer is the **injector hierarchy** — a tree of injectors, with lookup walking up from the component to the root.

### The idea in plain English
Think of nested folders. Each component, route, and module has its own "providers" folder. When a component asks for `LoggerService`, Angular looks in its folder first, then the parent's, then the parent's parent, all the way up. The first one that has it wins.

Two key things:
- `providedIn: 'root'` — register the service at the very top. One instance for the whole app. The default for `@Injectable`.
- **`InjectionToken`** — used when you want to inject something that isn't a class (a string, a config object). You can't `inject('apiUrl')` directly because there's no class; you create a token to stand in.

### Smallest working example
```typescript
import { Injectable, InjectionToken, inject } from '@angular/core';

// Token for non-class config
export const API_URL = new InjectionToken<string>('API_URL');

@Injectable({ providedIn: 'root' })
export class DataService {
    private url = inject(API_URL);            // get the string from DI
    list() { return fetch(`${this.url}/items`); }
}

// Provide at bootstrap
bootstrapApplication(AppComponent, {
    providers: [
        { provide: API_URL, useValue: 'https://api.example.com' },
    ],
});
```

`{ provide: TOKEN, useValue: ... }` is the simplest provider. There are also `useClass`, `useFactory`, and `useExisting` for richer cases.

### Drill (2 min)
A lazy-loaded feature module provides `LoggerService`. The root also provides it. Which instance does a component *inside* the lazy module get? *(Hint: the lazy module's own — DI walks up from the closest injector first. The lazy-module provider shadows the root one.)*

**Deep dive (later):** [angular/dependency-injection.md](../angular/dependency-injection.md)

---

## 2. Node.js — Promises & async/await

### Why this exists
JavaScript handles "do this when ready" with **Promises**. `async/await` is sugar over Promises that makes async code read like sync code. You need both, plus the four `Promise` combinators that handle multiple promises at once.

### The idea in plain English
A Promise is a state machine: `pending` → `fulfilled` or `rejected`. Once settled, it never changes. `async` functions always return a Promise. `await` unwraps the value, or throws if the Promise rejected.

The four combinators (for running multiple promises in parallel):

| Combinator | Resolves when | Rejects when |
|---|---|---|
| `Promise.all` | all fulfill | any rejects (fail-fast) |
| `Promise.allSettled` | all settle | never |
| `Promise.race` | first settles | first rejects (if first) |
| `Promise.any` | first fulfills | all reject |

**`all`** — strict parallel. **`allSettled`** — collect everything, including failures. **`race`** — first answer wins (often used for timeouts). **`any`** — first success wins.

### Smallest working example
```javascript
// Parallel fetch, tolerate partial failure
async function fetchDashboard(userId) {
    const [profile, orders, recs] = await Promise.allSettled([
        fetchProfile(userId),
        fetchOrders(userId),
        fetchRecommendations(userId),
    ]);
    return {
        profile: profile.status === 'fulfilled' ? profile.value : null,
        orders: orders.status  === 'fulfilled' ? orders.value  : [],
        recs:   recs.status    === 'fulfilled' ? recs.value    : [],
    };
}

// Race with timeout
const withTimeout = (p, ms) => Promise.race([
    p,
    new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), ms)),
]);
```

**Gotcha:** `array.forEach(async ...)` does *not* wait. Use `for...of` with `await` for serial, or `Promise.all(array.map(...))` for parallel.

### Drill (2 min)
When would you pick `Promise.allSettled` over `Promise.all`? *(Hint: when you want to wait for everything and tolerate partial failures — like fetching three dashboard widgets where one failing shouldn't break the others. Never use it for transactional writes where you need all-or-nothing.)*

**Deep dive (later):** [nodejs/03-promises-and-async-await.md](../nodejs/03-promises-and-async-await.md)

---

## 3. Spring Boot — Exception Handling

### Why this exists
Without proper handling, every exception leaks a stack trace as a 500 error. Real APIs need structured, sanitized error responses with the right status codes. Spring gives you **`@RestControllerAdvice`** to handle exceptions globally.

### The idea in plain English
You define handler methods that catch specific exception types and return a structured response. Spring routes any unhandled exception thrown in a controller to the matching handler. Most specific handler wins.

Spring 6 introduced **`ProblemDetail`** — the standard JSON error shape (RFC 7807). Use it instead of rolling your own.

### Smallest working example
```java
// A domain exception
public class NotFoundException extends RuntimeException {
    public NotFoundException(String msg) { super(msg); }
}

// Global handler — one for the whole app
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public ProblemDetail handleNotFound(NotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource Not Found");
        return pd;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleAny(Exception ex) {
        log.error("Unhandled", ex);                                  // log full trace internally
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "Unexpected error");   // sanitized externally
    }
}
```

Now any `NotFoundException` thrown from any controller becomes a 404 with a nice JSON body — no try/catch in the controller.

### Drill (2 min)
A controller throws `IllegalArgumentException`. There's no specific handler but there's `@ExceptionHandler(Exception.class)`. Will it match? *(Hint: yes. Handlers match by `instanceof`. `IllegalArgumentException` is an `Exception`, so the catch-all catches it. A more specific handler always wins if one exists.)*

**Deep dive (later):** [spring-next/04-spring-boot-rest-api.md](../spring-next/04-spring-boot-rest-api.md)

---

## 4. MongoDB — Embedded vs Referenced

### Why this exists
This is the single most important schema decision in Mongo, revisited with sharper rules. Get it right and queries are simple. Get it wrong and you fight Mongo for years.

### The idea in plain English
**Embed** child data inside the parent when:
- Data is always read together with the parent.
- The child has no independent identity.
- The child's growth is bounded.

**Reference** the child (store an ID) when:
- The child is queried independently.
- The relationship is many-to-many.
- The child grows unboundedly (user → events).

Three patterns map to three cases:

1. **Embed children inline** — order's line items.
2. **Reference from child to parent** ("one to squillions") — events with `userId`.
3. **Array of references on parent** — bounded many-to-many (student's enrolled courses).

### Smallest working example
```javascript
// EMBED — line items live with the order
{
  _id: "ORD-1",
  customer: "Kushal",
  lineItems: [
    { sku: "A1", qty: 2, price: 99 },
    { sku: "B7", qty: 1, price: 49 }
  ]
}

// REFERENCE — one user has unboundedly many events
// users
{ _id: "u1", email: "k@x.com" }
// events
{ _id: "e1", userId: "u1", type: "login" }
```

**The 16 MB rule:** every Mongo document is capped at 16 MB. If embedded data could grow beyond that, you must reference. Even at half that size, you should worry.

### Drill (2 min)
A blog has posts and comments. Comments can be in the thousands per post. Embed or reference? *(Hint: reference. Embedding risks doc bloat (the 16 MB cap), expensive partial updates, and lock contention on hot posts. A separate `comments` collection with `postId` is the right shape.)*

**Deep dive (later):** [mongodb/07-embedded-vs-referenced.md](../mongodb/07-embedded-vs-referenced.md)

---

## 5. Postgres — Transactions & ACID

### Why this exists
Money transfers, inventory deductions, signups — many operations must succeed *together* or not at all. **Transactions** wrap multiple statements into one atomic unit.

### The idea in plain English
The four ACID guarantees:

- **A**tomicity — all statements succeed, or all are rolled back. No partial state.
- **C**onsistency — every constraint (foreign keys, checks) holds before and after.
- **I**solation — concurrent transactions don't see each other's intermediate state.
- **D**urability — once committed, the data is on disk and survives crashes.

You start a transaction with `BEGIN`, make changes, and either `COMMIT` (save) or `ROLLBACK` (discard). If anything errors, the whole thing rolls back automatically.

### Smallest working example
```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
    -- check (in the app or via a constraint) that balance >= 0
COMMIT;
-- If anything in between failed, ROLLBACK rolls back both updates.
```

If the second `UPDATE` fails (account 2 doesn't exist), neither update is applied — the first one is rolled back. No money is lost or duplicated.

**Pitfall:** if a statement errors *inside* a transaction in psql, the rest of the transaction is "aborted" — every subsequent statement returns an error until you `ROLLBACK`. Savepoints (`SAVEPOINT`, `ROLLBACK TO`) let you recover.

### Drill (2 min)
Why do banks always use transactions for transfers but might skip them for analytics aggregations? *(Hint: banks need atomicity (both sides of the transfer must succeed) and isolation (no halfway state visible). Analytics tolerates eventual consistency, and the locking overhead of transactions on huge scans hurts throughput.)*

**Deep dive (later):** [postgres/08-transactions-and-acid.md](../postgres/08-transactions-and-acid.md)

---

## 6. High-Level Design — Storage & CDN

### Why this exists
Where do you put images, videos, backups, logs, ML model artifacts? Not in your database. **Object storage** (S3, GCS, Azure Blob) is the default. And to serve them fast worldwide, you put a **CDN** in front.

### The idea in plain English
**Object storage** is a giant key-value store for files. You PUT a blob with a key (path), you GET it back. Properties: very durable (11 nines), cheap, web-accessible via HTTP, scales infinitely. Not for low-latency random reads of small data — use a KV store for that.

A **CDN** (Content Delivery Network) is a globally-distributed cache. Edge servers near the user hold copies of your popular files. The first request to a region is slow (cache miss → fetch from origin). Subsequent requests are fast (cache hit at the edge).

```
User → DNS routes to nearest edge → cache hit? serve in 20ms
                                  → miss? fetch from origin (S3, app)
                                          store in cache, serve
```

**Cache control:** use long TTLs and fingerprinted file names (`app.abc123.js`) for static assets — when the content changes, the name changes, so the CDN naturally serves the new one. For truly dynamic content, use `Cache-Control: no-store`.

### A real example
```
1. Client uploads avatar           → S3 PUT to s3://bucket/u/123/avatar-v4.jpg
2. App records the URL in DB       → user.avatarUrl = '.../avatar-v4.jpg'
3. Next viewer requests the page   → app returns HTML with that URL
4. Viewer's browser fetches image  → routed via CDN
                                   → cache miss the first time, hit thereafter
```

When the user changes their avatar, the URL changes (`avatar-v5.jpg`). No cache invalidation needed.

### Drill (2 min)
Why use a versioned filename (`main.abc123.js`) instead of `main.js` plus cache invalidation? *(Hint: invalidation is slow and error-prone. Versioned URLs are immutable — the CDN can cache them forever. When you ship a new version, you just emit a different filename in the HTML. No invalidation API call needed.)*

**Deep dive (later):** [system-design/high-level-design/07-storage-and-cdn.md](../system-design/high-level-design/07-storage-and-cdn.md)

---

## 7. Low-Level Design — Parking Lot system

### Why this exists
The Parking Lot system is a classic LLD interview problem. It exercises identifying entities, choosing data structures, and applying a design pattern or two. You won't be asked anything ground-breaking — they're checking that you can decompose a fuzzy problem cleanly.

### The idea in plain English
Walk through requirements first, *out loud*. Don't dive into code.

Typical requirements:
- Multi-floor lot with multiple slot sizes (motorcycle, compact, large).
- Vehicle enters → get a ticket → park in a slot.
- Vehicle exits → return ticket → pay based on duration.
- Find the nearest available slot for a vehicle type.

Key entities: `Vehicle`, `ParkingSlot`, `Floor`, `Ticket`, `ParkingLot`, `FeeStrategy`. Slot has a state (`FREE`, `OCCUPIED`). The fee strategy is a separate interface so you can swap "hourly" for "tiered" later (Strategy pattern).

### A minimal sketch
```java
enum VehicleSize { MOTORCYCLE, COMPACT, LARGE }
enum SlotStatus  { FREE, OCCUPIED }

class Vehicle {
    String plate;
    VehicleSize size;
}

class ParkingSlot {
    String id;
    VehicleSize size;
    SlotStatus status = SlotStatus.FREE;
    boolean canFit(Vehicle v) {
        return status == SlotStatus.FREE && v.size.ordinal() <= size.ordinal();
    }
}

interface FeeStrategy {
    BigDecimal calculate(Ticket t);
}

class ParkingLot {
    List<Floor> floors;
    FeeStrategy fees;

    Ticket entry(Vehicle v) {
        // find first free slot, park, return ticket
    }
    BigDecimal exit(String ticketId) {
        // find ticket, free the slot, compute fee
    }
}
```

Things to mention in the interview:
- Use `EnumMap<VehicleSize, Queue<Slot>>` per floor for O(1) "find a free slot of size X."
- `FeeStrategy` is the Strategy pattern — easy to add weekend pricing, EV charging fees.
- Concurrency: synchronize entry/exit (multiple gates), or use per-floor locks.

### Drill (2 min)
How would you support **reservations** (pre-book a slot for a future time)? *(Hint: add a `RESERVED` slot status with a TTL. A scheduled job releases it back to `FREE` on no-show. Track reservations in a separate map keyed by `(slot, timeWindow)`.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Heaps / Priority Queue (Kth Largest Element)

### The problem
Given an unsorted array, find the **kth largest** element. Don't sort the whole array (`O(n log n)`) — there's a better way.

Input: `nums = [3, 2, 1, 5, 6, 4]`, `k = 2`
Output: `5` (the 2nd largest)

### The idea
Keep a **min-heap of size k** as you walk through the array. A heap (priority queue) gives you the smallest element at the top in O(log k) time per insert/remove.

For each number `n`:
1. Push it onto the heap.
2. If the heap now has more than k elements, pop the smallest.

After processing every number, the heap holds the k largest values you've seen, with the smallest of *those* on top. That smallest of the top-k is exactly the kth largest overall.

**Why min-heap, not max-heap?** We want to easily evict the smallest of our "current top k" whenever a bigger one shows up. A min-heap puts the smallest at the top — `O(1)` to peek, `O(log k)` to evict.

### Smallest working example
```java
import java.util.PriorityQueue;

public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(k);
    for (int n : nums) {
        minHeap.offer(n);
        if (minHeap.size() > k) minHeap.poll();   // evict smallest
    }
    return minHeap.peek();                         // kth largest is on top
}
```

**Time:** O(n log k). **Space:** O(k). Better than sorting when k is much smaller than n.

### Drill (2 min)
Why a min-heap of size k for the kth *largest*, not a max-heap? *(Hint: a max-heap puts the biggest on top — but you want to evict the *smallest* of "the top k so far" when a bigger value arrives. Min-heap of size k gives you O(1) peek of that smallest. Max-heap would force you to scan to find it.)*

**Deep dive (later):** [dsa/ds-java/heaps-priority-queues.md](../dsa/ds-java/heaps-priority-queues.md) · [dsa/ds-js/heaps-priority-queues.md](../dsa/ds-js/heaps-priority-queues.md)

---

## 9. Design Pattern — Decorator

### Intent
**Add behavior to an object dynamically** by wrapping it in another object that has the same interface. Unlike inheritance, you can stack decorators at runtime.

### When you'd use it
- You want optional features (logging, caching, retries) that you can mix and match.
- You'd otherwise need a combinatorial explosion of subclasses.

### Smallest working example
```java
interface Coffee {
    String description();
    double cost();
}

class SimpleCoffee implements Coffee {
    public String description() { return "Coffee"; }
    public double cost()        { return 3.00; }
}

abstract class CoffeeDecorator implements Coffee {
    protected final Coffee wrapped;
    CoffeeDecorator(Coffee c) { this.wrapped = c; }
}

class Milk extends CoffeeDecorator {
    Milk(Coffee c) { super(c); }
    public String description() { return wrapped.description() + ", milk"; }
    public double cost()        { return wrapped.cost() + 0.50; }
}

class Sugar extends CoffeeDecorator {
    Sugar(Coffee c) { super(c); }
    public String description() { return wrapped.description() + ", sugar"; }
    public double cost()        { return wrapped.cost() + 0.25; }
}

// Usage — stack decorators
Coffee order = new Sugar(new Milk(new SimpleCoffee()));
// description: "Coffee, milk, sugar"  cost: 3.75
```

Each decorator implements the same `Coffee` interface, holds a wrapped `Coffee`, and adds its bit before/after delegating.

You've seen this everywhere: `BufferedInputStream` wraps a `FileInputStream`; Express middleware decorates the request pipeline; Spring's `@Transactional` is decoration via a proxy.

### Drill (1 min)
Name three places in Java's standard library that use the Decorator pattern. *(Hint: I/O streams (`BufferedInputStream`, `DataInputStream`, etc.), `Collections.unmodifiableList(list)`, `Collections.synchronizedMap(map)`.)*

**Deep dive (later):** [design-patterns/common/structural/decorator.md](../design-patterns/common/structural/decorator.md)

---

## 10. DevOps — Kubernetes Fundamentals

### Why this exists
Kubernetes (K8s) is how teams run containerized apps in production. It schedules containers, restarts failed ones, scales them up and down, and gives them stable network addresses. You need a mental model of the main objects.

### The vocabulary
- **Cluster** — the whole installation (control plane + worker nodes).
- **Node** — a worker machine (VM or physical) that runs your containers.
- **Pod** — the smallest unit. One or more containers that share network and storage. You rarely create pods directly.
- **Deployment** — declares "I want 3 replicas of this image, rolling-update strategy." K8s creates/replaces Pods to match.
- **Service** — gives a stable name and virtual IP to a set of Pods (which come and go). Without it, you'd be chasing IPs.
- **ConfigMap** — non-sensitive config (key/value).
- **Secret** — sensitive data, base64-encoded (not encrypted by default — pair with KMS).
- **Namespace** — logical grouping for isolation (RBAC, quotas).

The core pattern: write YAML, `kubectl apply -f`, K8s makes reality match the spec.

### Smallest working example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: app
          image: my-registry/api:1.4.2
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector: { app: api }            # find pods with this label
  ports: [{ port: 80, targetPort: 8080 }]
```

The Deployment keeps 3 pods running. The Service load-balances across them. If a pod dies, the Deployment spawns a fresh one; the Service keeps pointing to the live ones.

### Drill (2 min)
You run `kubectl delete pod my-api-abc123`. What happens? *(Hint: the Pod dies, but the Deployment notices it's now short one replica and starts a fresh Pod within seconds. That's the standard way to "restart" a pod — also try `kubectl rollout restart deployment/my-api` to roll the whole set.)*

**Deep dive (later):** [devops/06-kubernetes-fundamentals.md](../devops/06-kubernetes-fundamentals.md)

---

## End-of-day checklist

- [ ] Angular: I can explain why a lazy module's provider shadows root's
- [ ] Node.js: I know the four Promise combinators and when each is right
- [ ] Spring: I wrote one `@RestControllerAdvice` returning `ProblemDetail`
- [ ] MongoDB: I can give three rules of thumb for embed vs reference
- [ ] Postgres: I wrote one `BEGIN ... COMMIT` block
- [ ] HLD: I can sketch the request path through a CDN to S3
- [ ] LLD: I can name the entities of a parking lot system
- [ ] DSA: I solved Kth Largest with a min-heap of size k
- [ ] DP: I can name three JDK uses of Decorator
- [ ] DevOps: I can name 5 Kubernetes object kinds and what each is for

**If you remember just one thing today:** every layer — DI providers, exception advices, decorators, Kubernetes objects — is composition over inheritance. You describe the pieces you want, and the framework wires them up at runtime.

**Tomorrow:** you've covered the breadth. The next phase is depth — pick the topics you found weakest and go to the deep-dive files linked at the bottom of each section.
