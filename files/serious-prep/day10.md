# Day 10 — HTTP Clients, Middleware, Entities, and Real-World Designs

> **Today's goal:** see how Angular calls APIs in a typed way, how Express layers behavior through middleware, how Spring maps tables to classes, and how a real system like Twitter is sketched at a whiteboard.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | HttpClient (typed responses) | 15m |
| 2 | Node.js | Express Middleware (the chain) | 15m |
| 3 | Spring Boot | Entity Mapping (@Entity, @Id, @Column) | 15m |
| 4 | MongoDB | Sharding (shard keys, chunks) | 10m |
| 5 | Postgres | Locking (FOR UPDATE, deadlocks) | 10m |
| 6 | HLD | Twitter feed walkthrough | 12m |
| 7 | LLD | ATM machine | 12m |
| 8 | DSA | Recursion (Permutations) | 15m |
| 9 | Design Pattern | Composite | 8m |
| 10 | DevOps | Helm & Kustomize | 10m |

---

## 1. Angular — HttpClient (typed responses)

### Why this exists
You could call APIs with the browser's `fetch`, but you'd have to manage subscriptions, retries, and types by hand. Angular's `HttpClient` returns an `Observable`, integrates with interceptors, and lets you say "this endpoint returns a `User`" so the rest of your code gets autocomplete and compile-time safety.

### The idea in plain English
You inject `HttpClient` into a service. You call `http.get<T>(url)` and it returns `Observable<T>`. Nothing actually runs until someone subscribes. Components usually let Angular's `| async` pipe subscribe for them, so you never write subscribe/unsubscribe boilerplate.

Three things to register:
1. Call `provideHttpClient()` in your app config (one line).
2. Type your responses with a generic: `<User>` or `<User[]>`.
3. Let the template subscribe via `| async`.

### Smallest working example
```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [provideHttpClient()],
};

// user.service.ts
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  list(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');   // typed!
  }
}

// users.component.ts (template uses | async)
@Component({
  selector: 'app-users',
  template: `<li *ngFor="let u of users()">{{ u.name }}</li>`,
})
export class UsersComponent {
  users = toSignal(inject(UserService).list(), { initialValue: [] });
}
```

`toSignal` converts the Observable into a Signal — modern Angular's preferred way to read async data.

