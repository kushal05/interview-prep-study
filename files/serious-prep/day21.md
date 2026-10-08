# Day 21 — Split the load: lazy routes, clusters, profiles, and payments

> **Today's goal:** load only what the user needs, run more Node processes, swap config per environment, and sketch a payment system that won't double-charge.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Lazy loading | 15m |
| 2 | Node.js | Clustering & PM2 | 15m |
| 3 | Spring Boot | Profiles & configuration | 15m |
| 4 | MongoDB | Migrations | 10m |
| 5 | Postgres | Roles & privileges | 10m |
| 6 | HLD | Payment system | 12m |
| 7 | LLD | Hotel booking system | 12m |
| 8 | DSA | Trie (prefix tree) | 15m |
| 9 | Design Pattern | Memento | 8m |
| 10 | DevOps | AWS basics | 10m |

---

## 1. Angular — Lazy Loading

### Why this exists
Every line of TypeScript your app ships in `main.js` is a line the browser must download before showing the first pixel. An admin panel that 1% of users see has no business slowing down the other 99%. **Lazy loading** ships it as a separate file that only downloads when the user visits `/admin`.

### The idea in plain English
A bookshop with everything on one giant shelf takes forever to enter. Move "Cookbooks" to a back room — customers who want bread go there; everyone else gets in faster. That's lazy loading: each feature area becomes its own *chunk* (a separate JS file). The router downloads the chunk only when needed.

The Angular CLI does the splitting — you just point a route at a dynamic `import()` and the build tool produces the chunk.

### Smallest working example
```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', component: HomeComponent },       // in main bundle
  {
    path: 'admin',
    loadChildren: () =>
      import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
  },
];

// admin/admin.routes.ts — only downloaded when /admin is visited
export const ADMIN_ROUTES: Routes = [
  { path: '', component: AdminDashboardComponent },
  { path: 'users', component: AdminUsersComponent },
];
```

Run `ng build` — you'll see a separate `admin-*.js` chunk. The main bundle is smaller.

### Drill (2 min)
Your home page is 600KB and feels slow. Where do you start? *(Hint: run `ng build --stats-json` + a bundle analyzer. Find big features (charts library? admin panel?). Move them behind `loadChildren` or `loadComponent`.)*

**Deep dive (later):** [angular/lazy-loading.md](../angular/lazy-loading.md)

---

## 2. Node.js — Clustering & PM2

### Why this exists
Node runs JavaScript on a **single thread**. A server with 8 CPU cores using one Node process uses ~12% of the machine. **Clustering** runs one Node process per core; they all listen on the same port and the OS load-balances.

### The idea in plain English
Imagine a bakery with 8 ovens but only one baker. You can buy 7 more bakers (cluster) so all ovens run. Each baker is independent — they don't share dough (memory). PM2 is the bakery manager: it hires bakers, restarts the dead ones, watches their output.

There's also `worker_threads` for *CPU-heavy* work inside one process — they *do* share memory via `SharedArrayBuffer`. Use clustering for HTTP scaling, workers for heavy computation that would block the event loop.

### Smallest working example
```javascript
// server.js — same code runs in every worker
const http = require('http');
http.createServer((req, res) => res.end(`pid ${process.pid}`)).listen(3000);
```

Run it with PM2 in cluster mode:
```bash
pm2 start server.js -i max         # one process per CPU
pm2 list                           # see all workers
pm2 logs                           # tail their output
pm2 reload server                  # zero-downtime reload
```

Each request lands on a random worker; the response shows a different `pid` each time.

