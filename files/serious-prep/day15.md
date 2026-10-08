# Day 15 — Re-renders, rules, rollbacks, restores

> **Today's goal:** learn what triggers Angular to re-paint, how to refuse bad input, how to bundle DB writes safely, and how to back up a database without panic.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Change Detection (Default vs OnPush) | 15m |
| 2 | Node.js | Validation (Zod or Joi) | 15m |
| 3 | Spring Boot | @Transactional (propagation, rollbackFor) | 15m |
| 4 | MongoDB | Backup & Recovery (mongodump, oplog) | 10m |
| 5 | Postgres | Partitioning (range/list/hash) | 10m |
| 6 | HLD | Notification System Design (push/email/SMS) | 12m |
| 7 | LLD | Splitwise (expense splitting) | 12m |
| 8 | DSA | Trees: Lowest Common Ancestor (BST variant) | 15m |
| 9 | Design Pattern | Command | 8m |
| 10 | DevOps | Logging Best Practices (structured, levels, correlation IDs) | 10m |

---

## 1. Angular — Change Detection

### Why this exists
Angular's job: keep what's on the screen in sync with your data. **Change detection** is the recurring check that decides "did anything change? do I need to re-paint?"

### The idea in plain English
Imagine a museum guard walking through every room every few minutes asking, "did anyone move a painting?" That's **Default** change detection — Angular checks every component, every binding, on every event.

For small apps that's fine. For big apps with thousands of bindings, you want to tell Angular: "only re-check this component when something *that affects it* changes." That's **OnPush** — re-check only when:
1. An `@Input()` reference changes (new object, not just a mutated field)
2. An event fires from inside this component (click, etc.)
3. An observable bound with `async` pipe emits

**Analogy:** Default = guard checks every room every cycle. OnPush = guard only checks rooms whose door alarm went off.

### Smallest working example
```typescript
import { Component, ChangeDetectionStrategy, Input } from '@angular/core';

@Component({
  selector: 'app-user-card',
  changeDetection: ChangeDetectionStrategy.OnPush,    // opt in
  template: `<div>{{ user.name }}</div>`,
})
export class UserCardComponent {
  @Input() user!: { name: string };
}
```

**Gotcha to remember:** if a parent does `this.user.name = 'new'` (mutates), OnPush won't see it. You need a new reference: `this.user = { ...this.user, name: 'new' }`.

### Drill (2 min)
Your OnPush component shows old data after an update. What did the parent likely do wrong? *(Hint: mutated the input object in place. OnPush compares object references — same reference → no re-render. Create a new object.)*

**Deep dive (later):** [angular/change-detection.md](../angular/change-detection.md)

---

## 2. Node.js — Validation

### Why this exists
Never trust input. Without validation, a single bad request can crash your server or corrupt your database. A good validator rejects bad input at the door with a clear error.

### The idea in plain English
Validation answers: "Is this JSON exactly the shape I expect?" Two popular tools in Node:
- **Zod** — TypeScript-first, you get a typed result for free
- **Joi** — older, framework-agnostic, similar API

Both work the same way: you write a **schema** (a description of the shape), then `parse(input)` returns either valid data or a clear error.

**Analogy:** a bouncer at a club with a checklist — age ≥ 18, ID present, name not blank. Anyone failing the checklist doesn't get in.

### Smallest working example
```javascript
const { z } = require('zod');

const CreateUserSchema = z.object({
  email: z.string().email(),
  age:   z.number().int().min(18),
  name:  z.string().min(1),
});

// In your Express handler:
app.post('/users', (req, res) => {
  const parsed = CreateUserSchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ errors: parsed.error.flatten() });
  }
  // parsed.data is now typed and safe to use
  createUser(parsed.data);
  res.status(201).end();
});
```

`safeParse` returns `{ success, data | error }` — no try/catch needed.

### Drill (2 min)
Why validate *before* your business logic, not inside it? *(Hint: clear separation. Business logic stays focused on rules, not "is this field a string?" Errors are also cheaper to return at the edge.)*

**Deep dive (later):** [nodejs/15-validation.md](../nodejs/15-validation.md)

---

## 3. Spring Boot — @Transactional

### Why this exists
Some business actions involve **multiple writes** that must all succeed or all fail (transfer money: debit A, credit B). A **database transaction** is the unit of "all-or-nothing." `@Transactional` wires that into Spring with one annotation.

