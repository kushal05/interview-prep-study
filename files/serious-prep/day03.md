# Day 3 — Binding data, lifecycles, and SQL CTEs

> **Today's goal:** see how Angular shows data on screen, how npm pins versions, how Spring beans are born and die, and how SQL queries can name and reuse sub-results.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Data Binding & Directives (*ngFor, *ngIf) | 15m |
| 2 | Node.js | npm & package.json (semver, lockfile) | 15m |
| 3 | Spring Boot | Bean Lifecycle & Scopes | 15m |
| 4 | MongoDB | Query Operators ($eq, $gt, $in, $elemMatch) | 10m |
| 5 | Postgres | Subqueries & CTEs (WITH clause) | 10m |
| 6 | HLD | Databases (SQL vs NoSQL choice) | 12m |
| 7 | LLD | UML & Class Diagrams (the 4 relationships) | 12m |
| 8 | DSA | Stacks/Queues (Valid Parentheses) | 15m |
| 9 | Design Pattern | Abstract Factory | 8m |
| 10 | DevOps | Git & VCS (branch, merge, rebase basics) | 10m |

---

## 1. Angular — Data Binding & Directives

### Why this exists
A component has data in its TypeScript class and HTML on the screen. **Binding** is how Angular keeps the two in sync. You need four kinds of binding and two directives to write 90% of templates.

### The idea in plain English
Imagine the component class is a clipboard and the HTML is a whiteboard. Binding is the marker that copies between them:

- **Interpolation `{{ value }}`** — write a class value into the HTML.
- **Property binding `[src]="url"`** — set an HTML attribute from a class value.
- **Event binding `(click)="save()"`** — when the user clicks, run a method.
- **Two-way binding `[(ngModel)]="name"`** — for forms; data flows both ways.

The two structural directives `*ngFor` (loop) and `*ngIf` (conditional show) handle most "list of things" and "show this when" needs.

### Smallest working example
```typescript
@Component({
  standalone: true,
  imports: [CommonModule],
  template: `
    <input [value]="name">                            <!-- property -->
    <button (click)="greet()">Greet</button>          <!-- event -->
    <p>{{ message }}</p>                              <!-- interpolation -->

    <ul>
      <li *ngFor="let item of items">{{ item }}</li>  <!-- *ngFor -->
    </ul>

    <p *ngIf="items.length === 0">No items.</p>       <!-- *ngIf -->
  `,
})
export class Demo {
  name = 'Kushal';
  message = '';
  items = ['apples', 'bananas'];
  greet() { this.message = 'Hello, ' + this.name; }
}
```

`*ngFor` and `*ngIf` come from `CommonModule` — that's why we import it.

### Drill (2 min)
You write `<input [value]="name">` and type in the box. The `name` field in your class stays unchanged. Why? *(Hint: `[value]` is one-way (class → HTML). To get changes back, use two-way binding `[(ngModel)]="name"` (and import `FormsModule`).)*

**Deep dive (later):** [angular/directives.md](../angular/directives.md)

---

## 2. Node.js — npm & package.json

### Why this exists
Every Node project has dependencies — libraries written by others. **npm** (Node Package Manager) installs them, and `package.json` records *which* ones and *what versions* are allowed.

### The idea in plain English
Think of `package.json` as a shopping list: "I want express, somewhere around version 4." The **lockfile** (`package-lock.json`) is the receipt: "I actually bought express 4.19.0, with these exact sub-deps." Without a lockfile, two developers can install slightly different versions and one's machine breaks.

**Semver** (semantic versioning) is `MAJOR.MINOR.PATCH`:
- MAJOR = breaking change.
- MINOR = backward-compatible new feature.
- PATCH = backward-compatible fix.

`^4.19.0` means "4.x.x as long as 4 stays the major." `~4.19.0` means "4.19.x only."

