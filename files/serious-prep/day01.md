# Day 1 — First steps: how the pieces fit together

> **Today's goal:** see one tiny example of each tool so the names stop feeling scary. Don't try to master anything — just recognize the words.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Components & Templates | 15m |
| 2 | Node.js | Event Loop (single-threaded JS) | 15m |
| 3 | Spring Boot | Spring Core / IoC | 15m |
| 4 | MongoDB | Document model | 10m |
| 5 | Postgres | Tables & basic SQL | 10m |
| 6 | HLD | Latency vs throughput | 12m |
| 7 | LLD | The four OOP ideas | 12m |
| 8 | DSA | Two Sum (warm-up problem) | 15m |
| 9 | Design Pattern | Singleton | 8m |
| 10 | DevOps | Linux essentials | 10m |

---

## 1. Angular — Components & Templates

### Why this exists
A web page is made of small parts (a header, a list, a button). Angular calls each part a **component**. Building UI from components is like building with Lego blocks — small, reusable, snap-together pieces.

### The idea in plain English
A component bundles three things:
- **HTML** — what the user sees (the template)
- **CSS** — how it looks (styles)
- **TypeScript** — what it does (the class)

You write the class, mark it with `@Component`, and Angular shows it on screen whenever the matching tag (the *selector*) appears in HTML.

### Smallest working example
```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-counter',          // <app-counter></app-counter> in HTML
  standalone: true,
  template: `
    <button (click)="count = count + 1">
      Clicked {{ count }} times
    </button>
  `,
})
export class CounterComponent {
  count = 0;
}
```

Three new things to notice:
- `{{ count }}` — show a variable in HTML (called *interpolation*).
- `(click)="..."` — listen for an event.
- The class has no `render()` method. Angular handles that for you.

### Drill (2 min)
What happens to the number on screen when you click the button? *(Hint: Angular notices `count` changed and updates the DOM for you. You did not have to write any update code.)*

**Deep dive (later):** [angular/components.md](../angular/components.md) · [angular/angular-core-concepts.md](../angular/angular-core-concepts.md)

---

## 2. Node.js — Event Loop (single-threaded JS)

### Why this exists
JavaScript was made for browsers — one tab, one user, one thread. Node.js took that same JavaScript and made it run servers. So how does a single-threaded language handle thousands of users at once? With the **event loop**.

### The idea in plain English
Think of a chef who cooks alone but never waits idly. While the rice boils (slow I/O), the chef chops onions (fast work). The chef checks the rice only when it's likely done. That's the event loop: **don't block, let slow things happen in the background, come back when they're ready**.

What blocks the loop = bad: long math, reading a huge file synchronously, infinite loops.
What doesn't block = good: async I/O (`fs.promises.readFile`, network calls, timers).

### Smallest working example
```javascript
console.log('A');                                  // runs now
setTimeout(() => console.log('B'), 0);             // runs later (after current code)
Promise.resolve().then(() => console.log('C'));    // runs later, before B
console.log('D');                                  // runs now

// Output: A D C B
```

