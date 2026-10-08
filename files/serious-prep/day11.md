# Day 11 — Interceptors, Routers, Relationships, and Fan-Out

> **Today's goal:** see how Angular adds auth headers everywhere automatically, how Express organizes big route files, how JPA models 1-to-many relationships, and the Twitter timeline fan-out trade-off in full.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | HTTP Interceptors (global auth header) | 15m |
| 2 | Node.js | Routing patterns in Express | 15m |
| 3 | Spring Boot | JPA Relationships (1-1, 1-N, N-N) | 15m |
| 4 | MongoDB | Mongoose ODM (schemas, models) | 10m |
| 5 | Postgres | Stored Procs & Triggers | 10m |
| 6 | HLD | Twitter Design (fan-out push vs pull) | 12m |
| 7 | LLD | TicTacToe game | 12m |
| 8 | DSA | Backtracking — N-Queens (intro) | 15m |
| 9 | Design Pattern | Bridge | 8m |
| 10 | DevOps | CI/CD Fundamentals | 10m |

---

## 1. Angular — HTTP Interceptors (auth headers, globally)

### Why this exists
Every API call needs the same `Authorization: Bearer ...` header. Adding it on every `http.get(...)` is repetitive and easy to forget. An **interceptor** is middleware for HttpClient — it sees every outgoing request and can add headers, log timing, retry, or transform errors in one place.

### The idea in plain English
An interceptor is a function that receives the request and a `next` function. You modify the request (e.g., add a header), then call `next(req)` to pass it down the chain. Multiple interceptors stack in registration order — like Express middleware, but for the browser's HTTP calls.

Analogy: Interceptors are the corporate stamp at the mailroom — every letter going out gets the company logo added before it reaches the post office.

### Smallest working example
```typescript
// auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  if (!token) return next(req);                      // no token? pass through

  const cloned = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });
  return next(cloned);
};

// app.config.ts — register the interceptor
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor])),
  ],
};
```

Every call from any `HttpClient` in the app now gets the header automatically.