### The idea in plain English
Wrap a method in `@Transactional` and Spring opens a DB transaction before it runs and commits when it returns. If a `RuntimeException` is thrown, Spring **rolls back** automatically — the DB looks as if nothing happened.

**Analogy:** a bank teller's "in-progress" tray. They process the whole transfer; if anything goes wrong before they file it, they sweep the tray into the bin. The ledger is never half-updated.

Two settings you'll be asked about:
- **propagation** — what to do if a transaction is already running. `REQUIRED` (default) joins it; `REQUIRES_NEW` starts a fresh one (suspending the outer).
- **rollbackFor** — by default Spring only rolls back on `RuntimeException`. Checked exceptions don't roll back unless you say so.

### Smallest working example
```java
@Service
public class TransferService {

    @Transactional(rollbackFor = Exception.class)
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        accountRepo.debit(fromId, amount);
        accountRepo.credit(toId, amount);
        // If either line throws — Spring rolls back both.
    }
}
```

The two `repo` calls are now atomic from the DB's view.

### Drill (2 min)
A method has `@Transactional` and throws `IOException` (checked). Does Spring roll back by default? *(Hint: no — default is rollback on `RuntimeException` only. Add `rollbackFor = IOException.class` or `rollbackFor = Exception.class`.)*

**Deep dive (later):** [spring-detailed/PART-16-transactions-proxies-aop.md](../spring-detailed/PART-16-transactions-proxies-aop.md)

---

## 4. MongoDB — Backup & Recovery

### Why this exists
Disks fail. Bugs delete the wrong rows. Without backups, "rm -rf" or a `db.dropDatabase()` ends your week. Mongo gives you two tools.

### The idea in plain English
- **mongodump / mongorestore** — like `pg_dump`. Snapshots the data as BSON files. Easy, but it's a *point-in-time* snapshot — you lose anything written after the dump.
- **Oplog** — the operation log in replica sets. Every write goes here as a record. You can replay the oplog on top of a dump to roll forward to a specific second.

**Analogy:** mongodump is a photograph of your bookshelf. The oplog is the diary of every book you added/removed since. Photo + diary lets you reconstruct any moment in time.

### Smallest working example
```bash
# Full dump of one database
mongodump --uri="mongodb://localhost:27017" --db=ordersdb --out=/backup/2026-05-20

# Restore later
mongorestore --uri="mongodb://localhost:27017" --db=ordersdb /backup/2026-05-20/ordersdb

# Production: schedule via cron + ship the dump to S3
```

For point-in-time recovery, run a replica set and use `--oplog` on dump + `--oplogReplay` on restore.

### Drill (2 min)
You took a dump at 02:00 AM. A bug deleted important data at 03:30 AM. Can you recover the 02:00–03:30 writes? *(Hint: only if you have the oplog from that window. Plain mongodump alone gets you back to 02:00; oplog replay covers the gap up to just before the bad delete.)*

**Deep dive (later):** [mongodb/14-backup-and-recovery.md](../mongodb/14-backup-and-recovery.md)

---

## 5. Postgres — Partitioning

### Why this exists
When a table grows huge (100M rows of events), every query and every index gets slow. **Partitioning** splits one logical table into many physical chunks so queries only scan the relevant chunk.

### The idea in plain English
Imagine your closet had 10 years of clothes in one pile. To find a winter jacket you'd dig through everything. Partition by year and season, and "winter 2024" is one labeled bin.

Three strategies:
- **Range** — by a sortable column (date ranges: Jan, Feb, Mar...). Most common for time-series.
- **List** — by a discrete value (`country IN ('IN', 'US')`).
- **Hash** — by a hash of a column. Useful when no natural range exists and you just want even distribution.

### Smallest working example
```sql
CREATE TABLE events (
    id        BIGSERIAL,
    user_id   BIGINT,
    occurred  TIMESTAMPTZ NOT NULL,
    payload   JSONB
) PARTITION BY RANGE (occurred);

-- One child partition per month
CREATE TABLE events_2026_05
  PARTITION OF events
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE TABLE events_2026_06
  PARTITION OF events
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
```

A query `WHERE occurred >= '2026-05-15'` scans only `events_2026_05` (called *partition pruning*).

### Drill (2 min)
Your `events` table has 5 years of data and most queries ask for "last 7 days." Which strategy? *(Hint: range partition on `occurred` by month or week. Old months sit idle on disk; recent queries scan a tiny range. Bonus: dropping old data is instant — just `DROP TABLE events_2020_01`.)*