### Drill (2 min)
You store user sessions in a JavaScript `Map` inside the Node process. After enabling cluster mode, users randomly get logged out. Why? *(Hint: each worker has its own memory. The session you stored in worker #3 isn't visible to worker #5. Move sessions to Redis or a database.)*

**Deep dive (later):** [nodejs/20-performance-and-clustering.md](../nodejs/20-performance-and-clustering.md)

---

## 3. Spring Boot — Profiles & Configuration

### Why this exists
Dev uses an in-memory DB; staging uses a small Postgres; prod uses a beefy Postgres in another region. You don't want to edit code or rebuild jars for each. Spring **profiles** swap configuration by environment.

### The idea in plain English
A profile is a named bundle of settings. Spring loads `application.yml` always, plus `application-{profile}.yml` if the profile is active. You activate a profile with `--spring.profiles.active=prod` or the env var `SPRING_PROFILES_ACTIVE=prod`. Same jar, three behaviors.

You can also tag whole beans with `@Profile("prod")` — they exist only when that profile is active. Useful for swapping a fake email sender in dev for a real one in prod.

### Smallest working example
```yaml
# application.yml — common defaults
server.port: 8080

# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb            # in-memory, no setup
logging.level.root: DEBUG

# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://db.prod.internal:5432/app
    username: ${DB_USER}               # from env var
    password: ${DB_PASS}
logging.level.root: WARN
```

```java
@Service
@Profile("dev")
class FakeEmailSender implements EmailSender { /* prints to console */ }

@Service
@Profile("prod")
class SesEmailSender implements EmailSender { /* calls AWS SES */ }
```

```bash
java -jar app.jar --spring.profiles.active=prod
```

### Drill (2 min)
You added `application-prod.yml` with a new DB URL, but the app still uses the dev DB in production. Why? *(Hint: the prod profile isn't active. Set `SPRING_PROFILES_ACTIVE=prod` in the container/server env, or pass `--spring.profiles.active=prod` on startup.)*

**Deep dive (later):** [spring-next/03-spring-boot-fundamentals.md](../spring-next/03-spring-boot-fundamentals.md) (profiles and configuration)

---

## 4. MongoDB — Migrations

### Why this exists
Schema-less doesn't mean schema-free. Real apps have a shape (`user.firstName` exists). When you rename it to `name.first`, you need to update existing documents — that's a migration. Postgres has built-in DDL; Mongo doesn't, so you write a script.

### The idea in plain English
A migration is a one-way script that transforms data from "schema version N" to "version N+1." Store the version number somewhere (a `migrations` collection) so you don't re-run scripts. Tools like `migrate-mongo` automate this.

Two common patterns:
- **Eager** — run a script that updates all documents in one shot. Simple but locks/loads the DB.
- **Lazy** — when your code reads an old document, transform it on the fly and write back. No batch downtime; works while you ship.

### Smallest working example
```javascript
// migrations/20260520-rename-firstName.js
module.exports = {
  async up(db) {
    await db.collection('users').updateMany(
      { firstName: { $exists: true } },
      [{ $set: { 'name.first': '$firstName' } }, { $unset: 'firstName' }]
    );
  },
  async down(db) {
    await db.collection('users').updateMany(
      { 'name.first': { $exists: true } },
      [{ $set: { firstName: '$name.first' } }, { $unset: 'name.first' }]
    );
  }
};
```

```bash
npx migrate-mongo up        # apply pending migrations
npx migrate-mongo status    # see what ran when
```

`up` moves forward; `down` reverts. Always test `down` in staging.

### Drill (2 min)
A migration takes 6 hours on production data and would lock writes. What's the safer approach? *(Hint: don't migrate in one shot. Make your code handle **both** shapes during a transition. Run a slow background script in batches. When 100% migrated, remove the old-shape code path.)*

**Deep dive (later):** [mongodb/13-migrations-and-schema-evolution.md](../mongodb/13-migrations-and-schema-evolution.md)

---

## 5. Postgres — Roles & Privileges

### Why this exists
Your app shouldn't run as the same user that can `DROP TABLE`. The reporting tool shouldn't be able to `UPDATE`. Postgres ships with a permission system: **roles** (users or groups), **privileges** (what they can do), and **GRANT/REVOKE** to assign them.

### The idea in plain English
A role is a username. Two flavors:
- **Login roles** — actual users that connect.
- **Group roles** — bundles of privileges; you `GRANT` the group to login roles so they inherit.

Each table can be granted `SELECT`, `INSERT`, `UPDATE`, `DELETE` independently. The rule of thumb: **least privilege** — give each connection only what it needs.

### Smallest working example
```sql
-- Create a group role with read-only access
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE app TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO readonly;   -- future tables too

-- Create a login user and put them in the group
CREATE USER analytics_bot WITH PASSWORD 'secret';
GRANT readonly TO analytics_bot;

-- App user with read/write but no DDL
CREATE USER app_user WITH PASSWORD 'app-pass';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

To revoke: `REVOKE SELECT ON users FROM analytics_bot;`.

### Drill (2 min)
You granted `SELECT ON ALL TABLES` to `readonly`. A week later you create a new table. Can `readonly` read it? *(Hint: not unless you use `ALTER DEFAULT PRIVILEGES ... GRANT SELECT ON TABLES TO readonly;`. Otherwise you must re-grant after every new table.)*

**Deep dive (later):** [postgres/14-roles-and-privileges.md](../postgres/14-roles-and-privileges.md)

---

## 6. High-Level Design — Payment System

### Why this exists
Money is the highest-stakes domain. A duplicate charge, lost transaction, or wrong amount becomes a refund ticket — or a lawsuit. Two ideas dominate payment design: **idempotency keys** and **ledgers**.

### The idea in plain English
- **Idempotency key** = "do this exactly once, even if I send the request three times." The client generates a UUID once. The server stores `(idempotency_key → response)`. If the same key arrives again, return the cached response instead of charging again. This survives network retries, mobile drops, double clicks.
- **Ledger** = an append-only log of money movements. Never UPDATE a balance directly. Insert a row that says "credit account A by $100, debit account B by $100." Balance is the sum of rows. This makes audits possible — you can replay history.

### The big picture
```
Client ── idempotency_key, amount ──▶  Payment Service
                                       │
                                       ├─ check key in `idempotency` table
                                       │   (hit → return prior response)
                                       │
                                       ├─ insert ledger entries (txn)
                                       │
                                       └─ call PSP (Stripe), record result
```

Five things to mention in an interview:
1. Idempotency key with a TTL (24h is common).
2. Double-entry ledger — every payment is two rows summing to zero.
3. Reconciliation — daily job compares your ledger with the PSP's reports.
4. Webhooks from the PSP can arrive out of order — process by `event.created_at`.
5. PCI-DSS — never store raw card numbers; let the PSP handle that.

### Drill (2 min)
A user double-clicks "Pay." Your service receives two identical requests one millisecond apart. How do you prevent two charges? *(Hint: both requests carry the same idempotency key. The first one inserts a row and proceeds. The second tries to insert and gets a unique-constraint error → returns the in-progress or completed response of the first.)*

**Deep dive (later):** [system-design/high-level-design/21-payments.md](../system-design/high-level-design/21-payments.md)

---

## 7. Low-Level Design — Hotel Booking

### The classes (start with nouns from the problem)
- `Hotel` — name, address, list of rooms.
- `Room` — number, type (Standard/Deluxe/Suite), capacity, price.
- `RoomAvailability` — per-room, per-night status.
- `Booking` — user, room, check-in, check-out, status.
- `User`, `Payment`.

### The hard parts
1. **Search** — "rooms in Mumbai, June 5–7, fits 2 people, < ₹5000." Index by `(city, capacity)` then filter by availability. A separate `(room_id, date)` table with one row per night is the most common availability representation.

2. **Concurrency** — two users book the last room at once. Solution:
   ```sql
   BEGIN;
   SELECT 1 FROM room_availability
     WHERE room_id = 42 AND date BETWEEN '2026-06-05' AND '2026-06-06'
     AND status = 'AVAILABLE'
     FOR UPDATE;   -- lock the rows
   UPDATE room_availability SET status = 'BOOKED' WHERE ...;
   INSERT INTO bookings (...) VALUES (...);
   COMMIT;
   ```
   The second user's `FOR UPDATE` waits, then sees the rows are no longer `AVAILABLE`.

3. **Cancellation** — refund logic by policy (full refund 24h before; 50% after). Keep the cancellation as a status change with timestamp, not a delete.

### Smallest skeleton
```java
class BookingService {
    @Transactional
    Booking book(long userId, long roomId, LocalDate in, LocalDate out) {
        var nights = availabilityRepo.findAndLock(roomId, in, out);
        if (nights.stream().anyMatch(n -> n.status() != AVAILABLE))
            throw new RoomTakenException();
        nights.forEach(n -> n.markBooked());
        return bookingRepo.save(new Booking(userId, roomId, in, out, CONFIRMED));
    }
}
```

### Drill (2 min)
What's the simplest way to model "room X is available from June 1 to June 10"? *(Hint: one row per night in a `room_availability(room_id, date, status)` table. Simple to query; easy to lock per night.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Trie (Prefix Tree)

### Why this exists
Searching "auto" in a million words by scanning each one is slow. A **trie** stores words as a tree of characters, sharing prefixes. Common in autocomplete, spellcheck, IP routing.

### The idea in plain English
A trie is the tree behind every **autocomplete** box. The root is empty. Each edge is a letter. Walking from the root spells a prefix. A node is marked "end of a word" if some real word ends there. Looking up "cat" walks `c → a → t` — three steps, regardless of dictionary size. Searching by prefix returns instantly because you stop where the prefix ends.

### Smallest working example
```javascript
class Trie {
  constructor() { this.root = {}; }
  insert(word) {
    let node = this.root;
    for (const ch of word) {
      if (!node[ch]) node[ch] = {};
      node = node[ch];
    }
    node.end = true;                  // marks "a word ends here"
  }
  search(word) {
    const node = this._walk(word);
    return !!(node && node.end);
  }
  startsWith(prefix) { return !!this._walk(prefix); }
  _walk(s) {
    let node = this.root;
    for (const ch of s) {
      if (!node[ch]) return null;
      node = node[ch];
    }
    return node;
  }
}

const t = new Trie();
['cat', 'car', 'cart'].forEach(w => t.insert(w));
t.search('car');       // true
t.search('ca');        // false (no .end at 'a')
t.startsWith('ca');    // true
```

Time: `O(L)` per operation where L is the word length. Space: `O(total chars)`.

### Drill (3 min)
Insert `['cat', 'car', 'cart']`. Draw the tree. *(Hint: root → c → a → {t (end), r (end) → t (end)}. Three words, six nodes, prefix `ca` is shared.)*

**Deep dive (later):** [dsa/ds-js/trie.md](../dsa/ds-js/trie.md) · [dsa/ds-java/trie.md](../dsa/ds-java/trie.md)

---

## 9. Design Pattern — Memento

### Intent
**Save and restore an object's state without exposing its internals.** It's the "undo" pattern.

### When you'd use it
- Undo in editors (text, drawing).
- Save game / load game.
- Checkpoint before a risky operation.

### Three roles
- **Originator** — the object whose state we save/restore.
- **Memento** — an opaque snapshot, with no setters.
- **Caretaker** — the stack of mementos; never inspects them.

### Smallest working example
```java
class TextEditor {
    private String content = "";

    public void type(String s) { content += s; }
    public String content()    { return content; }

    // Snapshot — immutable
    public record Memento(String snapshot) {}

    public Memento save()                 { return new Memento(content); }
    public void restore(Memento m)        { this.content = m.snapshot(); }
}

class UndoStack {
    private final Deque<TextEditor.Memento> stack = new ArrayDeque<>();
    void push(TextEditor.Memento m) { stack.push(m); }
    TextEditor.Memento pop()        { return stack.pop(); }
}

// Usage
var ed = new TextEditor();   var undo = new UndoStack();
ed.type("Hello");            undo.push(ed.save());
ed.type(" world");           // "Hello world"
ed.restore(undo.pop());      // back to "Hello"
```

The `UndoStack` only stores mementos; it can't peek inside them.

### Drill (1 min)
A game saves 100MB of state every frame. What's the smart fix? *(Hint: store only **diffs** from the previous memento, not full snapshots. Or sample (every 10th frame).)*

**Deep dive (later):** [design-patterns/common/behavioral/memento.md](../design-patterns/common/behavioral/memento.md)

---

## 10. DevOps — AWS Basics

### Why this exists
AWS has 200+ services. You're not asked to know them all. You're asked to know the five that show up everywhere: **EC2, S3, RDS, IAM, VPC**.

### The idea in plain English
- **EC2** — a virtual machine in the cloud. Pick CPU, RAM, OS. You SSH in. Billed per hour or per second.
- **S3** — file storage with a URL. Unlimited size, durable (11 nines), cheap. Buckets hold objects (files).
- **RDS** — managed Postgres/MySQL. AWS handles backups, patching, replicas. You connect to it like a normal DB.
- **IAM** — who can do what. Users, roles, policies. Without IAM you can't make any AWS call.
- **VPC** — your private network. Subnets, route tables, security groups (firewalls). Everything sits in a VPC.

A typical web app uses all five: code on EC2 (or a container), files on S3, data in RDS, all inside a VPC, with IAM controlling access.

### Smallest mental model
```
        ┌──────────────────── VPC ────────────────────┐
        │  Public subnet                              │
        │    [ EC2 web ]  ── security group           │
        │         │                                   │
        │  Private subnet                             │
        │    [ RDS Postgres ]                         │
        └─────────────────────────────────────────────┘
                ▲                       ▲
                │                       │
        IAM (who can call AWS APIs)   S3 (logs, uploads)
```

### Three IAM rules to memorize
1. Never give an EC2 instance long-lived AWS keys. Use an **instance role** — temporary credentials handed out automatically.
2. Policies follow **least privilege** — write `Allow s3:GetObject on arn:aws:s3:::my-bucket/*`, not `*`.
3. Root account = god mode. Lock it away, use IAM users for daily work.

### Drill (2 min)
Your EC2 instance needs to read from one S3 bucket. How do you give it access without storing keys? *(Hint: create an IAM role with `s3:GetObject` on that bucket only, attach the role to the EC2 instance. AWS SDKs pick it up automatically.)*

**Deep dive (later):** [devops/15-cloud-aws-basics.md](../devops/15-cloud-aws-basics.md)

---

## End-of-day checklist

- [ ] Angular: I can explain when to use `loadChildren` vs `loadComponent`
- [ ] Node: I know why clustered Node can't share an in-memory cache
- [ ] Spring: I can name two ways to activate a profile
- [ ] MongoDB: I can describe one eager and one lazy migration strategy
- [ ] Postgres: I can write a `CREATE ROLE` and one `GRANT` statement
- [ ] HLD: I can explain idempotency keys with the "double-click" example
- [ ] LLD: I sketched a hotel booking with row-level locking
- [ ] DSA: I implemented a Trie with insert / search / startsWith
- [ ] DP: I can name the three roles in Memento
- [ ] DevOps: I can list the five core AWS services and one purpose for each

**If you remember just one thing today:** systems get faster and safer by **splitting** — split the bundle, the process, the config, the privileges. Each split lets you scale or change one piece without touching others.

**Tomorrow:** async pipes, WebSockets, Actuator, Mongo validators, logical decoding, distributed storage, BookMyShow, segment trees, Visitor pattern, and GCP basics.
