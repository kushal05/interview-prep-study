# Day 9 — Guards, HTTP, Repositories, and Consensus

> **Today's goal:** see how Angular protects routes, how a Node server actually listens for HTTP, how Spring lets you skip writing SQL, and the basics of how distributed systems agree on anything.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Guards & Resolvers (CanActivate) | 15m |
| 2 | Node.js | HTTP module & Express basics | 15m |
| 3 | Spring Boot | Spring Data JPA (Repository abstraction) | 15m |
| 4 | MongoDB | Replication (replica sets, primary/secondary) | 10m |
| 5 | Postgres | MVCC (why vacuum exists) | 10m |
| 6 | HLD | Consistency & Consensus (CAP, quorum, Raft basics) | 12m |
| 7 | LLD | LRU Cache (with class diagram) | 12m |
| 8 | DSA | Graphs DFS (Number of Islands) | 15m |
| 9 | Design Pattern | Proxy | 8m |
| 10 | DevOps | K8s Storage (PV, PVC, StorageClass) | 10m |

---

## 1. Angular — Guards & Resolvers (CanActivate)

### Why this exists
Some pages are only for logged-in users. Some need data fetched **before** they render (so the page doesn't briefly flash empty). Guards say "can this route activate at all?" and resolvers say "fetch this data before activating."

### The idea in plain English
A guard is a function the router runs before navigating. Return `true` and you're in; return `false` (or a `UrlTree`) and the router redirects somewhere else (usually `/login`).

A resolver is a function the router runs before the component is created. It returns a value (or Observable/Promise), and the component reads that value from the route data — no spinner needed, no first-render flicker.

Analogy: the guard is the bouncer at the door; the resolver is the host who shows you to your table with the menu already on it.

### Smallest working example
```typescript
// auth.guard.ts
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isLoggedIn() ? true : router.createUrlTree(['/login']);
};

// user.resolver.ts
export const userResolver: ResolveFn<User> = (route) => {
  const id = route.paramMap.get('id')!;
  return inject(UserService).get(id);                // Observable<User>
};

// app.routes.ts
export const routes: Routes = [
  { path: 'admin', canActivate: [authGuard], component: AdminComponent },
  { path: 'users/:id', resolve: { user: userResolver }, component: UserComponent },
];
```

In `UserComponent`, you read the pre-fetched user via `this.route.snapshot.data['user']`.

### Drill (2 min)
A non-logged-in user types `/admin` in the address bar. What happens? *(Hint: `authGuard` runs, sees `isLoggedIn() === false`, returns a `UrlTree` to `/login`, and the router navigates there instead.)*

**Deep dive (later):** [angular/routing.md](../angular/routing.md)

---

## 2. Node.js — HTTP module & Express basics

### Why this exists
Node's built-in `http` module gives you a raw server: you handle every URL by hand with a giant `if/else`. Express is a tiny library on top that adds routing (`app.get('/users', ...)`), middleware, and a saner request/response API. Almost every Node web app uses Express or something inspired by it.

### The idea in plain English
The flow is the same everywhere:
1. **Listen** on a port (e.g., 3000).
2. When a request arrives, the framework matches the URL + method to a handler.
3. The handler reads the request, builds a response, and sends it.

Express's job is to make step 2 readable. You declare routes top to bottom: first match wins.

### Smallest working example
```javascript
// Plain Node — verbose
const http = require('http');
http.createServer((req, res) => {
  if (req.url === '/hello' && req.method === 'GET') {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hi');
  } else {
    res.writeHead(404).end();
  }
}).listen(3000);

// Same thing with Express — readable
const express = require('express');
const app = express();
app.get('/hello', (req, res) => res.send('Hi'));
app.listen(3000);
```

Two lines, same behavior. Express also handles JSON parsing, URL params, query strings — things you'd otherwise write by hand.

### Drill (2 min)
You add `app.get('/hello', (req, res) => res.send('first'))` and below it `app.get('/hello', (req, res) => res.send('second'))`. Which response does the client see? *(Hint: `'first'` — Express stops at the first matching route. The second handler never runs.)*

**Deep dive (later):** [nodejs/10-http-and-express.md](../nodejs/10-http-and-express.md)

---

## 3. Spring Boot — Spring Data JPA (Repository abstraction)

### Why this exists
Writing the same `findById`, `findAll`, `save`, `delete` SQL for every entity is tedious. Spring Data JPA generates those methods for you at runtime. You declare an interface, Spring writes the implementation.

### The idea in plain English
A **repository** is an interface that extends `JpaRepository<T, ID>`. Spring sees it at startup, creates a class behind the scenes that implements every method, and registers it as a bean. You inject it and call it — no SQL, no implementation file.

Bonus: you can add custom finders by naming convention. `findByEmail(String email)` tells Spring to write `SELECT * FROM users WHERE email = ?` automatically. This is called a **derived query**.

Analogy: it's like ordering off a menu — you say "I want the chicken" and the kitchen takes care of the rest.

### Smallest working example
```java
@Entity
public class User {
    @Id @GeneratedValue private Long id;
    private String email;
    private String name;
    // getters/setters
}

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);            // derived query
    List<User> findByNameContainingIgnoreCase(String q); // also derived
}

@Service
class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }

    public User register(String email, String name) {
        return repo.save(new User(email, name));         // no SQL!
    }
}
```

You never wrote a SQL string. Spring builds the query from the method name.

### Drill (2 min)
You add `List<User> findByEmailAndName(String email, String name);` — what SQL does Spring run? *(Hint: `SELECT * FROM users WHERE email = ? AND name = ?` — parameter order matches method-parameter order.)*

**Deep dive (later):** [spring-next/05-spring-data-jpa.md](../spring-next/05-spring-data-jpa.md)

---

## 4. MongoDB — Replication (replica sets, primary/secondary)

### Why this exists
One database server is a single point of failure. If the disk dies, you lose data. **Replication** keeps copies of your data on multiple servers so the system survives a hardware failure.

### The idea in plain English
A **replica set** is a group of mongod processes (usually 3) that hold the same data. One is the **primary** — it accepts writes. The others are **secondaries** — they continuously copy the primary's writes by replaying its operation log (the **oplog**).

If the primary dies, the secondaries hold an **election** and one of them becomes the new primary. Clients automatically reconnect to the new primary. By default, all reads also go to the primary (strong consistency); you can opt secondaries in via *read preference* if you can tolerate stale data.

Analogy: a primary teacher writes on the whiteboard; secondaries are students who copy every line. If the teacher leaves, the smartest student takes over the marker.

### Smallest working example
```javascript
// Connection string lists all replica set members
const uri = 'mongodb://host1,host2,host3/?replicaSet=rs0';

// Client auto-discovers the primary
const client = new MongoClient(uri);
await client.connect();

// Writes always go to primary
await client.db('app').collection('users').insertOne({ name: 'Jane' });

// Reads can target secondaries (eventually consistent)
const cur = client.db('app').collection('users')
  .find({}, { readPreference: 'secondaryPreferred' });
```

The client driver handles failover transparently — your code doesn't change when the primary changes.

### Drill (2 min)
You have a 3-node replica set. The primary's network cable is yanked. What happens? *(Hint: the two remaining secondaries detect the missing heartbeat, hold an election, and one becomes the new primary. Writes resume in a few seconds.)*

**Deep dive (later):** [mongodb/09-replication.md](../mongodb/09-replication.md)

---

## 5. Postgres — MVCC (why vacuum exists)

### Why this exists
If two transactions read and write the same row at the same time, how do you avoid blocking everyone? Postgres uses **MVCC (Multi-Version Concurrency Control)** — it keeps multiple versions of each row so readers never block writers and writers never block readers.

### The idea in plain English
When you `UPDATE` a row, Postgres doesn't overwrite it. It writes a **new version** and marks the old one as "obsolete after transaction X." Readers see the version that was visible when their transaction started. Writers see and update the latest visible version.

The downside: obsolete row versions pile up. They take disk space and slow queries. The cleanup job that removes them is **VACUUM**. Postgres runs **autovacuum** in the background, but on busy tables it can fall behind, and that's when DBAs get paged.

Analogy: every edit makes a new photocopy of the page; the old copy gets a "tombstone" sticker. VACUUM is the janitor who eventually shreds tombstoned pages.

### Smallest working example
```sql
-- Two sessions, run side by side:

-- Session A
BEGIN;
SELECT balance FROM accounts WHERE id = 1;   -- sees: 100

-- Session B (commits while A is still open)
UPDATE accounts SET balance = 200 WHERE id = 1;
COMMIT;

-- Back in Session A
SELECT balance FROM accounts WHERE id = 1;   -- still sees: 100 (snapshot!)
COMMIT;

-- Manually trigger cleanup of dead rows (or trust autovacuum)
VACUUM ANALYZE accounts;
```

Session A never blocks. It just sees the version that was current when its transaction began.

### Drill (2 min)
Why does a table that's only ever `UPDATE`d (never inserted into) keep growing on disk? *(Hint: each UPDATE writes a new row version. Without VACUUM, the dead versions pile up. After VACUUM, the space is reusable.)*

**Deep dive (later):** [postgres/10-mvcc.md](../postgres/10-mvcc.md)

---

## 6. HLD — Consistency & Consensus (CAP, quorum, Raft basics)

### Why this exists
The moment you have more than one server, you face the question: when they disagree, who's right? Distributed systems theory gives names to the trade-offs.

### The idea in plain English
**CAP theorem:** in a network partition (some servers can't reach each other), you must pick:
- **CP** — Consistency over Availability (refuse writes until partition heals; e.g., MongoDB primary).
- **AP** — Availability over Consistency (accept writes on both sides, reconcile later; e.g., Cassandra).
- You can't get both during a partition. You **always** have partition tolerance — networks fail.

**Quorum:** to agree on a value, a majority of nodes must say yes. With 3 nodes you need 2; with 5 you need 3. That's why replica sets prefer odd numbers — a 4-node set still only needs 3 votes, so the extra node buys nothing.

**Raft** is a consensus algorithm that elects a leader and replicates a log to followers, used by etcd (Kubernetes), Consul, CockroachDB. The key trick: every change goes through the leader, and the leader waits for a quorum of followers to acknowledge before committing.

### A picture
```
   3-node cluster, quorum = 2

   Leader  -->  Follower
       \-->  Follower

   Leader appends "x=1" to its log.
   Sends to followers. Two ack? Commit. One acks? Wait.
```

### Drill (2 min)
A 5-node Raft cluster suffers a network split: 2 nodes on one side, 3 on the other. Which side can still commit writes? *(Hint: the 3-node side — it holds a quorum (majority of 5). The 2-node side can't elect a leader and serves only reads (if at all).)*

**Deep dive (later):** [system-design/high-level-design/09-consistency-and-consensus.md](../system-design/high-level-design/09-consistency-and-consensus.md)

---

## 7. LLD — LRU Cache (with class diagram)

### Why this exists
Caches have limited size. When full and a new item arrives, you must evict someone. **LRU (Least Recently Used)** evicts the item that hasn't been touched the longest — a great default because recently-used items are often used again soon.

### The idea in plain English
We need two operations in **O(1)**:
- `get(key)` — return value, mark as most-recently-used.
- `put(key, value)` — insert; if over capacity, evict the least-recently-used.

The classic answer: combine a **HashMap** (O(1) lookup) with a **doubly linked list** (O(1) move-to-front and O(1) tail removal). The hash map stores `key → node`; the list orders nodes from most-recent (head) to least-recent (tail).

### Smallest working example
```java
class LRUCache {
    private final int cap;
    private final LinkedHashMap<Integer, Integer> map;

    public LRUCache(int capacity) {
        this.cap = capacity;
        // accessOrder=true ⇒ moves accessed entries to the end
        this.map = new LinkedHashMap<>(capacity, 0.75f, true) {
            protected boolean removeEldestEntry(Map.Entry<Integer,Integer> e) {
                return size() > cap;
            }
        };
    }

    public int get(int key) {
        return map.getOrDefault(key, -1);
    }

    public void put(int key, int value) {
        map.put(key, value);          // eldest auto-removed if over cap
    }
}
```

Java's `LinkedHashMap` already implements a list-backed hash map. For an interview, you may be asked to write the doubly-linked-list version by hand — see the deep-dive.

### Drill (2 min)
Capacity 2. Calls: `put(1,1)`, `put(2,2)`, `get(1)`, `put(3,3)`, `get(2)`. What does the last call return? *(Hint: `-1`. After `get(1)`, key 1 is most-recent; key 2 is least-recent and gets evicted when 3 is added.)*

**Deep dive (later):** [system-design/low-level-design/08-classic-lld-problems.md](../system-design/low-level-design/08-classic-lld-problems.md)

---

## 8. DSA — Graphs: DFS (Number of Islands)

### The problem
Given a 2D grid of `'1'`s (land) and `'0'`s (water), count the number of islands. An island is a group of `'1'`s connected horizontally or vertically (not diagonally).

```
Input:  [['1','1','0','0'],
         ['1','1','0','0'],
         ['0','0','1','0'],
         ['0','0','0','1']]
Output: 3
```

### The idea
Walk every cell. When you find an unvisited `'1'`, that's a new island — increment the count, then DFS from there to "sink" every connected `'1'` (mark it as visited or flip it to `'0'`). Move on. Each cell is visited once.

DFS (Depth-First Search) is recursion: visit a cell, then visit its neighbors, recursively. It's the natural fit when you only need to **mark** connected components — order doesn't matter (unlike shortest path, which needs BFS).

### Smallest working example
```javascript
function numIslands(grid) {
  const R = grid.length, C = grid[0].length;
  let count = 0;

  function sink(r, c) {
    if (r < 0 || c < 0 || r >= R || c >= C || grid[r][c] !== '1') return;
    grid[r][c] = '0';                          // mark visited
    sink(r+1, c); sink(r-1, c);
    sink(r, c+1); sink(r, c-1);
  }

  for (let r = 0; r < R; r++)
    for (let c = 0; c < C; c++)
      if (grid[r][c] === '1') { count++; sink(r, c); }

  return count;
}
```

**Time:** O(R × C) — each cell visited at most once. **Space:** O(R × C) for the recursion stack in the worst case.

### Drill (3 min)
Run through the 4×4 example by hand. *(You should find: island 1 at (0,0)-(1,1) block; island 2 at (2,2); island 3 at (3,3). The `sink` function flips each island's cells to `'0'` after counting.)*

**Deep dive (later):** [dsa/ds-js/graphs.md](../dsa/ds-js/graphs.md) · [dsa/ds-java/graphs.md](../dsa/ds-java/graphs.md)

---

## 9. Design Pattern — Proxy

### Intent
**Stand in front of another object** to control access, add behavior, or delay creation — without the caller knowing the proxy exists.

### When you'd use it
- **Lazy loading** — don't load a 500MB image until someone actually views it.
- **Access control** — wrap a service so only admins reach the inner methods.
- **Caching** — proxy returns a cached result, hits the real service only on miss.
- **Remote proxy** — the proxy lives in process, the real object lives across the network.

### Smallest working example
```java
interface Image { void display(); }

class RealImage implements Image {
    private final String filename;
    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();                              // expensive
    }
    private void loadFromDisk() { /* slow */ }
    public void display()       { System.out.println("Showing " + filename); }
}

class ProxyImage implements Image {
    private final String filename;
    private RealImage real;

    public ProxyImage(String filename) { this.filename = filename; }

    public void display() {
        if (real == null) real = new RealImage(filename);  // load on first use
        real.display();
    }
}
```

Spring uses proxies heavily — `@Transactional` works by wrapping your bean in a proxy that opens/commits the transaction around your method call.

### Drill (1 min)
Why is Proxy different from Decorator? *(Hint: both wrap an object. A Decorator **adds behavior** the caller asked for. A Proxy **controls access** — the caller often doesn't know it's there. Different intent, same shape.)*

**Deep dive (later):** [design-patterns/common/structural/proxy.md](../design-patterns/common/structural/proxy.md)

---

## 10. DevOps — K8s Storage (PV, PVC, StorageClass)

### Why this exists
Pods are ephemeral — their local disk is wiped when they restart. But databases need to keep data across restarts. Kubernetes splits storage into pieces so apps can ask for disk without caring whether it's AWS EBS, GCP PD, or NFS.

### The idea in plain English
- **PersistentVolume (PV)** — a chunk of real storage available in the cluster (e.g., a 10 GB EBS disk). Created by an admin or auto-provisioned.
- **PersistentVolumeClaim (PVC)** — a pod's request: "I need 5 GB, read-write, fast SSD." K8s matches the claim to a PV.
- **StorageClass** — a template for auto-creating PVs on demand. "Whenever a PVC needs `fast-ssd`, provision a new GP3 EBS volume."
- **StatefulSet** — like a Deployment but each pod gets a stable name (`db-0`, `db-1`) and its own PVC that survives restarts.

Analogy: PV = a parking spot. PVC = a parking ticket request. StorageClass = the policy ("paid spots are 20 ft, free spots are 15 ft").

### Smallest working example
```yaml
# 1. The pod's request
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 5Gi } }
  storageClassName: fast-ssd
---
# 2. The pod uses the claim like a normal volume
apiVersion: v1
kind: Pod
metadata: { name: db }
spec:
  containers:
  - name: db
    image: postgres:16
    volumeMounts: [{ name: data, mountPath: /var/lib/postgresql/data }]
  volumes:
  - name: data
    persistentVolumeClaim: { claimName: data }
```

If the pod dies and restarts on another node, K8s reattaches the same PV — the data is still there.

### Drill (2 min)
You scale a StatefulSet from 1 to 3 replicas. How many PVCs exist now? *(Hint: 3 — one per pod (`data-db-0`, `data-db-1`, `data-db-2`). Each pod owns its own claim, so each has its own disk.)*

**Deep dive (later):** [devops/08-kubernetes-storage.md](../devops/08-kubernetes-storage.md)

---

## End-of-day checklist

- [ ] Angular: I can write a `canActivate` guard that redirects unauthenticated users
- [ ] Node.js: I can stand up an Express server with one GET route in 4 lines
- [ ] Spring: I can extend `JpaRepository<User, Long>` and add a `findByEmail` method
- [ ] MongoDB: I can describe primary/secondary roles and what happens during failover
- [ ] Postgres: I can explain why an UPDATE-heavy table needs VACUUM
- [ ] HLD: I can state CAP in one sentence and explain why quorums prefer odd numbers
- [ ] LLD: I can list the two data structures behind an O(1) LRU cache
- [ ] DSA: I solved Number of Islands and explained when to pick DFS vs BFS
- [ ] DP: I can name two real uses of the Proxy pattern (Spring `@Transactional` is one)
- [ ] DevOps: I know what PV, PVC, and StorageClass each represent

**If you remember just one thing today:** **distributed systems exchange certainty for availability**. Replicas, quorums, and isolation levels are all dials on that same trade-off.

**Tomorrow:** typed HTTP clients, Express middleware chains, JPA entity mapping, sharding, locks, and the Twitter feed at high level.