Why this order? `A` and `D` run immediately (they're synchronous). Then the loop drains *microtasks* (Promises) first → prints `C`. Then it handles *macrotasks* (timers) → prints `B`.

### Drill (2 min)
If you replace `Promise.resolve().then(...)` with `setImmediate(...)`, what changes? *(Hint: `setImmediate` is a macrotask like `setTimeout(0)`, so it runs near `B`, not near `C`.)*

**Deep dive (later):** [nodejs/02-async-and-event-loop.md](../nodejs/02-async-and-event-loop.md)

---

## 3. Spring Boot — Spring Core / IoC

### Why this exists
In a normal program, classes create the objects they need: `new EmailService()`. In a big app that creates a tangled mess — `OrderService` builds an `EmailService` which builds an `SmtpClient` which builds a `Config`... Spring inverts this: **you declare what you need, Spring builds and wires everything**. That's **Inversion of Control (IoC)**.

### The idea in plain English
Imagine moving into a furnished apartment. You don't shop for a fridge — you walk in and the fridge is there, plugged in. Spring is your landlord: it furnishes ("injects") your class with everything it needs (dependencies). You just declare what you need in the constructor.

### Smallest working example
```java
@Service
public class GreetingService {
    public String greet(String name) {
        return "Hello, " + name;
    }
}

@RestController
public class HelloController {
    private final GreetingService greeting;

    // Spring sees this constructor and gives us a GreetingService for free.
    public HelloController(GreetingService greeting) {
        this.greeting = greeting;
    }

    @GetMapping("/hi/{name}")
    public String hi(@PathVariable String name) {
        return greeting.greet(name);
    }
}
```

`@Service` and `@RestController` tell Spring: "These are beans — manage them." Spring creates one instance of each (default scope = singleton) and connects them.

### Drill (2 min)
You never wrote `new GreetingService()` anywhere. So who creates it? *(Hint: at startup, Spring scans the classpath for `@Service`, `@Component`, `@RestController`, etc., creates the objects, and stores them in a container called the `ApplicationContext`.)*

**Deep dive (later):** [spring-detailed/PART-12-spring-core-internals.md](../spring-detailed/PART-12-spring-core-internals.md) · [spring-next/02-spring-core.md](../spring-next/02-spring-core.md)

---

## 4. MongoDB — Document model

### Why this exists
Relational databases (Postgres, MySQL) store data in tables with strict columns. That's great for structured data, but painful when shapes vary — like user profiles where some users have addresses, some don't, some have 5. MongoDB stores **documents** (JSON-like objects), letting each record have whatever shape fits.

### The idea in plain English
A document is just like a JavaScript object. Documents live inside **collections** (think folders of similar objects), and collections live inside **databases**. No tables, no joins by default — you put related data *inside* the document.

### Smallest working example
```javascript
// A user document — notice 'addresses' is an array inside the document itself
{
  _id: ObjectId("65f0..."),       // primary key (auto-generated)
  email: "jane@example.com",
  name: "Jane",
  addresses: [
    { type: "home", city: "Bengaluru" },
    { type: "work", city: "Pune" }
  ]
}

// Insert + find
db.users.insertOne({ email: "k@x.com", name: "K" });
db.users.findOne({ email: "k@x.com" });
```

In Postgres this would need two tables and a join. In Mongo it's one document.

### Drill (2 min)
A user has 3 home addresses. In Postgres you'd add 3 rows to an `addresses` table. How do you store that in Mongo? *(Hint: it's an array inside the user document, like the example above.)*

**Deep dive (later):** [mongodb/01-intro-and-document-model.md](../mongodb/01-intro-and-document-model.md)

---

## 5. Postgres — Tables & basic SQL

### Why this exists
Most business data has a known shape (a user has an email, name, age) and you query it in predictable ways. Relational databases store that data in tables and let you ask powerful questions with **SQL** (Structured Query Language).

### The idea in plain English
A table is a spreadsheet: rows are records, columns are fields. Each column has a **type** (text, number, date). The database enforces those types — try to put text into a number column and it refuses.

You **describe** the schema with DDL (`CREATE TABLE`), then **work with data** using DML (`INSERT`, `SELECT`, `UPDATE`, `DELETE`).

### Smallest working example
```sql
-- Create
CREATE TABLE users (
    id         BIGSERIAL PRIMARY KEY,    -- auto-incrementing number
    email      TEXT NOT NULL UNIQUE,
    name       TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Insert
INSERT INTO users (email, name) VALUES ('jane@x.com', 'Jane');

-- Read
SELECT id, name FROM users WHERE email = 'jane@x.com';

-- Update
UPDATE users SET name = 'Jane M' WHERE id = 1;

-- Delete
DELETE FROM users WHERE id = 1;
```

`PRIMARY KEY` means "this column uniquely identifies a row." `NOT NULL` means "must have a value." `UNIQUE` means "no duplicates."

### Drill (2 min)
What happens if you `INSERT` a user with `email = 'jane@x.com'` twice? *(Hint: the second insert errors because of the `UNIQUE` constraint on email.)*

**Deep dive (later):** [postgres/02-sql-basics-and-ddl.md](../postgres/02-sql-basics-and-ddl.md)

---

## 6. High-Level Design — Latency vs throughput

### Why this exists
Engineers throw around "fast" and "slow" loosely. In system design we use **precise** words so you can compare options. The two most important: **latency** and **throughput**.

### The idea in plain English
Imagine a coffee shop:
- **Latency** = how long *one* customer waits from "I'd like a latte" to "here's your latte." Measured in milliseconds.
- **Throughput** = how many lattes the shop can serve *per minute*.

