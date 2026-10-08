# Day 6 — Reactive forms, REST, EXPLAIN, and concurrency

> **Today's goal:** see how Angular handles complex forms, how Node's EventEmitter works, how Spring builds REST endpoints, and how to read Postgres EXPLAIN output without panicking.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Forms: Reactive (FormBuilder, validators) | 15m |
| 2 | Node.js | EventEmitter (custom events) | 15m |
| 3 | Spring Boot | REST Controllers (@RestController) | 15m |
| 4 | MongoDB | Schema Design Patterns (embed vs reference) | 10m |
| 5 | Postgres | Query Planner & EXPLAIN ANALYZE | 10m |
| 6 | HLD | Load Balancing & Proxies (L4 vs L7) | 12m |
| 7 | LLD | Concurrency & Threading | 12m |
| 8 | DSA | BST (Validate a BST) | 15m |
| 9 | Design Pattern | Adapter | 8m |
| 10 | DevOps | Networking Fundamentals (DNS, TCP, HTTP, TLS) | 10m |

---

## 1. Angular — Forms: Reactive

### Why this exists
Yesterday's template-driven forms are quick but awkward as forms grow. **Reactive forms** put the source of truth in TypeScript — the form is a class field. That makes complex forms (dynamic fields, conditional validation, multi-step) far cleaner and unit-testable.

### The idea in plain English
A reactive form is a tree of `FormControl` (single input), `FormGroup` (object of controls), and `FormArray` (list of controls). You build the tree in your component class and bind the template to it.

You get for free: validation rules in code, type safety, real-time `valueChanges` observables, and the ability to add/remove fields at runtime.

### Smallest working example
```typescript
import { Component } from '@angular/core';
import { FormBuilder, Validators, ReactiveFormsModule } from '@angular/forms';

@Component({
    standalone: true,
    imports: [ReactiveFormsModule],
    template: `
      <form [formGroup]="form" (ngSubmit)="submit()">
        <input formControlName="email" placeholder="email">
        <input formControlName="password" type="password" placeholder="password">
        <button type="submit" [disabled]="form.invalid">Sign up</button>
      </form>
    `,
})
export class SignupComponent {
    form = this.fb.group({
        email:    ['', [Validators.required, Validators.email]],
        password: ['', [Validators.required, Validators.minLength(8)]],
    });

    constructor(private fb: FormBuilder) {}

    submit() {
        if (this.form.valid) console.log(this.form.value);
    }
}
```

`FormBuilder` is just shorthand for creating `FormGroup` / `FormControl` instances. Each control's second array argument is its validator list. `form.value` gives you a plain object of the current values.

### Drill (2 min)
What's the difference between `form.value` and `form.getRawValue()`? *(Hint: `form.value` skips disabled controls. `getRawValue()` includes them. If you `disable()` a field and need it in the submit payload, use `getRawValue()`.)*

**Deep dive (later):** [angular/forms-reactive.md](../angular/forms-reactive.md)

---

## 2. Node.js — EventEmitter

### Why this exists
Node has a built-in publisher/subscriber primitive: **EventEmitter**. Streams, HTTP servers, and child processes are all EventEmitters under the hood. Writing your own is sometimes the right way to decouple parts of your app.

### The idea in plain English
An EventEmitter is a tiny in-process message bus. You attach **listeners** to **named events**, and `emit` fires them.

Two key methods:
- `on(event, fn)` — listener stays until removed.
- `once(event, fn)` — fires once, then auto-removes.

Listeners run **synchronously** in registration order. If a listener throws, later listeners don't run. EventEmitter is in-process only — for cross-service messaging, use Redis pub/sub or a real broker.

A common bug: adding listeners inside a per-request handler and never removing them. Memory grows; you'll see `MaxListenersExceededWarning`.

### Smallest working example
```javascript
import { EventEmitter } from 'node:events';

const bus = new EventEmitter();

bus.on('user:created', (user) => console.log('Welcome,', user.name));
bus.on('user:created', (user) => sendWelcomeEmail(user));

bus.emit('user:created', { id: 1, name: 'Kushal' });
// Logs "Welcome, Kushal" and triggers email
```

Multiple listeners can subscribe to the same event. Order is registration order.