### Smallest working example
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev":   "node --watch src/index.js",
    "test":  "node --test"
  },
  "dependencies": {
    "express": "^4.19.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0"
  }
}
```

`dependencies` go to production. `devDependencies` (build tools, test runners, linters) don't. Use `npm ci` (clean install) in CI — it refuses to install if the lockfile is missing or stale, which is what you want.

### Drill (2 min)
Your lockfile says `express@4.19.0`. Your `package.json` says `"express": "^4.19.0"`. A teammate runs `npm install`. They get `4.19.0`, not `4.19.4` (which is out). Why? *(Hint: `npm install` respects the lockfile if it's present. To bump, you'd have to delete the lockfile or run `npm update express`.)*

**Deep dive (later):** [nodejs/06-npm-and-package-json.md](../nodejs/06-npm-and-package-json.md)

---

## 3. Spring Boot — Bean Lifecycle & Scopes

### Why this exists
Spring creates your beans automatically — but *when*? And what if you need to run setup code (warm a cache) or cleanup code (close a file) at the right moment? That's the **lifecycle**. And what if you don't want a singleton? That's **scopes**.

### The idea in plain English
A bean's life has three big moments:
1. **Born** — the constructor runs and Spring injects its dependencies.
2. **Ready to use** — any `@PostConstruct` method runs (your chance to do setup that needs the injected fields).
3. **Closing down** — when the app shuts, `@PreDestroy` runs (your chance to clean up).

**Scopes** control *how many* of each bean exist:
- `singleton` (default) — one per app. Shared by everyone.
- `prototype` — a fresh one every time someone asks for it.
- `request` / `session` (web) — one per HTTP request / session.

Singleton is the default for a reason: it's simple and cheap. Use prototype only when you genuinely need a new instance each time.

### Smallest working example
```java
@Component
public class CacheWarmer {
    private final ProductRepository repo;
    private Map<Long, Product> cache;

    public CacheWarmer(ProductRepository repo) {
        this.repo = repo;
    }

    @PostConstruct
    void warm() {
        // Runs after construction + injection. Bean is not yet "live" externally.
        cache = repo.findAll().stream()
                    .collect(Collectors.toMap(Product::id, p -> p));
    }

    @PreDestroy
    void cleanup() {
        cache.clear();
    }
}
```

`@PostConstruct` is where you put "do this once, after Spring has wired me up."

### Drill (2 min)
A singleton bean has `Random random = new Random()` as a field. Two HTTP requests come in. Do they share the same `Random` object? *(Hint: yes — the bean is a singleton, so the `random` field is created once at startup and shared by every request. That's usually fine for `Random`, but think before using mutable shared state.)*

**Deep dive (later):** [spring-detailed/PART-13-ioc-container-deeply.md](../spring-detailed/PART-13-ioc-container-deeply.md)

---

## 4. MongoDB — Query Operators

### Why this exists
You rarely want "exact match." More often it's "age greater than 18" or "tag is one of these." Mongo's `$`-prefixed **operators** are how you express that.

### The idea in plain English
A Mongo query is a JSON pattern. Without operators, every field is an equality check. With operators, each field can carry a small condition object.

The ones you'll use 90% of the time:
- `$eq`, `$ne` — equals, not equals.
- `$gt`, `$gte`, `$lt`, `$lte` — greater/less than.
- `$in`, `$nin` — value is in (or not in) this list.
- `$exists` — the field is present.

For arrays of nested objects, **`$elemMatch`** is the must-know one: it asks "does *one* element satisfy *all* these conditions" (as opposed to "any element satisfies any condition," which is the default and often a bug).

### Smallest working example
```javascript
// Find adults named Alice or Bob
db.users.find({
  age: { $gte: 18 },
  name: { $in: ["Alice", "Bob"] }
});

// Document: { _id: 1, scores: [85, 92, 78] }

