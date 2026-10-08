# Day 19 — Effects, hardening, replicas, sharding

> **Today's goal:** see how NgRx handles async, learn the OWASP basics every Node dev should know, do method-level Spring auth, and pick a sharding strategy.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | NgRx Effects & Selectors | 15m |
| 2 | Node.js | Security (OWASP basics, helmet, rate limiting) | 15m |
| 3 | Spring Boot | Method Security (@PreAuthorize, @PostAuthorize) | 15m |
| 4 | MongoDB | Index Strategies (covering, partial, sparse) | 10m |
| 5 | Postgres | Read Replicas & Failover | 10m |
| 6 | HLD | Sharding Strategies (range, hash, geo) | 12m |
| 7 | LLD | Vending Machine | 12m |
| 8 | DSA | Greedy (Activity Selection / Jump Game) | 15m |
| 9 | Design Pattern | Chain of Responsibility | 8m |
| 10 | DevOps | Terraform (providers, resources, state, modules) | 10m |

---

## 1. Angular — NgRx Effects & Selectors

### Why this exists
Day 18 showed reducers — pure functions. But "the user clicked save → call the API → store the result" needs **side effects** (HTTP calls). NgRx Effects keep reducers pure by handling async separately.

### The idea in plain English
- **Effect** — listens to an action stream, performs a side effect (HTTP, localStorage, etc.), and emits a *new* action with the result. Reducers see only the new action; they stay pure.
- **Selector** — already covered, but here's the twist: `createSelector` is **memoized** — if its inputs haven't changed, it returns the cached result. Avoids unnecessary component re-renders.

**Analogy:** reducers are the spreadsheet formulas (deterministic). Effects are the spreadsheet's "fetch from web" button — they go talk to the outside world, then write a result back into a cell that the formulas can react to.

### Smallest working example
```typescript
// effects.ts
@Injectable()
export class UserEffects {
  constructor(private actions$: Actions, private api: UserApi) {}

  loadUser$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadUser),
      mergeMap(action =>
        this.api.getUser(action.id).pipe(
          map(user => loadUserSuccess({ user })),
          catchError(err => of(loadUserFailure({ err })))
        )
      )
    )
  );
}

// selectors.ts (memoized)
export const selectUserState = (s: AppState) => s.user;
export const selectUserName  = createSelector(selectUserState, u => u.name);
```

The component dispatches `loadUser({ id })`. The effect catches it, calls the API, dispatches success or failure. The reducer handles those new actions. The component re-renders via the memoized selector.

### Drill (2 min)
Why does the effect dispatch a *new* action instead of just storing the result directly? *(Hint: keeps reducers as the single source of state changes. The action log stays a complete history of "what happened" — auditable and replayable.)*

**Deep dive (later):** [angular/state-ngrx.md](../angular/state-ngrx.md)

---

## 2. Node.js — Security

### Why this exists
The internet is hostile. Without basic hardening, your Node service is one curl away from a leaked database or an XSS payload. The OWASP Top 10 and two libraries cover the majority of low-hanging fruit.

### The idea in plain English
The **OWASP Top 10** is the most common web vulnerabilities. The ones that matter most for a Node API:
1. **Injection** (SQL, NoSQL) — never concatenate user input into queries; use parameterized queries or an ORM.
2. **Broken auth** — hash passwords with bcrypt, use HttpOnly cookies, short JWT TTLs.
3. **Sensitive data exposure** — TLS everywhere, don't log secrets.
4. **XSS** — escape output, set Content-Security-Policy, never `eval` user input.
5. **CSRF** — for cookie-auth, use CSRF tokens or SameSite cookies.
6. **Insecure deserialization / SSRF** — validate URLs, don't trust user-supplied URLs in `fetch`.

Two libraries that handle a lot of this:
- **`helmet`** — sets a dozen security headers in one line (CSP, X-Frame-Options, HSTS).
- **`express-rate-limit`** — caps requests per IP per window so brute force / login spam dies fast.

### Smallest working example
```javascript
const express = require('express');
const helmet  = require('helmet');
const rateLimit = require('express-rate-limit');

const app = express();
app.use(helmet());                     // sensible defaults for many headers

app.use('/login', rateLimit({
  windowMs: 15 * 60 * 1000,             // 15 minutes
  max: 10,                              // 10 attempts per IP
  standardHeaders: true,
}));

app.use(express.json({ limit: '100kb' })); // refuse huge bodies (DoS basics)
```