For errors, the special `'error'` event must always have a listener — if it fires with no listeners, Node crashes the whole process.

### Drill (2 min)
Inside an Express handler, you write `db.on('reconnect', refreshCache)`. Why is this a memory leak? *(Hint: every request adds another listener to the long-lived `db` object. They're never removed. Memory grows, the listener cap warns, and on each reconnect `refreshCache` runs N times. Attach listeners once at module init.)*

**Deep dive (later):** [nodejs/09-event-emitter.md](../nodejs/09-event-emitter.md)

---

## 3. Spring Boot — REST Controllers

### Why this exists
A web API in Spring is a class annotated `@RestController`. Each method maps to an HTTP route. Spring handles JSON serialization, URL parameters, and request bodies — so you focus on the business logic.

### The idea in plain English
`@RestController` is `@Controller` plus "return values become the response body" (no view rendering). You map each method to a verb + path with `@GetMapping`, `@PostMapping`, etc.

Parameters come from different parts of the request:
- `@PathVariable` — from the URL (`/users/{id}` → `id`).
- `@RequestParam` — from the query string (`?page=0`).
- `@RequestBody` — from the JSON body (deserialized to a class).

A small but important rule: **return DTOs, not entities**. Returning a JPA entity from a controller leaks DB structure and can break in surprising ways. Use a `record` for the response shape.

### Smallest working example
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public UserDto get(@PathVariable Long id) {
        return userService.findById(id)
            .orElseThrow(() -> new NotFoundException("user " + id));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDto create(@RequestBody CreateUserRequest req) {
        return userService.create(req);
    }
}

public record UserDto(Long id, String email, String name) {}
public record CreateUserRequest(String email, String name) {}
```

The `@ResponseStatus(HttpStatus.CREATED)` on `create` makes the response a `201` instead of the default `200`.

### Drill (2 min)
Why use a `UserDto` record instead of returning the JPA `User` entity directly? *(Hint: the entity is tied to the DB. Returning it can serialize lazy-loaded relations (or fail trying), leak internal fields, and couple your API to schema changes. A DTO is a stable, intentional contract.)*

**Deep dive (later):** [spring-detailed/PART-15-spring-mvc-request-flow.md](../spring-detailed/PART-15-spring-mvc-request-flow.md)

---

## 4. MongoDB — Schema Design Patterns

### Why this exists
Mongo lets you store data many ways — embedded inside parents, referenced by ID, sharded across collections. The Mongo community has named a few common shapes so you can talk about them: **embed**, **reference**, **subset**, **bucket**, **polymorphic**.

### The idea in plain English
The two pillars:

- **Embed** — put related data *inside* the parent document. Best when data is read together, child is bounded, child has no independent life.
- **Reference** — store an ID; load the other collection separately. Best when child is large, unbounded, or queried on its own.

A few named patterns built on top:

- **Subset pattern** — embed only the *frequently-accessed* slice (e.g., a user's last 5 orders), reference the rest.
- **Bucket pattern** — for time-series, group many small docs into one bucket doc (e.g., 60 sensor readings into one "minute" doc) to reduce overhead.
- **Polymorphic pattern** — different "subtypes" share one collection with a `type` discriminator field.

### Smallest working example
```javascript
// EMBED — line items live and die with the order
{
  _id: "ORD-1",
  customer: "Kushal",
  items: [
    { sku: "A1", qty: 2, price: 99 },
    { sku: "B7", qty: 1, price: 49 }
  ],
  total: 247
}

// REFERENCE — events are unbounded, queried independently
// users collection
{ _id: "u1", email: "k@x.com" }
// events collection
{ _id: "e1", userId: "u1", type: "login", ts: ISODate("...") }
```

The rule of thumb: if you'd never read the parent without the child, embed. If the child is queried alone or grows unboundedly, reference.

### Drill (2 min)
You're storing a user's permanent address (one per user, always read with the user). Embed or reference? *(Hint: embed. Single, bounded, always read together — no reason for a separate collection.)*

**Deep dive (later):** [mongodb/06-schema-design.md](../mongodb/06-schema-design.md)

---

## 5. Postgres — Query Planner & EXPLAIN ANALYZE

### Why this exists
Postgres decides how to execute every query — index scan, sequential scan, hash join, merge join. Sometimes it picks badly. **EXPLAIN ANALYZE** shows you the plan it chose *and* how long each step actually took. It's the #1 tool for debugging slow queries.

### The idea in plain English
`EXPLAIN` shows the planned tree without running the query (estimates only). `EXPLAIN ANALYZE` runs the query and adds real timings.

Read the plan **from the inside out** — the innermost node runs first, the outermost last.

Two things you almost always want to see:
- `Index Scan` (good) vs `Seq Scan` (bad on large tables with selective predicates).
- Estimated rows vs actual rows — if they're wildly off, the planner is using stale statistics. Run `ANALYZE your_table;`.

### Smallest working example
```sql
EXPLAIN ANALYZE
SELECT name FROM users WHERE email = 'k@x.com';
```

Output (simplified):
```
Index Scan using users_email_key on users  (cost=0.29..8.31 rows=1 width=32)
                                           (actual time=0.02..0.02 rows=1 loops=1)
  Index Cond: (email = 'k@x.com'::text)
Planning Time: 0.10 ms
Execution Time: 0.04 ms
```

`Index Scan using users_email_key` — good, used the email index. `rows=1` estimated, `rows=1` actual — stats are accurate. `Execution Time: 0.04 ms` — fast.

Compare with a missing index:
```
Seq Scan on users  (cost=0.00..1850.00 rows=1 width=32) (actual time=15.2..15.2 rows=1 loops=1)
  Filter: (email = 'k@x.com'::text)
  Rows Removed by Filter: 99999
Execution Time: 15.2 ms
```

`Seq Scan` reading every row — 99,999 thrown away to find one. Add an index.

### Drill (2 min)
Your query was fast yesterday, slow today, same data. EXPLAIN now shows a Seq Scan instead of Index Scan. What might have changed? *(Hint: planner statistics are stale (run `ANALYZE`), the index was dropped, or data distribution shifted so the planner thinks the index isn't selective enough. Start with `ANALYZE your_table;`.)*

**Deep dive (later):** [postgres/07-query-planner-and-explain.md](../postgres/07-query-planner-and-explain.md)

---

## 6. High-Level Design — Load Balancing & Proxies (L4 vs L7)

### Why this exists
One server can only handle so much traffic. A **load balancer** sits in front of many servers and distributes requests. It also gives you health checking, failover, and a single stable address — clients don't care that backends come and go.

### The idea in plain English
Two layers of load balancing:

- **L4 (transport layer)** — looks only at TCP/IP. Routes packets by IP and port. Fast, simple, but can't read HTTP headers.
- **L7 (application layer)** — understands HTTP. Can route by URL path, hostname, or header. Slower, more CPU, but vastly more flexible.

**Examples:** AWS NLB is L4. AWS ALB, NGINX, Envoy are L7.

Routing algorithms:
- **Round-robin** — next server in line.
- **Least connections** — send to the server with the fewest active connections.
- **Hash-based** — hash a request key (user ID) to a specific backend. Useful for cache stickiness.

**Sticky sessions** pin a client to one backend (by cookie, IP, or token). Avoid when possible — stateless backends scale better.

### A picture
```
Client ──HTTPS──> L7 LB (NGINX / Envoy)
                    │  terminate TLS, inspect URL path
                    ├──> /api/*       → app pool (8 servers)
                    └──> /static/*    → CDN
```

### Drill (2 min)
You need to route `/api/v1/*` to one set of servers and `/api/v2/*` to another. Can an L4 load balancer do that? *(Hint: no. L4 only sees IP + port; it can't read the HTTP path. You need an L7 load balancer that can inspect the URL.)*

**Deep dive (later):** [system-design/high-level-design/06-load-balancing-and-proxies.md](../system-design/high-level-design/06-load-balancing-and-proxies.md)

---

## 7. Low-Level Design — Concurrency & Threading

### Why this exists
Modern CPUs have many cores. To use them, your code has to do multiple things at once — **concurrently**. That introduces a new class of bugs (race conditions, deadlocks) that single-threaded code doesn't have.

### The idea in plain English
Two ideas to keep separate:

- **Concurrency** — multiple tasks making progress in overlapping time. A *design* concept.
- **Parallelism** — multiple tasks actually running at the same instant. A *hardware* concept.

A web server is concurrent (many requests in flight) even on a single core. Add cores and it becomes parallel.

The hard part is **shared mutable state**. Two threads both doing `counter++` can lose increments because `++` is read → add → write, not atomic. Fixes:
- Use atomic types (`AtomicInteger` in Java) — uses CPU-level compare-and-swap.
- Use locks (`synchronized`, `ReentrantLock`) — only one thread inside at a time.
- Avoid shared state entirely — pass messages between threads (Go channels, Node Worker Threads).

### Smallest working example
```java
// BAD — race condition. Two threads can lose increments.
class Counter {
    private int count = 0;
    public void inc() { count++; }            // read, add, write — not atomic
}

// GOOD — atomic
class CounterSafe {
    private final AtomicInteger count = new AtomicInteger();
    public void inc() { count.incrementAndGet(); }
    public int get()  { return count.get(); }
}
```

If two threads call `inc()` a million times each on the BAD version, the final count is somewhere between 1,000,001 and 2,000,000 — non-deterministic. The atomic version always ends at exactly 2,000,000.

**Node note:** Node's main JS runs single-threaded (no shared-memory races in your own code). True parallelism uses `worker_threads` or child processes, which communicate by message-passing.

### Drill (2 min)
Two threads each do `counter++` one million times on a shared `int counter = 0`. What's the final value? *(Hint: somewhere between 1,000,001 and 2,000,000 — non-deterministic. `++` is read-modify-write; both threads can read the same value, both increment, both write — losing one increment. Use `AtomicInteger`.)*

**Deep dive (later):** [system-design/low-level-design/07-concurrency-and-threading.md](../system-design/low-level-design/07-concurrency-and-threading.md)

---

## 8. DSA — BST (Validate a BST)

### The problem
Given a binary tree, decide if it's a **valid BST** (binary search tree): every node's value is strictly greater than all values in its left subtree, and strictly less than all values in its right subtree.

### The naive (wrong) approach
At each node, check `node.left.val < node.val && node.right.val > node.val`. This **fails** because it only compares immediate children. A wrong value deeper down can slip through:

```
        10
       /  \
      5    15
          /  \
         6    20   ← 6 < 10 violates BST, but local check at 15 says 6 < 15 ✓
```

### The correct approach (bounds)
At each node, pass down **valid bounds**. The left child must be less than the current node *and* less than every ancestor that turned right. The right child must be greater than the current node *and* greater than every ancestor that turned left.

### Smallest working example
```javascript
function isValidBST(root, min = null, max = null) {
    if (root === null) return true;
    if (min !== null && root.val <= min) return false;
    if (max !== null && root.val >= max) return false;
    return isValidBST(root.left,  min, root.val) &&
           isValidBST(root.right, root.val, max);
}
```

For the broken tree above, when we recurse into node `6`, we pass `max = 10` (from the root, since we turned right then left). `6 < 10` is fine — but wait, we also need `min = 10`. We turned right at the root, so anything to the right must be > 10. `6 < 10` violates `min = 10` → return false.

**Time:** O(n). **Space:** O(h) for recursion.

### Drill (2 min)
Why pass `null` for "no bound yet" instead of `Number.MIN_SAFE_INTEGER`? *(Hint: if a node's value happened to equal the sentinel, you'd falsely reject a valid tree. Using `null` says "no bound at this side" unambiguously.)*

**Deep dive (later):** [dsa/ds-js/trees-graphs.md](../dsa/ds-js/trees-graphs.md) · [dsa/ds-java/trees-graphs.md](../dsa/ds-java/trees-graphs.md)

---

## 9. Design Pattern — Adapter

### Intent
**Make two incompatible interfaces work together** by wrapping one in a class that exposes the shape the other expects.

### When you'd use it
- Integrating a third-party library whose API you can't change.
- Bridging an old API to a new one during a migration.
- You depend on an interface; the external service speaks a different one.

### Smallest working example
```java
// What our code expects
public interface PaymentProcessor {
    PaymentResult charge(int cents, String customerId);
}

// Third-party SDK with a different shape
public class StripeSdk {
    public StripeResponse createCharge(int amountInCents, String currency,
                                       String customer, String description) {
        // proprietary call
    }
}

// Adapter — translates between the two
public class StripePaymentAdapter implements PaymentProcessor {
    private final StripeSdk sdk;
    public StripePaymentAdapter(StripeSdk sdk) { this.sdk = sdk; }

    public PaymentResult charge(int cents, String customerId) {
        StripeResponse r = sdk.createCharge(cents, "INR", customerId, "Order charge");
        return new PaymentResult(r.id(), r.success());
    }
}
```

Now your `OrderService` depends on `PaymentProcessor`. Swap Stripe for PayPal by writing another adapter — zero changes to business code.

You've already used this without naming it: `Arrays.asList(arr)` adapts a Java array to a `List`; `InputStreamReader` adapts a byte stream to a character stream.

### Drill (1 min)
What's the difference between Adapter and Decorator? *(Hint: Adapter changes the *interface* (different shape, same behavior). Decorator keeps the *same interface* and adds behavior. Adapter is "make B look like A"; Decorator is "wrap A and add a feature.")*

**Deep dive (later):** [design-patterns/common/structural/adapter.md](../design-patterns/common/structural/adapter.md)

---

## 10. DevOps — Networking Fundamentals

### Why this exists
Every distributed system runs on the same handful of network primitives. When something breaks in production, the root cause is often DNS, TLS, or TCP — be able to name the moving parts.

### A quick tour
- **DNS** — turns names like `example.com` into IPs. Records: `A` (IPv4), `AAAA` (IPv6), `CNAME` (alias to another name), `MX` (mail), `TXT` (free-form, often used for SPF and domain verification). Resolutions are cached by **TTL**.

- **TCP** — reliable byte stream. Setup costs one round-trip (the **3-way handshake**: SYN, SYN-ACK, ACK). That's why reusing connections (keep-alive) matters for perf.

- **HTTP** — your request/response protocol, plain text over TCP. `HTTPS` = HTTP over **TLS** (encryption). TLS adds another 1-2 RTT for the handshake.

- **CIDR** — IP block notation. `10.0.0.0/16` means "first 16 bits are the network, the rest are the host" — 65,536 addresses.

### A few commands
```bash
dig +short example.com A           # look up the A record
dig example.com MX                 # look up mail servers

curl -v https://example.com        # show full handshake + headers
# *   Trying 93.184.216.34:443...
# * TLSv1.3 (OUT), Client hello (1):
# > GET / HTTP/2
# < HTTP/2 200
```

### Drill (2 min)
What's the difference between an `A` record and a `CNAME`? *(Hint: an `A` record maps a name directly to an IPv4 address — one lookup. A `CNAME` is an alias pointing to *another name*, which then has to be resolved — one extra lookup. CNAMEs can't be at the zone apex (`example.com` itself) — that's why `ALIAS`/`ANAME` records exist.)*

**Deep dive (later):** [devops/22-networking-fundamentals.md](../devops/22-networking-fundamentals.md)

---

## End-of-day checklist

- [ ] Angular: I built a reactive form with `FormBuilder`
- [ ] Node.js: I know when EventEmitter listeners leak
- [ ] Spring: I wrote one `@GetMapping` returning a DTO record
- [ ] MongoDB: I can pick embed vs reference for one scenario
- [ ] Postgres: I can read `Index Scan` vs `Seq Scan` in EXPLAIN
- [ ] HLD: I can describe the difference between L4 and L7 load balancers
- [ ] LLD: I can explain why `counter++` is unsafe across threads
- [ ] DSA: I validated a BST using min/max bounds
- [ ] DP: I know when to use Adapter vs Decorator
- [ ] DevOps: I can explain what TLS adds on top of TCP

**If you remember just one thing today:** every layer of a real system — a form, a REST handler, an EXPLAIN plan, a load balancer — exposes its choices explicitly so you can reason about them without running the whole thing.

**Tomorrow:** Angular DI hierarchy, Promises and async/await, exception handling, embed vs reference, transactions, storage and CDN, classic LLD (parking lot), heaps, Decorator pattern, and Kubernetes fundamentals.