// $elemMatch — at least ONE element matches BOTH conditions
db.students.find({
  scores: { $elemMatch: { $gte: 80, $lte: 90 } }
});
// matches _id: 1 because 85 satisfies both
```

Without `$elemMatch`, `{ scores: { $gte: 80, $lte: 90 } }` would also match because 92 ≥ 80 *and* 78 ≤ 90 — different elements satisfying different conditions. Usually a bug.

### Drill (2 min)
A document is `{ tags: ["red", "blue", "green"] }`. Does the query `{ tags: ["red", "blue"] }` match it? *(Hint: no — without operators, that's an exact array equality check (same elements, same order). Use `{ tags: { $all: ["red", "blue"] } }` to check "contains both.")*

**Deep dive (later):** [mongodb/03-query-operators.md](../mongodb/03-query-operators.md)

---

## 5. Postgres — Subqueries & CTEs

### Why this exists
SQL queries often need an intermediate result — "the top 10 customers by revenue" — before the final answer. You *could* write a nested subquery, but **CTEs** (Common Table Expressions, using `WITH`) make it readable, like naming a step in a recipe.

### The idea in plain English
A CTE is a temporary named result you can reuse within a single query. Read top to bottom: define `high_value_customers`, then `SELECT` from it.

Compared to subqueries:
- Subquery: written inline, often hard to read when nested.
- CTE: named, defined up top, the main query reads cleanly.

There's also a **recursive CTE** — a query that refers to itself, used for hierarchies (managers above an employee, parents of a category). Powerful, but you'll meet it rarely.

### Smallest working example
```sql
-- A CTE makes a two-step query readable
WITH high_value_customers AS (
    SELECT user_id, SUM(total) AS lifetime
    FROM orders
    GROUP BY user_id
    HAVING SUM(total) > 10000
)
SELECT u.email, hvc.lifetime
FROM users u
JOIN high_value_customers hvc ON hvc.user_id = u.id
ORDER BY hvc.lifetime DESC;
```

The CTE block defines a "virtual table" that the main query joins against. The whole thing is still *one* query — the CTE doesn't get saved anywhere.

### Drill (2 min)
Could you rewrite the example without a CTE? *(Hint: yes — put the SELECT in `FROM (...) AS hvc` as a subquery. Same result. The CTE is just easier to read, especially with multiple steps.)*

**Deep dive (later):** [postgres/04-subqueries-and-ctes.md](../postgres/04-subqueries-and-ctes.md)

---

## 6. High-Level Design — Databases (SQL vs NoSQL)

### Why this exists
Every system has to pick where to store data. The first big fork: **SQL** (relational tables) or **NoSQL** (everything else — documents, key-value, graph). Picking wrong adds years of pain.

### The idea in plain English
SQL is a strict spreadsheet — every row has the same columns, with declared types. Best when data has a fixed shape and you need transactions ("subtract money here AND add it there, or neither").

NoSQL is an umbrella for several shapes:
- **Document** (Mongo) — JSON-ish objects with flexible shapes.
- **Key-Value** (Redis) — pure `get(key)`/`set(key, value)`. Used for caches.
- **Wide-column** (Cassandra) — huge tables sharded across many servers.
- **Graph** (Neo4j) — nodes and edges; great for "friend-of-friend" queries.

A simple decision table:

| Need | Pick |
|---|---|
| Transactions across rows (banking) | SQL |
| Variable shape per record (user profiles) | Document |
| Cache, sessions, rate-limiting | Key-value (Redis) |
| Massive write throughput | Wide-column |
| Friend-of-friend graph queries | Graph |

Real systems use several at once — Postgres for orders, Redis for cache, Elasticsearch for search. That's called **polyglot persistence**.

### Drill (2 min)
You're storing user profiles where some users have addresses, some have many, some have none. SQL or document? *(Hint: document. SQL forces you into a separate `addresses` table with joins. A document can hold an `addresses` array inline.)*

**Deep dive (later):** [system-design/high-level-design/03-databases.md](../system-design/high-level-design/03-databases.md)

---

## 7. Low-Level Design — UML & Class Diagrams

### Why this exists
When you discuss design with anyone — interviewer, teammate, future-you reading old notes — a sketch beats a paragraph. **UML class diagrams** are the visual shorthand. You need to recognize four relationship arrows.

### The idea in plain English
A class is a box with three sections: name, attributes (data), operations (methods). `+` means public, `-` means private.

The four relationships you'll see most:

| Relationship | What it means | Example |
|---|---|---|
| **Association** (`A → B`) | A uses B | `OrderService` uses `PaymentGateway` |
| **Aggregation** (`A ◇— B`) | A "has" B; B can outlive A | A `Team` has `Player`s |
| **Composition** (`A ◆— B`) | A "owns" B; B dies with A | An `Order` owns its `OrderItem`s |
| **Inheritance** (`A —▷ B`) | A extends B | `Dog extends Animal` |

Composition vs aggregation is the subtle one — composition is "if the parent dies, the child has no reason to exist" (an order line item without an order makes no sense). Aggregation is "the child can stand alone" (a team disbanding doesn't kill the players).

### A minimal sketch
```
┌─────────────────────────┐
│      OrderService       │
├─────────────────────────┤
│ - payment: PaymentGateway│      (association)
├─────────────────────────┤
│ + place(cart): Order    │
└─────────────────────────┘
           │
           ◆  (composition — Order owns its items)
           │
       OrderItem (one-to-many)