A barista who serves one customer at a time has low throughput. Add 4 baristas with one espresso machine — throughput goes up, but if the machine is the bottleneck, latency for any single drink may not improve at all.

You report latency as **percentiles**: `p50` (median), `p95`, `p99`. p99 = "99% of requests are faster than this." Outliers (the slowest 1%) often matter more than the average.

### Numbers worth memorizing
| Operation | Latency |
|---|---|
| L1 cache reference | ~1 ns |
| RAM access | ~100 ns |
| SSD read | ~100 µs |
| Network within data center | ~500 µs |
| HDD seek | ~10 ms |
| Network round-trip India ↔ US | ~150 ms |

### Drill (2 min)
A service runs at 100 requests/sec with `p99 = 500ms`. Is it "fast"? *(Hint: it depends. 500ms for a search result is fine; 500ms for a key press is terrible. Always ask "fast for what use case?")*

**Deep dive (later):** [system-design/high-level-design/01-fundamentals.md](../system-design/high-level-design/01-fundamentals.md)

---

## 7. Low-Level Design — The four OOP ideas

### Why this exists
Object-Oriented Programming (OOP) is the most common way to organize code in Java, C#, TypeScript, Python. Four ideas keep showing up in interviews: **Encapsulation, Abstraction, Inheritance, Polymorphism**. Know them by example, not by definition.

### Each idea in one sentence
- **Encapsulation** — keep data private; expose methods. Outside code can't reach in and break invariants.
- **Abstraction** — show *what* something does, not *how* it does it. Interfaces are the tool.
- **Inheritance** — `Dog extends Animal`. Reuse code from a parent class.
- **Polymorphism** — one method name, many implementations. The caller doesn't know which one runs.

### Smallest working example
```java
// Abstraction: a contract — "payment methods can charge."
interface PaymentMethod {
    void charge(int cents);
}

// Two implementations (polymorphism)
class CardPayment implements PaymentMethod {
    public void charge(int cents) {
        System.out.println("Charging card: " + cents);
    }
}
class UpiPayment implements PaymentMethod {
    public void charge(int cents) {
        System.out.println("Sending UPI request: " + cents);
    }
}

// Caller doesn't care which one (polymorphism in action)
void checkout(PaymentMethod method) {
    method.charge(99_00);  // ₹99 — but to the caller it's just "charge"
}
```

### Drill (2 min)
If we want to add `PayPalPayment` tomorrow, what changes in `checkout(...)`? *(Hint: nothing! That's the whole point of polymorphism — new implementations don't change the caller.)*

**Deep dive (later):** [system-design/low-level-design/01-oop-fundamentals.md](../system-design/low-level-design/01-oop-fundamentals.md)

---

## 8. DSA — Two Sum (warm-up problem)

### The problem
Given an array of integers `nums` and a target integer `target`, return the **indices** of the two numbers that add up to `target`. There's exactly one solution. Don't use the same element twice.

```
Input:  nums = [2, 7, 11, 15], target = 9
Output: [0, 1]    // because nums[0] + nums[1] == 2 + 7 == 9
```

### The naive approach (always start here)
Check every pair with two nested loops. Easy to think of, easy to code, but **O(n²)** time. For n = 10,000 that's 100 million operations — slow.

### The fast approach (hash map)
Walk through the array once. For each number `x`, the *complement* you need is `target - x`. If you've already seen the complement, return both indices. Otherwise remember `x` and its index.

```javascript
function twoSum(nums, target) {
  const seen = new Map();           // value → index
  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (seen.has(complement)) {
      return [seen.get(complement), i];
    }
    seen.set(nums[i], i);
  }
}

twoSum([2, 7, 11, 15], 9);  // → [0, 1]
```

**Why it works:** when you reach index `i`, the map holds every value before `i`. If `target - nums[i]` is in there, you've found the pair.

**Time complexity:** O(n) — one loop. **Space:** O(n) — the map.

### Drill (3 min)
Type this code out by hand once. Trace what `seen` looks like at each step for `nums = [3, 2, 4], target = 6`. *(You should see: i=0 → seen = {3:0}; i=1 → seen = {3:0, 2:1}; i=2 → complement is 2, found in map → return [1, 2].)*

**Deep dive (later):** [dsa/ds-js/arrays-strings.md](../dsa/ds-js/arrays-strings.md) · [dsa/ds-java/arrays-strings.md](../dsa/ds-java/arrays-strings.md)