**Deep dive (later):** [postgres/16-partitioning.md](../postgres/16-partitioning.md)

---

## 6. High-Level Design — Notification System

### Why this exists
Almost every app sends notifications: "Your order shipped," "New message," "Suspicious login." Doing this well at scale needs more thought than "just call Twilio in your handler."

### The idea in plain English
Three channels to support: **push** (mobile/web), **email**, **SMS**. Each has different rate limits, latencies, costs, and providers. You don't want your order service to know about any of them.

**Core flow:**
1. App publishes a logical event (`order.shipped`) to a message queue.
2. A **notification service** consumes the event, decides which channels apply (based on user preferences), and renders a template.
3. Channel-specific workers (push worker, email worker, SMS worker) call the providers.
4. Failures go to a **retry queue** with exponential backoff; permanent failures to a dead-letter queue.

### A minimal sketch
```
[Order Service] → publishes "order.shipped" event
        │
        ▼
   [Kafka topic]
        │
        ▼
[Notification Orchestrator] — loads user prefs, picks channels
        │   │   │
        ▼   ▼   ▼
   [Push] [Email] [SMS]   workers → FCM / SES / Twilio
```

User preferences ("don't SMS me at night") live in the orchestrator, not the channel workers — keep workers dumb.

### Drill (3 min)
A user disabled email but enabled push. Where in your design do you enforce that? *(Hint: in the orchestrator before fan-out — it's the only thing that knows about user preferences. Channel workers blindly send. Decoupling = each box has one job.)*

**Deep dive (later):** [system-design/high-level-design/13-notification-system.md](../system-design/high-level-design/13-notification-system.md)

---

## 7. LLD — Splitwise

### Why this exists
Splitwise interviews test how you model relationships and money — easy to get wrong, easy to over-engineer.

### Core entities
- **User** — id, name
- **Group** — has many `User`s
- **Expense** — who paid, total amount, list of `Split`s
- **Split** — for one user, how much they owe (or are owed)
- **BalanceSheet** — net amount each user owes each other user

### A clean class sketch
```java
enum SplitType { EQUAL, EXACT, PERCENTAGE }

class Split {
    User user;
    BigDecimal amount;        // signed: + means user owes payer, - means payer owes user
}

class Expense {
    User paidBy;
    BigDecimal total;
    SplitType type;
    List<Split> splits;
}

class BalanceSheet {
    // owes[A][B] = how much A owes B
    Map<User, Map<User, BigDecimal>> owes = new HashMap<>();

    void apply(Expense e) {
        for (Split s : e.splits) {
            if (s.user.equals(e.paidBy)) continue;
            owes.computeIfAbsent(s.user, k -> new HashMap<>())
                .merge(e.paidBy, s.amount, BigDecimal::add);
        }
    }
}
```

Always use `BigDecimal` for money. `double` will betray you on rounding.

### Drill (3 min)
Alice pays $90, split equally among 3 (Alice, Bob, Charlie). What does each row of the balance sheet look like after `apply`? *(Hint: Bob owes Alice $30; Charlie owes Alice $30. Alice's own split cancels out — she doesn't owe herself.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Lowest Common Ancestor (BST)

### The problem
Given a Binary Search Tree (BST) and two nodes `p` and `q`, find the lowest (deepest) ancestor that has both in its subtree.

```
        6
       / \
      2   8
     / \ / \
    0  4 7  9
```
LCA of 2 and 8 → 6. LCA of 0 and 4 → 2.

### The BST property that saves us
In a BST: left subtree < node < right subtree. So if both `p` and `q` are *less than* the current node, the LCA is in the left subtree. If both are *greater*, it's in the right. Otherwise, the current node *is* the LCA (the split point).

### The implementation
```javascript
function lowestCommonAncestor(root, p, q) {
  let node = root;
  while (node) {
    if (p.val < node.val && q.val < node.val) {
      node = node.left;             // both on the left
    } else if (p.val > node.val && q.val > node.val) {
      node = node.right;            // both on the right
    } else {
      return node;                  // split point — this is the LCA
    }
  }
}
```

**Time:** O(h) where h is the tree height. **Space:** O(1) — no recursion needed.

### Why it works
The first time `p` and `q` are on different sides of a node (or one equals the node), that node is the deepest split — exactly the LCA.

### Drill (3 min)
For the tree above, trace LCA(0, 4) step by step. *(You should see: at 6, both < 6 → go left to 2. At 2, 0 < 2 but 4 > 2 → split! return 2.)*

**Deep dive (later):** [dsa/ds-js/trees.md](../dsa/ds-js/trees.md) · [dsa/ds-java/trees.md](../dsa/ds-java/trees.md)

---

## 9. Design Pattern — Command

### Intent
**Wrap a request as an object** so it can be parameterized, queued, undone, or logged.

### When you'd use it
- Undo/redo in editors (each user action is a Command)
- Job queues (each job is a Command waiting to run)
- Macro recording (a list of Commands replayed in order)

### The simplest version (Java)
```java
interface Command {
    void execute();
    void undo();
}

class AddTextCommand implements Command {
    private final Document doc;
    private final String text;
    AddTextCommand(Document doc, String text) { this.doc = doc; this.text = text; }
    public void execute() { doc.append(text); }
    public void undo()    { doc.deleteLast(text.length()); }
}

// History
Deque<Command> history = new ArrayDeque<>();
Command c = new AddTextCommand(doc, "hello");
c.execute();
history.push(c);
// Ctrl+Z:
history.pop().undo();
```

The caller (the editor) doesn't know what the command *does* — only that it can be executed and undone.

### Drill (1 min)
Why does the Command object hold a reference to the `Document`? *(Hint: so `undo()` knows what to undo on. Each command is self-contained — it carries everything it needs.)*

**Deep dive (later):** [design-patterns/common/behavioral/command.md](../design-patterns/common/behavioral/command.md)

---

## 10. DevOps — Logging Best Practices

### Why this exists
Logs are the cheapest debugging tool you have. Done well, they pinpoint bugs in seconds. Done badly, they're a swamp of `console.log("here")`.

### The idea in plain English
Three rules that turn logs from junk to gold:

1. **Structured logs** — log JSON, not free text. `{"level":"error","userId":42,"err":"NOT_FOUND"}` is searchable. `"User 42 not found"` is not.
2. **Log levels** — `debug` (chatty, off in prod), `info` (key events), `warn` (something odd but recoverable), `error` (a request failed), `fatal` (process is dying). Set the *minimum* level in prod to `info` to keep volume sane.
3. **Correlation ID** — generate a `request-id` at the edge and put it on every log line for that request. When a request fails, you can grep one ID and see the whole story across services.

**Analogy:** structured logs are filing cabinets with labeled folders; free-text logs are a pile on the floor. You can grep the pile, but you can't aggregate it.

### Smallest working example (Node.js with pino)
```javascript
const pino = require('pino')();

// Express middleware that attaches a correlation id and a child logger
app.use((req, res, next) => {
  const requestId = req.headers['x-request-id'] || crypto.randomUUID();
  req.log = pino.child({ requestId });
  res.setHeader('x-request-id', requestId);
  next();
});

app.get('/order/:id', (req, res) => {
  req.log.info({ orderId: req.params.id }, 'fetching order');
  // ... if you log inside services, pass req.log down or use AsyncLocalStorage
});
```

Now every log line for one request carries the same `requestId`. Ship logs to ELK / Loki / Datadog and search by it.

### Drill (2 min)
A user complains "my request was slow." You have 20 services and 100 GB/day of logs. How do you find their request? *(Hint: ask them for the `x-request-id` from the response — grep that ID across all log streams to get the full trace in seconds.)*

**Deep dive (later):** [devops/19-logging-best-practices.md](../devops/19-logging-best-practices.md)

---

## End-of-day checklist

- [ ] Angular: I can explain when OnPush re-renders and when it doesn't
- [ ] Node.js: I wrote one Zod schema and used `safeParse`
- [ ] Spring: I can explain why a checked exception doesn't roll back by default
- [ ] MongoDB: I can describe mongodump + oplog for point-in-time recovery
- [ ] Postgres: I can choose range vs list vs hash partitioning for a use case
- [ ] HLD: I can sketch the order-event → orchestrator → channel-worker flow
- [ ] LLD: I can model Splitwise expenses and BigDecimal balances
- [ ] DSA: I can implement BST LCA iteratively in O(h)
- [ ] DP: I can describe Command pattern with an undo example
- [ ] DevOps: I know what structured logs, levels, and correlation IDs each give me

**If you remember just one thing today:** every layer benefits from "be lazy when it's safe" — OnPush, partition pruning, log levels in prod, transactions only when needed. Avoid unnecessary work.

**Tomorrow:** Signals, DB drivers, Spring Security filter chain, change streams, vacuum, search engines, Snake, DP intro, Iterator, Prometheus.