Five lines, big security uplift. Add input validation (Zod, Day 15) and parameterized queries (Day 16) and you've covered the worst of OWASP.

### Drill (2 min)
A bug lets a user POST 50 MB of JSON. What's the simple defense? *(Hint: `express.json({ limit: '100kb' })`. Any body over 100kb gets rejected with 413 before your handler runs. Cheap protection against memory-exhaustion DoS.)*

**Deep dive (later):** [nodejs/19-security.md](../nodejs/19-security.md)

---

## 3. Spring Boot — Method Security

### Why this exists
URL-level rules (Day 16's `requestMatchers`) are coarse. Sometimes the rule depends on the data: "user can edit a post only if they own it." Method security lets you put the check on the method itself.

### The idea in plain English
Enable global method security, then annotate methods:
- **`@PreAuthorize("...")`** — checked **before** the method runs. Throws `AccessDeniedException` if the SpEL expression is false.
- **`@PostAuthorize("...")`** — checked **after** the method runs. Useful when authorization depends on the *return value* (the post you just loaded).

You write a **SpEL** (Spring Expression Language) string with access to `authentication`, `principal`, method arguments (`#id`), and the return value (`returnObject`).

### Smallest working example
```java
@Configuration
@EnableMethodSecurity   // turns on @PreAuthorize / @PostAuthorize
class SecurityConfig { }

@Service
class PostService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteAny(Long id) { /* admin only */ }

    @PreAuthorize("#userId == authentication.principal.id")
    public Profile loadOwn(Long userId) { /* user can only load themselves */ }

    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    public Post getPost(Long id) { /* user can only see posts they own */ }
}
```

If the expression is false, Spring throws — never your code. Stays clean.

### Drill (2 min)
Why use `@PostAuthorize` instead of `@PreAuthorize` for "you can only see your own posts"? *(Hint: at pre-time you only know the `id`, not the owner. You'd have to load the post to check — that's exactly what the method already does. Use post-auth and check the return value once.)*

**Deep dive (later):** [spring-next/06-spring-security.md](../spring-next/06-spring-security.md)

---

## 4. MongoDB — Index Strategies

### Why this exists
Plain B-tree indexes get you started. Three specialized variants — **covering, partial, sparse** — solve real performance and storage problems.

### The idea in plain English
- **Covering index** — an index that contains *all* the fields your query needs, including the projected ones. Mongo answers from the index alone, never touching the document. Fastest reads possible.
- **Partial index** — index only documents matching a filter (e.g., `{ active: true }`). Smaller index, faster, often free for the inactive 90%.
- **Sparse index** — index only documents that *have* the field. Useful for optional fields (e.g., `email` on users where some don't have one).

**Analogy:** a regular index is a phonebook. A *covering* index is a phonebook that already lists the data you want next to the name (no need to flip back). A *partial* index is a phonebook for "members only." A *sparse* index is a phonebook that skips people without a phone.

### Smallest working example
```javascript
// Covering: query and projection only use these fields
db.users.createIndex({ email: 1, name: 1 });
db.users.find({ email: "k@x.com" }, { _id: 0, email: 1, name: 1 });
// .explain() will show "PROJECTION_COVERED" — no fetch step

// Partial: only active users
db.orders.createIndex(
  { customerId: 1 },
  { partialFilterExpression: { status: "ACTIVE" } }
);

// Sparse: only docs that have the field
db.users.createIndex({ phone: 1 }, { sparse: true });
```

### Drill (2 min)
You have an index `{ email: 1 }`. Query `find({email:"a@b"}, {email:1, name:1})`. Why is this *not* covered? *(Hint: the index has only `email`. To return `name`, Mongo must fetch the document. Add `name` to the index to make it covering.)*

**Deep dive (later):** [mongodb/04-indexes.md](../mongodb/04-indexes.md)

---

## 5. Postgres — Read Replicas & Failover

### Why this exists
Read-heavy apps can scale reads horizontally by adding **read replicas**. And if the primary dies, a replica should take over with minimal disruption — **failover**.

### The idea in plain English
- A **read replica** is a streaming-replication standby (see Day 14). It's read-only. You point heavy read traffic at it, leaving the primary for writes.
- **Failover** is promoting a standby to be the new primary when the old primary dies.
- **Replication lag** is unavoidable — the replica is a few milliseconds (or more) behind. Reading from a replica can show stale data.

**Analogy:** replicas are photocopies of the master ledger updated continuously. Heavy readers use a photocopy. If the master burns down, you officially crown a photocopy as the new master.

**Failover tools:** `pg_auto_failover`, Patroni, or cloud-managed (RDS Multi-AZ).

### Smallest working example (lag check)
```sql
-- Run on the replica
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- Promote a standby to primary (manual failover)
SELECT pg_promote();
```

In app code, separate read and write data sources:
```yaml
spring.datasource.primary.url=jdbc:postgresql://db-primary:5432/app
spring.datasource.replica.url=jdbc:postgresql://db-replica:5432/app
```

Use the primary for writes, replica for analytics and "eventually consistent" reads.

### Drill (2 min)
A user updates their profile then immediately re-reads it and sees old data. Likely cause? *(Hint: the write hit the primary but the read went to a replica that hasn't caught up. Either route post-write reads back to the primary for a few seconds (read-your-writes) or wait for sync.)*

**Deep dive (later):** [postgres/15-replication.md](../postgres/15-replication.md)

---

## 6. High-Level Design — Sharding Strategies

### Why this exists
One DB server has finite disk and CPU. **Sharding** splits one logical database into many physical ones, each holding part of the data. The question is *how* you split.

### The idea in plain English
- **Range sharding** — split by sorted value (user_id 1–1M on shard A, 1M–2M on shard B). Pros: range queries hit one shard. Cons: hot shard if new users get all the writes.
- **Hash sharding** — split by `hash(key) mod N`. Pros: even distribution. Cons: range queries scatter across all shards.
- **Geo sharding** — split by location (EU users on the EU shard). Pros: low latency for nearby users, helps with data residency laws. Cons: cross-region queries are slow.

**Analogy:** a library with three branches.
- Range: A–H here, I–P there, Q–Z over there. Easy to find "all books from S to T" (one branch). New "Z" book floods one branch.
- Hash: assign each book a random branch via lookup table. Even spread; can't ask "all books from S to T" without visiting all three.
- Geo: each branch only stocks books popular in that city. Fast locally, painful for tourists.

Real systems often combine: hash by user_id, but geo-pin per region.

### A minimal sketch
```
Range (by user_id):
   shard A: users 1 – 1,000,000
   shard B: users 1,000,001 – 2,000,000
   ...

Hash (by user_id):
   shard = hash(user_id) % N
   user 7 lives on shard hash(7) % 4 = 3

Geo (by region):
   eu-west:    users in EU
   us-east:    users in NA
   ap-south:   users in IN
```

For range/geo, you also need a **shard router** (or directory) so the app knows which shard owns which user.

### Drill (3 min)
You're building a global chat. Most messages are between people in the same country. Which strategy? *(Hint: geo-sharding. EU↔EU traffic stays on EU shards (fast); cross-country is a small minority. Compliance also benefits (GDPR data stays in EU).)*

**Deep dive (later):** [system-design/high-level-design/17-sharding.md](../system-design/high-level-design/17-sharding.md)

---

## 7. LLD — Vending Machine

### Why this exists
Vending Machine is the canonical state-machine LLD. Tests inventory, money handling, and (perfectly) the State pattern from Day 17.

### Core entities
- **Item** — code, name, price
- **Inventory** — `Map<Item, Integer>` count per item
- **MachineState** — `IDLE`, `HAS_MONEY`, `DISPENSING`
- **VendingMachine** — facade for `insertCoin`, `selectItem`, `cancel`, `dispense`

### A clean class sketch
```java
enum State { IDLE, HAS_MONEY, DISPENSING }

class VendingMachine {
    private State state = State.IDLE;
    private int credit = 0;
    private Map<String, Item> items;
    private Map<String, Integer> stock;

    public void insertCoin(int cents) {
        if (state == State.DISPENSING) throw new IllegalStateException();
        credit += cents;
        state = State.HAS_MONEY;
    }

    public Item select(String code) {
        if (state != State.HAS_MONEY) throw new IllegalStateException("insert coin first");
        Item item = items.get(code);
        if (stock.getOrDefault(code, 0) == 0) throw new OutOfStockException();
        if (credit < item.price) throw new InsufficientFundsException();
        state = State.DISPENSING;
        return item;
    }

    public Item dispense() {
        // ... reduce stock, compute change, reset credit
        Item dispensed = /* from select() */;
        state = State.IDLE;
        return dispensed;
    }

    public int cancel() {
        int refund = credit;
        credit = 0;
        state = State.IDLE;
        return refund;
    }
}
```

The `state` field guards every operation. Try to `dispense` before `select` → exception. That's the state machine in a few lines. In a richer design, you'd extract `IdleState`, `HasMoneyState`, etc. (Day 17 pattern).

### Drill (3 min)
The user inserts $1, selects an item that costs $0.75. They want $0.25 change. Which collection holds the cash levels? *(Hint: a `Map<Coin, Integer>` of denominations on hand. Computing change is itself a small DP/greedy: prefer fewest coins; if you can't make exact change, reject the sale or refund.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Greedy

### What "greedy" means
At each step, take the choice that looks best **right now**, without looking back or planning ahead. Sometimes that gives the global optimum; sometimes it doesn't. The art is recognizing when it does.

### Problem 1: Activity Selection
Given `n` activities with start and end times, pick the **maximum number** of non-overlapping activities you can attend.

**Greedy:** sort by **end time**. Pick the first activity. Then pick the next one that starts ≥ the previous end. Repeat.

```javascript
function maxActivities(intervals) {
  intervals.sort((a, b) => a.end - b.end);
  let count = 0, lastEnd = -Infinity;
  for (const { start, end } of intervals) {
    if (start >= lastEnd) {   // doesn't overlap
      count++;
      lastEnd = end;
    }
  }
  return count;
}
```

**Why it works:** picking the activity that *frees you up earliest* leaves the most room for future picks. Mathematically proven optimal.

### Problem 2: Jump Game
Given `nums = [2, 3, 1, 1, 4]`, where each `nums[i]` is the max jump length from index `i`, can you reach the last index from index 0?

**Greedy:** track the farthest index reachable as you walk. If you ever stand on an index beyond what's reachable, return false.

```javascript
function canJump(nums) {
  let maxReach = 0;
  for (let i = 0; i < nums.length; i++) {
    if (i > maxReach) return false;        // can't even get here
    maxReach = Math.max(maxReach, i + nums[i]);
  }
  return true;
}
```

**Time:** O(n). Single pass.

### When greedy fails
Coin change with `[1, 5, 6, 9]` and target `11`. Greedy picks 9 + (then needs 2) = 9 + 1 + 1 (3 coins). Optimal is 5 + 6 (2 coins). Greedy is wrong here — you need DP. Always sanity-check whether the "look only ahead" approach truly gives the optimum for your problem.

### Drill (3 min)
For Jump Game with `nums = [3, 2, 1, 0, 4]`, the answer is false. Trace `maxReach`. *(Hint: i=0 maxReach=3; i=1 maxReach=3; i=2 maxReach=3; i=3 maxReach=3 (3+0=3); i=4 → 4 > 3 → return false.)*

**Deep dive (later):** [dsa/ds-js/greedy.md](../dsa/ds-js/greedy.md)

---

## 9. Design Pattern — Chain of Responsibility

### Intent
**Pass a request along a chain of handlers**; each one decides whether to handle or to pass on. The sender doesn't know which one will handle it.

### When you'd use it
- HTTP middleware (Express, Spring's filter chain) — each filter handles or forwards
- Logging frameworks (level filter → enrichment → appender)
- Approval workflows (manager → director → VP)

### The simplest version (Java)
```java
abstract class Approver {
    private Approver next;
    public Approver chain(Approver n) { this.next = n; return n; }

    public final void handle(Request r) {
        if (canHandle(r)) approve(r);
        else if (next != null) next.handle(r);
        else reject(r);
    }
    protected abstract boolean canHandle(Request r);
    protected abstract void approve(Request r);
    protected void reject(Request r) { System.out.println("nobody approved"); }
}

class Manager extends Approver {
    protected boolean canHandle(Request r) { return r.amount <= 1000; }
    protected void approve(Request r)      { System.out.println("manager approved " + r.amount); }
}
class Director extends Approver {
    protected boolean canHandle(Request r) { return r.amount <= 10_000; }
    protected void approve(Request r)      { System.out.println("director approved " + r.amount); }
}

var chain = new Manager();
chain.chain(new Director());
chain.handle(new Request(500));     // manager approves
chain.handle(new Request(5_000));   // director approves
chain.handle(new Request(50_000));  // nobody approved
```

Add new approvers without touching old code. Each handler has one rule.

### Drill (1 min)
Day 16's Spring Security filter chain — which pattern? *(Hint: Chain of Responsibility. Each filter either short-circuits the response or calls `chain.doFilter(...)` to pass to the next.)*

**Deep dive (later):** [design-patterns/common/behavioral/chain-of-responsibility.md](../design-patterns/common/behavioral/chain-of-responsibility.md)

---

## 10. DevOps — Terraform

### Why this exists
Clicking around an AWS console to provision infrastructure is slow, error-prone, and impossible to review or roll back. **Infrastructure as Code (IaC)** treats infra like software — versioned, reviewed, repeatable. Terraform is the most common tool.

### The idea in plain English
You write `.tf` files describing the **desired state** of your infrastructure (a server, a DB, a load balancer). You run `terraform apply`. Terraform compares desired state to current state and figures out what to create/update/delete.

Four core concepts:
- **Provider** — the plugin that talks to a specific cloud (AWS, GCP, Azure, etc.)
- **Resource** — a thing managed by the provider (`aws_instance`, `aws_s3_bucket`)
- **State** — Terraform's record of what currently exists. Stored in a `tfstate` file (locally) or remotely (S3 + DynamoDB for locking). Treat it like a precious database — don't edit by hand.
- **Module** — a reusable bundle of resources (a "VPC module" used in every environment)

**Analogy:** Terraform is a recipe + an inventory of what's in your kitchen. You write the recipe ("I want a sourdough loaf"). Terraform looks at your kitchen ("you have flour, you don't have a loaf"). It bakes one. Next run: "already there, nothing to do."

### Smallest working example
```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "logs" {
  bucket = "myapp-logs-2026"
}

resource "aws_s3_bucket_versioning" "logs" {
  bucket = aws_s3_bucket.logs.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

Run:
```bash
terraform init    # download providers
terraform plan    # preview what will change
terraform apply   # do it
```

Commit the `.tf` files to git. Review changes in pull requests. **Never** commit the `tfstate` file with secrets — use a remote backend.

### Drill (2 min)
Two engineers run `terraform apply` at the same time. What can go wrong? *(Hint: race on the state file — both think they own the latest version, one overwrites the other's record of changes. Solution: remote state with a lock (S3 + DynamoDB, or Terraform Cloud) so only one apply at a time.)*

**Deep dive (later):** [devops/13-iac-terraform.md](../devops/13-iac-terraform.md)

---

## End-of-day checklist

- [ ] Angular: I can describe what an effect does and why reducers stay pure
- [ ] Node.js: I added `helmet` and `express-rate-limit` to an app
- [ ] Spring: I used `@PreAuthorize` and `@PostAuthorize` and know when each fits
- [ ] MongoDB: I can pick between covering, partial, and sparse for a scenario
- [ ] Postgres: I can spot a "read-your-writes" issue caused by replica lag
- [ ] HLD: I can compare range vs hash vs geo sharding for a given workload
- [ ] LLD: I can model Vending Machine with a state field and guarded methods
- [ ] DSA: I solved Jump Game with the greedy max-reach trick
- [ ] DP: I can name Chain of Responsibility examples (filter chains, approvals)
- [ ] DevOps: I know what state file locking is and why it matters

**If you remember just one thing today:** **declare desired state, let the system reconcile.** That's NgRx reducers, Terraform, Kubernetes, even DP tables — the pattern repeats at every scale.

**Tomorrow:** we cross into the final third of the prep — Web Vitals, queues, microservices, full-text search, GIN, distributed transactions, payments, graphs, Visitor, GitOps.
