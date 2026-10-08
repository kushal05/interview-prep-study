# Day 2 — Modules, joins, and how code is organized

> **Today's goal:** see how big projects are split into pieces (Angular modules, Node modules, Spring beans), and how data spread across tables is brought back together with joins.
>
> **Total time:** ~2 hours. Take a 5-minute break after every 3 sections.

| # | Pillar | Topic | Time |
|---|--------|-------|------|
| 1 | Angular | Modules & Standalone Components | 15m |
| 2 | Node.js | Modules (CommonJS vs ES Modules) | 15m |
| 3 | Spring Boot | Dependency Injection & Beans | 15m |
| 4 | MongoDB | CRUD Operations | 10m |
| 5 | Postgres | Joins (INNER, LEFT) | 10m |
| 6 | HLD | Networking & Protocols (TCP, HTTP, DNS) | 12m |
| 7 | LLD | SOLID Principles | 12m |
| 8 | DSA | Hash Tables (Group Anagrams) | 15m |
| 9 | Design Pattern | Factory Method | 8m |
| 10 | DevOps | Bash & Shell Scripting | 10m |

---

## 1. Angular — Modules & Standalone Components

### Why this exists
A real app has dozens of components. Angular needs to know which ones a given component is allowed to use. Older Angular grouped components into **NgModules** (one big container per feature). New Angular lets each component declare its own dependencies — that's a **standalone component**.

### The idea in plain English
Think of NgModules as a school assembly: every component sits in one classroom (module), and the classroom decides what tools (other components, directives, pipes) are available. That worked, but it meant lots of bookkeeping.

A **standalone component** is like a kid who carries their own backpack. The component declares its imports right inside its own file. No classroom needed. New projects should default to standalone — it's simpler and Angular treats it as the future.

### Smallest working example
```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-search',
  standalone: true,                          // the key flag
  imports: [CommonModule],                   // tools this template uses
  template: `
    <ul>
      <li *ngFor="let item of items">{{ item }}</li>
    </ul>
  `,
})
export class SearchComponent {
  items = ['apples', 'bananas', 'cherries'];
}
```

`CommonModule` is what gives you `*ngFor` and `*ngIf`. Forget to import it and the build complains.

### Drill (2 min)
You use `*ngFor` but forgot `CommonModule` in `imports`. What happens? *(Hint: a build-time error like "ngFor is not a known directive." Angular needs to know where `*ngFor` comes from.)*

**Deep dive (later):** [angular/modules.md](../angular/modules.md)

---

## 2. Node.js — Modules (CommonJS vs ES Modules)

### Why this exists
You can't put all your code in one file. JavaScript needs a way to split code across files and bring it back together. Node has **two** module systems that grew at different times — knowing both saves headaches.

### The idea in plain English
A module is a file that **exports** some things and **imports** others. Two flavors:

- **CommonJS (CJS)** — Node's original system. Uses `require()` and `module.exports`. Loads synchronously.
- **ES Modules (ESM)** — the JavaScript standard, also used by browsers. Uses `import` / `export`. Loads asynchronously, more strict.

Node picks based on file extension (`.cjs` = CJS, `.mjs` = ESM) or the `"type"` field in your `package.json` (`"module"` = ESM, missing or `"commonjs"` = CJS).

New code should default to ESM. CJS still exists everywhere because of legacy.

### Smallest working example
```javascript
// math.mjs (ESM)
export function add(a, b) { return a + b; }

// app.mjs
import { add } from './math.mjs';
console.log(add(2, 3));   // 5

// ---- CJS equivalent ----
// math.cjs
function add(a, b) { return a + b; }
module.exports = { add };

// app.cjs
const { add } = require('./math');
console.log(add(2, 3));
```

Same idea, different keywords. The two systems don't mix freely — that's the painful part.