---

## 9. Design Pattern — Singleton

### Intent
**Make sure a class has exactly one instance**, and give everyone an easy way to get it.

### When you'd use it
- A logger (one place that writes logs)
- A configuration registry (one source of truth for settings)
- A connection pool (one pool shared by the whole app)

### The simplest version (Java)
```java
public class Logger {
    private static Logger instance;        // the one and only

    private Logger() {}                    // private — nobody else can `new` it

    public static Logger getInstance() {
        if (instance == null) instance = new Logger();
        return instance;
    }

    public void log(String msg) {
        System.out.println("[LOG] " + msg);
    }
}

// Use it
Logger.getInstance().log("App started");
```

The `private` constructor blocks `new Logger()` from outside. `getInstance()` creates the object the first time, then returns the same one forever.

### When NOT to use it
If you only need "a single shared object," let your framework (Spring, Angular DI, Node's module cache) handle it. Singletons are global state — they make testing harder. Use them sparingly.

### Drill (1 min)
Why is the constructor `private`? *(Hint: so the rest of the code can't accidentally create a second Logger. The class itself controls how many exist.)*

**Deep dive (later):** [design-patterns/common/creational/singleton.md](../design-patterns/common/creational/singleton.md)

---

## 10. DevOps — Linux essentials

### Why this exists
Almost every server runs Linux. You will SSH into servers, look at files, check who owns what, restart services. The few commands below cover ~80% of day-to-day Linux work.

### File permissions
Every file has an **owner**, a **group**, and **everyone else**. Each can `r`ead, `w`rite, and e`x`ecute. You see them with `ls -l`:

```
-rwxr-x---  1  app  app  1.2K  May 20  deploy.sh
 │└┬┘└┬┘└┬┘
 │ │  │  └── others: no access
 │ │  └───── group: read + execute
 │ └──────── owner: read + write + execute
 └────────── file type: '-' = file, 'd' = directory
```

Change with `chmod`: `chmod 750 deploy.sh` means owner=7 (rwx), group=5 (r-x), others=0 (none). The numbers are r=4, w=2, x=1 added together.

### Processes & signals
```bash
ps aux | grep node           # list processes matching "node"
kill 1234                    # polite ask to stop process 1234 (SIGTERM)
kill -9 1234                 # force-kill (SIGKILL) — last resort
systemctl restart nginx      # restart a service via systemd
journalctl -u nginx -f       # tail nginx logs in real time
```

**SIGTERM** vs **SIGKILL** is the most common interview question:
- `SIGTERM` (signal 15) is a polite request. The app can catch it, finish current work, close connections, then exit. This is what you want.
- `SIGKILL` (signal 9) is the kernel killing the app instantly. The app can't catch or handle it. Use only when SIGTERM doesn't work — you may lose data.

### Drill (2 min)
Your Node.js server is using too much memory and you need to restart it gracefully (let it finish in-flight requests). Which signal? *(Answer: SIGTERM. SIGKILL would drop in-flight requests.)*

**Deep dive (later):** [devops/01-linux-essentials.md](../devops/01-linux-essentials.md)

---

## End-of-day checklist

- [ ] Angular: I can name the three parts of a component (HTML, CSS, TS class)
- [ ] Node.js: I can explain "single-threaded but not blocking" using the chef analogy
- [ ] Spring: I can explain why we don't write `new` for services in Spring
- [ ] MongoDB: I can describe a document as a JSON-like object inside a collection
- [ ] Postgres: I wrote one `CREATE TABLE` and one `INSERT`/`SELECT` pair
- [ ] HLD: I can define latency and throughput in one sentence each
- [ ] LLD: I can list the four OOP ideas and give one example for each
- [ ] DSA: I solved Two Sum and traced through one example by hand
- [ ] DP: I explained why a Singleton's constructor is private
- [ ] DevOps: I know the difference between SIGTERM and SIGKILL

**If you remember just one thing today:** every modern stack (Angular components, Spring beans, Node modules) lets the framework wire pieces together so you can focus on *what* each piece does, not on who *creates* whom.

**Tomorrow:** we go one level deeper on each — modules, ES modules, Spring DI, MongoDB CRUD, Postgres joins, networking, SOLID, hash tables, Factory pattern, and bash scripting.