### Drill (2 min)
You call `userService.list()` in `ngOnInit` but assign it to a property without subscribing. No HTTP request fires. Why? *(Hint: Observables are lazy. They don't run until something subscribes — either `.subscribe(...)`, an `| async` pipe, or `toSignal`.)*

**Deep dive (later):** [angular/httpclient.md](../angular/httpclient.md)

---

## 2. Node.js — Express Middleware (the chain)

### Why this exists
Every web request needs the same chores: parse the body, log it, check auth, set CORS headers. You don't want to write that on every route. Middleware lets you stack those chores in a chain so each route gets them for free.

### The idea in plain English
**Middleware is a function `(req, res, next) => { ... }`.** Express runs them in the order you register them. Each middleware can:
- Inspect / modify `req` and `res`.
- Call `next()` to pass control to the next link.
- Call `next(err)` to jump to the error handler.
- End the response (`res.send(...)`) to short-circuit the chain.

Analogy: airport security. You walk through checkpoints in order — ID check, baggage scan, body scan. Each checkpoint can let you through or stop you. The route handler is the gate at the end of the line.

### Smallest working example
```javascript
const express = require('express');
const app = express();

// 1. JSON body parser
app.use(express.json());

// 2. Logger (custom middleware)
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// 3. The actual route
app.get('/hello', (req, res) => res.json({ msg: 'Hi' }));

// 4. Error handler — Express recognizes the 4-arg signature
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: 'Server error' });
});

app.listen(3000);
```

Order matters. `express.json()` must come before routes that read `req.body`. The error handler must come last.

### Drill (2 min)
Your logger forgets to call `next()`. What happens to the request? *(Hint: it hangs forever. The chain stops, and you never send a response. Always either `next()` or `res.send(...)`.)*

**Deep dive (later):** [nodejs/11-express-middleware-patterns.md](../nodejs/11-express-middleware-patterns.md)

---

## 3. Spring Boot — Entity Mapping (@Entity, @Id, @Column)

### Why this exists
JPA (Java Persistence API) is the standard ORM (Object-Relational Mapper) — it translates between Java objects and SQL rows. To make that work, you annotate your class so JPA knows which table, which column, and which field is the primary key.

### The idea in plain English
- `@Entity` — "this class maps to a database table."
- `@Table(name = "users")` — the table name (default: lower-cased class name).
- `@Id` — the primary key field.
- `@GeneratedValue` — let the database generate the ID (auto-increment).
- `@Column(name = "...", nullable = false)` — column metadata.

Analogy: a JPA entity is a translator between Java and SQL. The annotations are the dictionary entries — "in Java I'm `firstName`, in SQL I'm `first_name`."

### Smallest working example
```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(name = "first_name", length = 100)
    private String firstName;

    @Column(name = "created_at", updatable = false)
    private Instant createdAt = Instant.now();

    // getters/setters omitted
}
```

With this entity plus a `UserRepository extends JpaRepository<User, Long>`, Spring will create the table on startup (if you set `spring.jpa.hibernate.ddl-auto=update`) and generate SQL for you.

### Drill (2 min)
You name the field `firstName` in Java but the column in SQL is `first_name`. How does JPA bridge them? *(Hint: `@Column(name = "first_name")` on the field. Without it, Spring's default naming strategy maps camelCase Java → snake_case SQL automatically — but the explicit annotation is safer.)*

**Deep dive (later):** [spring-next/05-spring-data-jpa.md § 2. Entity Mapping](../spring-next/05-spring-data-jpa.md#2-entity-mapping)

---

## 4. MongoDB — Sharding (shard keys, chunks)

### Why this exists
A single replica set has a ceiling: one machine's disk and RAM. To scale past that, you **shard** — split the data across many replica sets, each holding a slice of the collection. Each slice is called a **shard**.

### The idea in plain English
You pick a **shard key** — a field (or fields) that decides which shard each document lives on. Mongo divides the key range into **chunks** and assigns chunks to shards. A router process (`mongos`) sits in front and forwards each query to the right shard.

Picking the shard key is the most important — and irreversible — decision. Good keys spread writes evenly and let common queries hit one shard. Bad keys create hot shards.

Analogy: sharding is like splitting a library across multiple branches. The shard key is "first letter of title." Books A–F go to branch 1, G–M to branch 2, etc. A reader who knows the title goes to the right branch directly.

### Smallest working example
```javascript
// 1. Enable sharding on the database
sh.enableSharding('app');

// 2. Choose a shard key for a collection.
//    Hashed key spreads writes evenly across shards.
sh.shardCollection('app.events', { userId: 'hashed' });

// 3. Inserts get routed automatically
db.events.insertOne({ userId: 'u123', type: 'click', ts: new Date() });
```

If a query includes `userId`, mongos sends it to one shard (fast). If not, mongos broadcasts to **all shards** (slow — called a *scatter-gather*).

### Drill (2 min)
You shard `orders` by `{ orderDate: 1 }` and most writes go to today's date. What goes wrong? *(Hint: a hot shard — every new insert lands on the same chunk, the same shard. Use a hashed key, or a compound key with a random/high-cardinality prefix.)*

**Deep dive (later):** [mongodb/10-sharding.md](../mongodb/10-sharding.md)

---

## 5. Postgres — Locking (FOR UPDATE, deadlocks)

### Why this exists
When two transactions try to update the same row, the database must order them. **Row-level locks** make a writer wait until the previous writer commits. Sometimes you also need a lock for a *read* — that's where `SELECT ... FOR UPDATE` comes in.

### The idea in plain English
`SELECT ... FOR UPDATE` tells Postgres: "I'm reading this row but I'm going to UPDATE it in a moment — block anyone else who wants the same lock until I commit." This is how you safely do read-then-write logic like "if balance >= 100, subtract 100."

A **deadlock** is when two transactions each hold a lock the other wants. Postgres detects it and aborts one with an error. Your code retries. Avoid them by always locking rows in the **same order** across transactions.

### Smallest working example
```sql
-- Safe debit: lock row, check, update, commit
BEGIN;
SELECT balance
  FROM accounts
  WHERE id = 'A'
  FOR UPDATE;                          -- lock until COMMIT

-- (app code checks balance, then runs the UPDATE)
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
COMMIT;
```

Without `FOR UPDATE`, two threads can read `balance = 100`, both decide it's enough, and both subtract — you end up at `-100`.

### Drill (2 min)
Transaction T1 locks row A then row B. Transaction T2 locks row B then row A. Both are waiting. What does Postgres do? *(Hint: detects the deadlock and aborts one with `ERROR: deadlock detected`. The app retries. Prevent by always locking in the same order — e.g., by row id ascending.)*

**Deep dive (later):** [postgres/11-locking.md](../postgres/11-locking.md)

---

## 6. HLD — Twitter feed walkthrough (high level)

### Why this exists
Showing each user a feed of "posts from the people I follow, freshest first" sounds easy until you have 500M users and celebrities with 100M followers. This is the canonical HLD interview because it forces you to think about reads vs writes.

### The idea in plain English
Two extremes for building the feed:
- **Pull (fan-in on read)** — when Alice opens her app, fetch posts from every account she follows, sort by time, return. Simple, but slow when she follows 1000 people.
- **Push (fan-out on write)** — when Bob tweets, copy the tweet into a feed list for every follower. Reads are fast (just fetch Alice's pre-built list), writes are slow. A bad fit for celebrities — one tweet, 100M inserts.

Real systems use a **hybrid**:
- Push for most users (cheap writes, fast reads).
- Pull for celebrities (don't fan-out, fetch their recent tweets at read time and merge).
- Heavy caching (Redis) for hot feeds.

### A picture
```
    Bob tweets ─────┐
                    ├─► Fan-out service:
                    │     - normal user? push to followers' feed cache
                    │     - celebrity?    skip — pulled on read
                    │
   Alice opens app  │
        └─► Feed service:
              merge( cached pushed feed,
                     live pull from celebrity tweets )
              + cache the result in Redis
```

### Drill (2 min)
A startup at 10K users — push, pull, or hybrid? *(Hint: pull. At 10K users with modest follow graphs, fan-in on read is trivial. Don't engineer for scale you don't have. Move to hybrid when feeds get slow.)*

**Deep dive (later):** [system-design/high-level-design/10-real-world-system-designs.md](../system-design/high-level-design/10-real-world-system-designs.md)

---

## 7. LLD — ATM machine

### Why this exists
ATM is a state-machine LLD problem: card in, PIN entered, withdraw, dispense, eject. It tests OOP, state transitions, and clear separation of concerns.

### The idea in plain English
Three actors:
- **ATM** — the machine. Holds cash, current card, current state.
- **State** — `Idle`, `CardInserted`, `Authenticated`, `Dispensing`. Each state allows different actions.
- **Bank** — external service: `verifyPin`, `getBalance`, `debit`.

Use the **State pattern** so each state is its own class — no giant `if (state == ...)` chain. The ATM holds a reference to the current state object and forwards calls to it.

### Smallest working example (skeleton)
```java
interface AtmState {
    void insertCard(Atm atm, Card card);
    void enterPin(Atm atm, String pin);
    void withdraw(Atm atm, int amount);
    void ejectCard(Atm atm);
}

class IdleState implements AtmState {
    public void insertCard(Atm atm, Card card) {
        atm.setCurrentCard(card);
        atm.setState(new CardInsertedState());
    }
    public void enterPin(Atm atm, String pin)     { /* error */ }
    public void withdraw(Atm atm, int amount)     { /* error */ }
    public void ejectCard(Atm atm)                { /* error */ }
}

// CardInsertedState, AuthenticatedState... etc.
```

The ATM never asks "what state am I in" — it just calls `state.withdraw(...)` and trusts the state object to do the right thing.

### Drill (2 min)
User tries to withdraw before inserting a card. What happens? *(Hint: `IdleState.withdraw(...)` is called — it raises an error or no-ops. Each state silently rejects illegal transitions, replacing what would be a giant switch in a monolithic design.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Recursion: Permutations

### The problem
Given a list of distinct integers, return every possible ordering.

```
Input:  [1, 2, 3]
Output: [[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]   // 3! = 6
```

### The idea
Recursion is "solve a smaller version, then build up." For permutations:
- Pick each element in turn as the **first** of the result.
- Recursively permute the **remaining** elements.
- Glue them together.

The structure: a `current` partial list + a set of "still available" numbers. When `current` is full, you've found one permutation. Otherwise, try each available number, recurse, then **backtrack** (remove it and try the next).

### Smallest working example
```javascript
function permute(nums) {
  const out = [];

  function backtrack(current, remaining) {
    if (remaining.length === 0) { out.push([...current]); return; }
    for (let i = 0; i < remaining.length; i++) {
      current.push(remaining[i]);
      const next = remaining.slice(0, i).concat(remaining.slice(i+1));
      backtrack(current, next);
      current.pop();                          // backtrack
    }
  }

  backtrack([], nums);
  return out;
}
```

**Time:** O(n × n!). **Space:** O(n) for the recursion depth (plus the output).

### Drill (3 min)
Trace `permute([1, 2])` step by step. *(You should see: backtrack([], [1,2]) → pick 1 → backtrack([1], [2]) → pick 2 → push [1,2]; backtrack pops; pick 2 from outer → backtrack([2], [1]) → push [2,1]. Result: [[1,2],[2,1]].)*

**Deep dive (later):** [dsa/ds-js/recursion-backtracking.md](../dsa/ds-js/recursion-backtracking.md) · [dsa/ds-java/recursion-backtracking.md](../dsa/ds-java/recursion-backtracking.md)

---

## 9. Design Pattern — Composite

### Intent
**Treat a single object and a group of objects the same way.** A file and a folder both implement `size()` — you don't care which one you're calling.

### When you'd use it
- File system: files and folders both have `size`, `name`, `delete()`.
- UI trees: a button and a panel both have `render()`.
- Org charts: an employee and a manager (who has employees) both have `getSalary()`.

### Smallest working example
```java
interface FsNode {
    int size();
}

class File implements FsNode {
    private final int bytes;
    public File(int bytes) { this.bytes = bytes; }
    public int size() { return bytes; }
}

class Folder implements FsNode {
    private final List<FsNode> children = new ArrayList<>();
    public void add(FsNode n) { children.add(n); }
    public int size() {
        return children.stream().mapToInt(FsNode::size).sum();
    }
}

// Usage — caller doesn't care
FsNode root = new Folder();
((Folder) root).add(new File(100));
((Folder) root).add(new File(50));
System.out.println(root.size());   // 150
```

A `Folder` contains `FsNode`s — which can themselves be Folders. Recursion is built in.

### Drill (1 min)
Why is a tree of `FsNode`s easier to walk than a separate `File[]` and `Folder[]`? *(Hint: one type, one `size()` call. You don't write two loops or two if-branches per operation. The structure absorbs the recursion.)*

**Deep dive (later):** [design-patterns/common/structural/composite.md](../design-patterns/common/structural/composite.md)

---

## 10. DevOps — Helm & Kustomize

### Why this exists
A Kubernetes app is many YAML files (Deployment, Service, ConfigMap, Ingress...). You probably want to install it across 3 environments (dev, staging, prod) with slight differences. Writing 3 copies is a maintenance disaster.

### The idea in plain English
Two solutions, different philosophies:

**Helm** — templating with values. A **chart** is a folder of templated YAML plus a `values.yaml` file. You `helm install myapp ./mychart --values prod.yaml` and Helm renders the templates with your values. Like a recipe book: same recipe, different ingredients per environment.

**Kustomize** — overlay-based, no templates. You have a `base/` folder with plain YAML, and `overlays/prod/` with patches that modify the base. Built into `kubectl` (`kubectl apply -k`). Like Photoshop layers: don't change the original, paint on top.

Rule of thumb: Helm for *packaging and distributing* apps (other teams install your chart). Kustomize for *managing your own multi-environment deploys*.

### Smallest working example
```yaml
# Helm — values.yaml
replicaCount: 3
image: myapp:1.2.0

# Helm template (templates/deployment.yaml)
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - image: {{ .Values.image }}
```

```yaml
# Kustomize — overlays/prod/kustomization.yaml
resources: [../../base]
patches:
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
  target: { kind: Deployment, name: web }
```

### Drill (2 min)
You want to install someone else's open-source app on your cluster, with custom config. Helm or Kustomize? *(Hint: Helm — most public Kubernetes apps publish a Helm chart you can install with one command and override via values.)*

**Deep dive (later):** [devops/09-helm-and-kustomize.md](../devops/09-helm-and-kustomize.md)

---

## End-of-day checklist

- [ ] Angular: I can fetch a typed list from an API with HttpClient
- [ ] Node.js: I can describe the middleware signature `(req, res, next)`
- [ ] Spring: I can annotate a class with `@Entity`, `@Id`, `@GeneratedValue`, `@Column`
- [ ] MongoDB: I can describe what a shard key does and what makes one "bad"
- [ ] Postgres: I know when to use `SELECT ... FOR UPDATE` and what causes deadlocks
- [ ] HLD: I can sketch push vs pull for a feed and say when to use a hybrid
- [ ] LLD: I can map an ATM to the State pattern with at least three state classes
- [ ] DSA: I implemented `permute(nums)` and traced one example
- [ ] DP: I can give a "file/folder" example for the Composite pattern
- [ ] DevOps: I can choose between Helm and Kustomize given a scenario

**If you remember just one thing today:** **layering**. Middleware layers in Express, decorators on Spring entities, overlays in Kustomize, fan-out layers in a feed — same idea everywhere.

**Tomorrow:** interceptors, routing patterns, JPA relationships, ODMs, stored procs, and the full Twitter feed design.