### Drill (2 min)
Why do we `req.clone({ ... })` instead of `req.headers.set(...)`? *(Hint: `HttpRequest` is immutable — you can't mutate it. `clone(...)` returns a new request with the changes applied.)*

**Deep dive (later):** [angular/http-interceptors.md](../angular/http-interceptors.md)

---

## 2. Node.js — Routing patterns in Express

### Why this exists
One file with 50 routes gets unreadable fast. Express has `Router` — a mini-app you can mount under a path prefix. Each feature gets its own router file; `app.use('/users', usersRouter)` plugs it in.

### The idea in plain English
- **Route params** — `:id` in the path, read via `req.params.id`.
- **Query params** — `?status=open`, read via `req.query.status`.
- **Router** — a self-contained module that exports its own routes; mounted under a path prefix in the main app.

Analogy: routers are like USB hubs. The main app has one socket (`/users`), and the user router exposes many sub-endpoints under it.

### Smallest working example
```javascript
// routes/users.js
const router = require('express').Router();

router.get('/', (req, res) => {
  res.json({ status: req.query.status || 'all' });        // ?status=open
});

router.get('/:id', (req, res) => {
  res.json({ id: req.params.id });                        // /users/42
});

router.post('/', (req, res) => {
  res.status(201).json(req.body);
});

module.exports = router;

// app.js
const app = require('express')();
app.use(express.json());
app.use('/users', require('./routes/users'));             // mount the router
app.listen(3000);
```

Now `GET /users?status=open` and `GET /users/42` both work — handled in `routes/users.js`.

### Drill (2 min)
You want a route `PUT /users/:id`. Where do you add it? *(Hint: in `routes/users.js` as `router.put('/:id', ...)` — the `/users` prefix is added by the mount in `app.js`.)*

**Deep dive (later):** [nodejs/10-http-and-express.md](../nodejs/10-http-and-express.md)

---

## 3. Spring Boot — JPA Relationships (1-1, 1-N, N-N)

### Why this exists
Real data is connected: a User has many Orders, an Order has many Items, an Item belongs to many Categories. JPA models these with `@OneToMany`, `@ManyToOne`, `@OneToOne`, `@ManyToMany` annotations so you can navigate from one entity to another in code as if they were Java references.

### The idea in plain English
Pick the side that **owns the foreign key**. By default, the "many" side does (each order row stores its `user_id`). The other side uses `mappedBy` to say "I'm the inverse — look at the field on the other entity."

Three common shapes:
- `@ManyToOne` — many orders belong to one user. FK on the orders table.
- `@OneToMany(mappedBy = "user")` — the inverse, on the user side.
- `@ManyToMany` — needs a join table (e.g., `student_course`).

Also know **fetch types**: `FetchType.LAZY` (default for `@OneToMany`/`@ManyToMany`; loads on first access) vs `EAGER` (always loaded). Lazy + a missing transaction = the classic `LazyInitializationException`.

### Smallest working example
```java
@Entity
public class User {
    @Id @GeneratedValue private Long id;
    private String name;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();
}

@Entity
public class Order {
    @Id @GeneratedValue private Long id;
    private BigDecimal total;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")              // FK column on orders table
    private User user;
}
```

`mappedBy = "user"` tells JPA: "to find a user's orders, look at the `user` field on the Order side."

### Drill (2 min)
You read a `User` from the repository, return it from a controller as JSON, and get a `LazyInitializationException`. Why? *(Hint: serialization tried to access `orders` after the transaction closed. Fix: fetch eagerly with a `JOIN FETCH` query, or return a DTO with only the fields you want.)*

**Deep dive (later):** [spring-next/05-spring-data-jpa.md § 3. Relationships](../spring-next/05-spring-data-jpa.md#3-relationships)

---

## 4. MongoDB — Mongoose ODM (schemas, models)

### Why this exists
The native MongoDB driver gives you raw objects — no validation, no types, no helpers. **Mongoose** is the most popular ODM (Object Document Mapper) for Node.js: it adds schemas, type casting, validation, and middleware to MongoDB.

### The idea in plain English
You define a **schema** (the shape and rules) and turn it into a **model** (the class you use to query and create documents). Mongoose runs every save through the schema — invalid fields are rejected before they reach Mongo.

Analogy: Mongoose is Mongo's TypeScript-aware secretary. It checks your forms (validation), stamps them (timestamps), and remembers what each form is supposed to look like (schema).

### Smallest working example
```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true, lowercase: true },
  name:  { type: String, required: true, trim: true, minlength: 2 },
  role:  { type: String, enum: ['user', 'admin'], default: 'user' },
}, { timestamps: true });                            // adds createdAt, updatedAt

const User = mongoose.model('User', userSchema);

// Usage — validation runs automatically
const u = new User({ email: 'JANE@X.COM ', name: 'Jane' });
await u.save();                                      // email becomes 'jane@x.com'

const all = await User.find({ role: 'admin' });
```

The model gives you `find`, `findById`, `findOne`, `updateOne`, etc., already wired to the right collection.

### Drill (2 min)
You try to save a user with `role: 'superuser'`. What happens? *(Hint: validation fails — `enum: ['user', 'admin']` rejects unknown values. Mongoose throws a `ValidationError` before sending anything to Mongo.)*

**Deep dive (later):** [mongodb/11-mongoose-odm.md](../mongodb/11-mongoose-odm.md)

---

## 5. Postgres — Stored Procs & Triggers

### Why this exists
Sometimes you want logic to run **inside** the database — for performance, atomicity, or to enforce a rule no matter which app inserts data. **Stored procedures** are SQL functions; **triggers** fire stored procedures automatically on `INSERT`/`UPDATE`/`DELETE`.

### The idea in plain English
- **Function (stored procedure)** — a named block of SQL/PL-pgSQL you call like `SELECT my_fn(args)`.
- **Trigger** — "every time someone modifies this table, run that function." Choose `BEFORE` or `AFTER`, and `ROW` (once per row) or `STATEMENT` (once per query).

Common use case: a `updated_at` column that the database keeps fresh, so apps never forget.

### Smallest working example
```sql
-- 1. The trigger function
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 2. The trigger that calls it
CREATE TRIGGER users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

-- 3. Now any UPDATE auto-refreshes updated_at
UPDATE users SET name = 'Jane M' WHERE id = 1;
-- updated_at is set to now() — without the app touching it
```

`NEW` is the row being inserted/updated; `OLD` is the previous version (for UPDATE/DELETE).

### Drill (2 min)
Should you put complex business rules (e.g., fraud checks) in a trigger? *(Hint: rarely. Triggers are invisible — debugging "why did this insert fail?" is painful when the cause is hidden in a trigger. Keep triggers for simple, universal rules like timestamps and audit logs.)*

**Deep dive (later):** [postgres/12-stored-procedures-and-triggers.md](../postgres/12-stored-procedures-and-triggers.md)

---

## 6. HLD — Twitter Design (fan-out: push vs pull)

### Why this exists
This is the canonical HLD problem. You've seen the high-level sketch (Day 10); today, lock in the fan-out trade-off.

### The idea in plain English
The question is: when do you compute Alice's feed?

**Push (fan-out on write):** when Bob tweets, the system writes a copy of the tweet ID into a list for every one of Bob's followers. Alice's feed is already there when she opens the app — one read.
- Wins: blazing-fast reads.
- Costs: a celebrity with 100M followers triggers 100M inserts per tweet. Storage explodes; write latency suffers.

**Pull (fan-in on read):** Alice opens the app; the system fetches the latest tweets from every account she follows and merges them in memory.
- Wins: cheap writes, no storage explosion.
- Costs: slow reads if she follows many people; CPU-heavy at the edge.

**Hybrid (real Twitter):** push for normal users (most cases, fast reads). For celebrities, **don't push** — store their tweets once and pull them at read time. Then merge "pushed feed" + "pulled celebrity tweets" + cache the result.

### A picture
```
Bob (normal)  tweets ──► push to each follower's redis list (fan-out)
LeBron (celeb) tweets ──► store; do NOT fan out

Alice opens app:
  feed = merge(
    pushed-list[alice],        // pre-built (fast)
    pull(celebs_alice_follows) // computed (small)
  )
```

### Drill (2 min)
A user follows 1000 normal accounts and 5 celebrities. Which path computes her feed? *(Hint: hybrid. The 1000 normal accounts pushed into her cached feed; the 5 celebrities pulled at read time and merged. Best of both.)*

**Deep dive (later):** [system-design/high-level-design/10-real-world-system-designs.md](../system-design/high-level-design/10-real-world-system-designs.md)

---

## 7. LLD — TicTacToe game

### Why this exists
A tiny, complete state-machine problem you can finish in 15 minutes. Tests OOP, win detection, and turn handling.

### The idea in plain English
Three classes:
- **Board** — a 3×3 grid; methods `place(row, col, mark)` and `checkWinner()`.
- **Player** — name and mark (`X` or `O`).
- **Game** — the controller: alternates turns, asks board for moves, checks winner or draw.

The interesting part is `checkWinner()`: check 3 rows, 3 columns, 2 diagonals — 8 lines total. Or maintain row/col/diagonal counters that update on each move (O(1) check per move).

### Smallest working example (skeleton)
```java
class Board {
    private final char[][] g = new char[3][3];
    private final int[] rows = new int[3], cols = new int[3];
    private int diag = 0, anti = 0;
    private int moves = 0;

    /** Returns 'X', 'O', or '.' for no-winner. */
    public char place(int r, int c, char mark) {
        g[r][c] = mark;
        moves++;
        int delta = mark == 'X' ? 1 : -1;
        rows[r] += delta; cols[c] += delta;
        if (r == c)            diag += delta;
        if (r + c == 2)        anti += delta;
        if (Math.abs(rows[r]) == 3 || Math.abs(cols[c]) == 3
         || Math.abs(diag)    == 3 || Math.abs(anti)    == 3) return mark;
        return moves == 9 ? 'D' : '.';   // D = draw
    }
}
```

Each move runs in O(1); we never re-scan the board.

### Drill (2 min)
Why use ±1 counters instead of re-checking the board each move? *(Hint: O(1) per move vs O(N) per check. For a 3×3 board it's negligible; for N×N (e.g., Connect Four 7×6) it's a real win.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Backtracking: N-Queens (intro)

### The problem
Place N queens on an N×N chessboard so that no two attack each other (no shared row, column, or diagonal). Return the count of valid arrangements (or one example board).

```
N = 4 → 2 solutions
N = 8 → 92 solutions
```

### The idea
**Backtracking** = "try a choice, recurse, if it fails undo and try the next." For N-Queens:
- Place queens row by row.
- For row `r`, try each column `c`.
- If `(r, c)` is attacked by any earlier queen, skip.
- Otherwise, mark it and recurse to row `r+1`.
- When all N rows are placed, you found one solution. Otherwise, backtrack.

Three sets give O(1) "is this square attacked?" checks: `cols`, `diag1` (`r - c`), `diag2` (`r + c`).

### Smallest working example
```javascript
function totalNQueens(n) {
  let count = 0;
  const cols = new Set(), d1 = new Set(), d2 = new Set();

  function place(row) {
    if (row === n) { count++; return; }
    for (let c = 0; c < n; c++) {
      if (cols.has(c) || d1.has(row - c) || d2.has(row + c)) continue;
      cols.add(c); d1.add(row - c); d2.add(row + c);
      place(row + 1);
      cols.delete(c); d1.delete(row - c); d2.delete(row + c);    // backtrack
    }
  }

  place(0);
  return count;
}

totalNQueens(4);   // 2
```

**Time:** much less than O(N!) thanks to pruning. **Space:** O(N) for the sets plus the recursion stack.

### Drill (3 min)
Why is `row - c` constant along a diagonal? *(Hint: every square on the same `\` diagonal has the same `row - col`. Same logic: `row + col` is constant along `/` diagonals. That's why two sets are enough to track both diagonal directions.)*

**Deep dive (later):** [dsa/ds-js/recursion-backtracking.md](../dsa/ds-js/recursion-backtracking.md) · [dsa/ds-java/recursion-backtracking.md](../dsa/ds-java/recursion-backtracking.md)

---

## 9. Design Pattern — Bridge

### Intent
**Split a thing that varies in two dimensions into two separate hierarchies**, then connect them at runtime — so you don't get an explosion of subclasses.

### When you'd use it
You have shapes (Circle, Square) and renderers (SVG, Canvas). Without Bridge, you'd write `SvgCircle`, `CanvasCircle`, `SvgSquare`, `CanvasSquare` — and N × M classes when adding a third shape. With Bridge, you split into Shape (abstraction) + Renderer (implementation) and combine them at runtime.

### Smallest working example
```java
interface Renderer {
    void drawCircle(double x, double y, double r);
}

class SvgRenderer    implements Renderer { /* draws SVG    */ }
class CanvasRenderer implements Renderer { /* draws Canvas */ }

abstract class Shape {
    protected final Renderer renderer;
    public Shape(Renderer r) { this.renderer = r; }
    public abstract void draw();
}

class Circle extends Shape {
    private final double x, y, r;
    public Circle(Renderer r, double x, double y, double rad) {
        super(r); this.x = x; this.y = y; this.r = rad;
    }
    public void draw() { renderer.drawCircle(x, y, r); }
}

// Mix and match at runtime
new Circle(new SvgRenderer(), 0, 0, 5).draw();
new Circle(new CanvasRenderer(), 0, 0, 5).draw();
```

### Drill (1 min)
What's the difference between Bridge and Strategy? *(Hint: Strategy swaps **one algorithm**. Bridge separates **two whole hierarchies** so each can evolve independently. They look similar but the intent and scope are different.)*

**Deep dive (later):** [design-patterns/common/structural/bridge.md](../design-patterns/common/structural/bridge.md)

---

## 10. DevOps — CI/CD Fundamentals

### Why this exists
"It worked on my machine" is not a release process. **Continuous Integration (CI)** automates build + test on every commit. **Continuous Delivery (CD)** automates deploys to the next environment. Together they let teams ship many times a day with confidence.

### The idea in plain English
A pipeline is a recipe: **build → test → deploy**. Each step runs in a fresh container so it's reproducible. Failure at any step stops the pipeline. The result: every commit either reaches production or fails fast with a red X.

Three deployment strategies you should know:
- **Rolling** — replace pods one at a time. Cheap, default. Slight version overlap during the rollout.
- **Blue-Green** — run both versions side by side, switch the router from blue to green in one move. Fast rollback. Doubles cost during the switch.
- **Canary** — release to 1% of users first; watch metrics; if healthy, expand to 10%, 100%. Safer for risky changes.

### Smallest working example
```yaml
# A generic pipeline (think GitHub Actions / GitLab CI / Jenkins)
stages: [build, test, deploy]

build:
  stage: build
  script:
    - npm ci
    - npm run build
  artifacts: { paths: [dist/] }

test:
  stage: test
  script: [npm test]

deploy_prod:
  stage: deploy
  script:
    - kubectl set image deployment/web web=myapp:${CI_COMMIT_SHA}    # rolling
  when: manual                                                       # gate
  only: [main]
```

`when: manual` adds a human gate before production. `only: [main]` ensures only commits on `main` can deploy.

### Drill (2 min)
A change touches the payment system. Rolling, blue-green, or canary? *(Hint: canary — start with 1% of traffic, watch error rates, expand only if metrics stay clean. For risky changes, the slower rollout is the safer rollout.)*

**Deep dive (later):** [devops/10-ci-cd-fundamentals.md](../devops/10-ci-cd-fundamentals.md)

---

## End-of-day checklist

- [ ] Angular: I can add an HTTP interceptor that injects a Bearer token globally
- [ ] Node.js: I can split routes into a separate file and `app.use(...)` it
- [ ] Spring: I can annotate a `User`/`Order` pair with `@OneToMany` and `@ManyToOne`
- [ ] MongoDB: I can write a Mongoose schema with validation and a model from it
- [ ] Postgres: I can write a BEFORE UPDATE trigger that maintains `updated_at`
- [ ] HLD: I can explain push vs pull vs hybrid in 30 seconds
- [ ] LLD: I can sketch a TicTacToe board with O(1) win detection
- [ ] DSA: I implemented N-Queens with three sets and explained the diagonals trick
- [ ] DP: I can give one example of Bridge vs Strategy
- [ ] DevOps: I can name three deployment strategies and pick one per scenario

**If you remember just one thing today:** **the chain is everywhere.** Interceptors, middleware, JPA cascades, CI/CD stages, fan-out paths — they're all "do this, then that, in order, with the option to short-circuit."

**Tomorrow:** RxJS observables, error handling, JPQL, JSONB, URL shortener design, and chess as LLD.