### Drill (2 min)
Your file is named `math.js` and your `package.json` has `"type": "module"`. Will `require('./math')` work in another file? *(Hint: no — the file is treated as ESM. You'd use `import` instead, or rename the file to `.cjs`.)*

**Deep dive (later):** [nodejs/05-modules-cjs-vs-esm.md](../nodejs/05-modules-cjs-vs-esm.md)

---

## 3. Spring Boot — Dependency Injection & Beans

### Why this exists
Yesterday you saw the IoC container. Today: how does Spring actually find your classes and connect them? The answer is **beans** — objects Spring manages — and **dependency injection** — Spring handing them to whoever asks.

### The idea in plain English
Imagine a kitchen with shared equipment: one oven, one fridge, one mixer. Every recipe (your service) declares "I need an oven and a mixer." The kitchen manager (Spring) hands them over. You never go shopping for your own oven.

Spring finds your classes by **scanning** — at startup, it looks for classes tagged `@Component`, `@Service`, `@Repository`, or `@RestController` and creates one of each. Then for every class that has a constructor needing those types, Spring passes them in.

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

    // Spring sees this constructor and passes the GreetingService bean.
    public HelloController(GreetingService greeting) {
        this.greeting = greeting;
    }

    @GetMapping("/hi/{name}")
    public String hi(@PathVariable String name) {
        return greeting.greet(name);
    }
}
```

No `new GreetingService()` anywhere. Spring created it, and Spring handed it to `HelloController`'s constructor.

### Drill (2 min)
What's the difference between `@Service` and `@Component`? *(Hint: nothing functionally — they're both bean-creating annotations. `@Service` just labels the role as "business logic" for humans reading the code.)*

**Deep dive (later):** [spring-detailed/PART-13-ioc-container-deeply.md](../spring-detailed/PART-13-ioc-container-deeply.md)

---

## 4. MongoDB — CRUD Operations

### Why this exists
Once you have a document database, you need the four basic operations: **C**reate, **R**ead, **U**pdate, **D**elete. Knowing these four covers 90% of daily Mongo work.

### The idea in plain English
Mongo's query language *looks* like JavaScript objects. To find documents you write the shape you're looking for; to update, you describe the change with `$` operators (like `$set`, `$inc`).

The most common gotcha: `updateOne(filter, { name: "K" })` does **not** set the `name` field — it *replaces the entire document*. Always use `$set` unless you really mean "throw out everything else."

### Smallest working example
```javascript
// CREATE
db.users.insertOne({ email: "k@x.com", name: "K", age: 28 });

// READ
db.users.findOne({ email: "k@x.com" });
db.users.find({ age: { $gte: 18 } });           // 18 or older

// UPDATE — always use $set
db.users.updateOne(
  { email: "k@x.com" },
  { $set: { age: 29 } }
);

// DELETE
db.users.deleteOne({ email: "k@x.com" });
```

`$gte` means "greater than or equal to." Other comparisons: `$gt`, `$lt`, `$lte`, `$ne`, `$in`.

### Drill (2 min)
You run `db.users.updateOne({ email: "k@x.com" }, { name: "Kushal" })`. The user's `age` and `email` fields disappear. Why? *(Hint: without `$set`, Mongo treats the second argument as a replacement document. Use `{ $set: { name: "Kushal" } }`.)*

**Deep dive (later):** [mongodb/02-crud-operations.md](../mongodb/02-crud-operations.md)

---

## 5. Postgres — Joins (INNER, LEFT)

### Why this exists
Relational data is split across tables to avoid duplication. A `users` table and an `orders` table both exist; to answer "who placed which order," you have to *join* them. Two joins cover 95% of cases: `INNER` and `LEFT`.

### The idea in plain English
Imagine two stacks of cards: one with users, one with orders. You want pairs where they match on `user_id`.

- **INNER JOIN** — only keep pairs where both sides match. Users with no orders disappear; orders with no user disappear.
- **LEFT JOIN** — keep *every* row from the left table. If the right side has no match, you get `NULL` over there.

Use LEFT when you want to include "the missing case" — like listing every user even if they've never ordered anything.

### Smallest working example
```sql
-- Tables
-- users(id, name)           orders(id, user_id, total)
-- 1, Alice                  101, 1, 100
-- 2, Bob                    102, 1, 200
-- 3, Carol                  103, 2, 150

-- INNER JOIN — only matched pairs
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON o.user_id = u.id;
-- Alice 100, Alice 200, Bob 150     (Carol is missing)

-- LEFT JOIN — every user, even without orders
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
-- Alice 100, Alice 200, Bob 150, Carol NULL
```

The `ON` clause is the join condition — usually a foreign key matching a primary key.

### Drill (2 min)
You want a list of users who have never placed an order. How? *(Hint: `LEFT JOIN` users with orders, then filter `WHERE o.id IS NULL` — that catches the rows where there was no matching order.)*

**Deep dive (later):** [postgres/03-joins.md](../postgres/03-joins.md)

---

## 6. High-Level Design — Networking & Protocols

### Why this exists
Every request your app makes — a database query, an API call, a web page load — travels through a stack of network protocols. You don't need to know every byte, but you need to know the names so you can read error messages and architecture diagrams.

### The idea in plain English
Think of sending a package internationally:
- **IP** is the address (where it goes).
- **TCP** is the careful courier — guarantees the package arrives in order, every piece intact. Slower setup, reliable.
- **UDP** is the postcard — fast, but it might get lost or arrive out of order.
- **HTTP** is the message inside (the request: "give me /users/123").
- **DNS** is the phone book — turns names like `google.com` into IP addresses like `142.250.x.x`.

When you visit a website: DNS lookup → TCP connection (3-way handshake: SYN, SYN-ACK, ACK) → TLS handshake (for HTTPS) → HTTP request → HTTP response.

### A flow worth picturing
```
You type "example.com" in browser:

1. DNS lookup        →  resolver returns 1.2.3.4
2. TCP SYN/SYN-ACK   →  3-way handshake with 1.2.3.4
3. TLS handshake     →  agree on encryption keys
4. HTTP GET /        →  send the actual request
5. HTTP 200 OK       →  server returns HTML
```

That's why the *first* request to a new domain feels slow — five things have to happen before any data flows.

### A few status codes to recognize
- `200` OK · `201` Created · `204` No Content
- `400` Bad Request · `401` Unauthorized (no creds) · `403` Forbidden (creds but denied) · `404` Not Found
- `500` Server Error · `502` Bad Gateway · `503` Service Unavailable

### Drill (2 min)
Why is the *first* HTTPS request to a new website noticeably slow, but later requests to the same site are fast? *(Hint: DNS, TCP handshake, and TLS handshake all happen up front. Later requests reuse the open connection — this is called keep-alive.)*

**Deep dive (later):** [system-design/high-level-design/02-networking-and-protocols.md](../system-design/high-level-design/02-networking-and-protocols.md)

---

## 7. Low-Level Design — SOLID Principles

### Why this exists
Code that grows over years often turns into a mess — small changes break unrelated things, tests are impossible, every feature is risky. **SOLID** is five rules that, applied loosely, keep code easy to change. You don't need to memorize the formal definitions; you need to recognize the smells.

### The five letters in one line each
- **S — Single Responsibility:** one class, one reason to change. A `User` class that handles DB saves *and* email *and* validation will be edited by three people for three reasons.
- **O — Open/Closed:** add new behavior by adding new code, not editing existing code. Big `if/else` chains for "type" are the smell.
- **L — Liskov Substitution:** a subclass should behave like its parent. If `Square extends Rectangle` but breaks when you `setWidth`, you've violated it.
- **I — Interface Segregation:** don't force a class to implement methods it doesn't need. Split big interfaces into small ones.
- **D — Dependency Inversion:** depend on interfaces, not concrete classes. Lets you swap implementations for tests.

### Smallest working example
```java
// BAD — violates Open/Closed. Adding a new method = editing this class.
public double shipCost(Order o, String method) {
    if (method.equals("standard")) return o.weight() * 0.5;
    if (method.equals("express"))  return o.weight() * 1.2 + 5;
    // every new shipping option means editing this file
    throw new IllegalArgumentException(method);
}

// GOOD — open for extension, closed for modification
public interface ShippingStrategy {
    double cost(Order o);
}
public class StandardShipping implements ShippingStrategy {
    public double cost(Order o) { return o.weight() * 0.5; }
}
public class ExpressShipping implements ShippingStrategy {
    public double cost(Order o) { return o.weight() * 1.2 + 5; }
}
// Now adding "OvernightShipping" = adding one new file. No edits.
```

SOLID and Dependency Injection go hand in hand. Once you stop saying `new SomeService()`, swapping implementations becomes free.

### Drill (2 min)
A method has 8 `else if (animal instanceof Dog) ... else if (animal instanceof Cat) ...` branches. Which SOLID principle is being violated? *(Hint: Open/Closed. Replace the branches with a polymorphic method like `animal.speak()` so adding a new animal doesn't edit this method.)*

**Deep dive (later):** [system-design/low-level-design/02-solid-principles.md](../system-design/low-level-design/02-solid-principles.md)

---

## 8. DSA — Hash Tables (Group Anagrams)

### The problem
Given an array of strings, group strings that are anagrams of each other (same letters, different order).

Input: `["eat","tea","tan","ate","nat","bat"]`
Output: `[["eat","tea","ate"], ["tan","nat"], ["bat"]]`

### The idea
Two anagrams share the same letters. If you **sort the letters** of each string, all anagrams produce the same sorted key. So:

1. For each string, compute a key (the sorted letters).
2. Group strings under their key in a hash map.
3. Return the groups.

A hash map (HashMap in Java, Map in JS, `dict` in Python) is just a "value → bucket" lookup with average O(1) operations.

### Smallest working example
```javascript
function groupAnagrams(strs) {
    const groups = new Map();
    for (const s of strs) {
        const key = [...s].sort().join('');     // canonical anagram key
        if (!groups.has(key)) groups.set(key, []);
        groups.get(key).push(s);
    }
    return [...groups.values()];
}

groupAnagrams(["eat","tea","tan","ate","nat","bat"]);
// → [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

**Why it works:** "eat", "tea", "ate" all sort to "aet". The map naturally clusters them under that key.

**Time:** O(n · k log k) where n = number of strings, k = average length (the sort dominates).

