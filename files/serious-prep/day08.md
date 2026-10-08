# Day 8 — Routes, Types, Validation, and Isolation

> **Today's goal:** see how URLs find their handlers, how we catch bad input before it hits the database, and how databases keep multiple users from stepping on each other.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Routing (route config, params) | 15m |
| 2 | Node.js | TypeScript for Node (interfaces, generics basics) | 15m |
| 3 | Spring Boot | Bean Validation (@Valid, @NotBlank) | 15m |
| 4 | MongoDB | Transactions (multi-document ACID) | 10m |
| 5 | Postgres | Isolation Levels | 10m |
| 6 | HLD | Microservices Architecture | 12m |
| 7 | LLD | Elevator system | 12m |
| 8 | DSA | Graphs — BFS shortest path on a grid | 15m |
| 9 | Design Pattern | Facade | 8m |
| 10 | DevOps | K8s Networking (Services & Ingress) | 10m |

---

## 1. Angular — Routing (route config, params)

### Why this exists
A single-page app (SPA) doesn't reload the page when you click a link — yet `/users/42` should still show a different screen than `/dashboard`. The Angular Router watches the URL and swaps components in and out without a full page reload.

### The idea in plain English
Think of the router as a switchboard operator. You hand it a list of URL patterns and the component to show for each. When the URL changes, it looks up the match and renders that component into a placeholder called `<router-outlet>`. Dynamic pieces of the URL like `42` in `/users/42` are called **route params** — you read them with `ActivatedRoute`.

Three terms to know:
- **Route** — one entry: `{ path, component }`.
- **Param** — variable piece of the path, prefixed with `:` (e.g., `:id`).
- **Outlet** — `<router-outlet>` in your HTML; the router draws here.

### Smallest working example
```typescript
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: 'home', pathMatch: 'full' },
  { path: 'home', component: HomeComponent },
  { path: 'users/:id', component: UserComponent },   // :id is a param
];

// user.component.ts — read the param
export class UserComponent {
  private route = inject(ActivatedRoute);
  id = this.route.snapshot.paramMap.get('id');       // "42" if URL is /users/42
}
```

`pathMatch: 'full'` on the empty redirect prevents accidental matches on every URL.

### Drill (2 min)
The URL is `/users/7`. What value does `paramMap.get('id')` return? *(Hint: the string `"7"` — route params are always strings; convert with `Number(...)` if you need an int.)*

**Deep dive (later):** [angular/routing.md](../angular/routing.md)

---

## 2. Node.js — TypeScript for Node (interfaces, generics basics)

### Why this exists
JavaScript only checks types at runtime, so a typo like `user.naem` becomes a `TypeError` your customer sees. TypeScript adds a **compile-time** type checker on top of JS — it catches the typo before the code ever runs.

### The idea in plain English
TypeScript is JavaScript with sticky notes. You annotate variables and functions with what kind of value they hold (`string`, `number`, an object shape) and the compiler enforces it. Two ideas you'll see daily:

- **Interface** — describes the shape of an object. Like a form everyone must fill in.
- **Generic** — a placeholder type that gets filled in later. `Array<string>` means "array of strings"; `<T>` lets you write a function that works for any type without losing type safety.

Analogy: an interface is the cookie cutter, a generic is "give me a function that bakes whatever dough you hand it."

### Smallest working example
```typescript
interface User {
  id: number;
  name: string;
  email?: string;            // ? = optional
}

function firstOrNull<T>(items: T[]): T | null {
  return items.length > 0 ? items[0] : null;
}

const users: User[] = [{ id: 1, name: 'Jane' }];
const first = firstOrNull(users);   // TypeScript knows: User | null
console.log(first?.name);           // safe — uses optional chaining
```

`<T>` is a type variable — when you call `firstOrNull(users)`, `T` becomes `User` automatically.

