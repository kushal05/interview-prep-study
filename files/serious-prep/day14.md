# Day 14 — Two-week mark: streams, auth, joins, replication

> **Today's goal:** start the "real-world plumbing" half of the prep — multi-cast events, token auth, smart queries, replicas, chat systems. One small example per topic, no heroics.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | RxJS Subjects (Subject, BehaviorSubject, ReplaySubject) | 15m |
| 2 | Node.js | JWT Auth (header.payload.signature) | 15m |
| 3 | Spring Boot | N+1 Problem (causes, JOIN FETCH) | 15m |
| 4 | MongoDB | Security (auth, RBAC, TLS) | 10m |
| 5 | Postgres | Replication (streaming vs logical, WAL basics) | 10m |
| 6 | HLD | Chat System Design (1-to-1, presence, delivery) | 12m |
| 7 | LLD | Library Management System | 12m |
| 8 | DSA | Linked List Cycle Detection (Floyd's algorithm) | 15m |
| 9 | Design Pattern | Observer | 8m |
| 10 | DevOps | Load Balancers & Proxies (Nginx, L4 vs L7) | 10m |

---

## 1. Angular — RxJS Subjects

### Why this exists
Sometimes one piece of code needs to **announce** something ("user logged in!") and many other pieces need to **react**. A Subject is RxJS's built-in walkie-talkie for that.

### The idea in plain English
An **Observable** is a one-way TV channel — you watch, it broadcasts.
A **Subject** is a TV channel *and* a microphone — you can also push values into it. That makes it perfect for sharing state between components.

Three flavors you'll meet:
- **Subject** — late subscribers miss past values (like joining a live radio show mid-song)
- **BehaviorSubject** — always remembers the *current* value; new subscribers get it immediately (like turning on the radio and hearing the song already playing)
- **ReplaySubject** — replays the last *N* values to every new subscriber (like Spotify replaying the last 3 songs you missed)

### Smallest working example
```typescript
import { BehaviorSubject } from 'rxjs';

// Shared "current user" state — starts as null
const user$ = new BehaviorSubject<string | null>(null);

// Subscriber A — subscribes BEFORE anyone logs in
user$.subscribe(u => console.log('A sees:', u));   // A sees: null

user$.next('jane');                                // both A and B will see this
user$.subscribe(u => console.log('B sees:', u));   // B sees: jane (immediately!)

user$.next('mike');                                // A sees: mike, B sees: mike
```

Plain `Subject` would have given B `undefined` until the next `.next()`. `BehaviorSubject` hands B the current value the moment B subscribes.

### Drill (2 min)
Which Subject should an `AuthService.currentUser$` use? *(Hint: BehaviorSubject — every new component that subscribes should immediately know who's logged in, not have to wait for the next login event.)*

**Deep dive (later):** [angular/rxjs-subjects.md](../angular/rxjs-subjects.md)

---

## 2. Node.js — JWT Auth

### Why this exists
HTTP is stateless — the server forgets you between requests. With JWT, the server gives you a signed wristband; show it on every request and the server trusts you without a database lookup.

### The idea in plain English
A **JWT (JSON Web Token)** is three Base64 strings glued by dots: `header.payload.signature`.
- **Header** — algorithm used (e.g. `HS256`)
- **Payload** — claims about you (user id, expiry)
- **Signature** — proof the server signed it (anyone tampers → signature breaks)

Think of it as a concert wristband with a hologram. The bouncer (server) doesn't keep a list — they just check the hologram. If the wristband is altered, the hologram won't match.

### Smallest working example
```javascript
const jwt = require('jsonwebtoken');
const SECRET = process.env.JWT_SECRET || 'dev-only';

// Issue on login
const token = jwt.sign(
  { sub: 'user-123', role: 'admin' },
  SECRET,
  { expiresIn: '1h' }
);
console.log(token);   // xxx.yyy.zzz

// Verify on every request
try {
  const payload = jwt.verify(token, SECRET);
  console.log('Verified user:', payload.sub);   // user-123
} catch (e) {
  console.log('Invalid or expired token');
}
```

The client sends it in `Authorization: Bearer <token>`. The server verifies and trusts.

### Drill (2 min)
Why never put a password in the payload? *(Hint: the payload is Base64-encoded, not encrypted — anyone with the token can read it. JWT proves authenticity, not confidentiality.)*

**Deep dive (later):** [nodejs/14-authentication-and-jwt.md](../nodejs/14-authentication-and-jwt.md)

---

## 3. Spring Boot — N+1 Problem

### Why this exists
The N+1 problem is the #1 cause of "why is my JPA app slow?" tickets. Once you've seen it, you'll spot it everywhere.

### The idea in plain English
You fetch a list of N items, then accidentally fire one extra query per item to load a related thing — total: 1 + N queries. For 1,000 orders with customers, that's 1,001 queries.

**Analogy:** instead of asking the store clerk "what's in this aisle?" once for the whole store, you ask aisle-by-aisle 1,000 times. Same answer, 1,000× slower.

The fix is **JOIN FETCH** (or an entity graph) — tell JPA "while you're loading orders, grab their customers in the same SQL."

### Smallest working example
```java
// BAD: N+1 — one query for orders, then one per order for customer
@Query("select o from Order o")
List<Order> findAll();
// Then o.getCustomer() in a loop fires a fresh SELECT per order.

// GOOD: 1 query, joined
@Query("select o from Order o join fetch o.customer")
List<Order> findAllWithCustomer();
```

`join fetch` tells Hibernate to load the related entity in the same SQL via SQL JOIN. One round-trip, not N+1.

### Drill (2 min)
You see 51 SQL queries in your log when you load 50 orders. Likely cause? *(Hint: lazy-loaded `customer` association — Hibernate fired 1 query for the list and 1 per row when you touched `getCustomer()`. Add `join fetch` or an `@EntityGraph`.)*

**Deep dive (later):** [spring-detailed/PART-18-spring-data-jpa.md](../spring-detailed/PART-18-spring-data-jpa.md)

---

## 4. MongoDB — Security

### Why this exists
A MongoDB started without auth was famously the source of thousands of public-internet data leaks. Three layers keep it safe: **authentication, authorization, encryption in transit**.

### The idea in plain English
- **Authentication (authN)** — "who are you?" Mongo verifies a username/password (or x509 cert).
- **Authorization (authZ / RBAC)** — "what can you do?" Mongo gives you roles like `read`, `readWrite`, `dbAdmin` scoped to a database.
- **TLS** — encrypts traffic between client and server so passwords and data aren't readable on the wire.

Think of it as a building: authentication is the badge swipe at the door, RBAC decides which floors your badge opens, TLS is the soundproof glass on the corridors.

### Smallest working example
```javascript
// Create a least-privilege user
use admin;
db.createUser({
  user: "orders_app",
  pwd:  "S3cret!",
  roles: [
    { role: "readWrite", db: "ordersdb" }   // only this DB, only data ops
  ]
});

// Connect using that user over TLS:
// mongodb://orders_app:S3cret!@host:27017/ordersdb?authSource=admin&tls=true
```

`readWrite` lets the app do CRUD but not create users, drop the DB, or read other DBs.

### Drill (2 min)
Your app needs to read one collection and that's it. Which built-in role? *(Hint: `read` on that database — never grant `readWrite` "just in case." Least privilege is the rule.)*

**Deep dive (later):** [mongodb/13-security.md](../mongodb/13-security.md)

---

## 5. Postgres — Replication

### Why this exists
One database server is a single point of failure. **Replication** keeps a second server in sync so you can fail over, or so you can offload heavy reads.

### The idea in plain English
Postgres writes every change to a **WAL** (Write-Ahead Log) — a journal of "what just happened." Replication ships that journal to another server which replays it.

Two flavors:
- **Streaming (physical) replication** — replica is a byte-for-byte copy. Used for high availability (failover) and read replicas. Same Postgres version required.
- **Logical replication** — replica receives a stream of *row changes* by table. Used to move data between different versions or to a subset of tables (CDC-style).

Analogy: physical = photocopying the whole ledger; logical = dictating "row 42 changed to X" so the other side can update its own ledger.

### Smallest working example
```ini
# postgresql.conf on the primary
wal_level = replica            # 'logical' if you need logical replication
max_wal_senders = 10
```

```sql
-- On the primary, create a replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'r3p';

-- Then on the replica, run pg_basebackup pointing at primary, and start it.
```

The replica connects, streams WAL, and applies it continuously.

### Drill (2 min)
You want to upgrade Postgres from 14 to 16 with near-zero downtime. Which replication? *(Hint: logical — physical requires identical major versions. Logical lets the new-version replica catch up, then you switch traffic.)*

**Deep dive (later):** [postgres/15-replication.md](../postgres/15-replication.md)

---

## 6. High-Level Design — Chat System

### Why this exists
Real-time chat (WhatsApp, Slack DMs) touches half the topics interviewers love: long-lived connections, fan-out, ordering, presence, delivery guarantees.

### The idea in plain English
The web is request/response — client asks, server replies, done. Chat needs **server-pushed** messages. You hold a long-lived **WebSocket** connection (or use long-polling) so the server can push the instant a message arrives.

**Core flow for 1-to-1 chat:**
1. Alice's app holds a WebSocket to a **gateway** server.
2. Bob's app holds a WebSocket to (possibly a different) gateway.
3. Alice sends → gateway → message service → store in DB → look up Bob's gateway → push to Bob.
4. Server ACKs Alice ("sent"), then ACKs again when Bob's app reads ("delivered/read").

**Presence** ("Bob is online") is a cache (Redis) keyed by user, updated on WS connect/disconnect with a TTL.

### A minimal sketch
```
[Alice]──ws──┐                                      ┌──ws──[Bob]
             │                                      │
        [Gateway A]───────[Message Service]──────[Gateway B]
                              │
                      [Messages DB]  [Presence cache (Redis)]
```

Messages are written before fanning out → if Bob is offline, Alice's message still survives and is delivered when Bob reconnects.

### Drill (3 min)
Why store the message in the DB **before** pushing to Bob? *(Hint: durability. If you push first and the server crashes mid-flight, Bob never sees it and Alice thinks it's sent. DB-first guarantees the message exists even if the push fails — Bob's app can re-fetch on reconnect.)*

**Deep dive (later):** [system-design/high-level-design/12-chat-system.md](../system-design/high-level-design/12-chat-system.md)

---

## 7. LLD — Library Management System

### Why this exists
Library Management is the "FizzBuzz" of LLD interviews — small enough to design in 30 min, rich enough to show OOP, state transitions, and inheritance.

### Core entities
- **Book** — title, author, ISBN (catalog-level info)
- **BookItem** — a physical copy (multiple `BookItem`s per `Book`)
- **Member** — has an id, name, and a list of current loans
- **Loan** — links one BookItem to one Member with a due date
- **Librarian** — staff role that can add books, fine members, etc.

### A clean class sketch
```java
enum BookStatus { AVAILABLE, LOANED, LOST }

class BookItem {
    String barcode;
    Book book;
    BookStatus status = BookStatus.AVAILABLE;
}

class Member {
    String id;
    List<Loan> activeLoans = new ArrayList<>();
}

class Library {
    Loan checkout(Member m, BookItem item) {
        if (item.status != BookStatus.AVAILABLE) throw new IllegalStateException();
        item.status = BookStatus.LOANED;
        Loan loan = new Loan(m, item, LocalDate.now().plusDays(14));
        m.activeLoans.add(loan);
        return loan;
    }
}
```

Notice: **state lives on `BookItem`**, not on `Book`. Two copies of "Clean Code" can be in different states.

### Drill (3 min)
A member wants to reserve a book that's currently loaned. How do you model that? *(Hint: add a `Reservation` entity with a queue. When the book is returned, the next reservation in the queue gets notified. Avoid putting "reservedBy" on `BookItem` — multiple people may want it.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Linked List Cycle Detection

### The problem
Given the head of a singly-linked list, return `true` if there's a cycle (a node points back to an earlier node), else `false`.

```
1 → 2 → 3 → 4
        ↑   ↓
        6 ← 5      (cycle!)
```

### The slow approach
Store every visited node in a `HashSet`. If you ever see the same reference twice, you've cycled.
- Time: O(n)
- Space: **O(n)** — that's the catch.

### Floyd's tortoise and hare (the elegant one)
Walk two pointers. `slow` moves 1 step at a time, `fast` moves 2. If there's a cycle, `fast` eventually laps `slow` and they meet. If there's no cycle, `fast` reaches `null`.

```javascript
function hasCycle(head) {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow === fast) return true;
  }
  return false;
}
```

**Time:** O(n). **Space:** O(1). Same answer, no extra memory.

### Why it works (intuition)
Two cars on a circular track, one twice as fast — they will collide. Two cars on a straight road, one twice as fast — the fast one drives off the end and never meets the slow one.

### Drill (3 min)
What if `fast` was 3 steps at a time? *(Hint: still works as long as `fast` is strictly faster than `slow` and they're both inside the cycle. 2 is just the simplest and avoids skipping over `slow`.)*

**Deep dive (later):** [dsa/ds-js/linked-lists.md](../dsa/ds-js/linked-lists.md) · [dsa/ds-java/linked-lists.md](../dsa/ds-java/linked-lists.md)

---

## 9. Design Pattern — Observer

### Intent
**Let many objects ("observers") react when one object ("subject") changes**, without the subject knowing who's listening.

### When you'd use it
- A spreadsheet cell formula recalculating when an input cell changes
- UI re-renders when state changes (this is how RxJS, React, Vue all work)
- Event buses in microservices

### The simplest version (Java)
```java
interface Observer { void update(String event); }

class Subject {
    private List<Observer> observers = new ArrayList<>();
    public void subscribe(Observer o)   { observers.add(o); }
    public void publish(String event)   {
        for (Observer o : observers) o.update(event);
    }
}

// Use it
Subject feed = new Subject();
feed.subscribe(e -> System.out.println("UI got: " + e));
feed.subscribe(e -> System.out.println("Logger got: " + e));
feed.publish("user-logged-in");
// UI got: user-logged-in
// Logger got: user-logged-in
```

The `Subject` has no idea what `UI` or `Logger` do — only that they implement `Observer`. Add/remove subscribers any time.

### Drill (1 min)
You saw `BehaviorSubject` in section 1. Which pattern is it? *(Hint: Observer — `subscribe()` registers an observer, `next()` is publish.)*

**Deep dive (later):** [design-patterns/common/behavioral/observer.md](../design-patterns/common/behavioral/observer.md)

---

## 10. DevOps — Load Balancers & Proxies

### Why this exists
One server can handle X requests/sec. To serve 10× the traffic, you put a **load balancer** in front of many servers. The same load balancer also gives you TLS termination, retries, and a single public IP.

### The idea in plain English
A load balancer is the host at a restaurant: customers (requests) line up at the door, the host sends each to whichever table (server) is free.

Two layers to know:
- **L4 (transport layer)** — balances by IP/port. Fast, dumb. Doesn't read the request body. Tools: AWS NLB, HAProxy in TCP mode.
- **L7 (application layer)** — balances by HTTP info: URL path, headers, cookies. Can do A/B routing, rate limiting, gzip. Tools: Nginx, Envoy, AWS ALB.

A **reverse proxy** sits in front of servers (Nginx in front of Node). A **forward proxy** sits in front of clients (corporate web filter).

### Smallest working Nginx config
```nginx
upstream api {
    server app1.local:3000;
    server app2.local:3000;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://api;                 # forward to one of the upstreams
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

Nginx picks an upstream (round-robin by default) and forwards. Add `least_conn` for "send to the least busy."

### Drill (2 min)
You want requests with `/admin/*` to go to a separate "admin" pool of servers. L4 or L7? *(Hint: L7 — only L7 sees the URL path. L4 only sees TCP/IP.)*

**Deep dive (later):** [devops/23-load-balancers-and-proxies.md](../devops/23-load-balancers-and-proxies.md)

---

## End-of-day checklist

- [ ] Angular: I can explain the difference between Subject, BehaviorSubject, ReplaySubject
- [ ] Node.js: I can describe the three parts of a JWT and why the signature matters
- [ ] Spring: I can spot the N+1 problem from a log and propose `join fetch` as the fix
- [ ] MongoDB: I can create a least-privilege user with `db.createUser`
- [ ] Postgres: I can pick between streaming and logical replication for a use case
- [ ] HLD: I can sketch the gateway / message-service / DB flow for chat
- [ ] LLD: I can model Library with Book vs BookItem and a Loan
- [ ] DSA: I can implement Floyd's cycle detection from scratch
- [ ] DP: I can explain Observer in one breath
- [ ] DevOps: I can choose L4 vs L7 for a given routing rule

**If you remember just one thing today:** every "real-time" feature (RxJS Subjects, chat fan-out, Observer pattern) is the same idea — a publisher and many subscribers — just at different scales.

**Tomorrow:** change detection, validation, transactions, backups, partitioning, notifications, Splitwise, trees, Command pattern, structured logs.