### Drill (3 min)
Trace what `groups` looks like after processing `["eat", "tea", "bat"]`. *(After "eat": `{"aet": ["eat"]}`. After "tea": `{"aet": ["eat","tea"]}`. After "bat": `{"aet": ["eat","tea"], "abt": ["bat"]}`.)*

**Deep dive (later):** [dsa/ds-js/hash-tables.md](../dsa/ds-js/hash-tables.md) · [dsa/ds-java/hash-tables.md](../dsa/ds-java/hash-tables.md)

---

## 9. Design Pattern — Factory Method

### Intent
**Don't create objects with `new` directly.** Have a method (the factory) decide which concrete class to instantiate. Useful when the choice depends on input or configuration.

### When you'd use it
- You read a config that says `"provider": "stripe"` and want a `StripeProcessor`; tomorrow it might say `"paypal"`.
- You want one place to make "which class" decisions — not a `switch` scattered across the codebase.

### Smallest working example
```java
public interface PaymentProcessor {
    void process(double amount);

    // Static factory method
    static PaymentProcessor of(String provider) {
        return switch (provider) {
            case "stripe"   -> new StripeProcessor();
            case "paypal"   -> new PayPalProcessor();
            default -> throw new IllegalArgumentException("Unknown: " + provider);
        };
    }
}

// Usage — the caller doesn't know which class it gets
PaymentProcessor p = PaymentProcessor.of("stripe");
p.process(99.99);
```

The caller asks for "a payment processor" and gets back the right one. The decision is centralized.

### Drill (1 min)
Where in Java's standard library have you seen this pattern? *(Hint: `List.of(1,2,3)`, `Calendar.getInstance()`, `Optional.of(value)`. They're all static factory methods returning some implementation.)*

**Deep dive (later):** [design-patterns/common/creational/factory-method.md](../design-patterns/common/creational/factory-method.md)

---

## 10. DevOps — Bash & Shell Scripting

### Why this exists
Deploy scripts, CI jobs, incident playbooks — they're nearly always bash. You don't need to be a wizard; you need 5 commands and the "safe mode" header.

### The idea in plain English
Bash is a glue language: chain commands with `|` (pipes), check return codes (`0` = success, anything else = failure), and use shell variables. The most important habit: **start every script with safe mode** so a single command failing doesn't silently let the rest run.

`set -e` — stop on the first error.
`set -u` — error if you use an unset variable (catches typos).
`set -o pipefail` — fail if any command in a pipeline fails, not just the last.

Combined: `set -euo pipefail` at the top of every script.

### Smallest working example
```bash
#!/usr/bin/env bash
set -euo pipefail              # safe mode — fail fast on errors

INPUT="${1:?usage: $0 <file>}" # error out if no argument given
TMP="$(mktemp)"                # safe temp file
trap 'rm -f "$TMP"' EXIT       # always clean up, even on error

if [[ ! -r "$INPUT" ]]; then
    echo "cannot read $INPUT" >&2
    exit 2
fi

grep -v '^#' "$INPUT" | sort -u > "$TMP"
count=$(wc -l < "$TMP")
echo "Found $count unique non-comment lines."
```

`"${1:?...}"` is "use $1, or exit with this error message if it's missing." `trap '... ' EXIT` runs on script exit (success or failure) — perfect for cleanup.

### Drill (2 min)
Why does `set -e` alone not catch a failure inside `grep foo bar.txt | sort`? *(Hint: by default, only the last command's exit code is checked. `set -o pipefail` makes the pipeline fail if any command in it fails. That's why we use all three together.)*

**Deep dive (later):** [devops/02-bash-and-shell-scripting.md](../devops/02-bash-and-shell-scripting.md)

---

## End-of-day checklist

- [ ] Angular: I can explain what `standalone: true` lets you skip
- [ ] Node.js: I can name two ways Node decides if a file is CJS or ESM
- [ ] Spring: I can explain how Spring finds my `@Service` classes
- [ ] MongoDB: I know why `$set` matters in `updateOne`
- [ ] Postgres: I can describe what LEFT JOIN gives you that INNER JOIN doesn't
- [ ] HLD: I can list the steps from typing a URL to seeing the page
- [ ] LLD: I can name the five SOLID letters and one example each
- [ ] DSA: I solved Group Anagrams using a hash map
- [ ] DP: I gave one Java stdlib example of Factory Method
- [ ] DevOps: I know what `set -euo pipefail` does

**If you remember just one thing today:** modules (Angular, Node) and beans (Spring) all solve the same problem — making it explicit which pieces a chunk of code is allowed to use, so you can swap pieces without rewriting the rest.

**Tomorrow:** data binding, semver and lockfiles, bean lifecycles, Mongo query operators, recursive CTEs, SQL vs NoSQL, UML class diagrams, stacks for matching brackets, Abstract Factory, and Git fundamentals.