### Drill (2 min)
What does `firstOrNull<number>([1, 2, 3])` return as a type? *(Hint: `number | null` — generics flow through the function's return type.)*

**Deep dive (later):** [nodejs/04-typescript-for-node.md](../nodejs/04-typescript-for-node.md)

---

## 3. Spring Boot — Bean Validation (@Valid, @NotBlank)

### Why this exists
Bad input is the #1 cause of bugs. If a `POST /users` lets `email` be empty, you'll find out at 3 AM when a NullPointerException blows up your service. Bean Validation lets you declare rules **on the data class itself** so Spring rejects invalid requests before your controller code runs.

### The idea in plain English
You decorate fields with annotations like `@NotBlank`, `@Email`, `@Min(0)`. Then you tag the controller parameter with `@Valid`. When a request comes in, Spring deserializes the JSON into your object **and then validates it**. If anything fails, Spring sends back a 400 Bad Request automatically — you write zero validation code.

Think of `@Valid` as the bouncer at a club: it checks each guest's ID at the door so the inside never sees bad guests.

### Smallest working example
```java
public class CreateUserDto {
    @NotBlank(message = "name is required")
    private String name;

    @Email
    @NotBlank
    private String email;

    // getters/setters
}

@RestController
@RequestMapping("/users")
public class UserController {

    @PostMapping
    public User create(@Valid @RequestBody CreateUserDto dto) {
        // If we got here, dto is already validated.
        return userService.save(dto);
    }
}
```

Without `@Valid`, the annotations on `CreateUserDto` are just decoration — they don't fire. With it, Spring runs every annotation before your method body.

### Drill (2 min)
A client sends `{ "name": "", "email": "x" }`. What status code comes back? *(Hint: 400 Bad Request — `@NotBlank` rejects empty name, `@Email` rejects malformed email, and you never enter the method.)*

**Deep dive (later):** [spring-next/04-spring-boot-rest-api.md § 6. Validation](../spring-next/04-spring-boot-rest-api.md#6-validation)

---

## 4. MongoDB — Transactions (multi-document ACID)

### Why this exists
A bank transfer is two writes: subtract from A, add to B. If the server crashes after step 1, money disappears. Transactions let you say "both writes happen, or neither does." Mongo originally avoided them (single-document writes were always atomic), but added multi-document transactions in v4.0 for cases that span documents.

### The idea in plain English
A transaction wraps several operations in a **session**. You either `commitTransaction()` (apply everything) or `abortTransaction()` (throw it all away). While the transaction is open, other readers don't see your half-finished changes.

ACID stands for: **A**tomic (all or nothing), **C**onsistent (rules respected), **I**solated (others don't see in-progress), **D**urable (survives a crash). Mongo transactions give you all four — but they're slower than single writes, so use them only when you truly need multi-document atomicity.

### Smallest working example
```javascript
const session = client.startSession();
try {
  session.startTransaction();

  await accounts.updateOne({ _id: 'A' }, { $inc: { balance: -100 } }, { session });
  await accounts.updateOne({ _id: 'B' }, { $inc: { balance:  100 } }, { session });

  await session.commitTransaction();         // both writes go live together
} catch (err) {
  await session.abortTransaction();          // undo everything
} finally {
  await session.endSession();
}
```

Notice every operation passes `{ session }`. That's how Mongo knows these belong together.

### Drill (2 min)
The server crashes between the two `updateOne` calls. What happens to A's balance? *(Hint: nothing changes. The transaction wasn't committed, so when the server restarts, neither write is visible.)*

**Deep dive (later):** [mongodb/08-transactions.md](../mongodb/08-transactions.md)

---

## 5. Postgres — Isolation Levels

### Why this exists
Two users buying the last concert ticket at the same time can both think it's available. Isolation levels are the dial that controls how strictly the database hides concurrent changes from you — stricter = safer, but slower.

### The idea in plain English
Picture three people editing the same shared Google Doc:
- **Read Committed** (default) — you see whatever was saved at the moment you read each line. Other people's saves appear as you scroll.
- **Repeatable Read** — you get a frozen snapshot of the doc when you opened it. Re-reading the same line gives the same answer for your whole session.
- **Serializable** — Postgres pretends transactions ran one at a time, even if they ran in parallel. If two would conflict, one is aborted with a *serialization failure* and your app retries.

The trade: stricter levels prevent more anomalies but cause more retries / blocking.

### Smallest working example
```sql
-- Each connection picks its level once per transaction
BEGIN ISOLATION LEVEL SERIALIZABLE;

SELECT balance FROM accounts WHERE id = 'A';   -- read
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';

COMMIT;        -- may raise "could not serialize access" — app retries
```

If `COMMIT` raises a serialization failure (SQLSTATE 40001), the app simply runs the same transaction again. Most apps wrap this in a retry loop.

### Drill (2 min)
You read a row inside a transaction, then read it again 5 seconds later. Under **Repeatable Read**, do you see another user's update? *(Hint: no — your snapshot is frozen at the transaction's start. Under Read Committed, you would.)*

**Deep dive (later):** [postgres/09-isolation-levels.md](../postgres/09-isolation-levels.md)

---

## 6. HLD — Microservices Architecture

### Why this exists
A monolith is one big codebase deployed as one process. Easy to build, but as a company grows, the codebase becomes too large to coordinate, deploys get scary, and one bad change can break everything. Microservices split the app into small, independently deployable services that talk over the network.

### The idea in plain English
A monolith is a single restaurant: one menu, one kitchen, one chef. Microservices are a food court: one shop for pizza, one for sushi, one for coffee. Each shop owns its kitchen (database) and can update its menu without closing the other shops.

The trade:
- **Wins:** teams ship independently, scale services separately, use different languages/DBs.
- **Costs:** the network is now part of your app — calls fail, services time out, data is eventually consistent. You need service discovery, distributed tracing, and a smarter deployment story.

Rule of thumb: **start with a monolith**. Split into services only when team size or scaling needs demand it.

### Smallest working example (sketch)
```
Monolith:                  Microservices:
[ web + auth + orders +    [web] → [auth-svc] → [users-db]
  inventory + email ]            ↘ [orders-svc] → [orders-db]
        ↓                         ↘ [email-svc]  → (SMTP)
   [one db]
```

Each box is its own deploy, its own DB, its own team.

### Drill (2 min)
You move "send confirmation email" out of the monolith into its own service. The email service is down. Should the order still be placed? *(Hint: yes — decouple via a queue. The order service writes the order, drops a "send email" message into a queue, and returns success. The email service consumes from the queue when it's back up.)*

**Deep dive (later):** [system-design/high-level-design/08-microservices-and-architecture-patterns.md](../system-design/high-level-design/08-microservices-and-architecture-patterns.md)

---

## 7. LLD — Elevator system

### Why this exists
Elevators are a classic LLD interview because they exercise everything: state, requests in a queue, multiple "workers" (cars), and a smart algorithm to pick which car serves which request.

### The idea in plain English
Model the world as three actors:
- **Elevator** — one car. Has a current floor, a direction (UP/DOWN/IDLE), and a list of stops.
- **Request** — "I'm on floor 5 and want to go up" (external) or "Take me to floor 9" (internal).
- **Controller** — picks the best elevator for an external request. The classic rule: pick the nearest car already moving in the right direction; otherwise pick the nearest idle car.

This is SCAN/LOOK scheduling — the same idea disk drives use.

### Smallest working example (skeleton)
```java
enum Direction { UP, DOWN, IDLE }

class Elevator {
    int id, currentFloor = 0;
    Direction direction = Direction.IDLE;
    TreeSet<Integer> stops = new TreeSet<>();

    void addStop(int floor) { stops.add(floor); }

    void step() {
        if (stops.isEmpty()) { direction = Direction.IDLE; return; }
        int next = (direction == Direction.UP)
            ? stops.higher(currentFloor - 1)
            : stops.lower(currentFloor + 1);
        currentFloor += (next > currentFloor) ? 1 : -1;
        if (currentFloor == next) stops.remove(next);
    }
}
```

The `Controller` (not shown) iterates over elevators and assigns the cheapest one to each external request.

### Drill (2 min)
Two elevators are at floor 1 (idle) and floor 8 (going down). A request comes in at floor 5 going down. Which serves it? *(Hint: the one at floor 8 — it's already going down and will naturally pass floor 5. Sending the idle car would mean both cars travel.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Graphs: BFS shortest path on a grid

### The problem
Given a 2D grid where `0` is open and `1` is a wall, find the length of the shortest path from top-left to bottom-right, moving up/down/left/right one step at a time. Return `-1` if no path exists.

```
Input:  [[0,0,0],
         [1,1,0],
         [0,0,0]]
Output: 4   // (0,0) → (0,1) → (0,2) → (1,2) → (2,2)
```

### The idea
Think of the grid as a graph: each open cell is a node, neighbors are nodes you can move to. **BFS (Breadth-First Search)** explores level by level — first all cells 1 step away, then all cells 2 steps away, etc. The first time you visit the goal, you've found the shortest path because BFS guarantees the lowest depth wins.

Why not DFS? DFS goes deep down one path first — it might reach the goal via a long detour and call it "shortest" by mistake. BFS is the right tool whenever every edge has the same weight.

### Smallest working example
```javascript
function shortestPath(grid) {
  const R = grid.length, C = grid[0].length;
  if (grid[0][0] === 1 || grid[R-1][C-1] === 1) return -1;

  const queue = [[0, 0, 1]];                       // [row, col, distance]
  const seen  = new Set(['0,0']);
  const dirs  = [[1,0],[-1,0],[0,1],[0,-1]];

  while (queue.length) {
    const [r, c, d] = queue.shift();
    if (r === R-1 && c === C-1) return d;
    for (const [dr, dc] of dirs) {
      const nr = r+dr, nc = c+dc, key = `${nr},${nc}`;
      if (nr<0||nc<0||nr>=R||nc>=C||grid[nr][nc]===1||seen.has(key)) continue;
      seen.add(key);
      queue.push([nr, nc, d+1]);
    }
  }
  return -1;
}
```

**Time:** O(R × C) — every cell visited at most once. **Space:** O(R × C) for `seen`.

### Drill (3 min)
Trace the queue for the 3×3 example above for the first 3 steps. *(You should see: start with `[0,0,1]`; pop and push `[0,1,2]` and `[1,0,2]` — but `[1,0]` is a wall so only `[0,1,2]` is queued. Continue from there.)*

**Deep dive (later):** [dsa/ds-js/graphs.md](../dsa/ds-js/graphs.md) · [dsa/ds-java/graphs.md](../dsa/ds-java/graphs.md)

---

## 9. Design Pattern — Facade

### Intent
**Provide a simple front door to a complex subsystem.** Hide the messy details behind one clean API.

### When you'd use it
- Your code talks to 5 internal services to "place an order" — wrap it in one `OrderFacade.place()`.
- A 3rd-party library has 20 classes; expose only the 3 your team needs.
- You're migrating an old system — the facade is a stable surface while the inside changes.

### Smallest working example
```java
// Three messy subsystems
class PaymentSvc   { void charge(int amount) { /*...*/ } }
class InventorySvc { void reserve(String sku) { /*...*/ } }
class EmailSvc     { void confirm(String to)  { /*...*/ } }

// The facade: one method, three calls inside.
class OrderFacade {
    private final PaymentSvc   payment   = new PaymentSvc();
    private final InventorySvc inventory = new InventorySvc();
    private final EmailSvc     email     = new EmailSvc();

    public void placeOrder(String sku, int amount, String userEmail) {
        inventory.reserve(sku);
        payment.charge(amount);
        email.confirm(userEmail);
    }
}
```

The caller does `new OrderFacade().placeOrder(...)`. They don't need to know there are three services underneath.

### Drill (1 min)
What's the difference between a Facade and an Adapter? *(Hint: Facade simplifies a complex subsystem; Adapter changes an interface to match what the caller expects. Facade = friendlier front door; Adapter = plug converter.)*

**Deep dive (later):** [design-patterns/common/structural/facade.md](../design-patterns/common/structural/facade.md)

---

## 10. DevOps — K8s Networking (Services & Ingress)

### Why this exists
Pods in Kubernetes come and go — their IPs change every restart. You can't tell users "go to 10.42.0.7." You need a stable address that points to "whatever pods are currently healthy." That's a **Service**. An **Ingress** then exposes services to the outside world via HTTP routes.

### The idea in plain English
- **ClusterIP** (default) — a fixed virtual IP inside the cluster. Other pods reach your pods via this IP. Not reachable from outside.
- **NodePort** — opens the same port on every node's IP. Quick demo, ugly in production.
- **LoadBalancer** — asks the cloud (AWS/GCP/Azure) to give you a real external IP with a load balancer in front.
- **Ingress** — one shared LoadBalancer in front of *many* services, routing by hostname or URL path. Like an Nginx reverse proxy that Kubernetes manages for you.

### Smallest working example
```yaml
# Service: stable in-cluster address for app=web pods
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  selector: { app: web }
  ports: [{ port: 80, targetPort: 8080 }]
---
# Ingress: route api.example.com/* to the web service
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: web-ingress }
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: web, port: { number: 80 } } }
```

The Service finds pods via label selector; the Ingress sends external HTTP traffic to the Service.

### Drill (2 min)
You want a single public URL `https://api.example.com` that routes `/users` and `/orders` to two different services. ClusterIP + NodePort, or Ingress? *(Hint: Ingress. It does host- and path-based routing in front of multiple services with one external IP.)*

**Deep dive (later):** [devops/07-kubernetes-networking.md](../devops/07-kubernetes-networking.md)

---

## End-of-day checklist

- [ ] Angular: I can write a route with a `:param` and read it in the component
- [ ] Node.js: I can describe what an interface and a generic do in TypeScript
- [ ] Spring: I can explain why `@Valid` is needed (the field annotations alone don't fire)
- [ ] MongoDB: I know transactions need a `session` passed to every operation
- [ ] Postgres: I can name the three isolation levels in order of strictness
- [ ] HLD: I can give one concrete win and one cost of microservices
- [ ] LLD: I can sketch an Elevator class with floor, direction, and a stops set
- [ ] DSA: I solved grid BFS and explained why BFS (not DFS) gives the shortest path
- [ ] DP: I can describe a Facade in one sentence
- [ ] DevOps: I know when to use ClusterIP vs LoadBalancer vs Ingress

**If you remember just one thing today:** **defend the edges** — validate input at the controller, isolate writes at the database, and gate URLs at the router. Bugs that escape the edge are 10× harder to fix later.

**Tomorrow:** route guards, replica sets, MVCC, consensus algorithms, LRU caches, DFS islands — the next layer down on most of today's topics.