```

You don't need a tool to draw this — pencil and paper is fine.

### Drill (2 min)
A `User` has an `Address`. Should that be composition or aggregation? *(Hint: aggregation — an address can exist independently (think of an `addresses` table reused across users). Composition fits things like an order and its line items, where the child has no life outside the parent.)*

**Deep dive (later):** [system-design/low-level-design/03-uml-and-class-diagrams.md](../system-design/low-level-design/03-uml-and-class-diagrams.md)

---

## 8. DSA — Stacks/Queues (Valid Parentheses)

### The problem
Given a string of only `()[]{}`, decide if it's valid: every opener has a matching closer in the right order.

Valid: `()`, `()[]{}`, `([{}])`.
Invalid: `(]`, `([)]`, `(`.

### The idea
A stack is a "last in, first out" pile of plates. Walk the string left to right:
- If you see an opener (`(`, `[`, `{`), push it on the stack.
- If you see a closer, pop the top — if it's the matching opener, good. If not, fail.
- At the end, the stack must be empty (otherwise there were leftover openers).

A stack is perfect because brackets nest, and the **most recently opened** bracket is the **first one that must close**. That's LIFO.

### Smallest working example
```javascript
function isValid(s) {
    const stack = [];
    const pairs = { ')': '(', ']': '[', '}': '{' };
    for (const c of s) {
        if (c === '(' || c === '[' || c === '{') {
            stack.push(c);
        } else {
            // closer: stack must be non-empty AND top must match
            if (stack.pop() !== pairs[c]) return false;
        }
    }
    return stack.length === 0;
}

isValid("([{}])");   // true
isValid("([)]");     // false
```

`stack.pop()` returns `undefined` on an empty array, which won't equal any opener — so we don't need a separate "empty stack" check.

### Drill (2 min)
Trace what the stack looks like as you process `"({[]})"`. *((push `(`) → `[(]` (push `{`) → `[(,{]` (push `[`) → `[(,{,[]` (pop `[`, matches `]`) → `[(,{]` (pop `{`, matches `}`) → `[(]` (pop `(`, matches `)`) → `[]`. Empty stack at end → valid.)*

**Deep dive (later):** [dsa/ds-js/stacks-queues.md](../dsa/ds-js/stacks-queues.md) · [dsa/ds-java/stacks-queues.md](../dsa/ds-java/stacks-queues.md)

---

## 9. Design Pattern — Abstract Factory

### Intent
**Yesterday's Factory Method made one product. Abstract Factory makes a *family* of related products** — and guarantees the parts match.

### When you'd use it
- You have multiple product families (themes, platforms) and want to ensure the parts go together: a `MacButton` should pair with a `MacCheckbox`, not a `WindowsCheckbox`.
- The set of product *types* is fixed; only the *variants* (which family) change.

### Smallest working example
```java
// Products
public interface Button   { void render(); }
public interface Checkbox { void render(); }

// Family A — macOS
class MacButton   implements Button   { public void render() { /* mac style */ } }
class MacCheckbox implements Checkbox { public void render() { /* mac style */ } }

// Family B — Windows
class WinButton   implements Button   { public void render() { /* win style */ } }
class WinCheckbox implements Checkbox { public void render() { /* win style */ } }

// The Abstract Factory
public interface UIFactory {
    Button   createButton();
    Checkbox createCheckbox();
}
class MacUIFactory implements UIFactory {
    public Button   createButton()   { return new MacButton(); }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}
class WinUIFactory implements UIFactory {
    public Button   createButton()   { return new WinButton(); }
    public Checkbox createCheckbox() { return new WinCheckbox(); }
}

// Caller picks ONE factory at startup; uses it for everything
UIFactory ui = isMac ? new MacUIFactory() : new WinUIFactory();
ui.createButton().render();
ui.createCheckbox().render();
```

You can't accidentally mix Mac and Win parts because each factory only knows how to make its own family.

### Drill (1 min)
In one sentence, what's the difference between Factory Method and Abstract Factory? *(Hint: Factory Method makes one product; Abstract Factory makes a family of related products that are guaranteed to match.)*

**Deep dive (later):** [design-patterns/common/creational/abstract-factory.md](../design-patterns/common/creational/abstract-factory.md)

---

## 10. DevOps — Git & VCS

### Why this exists
Every codebase uses Git. Beyond `add`, `commit`, `push`, you need branches (to work in parallel), merges and rebases (to bring branches together), and the reflog (your safety net when you mess up).

### The idea in plain English
Git is a tree of snapshots called **commits**. Each commit points to its parent.

- **Branch** = a sticky note saying "this commit is the latest on this line of work." Cheap and disposable.
- **Merge** = "join branch B into branch A" — creates a special merge commit with two parents. History looks forked.
- **Rebase** = "replay my commits as if they were made on top of the latest A" — produces a clean, straight history.

**Rule of thumb:** rebase your *own* feature branch to keep it tidy before opening a PR. Never rebase a shared branch — others have based work on it; rewriting hurts.

### Smallest working example
```bash
# Start a feature branch
git checkout -b feature/login

# ... work, commit ...
git add .
git commit -m "feat: add login form"

# Bring in latest main without a merge commit
git fetch origin
git rebase origin/main

# Push and open a PR
git push -u origin feature/login

# Oh no — I deleted important commits with a bad reset!
git reflog                     # shows every move HEAD has made
git reset --hard HEAD@{5}      # jump back to the state 5 moves ago — recovered
```

The **reflog** is Git's undo history. Even commits you think are lost usually live there for ~90 days.

### Drill (2 min)
When would you choose `merge` over `rebase`? *(Hint: when integrating a branch others have shared. Rebasing rewrites history; if they've based work on the original commits, their copy and yours diverge. Merge preserves both histories with a merge commit.)*

**Deep dive (later):** [devops/03-git-and-vcs.md](../devops/03-git-and-vcs.md)

---

## End-of-day checklist

- [ ] Angular: I can name the four binding syntaxes
- [ ] Node.js: I know what the lockfile is for and why `npm ci` exists
- [ ] Spring: I can explain `@PostConstruct` in one sentence
- [ ] MongoDB: I know why `$elemMatch` exists
- [ ] Postgres: I wrote one CTE using `WITH ... AS (...)`
- [ ] HLD: I can pick SQL vs document for a small example
- [ ] LLD: I know the difference between composition and aggregation
- [ ] DSA: I solved Valid Parentheses with a stack
- [ ] DP: I can give the one-sentence difference between Factory Method and Abstract Factory
- [ ] DevOps: I know what `git reflog` is for

**If you remember just one thing today:** every framework gives you tools for the same problem — keep code, data, and structure in sync as the system grows. Binding does it for UI, lifecycles for beans, CTEs for queries.

**Tomorrow:** pipes, core Node modules, Spring AOP, MongoDB indexes, window functions, caching, REST design, linked lists, Builder pattern, and Docker basics.
