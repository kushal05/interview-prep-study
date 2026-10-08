# Node.js Design Patterns

Essential design patterns for Node.js development, organized by category with practical examples and interview tips.

---

## Creational Patterns

### 1. Singleton

**What:** Ensures only one instance of a class exists throughout the application.

**When to use:** Database connections, configuration objects, caching services, logging.

```js
// db.js - Node.js modules are cached, so this naturally creates a singleton
let instance = null;

class Database {
  constructor() {
    if (instance) return instance;
    instance = this;
    this.connection = null; // initialize connection here
  }

  connect(url) {
    this.connection = url; // simplified
    return this;
  }
}

module.exports = new Database();
```

**Pros:** Shared state, memory-efficient, controlled access
**Cons:** Hidden dependencies, harder to test in isolation (use dependency injection instead)

> **Interview Tip:** In Node.js, `require()` caches modules by default, making every module export a natural singleton. Explain why you would still use the explicit pattern (e.g., ensuring single instance across different import paths).

---

### 2. Factory

**What:** Creates objects without exposing instantiation logic. Returns different types based on input.

**When to use:** Creating different user roles, database adapters, notification services.

```js
function createUser(role) {
  const permissions = {
    admin: ['read', 'write', 'delete'],
    editor: ['read', 'write'],
    guest: ['read'],
  };
  return { role, permissions: permissions[role] || permissions.guest };
}

const admin = createUser('admin');
const guest = createUser('guest');
```

**Pros:** Decouples creation from usage, easy to extend
**Cons:** Can be overkill for simple objects

> **Interview Tip:** Factories shine when object creation involves complex logic or when the type of object depends on runtime conditions.

---

### 3. Builder

**What:** Constructs complex objects step-by-step using method chaining.

**When to use:** Query builders (Knex, Prisma), HTTP request construction, configuration objects.

```js
class QueryBuilder {
  constructor() { this.query = {}; }
  table(name) { this.query.table = name; return this; }
  where(condition) { this.query.where = condition; return this; }
  limit(n) { this.query.limit = n; return this; }
  build() {
    return `SELECT * FROM ${this.query.table} WHERE ${this.query.where} LIMIT ${this.query.limit}`;
  }
}

const sql = new QueryBuilder().table('users').where('active = true').limit(10).build();
```

**Pros:** Readable fluent API, separates construction from representation
**Cons:** More boilerplate code

> **Interview Tip:** Real-world example - Knex.js uses the builder pattern: `knex('users').where('id', 1).select('name')`.

---

## Structural Patterns

### 4. Module

**What:** Encapsulates related code into separate files with explicit exports.

**When to use:** Every Node.js file. This is the fundamental organizational pattern.

```js
// CommonJS (Node.js default)
const { add } = require('./math');

// ES Modules (modern)
import { add } from './math.js';
```

**Pros:** Encapsulation, reusability, clear dependencies
**Cons:** Circular dependencies can cause issues

> **Interview Tip:** Know the differences: CommonJS is synchronous and uses `require()`; ESM is async and uses `import/export`. CommonJS wraps modules in a function; ESM has true static analysis.

---

### 5. Proxy

**What:** A wrapper that controls access to another object.

**When to use:** Access control, lazy loading, logging, input validation, rate limiting.

```js
const user = { name: 'Alice', age: 30, _password: 'secret' };

const safeUser = new Proxy(user, {
  get(target, prop) {
    if (prop.startsWith('_')) throw new Error('Access denied');
    return target[prop];
  },
  set(target, prop, value) {
    if (prop === 'age' && typeof value !== 'number') {
      throw new TypeError('Age must be a number');
    }
    target[prop] = value;
    return true;
  }
});

safeUser.name;      // "Alice"
safeUser._password;  // Error: Access denied
```

**Pros:** Powerful, transparent interception layer
**Cons:** Performance overhead, can obscure debugging

> **Interview Tip:** Proxies are used in frameworks like Vue 3 for reactivity. Mention real-world uses: validation, property access logging, lazy initialization.

---

### 6. Decorator

**What:** Adds behavior to a function or object without modifying its source code.

**When to use:** Logging, caching, authorization checks, retry logic.

```js
function withLogging(fn) {
  return function(...args) {
    console.log(`Calling ${fn.name} with`, args);
    const result = fn(...args);
    console.log(`Result:`, result);
    return result;
  };
}

function add(a, b) { return a + b; }
const loggedAdd = withLogging(add);
loggedAdd(2, 3); // Logs: Calling add with [2, 3] -> Result: 5

// Practical: retry decorator for API calls
function withRetry(fn, retries = 3) {
  return async function(...args) {
    for (let i = 0; i < retries; i++) {
      try { return await fn(...args); }
      catch (err) { if (i === retries - 1) throw err; }
    }
  };
}
```

**Pros:** Composable, non-invasive, follows open-closed principle
**Cons:** Deeply nested decorators become hard to debug

> **Interview Tip:** Decorators are essentially higher-order functions. TypeScript/NestJS use `@Decorator` syntax which compiles to this pattern.

---

## Behavioral Patterns

### 7. Observer (EventEmitter)

**What:** Objects subscribe to events and get notified when those events occur.

**When to use:** Real-time features (chat, notifications), logging, decoupled module communication.

```js
const EventEmitter = require('events');

class OrderService extends EventEmitter {
  placeOrder(order) {
    // process order...
    this.emit('orderPlaced', order);
  }
}

const service = new OrderService();
service.on('orderPlaced', (order) => console.log('Send email for', order));
service.on('orderPlaced', (order) => console.log('Update inventory for', order));
service.placeOrder({ id: 1, item: 'Book' });
```

**Pros:** Loose coupling, easy to add new listeners without changing emitter
**Cons:** Hard to trace event flow in large systems, potential memory leaks (too many listeners)

> **Interview Tip:** Know the difference between `on` (persistent) and `once` (single-use). Also: `emitter.setMaxListeners(n)` to avoid the memory leak warning.

---

### 8. Middleware

**What:** Functions that form a processing pipeline, each able to modify the request/response or pass control forward.

**When to use:** Express.js request handling, any pipeline processing (auth, logging, validation, error handling).

```js
// Express middleware chain
const express = require('express');
const app = express();

// Logging middleware
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// Auth middleware
function requireAuth(req, res, next) {
  if (!req.headers.authorization) return res.status(401).json({ error: 'Unauthorized' });
  next();
}

// Error-handling middleware (4 params)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something broke' });
});
```

**Pros:** Clean separation of concerns, reusable, composable
**Cons:** Order-dependent (bugs if misordered), hard to debug long chains

> **Interview Tip:** If you do not call `next()`, the request hangs. Error-handling middleware must have exactly 4 parameters. These are top questions.

---

### 9. Strategy

**What:** Select an algorithm or behavior at runtime without changing the calling code.

**When to use:** Authentication methods, payment processors, sorting algorithms, notification channels.

```js
const strategies = {
  jwt: (token) => { /* verify JWT */ return { userId: 1 }; },
  apiKey: (key) => { /* lookup API key */ return { userId: 2 }; },
  oauth: (code) => { /* exchange OAuth code */ return { userId: 3 }; },
};

function authenticate(type, credential) {
  const strategy = strategies[type];
  if (!strategy) throw new Error(`Unknown auth type: ${type}`);
  return strategy(credential);
}

authenticate('jwt', 'eyJhbGc...');
```

**Pros:** Eliminates long if/else chains, easy to add new strategies, open-closed principle
**Cons:** Requires upfront structure

> **Interview Tip:** Passport.js is a real-world example - it uses strategies for different auth providers.

---

### 10. Command

**What:** Encapsulates an action as an object, enabling queuing, undo/redo, and logging.

**When to use:** Job queues (Bull, BullMQ), undo/redo, task scheduling, audit logging.

```js
class Command {
  execute() { throw new Error('Implement execute()'); }
  undo() { throw new Error('Implement undo()'); }
}

class AddItemCommand extends Command {
  constructor(cart, item) {
    super();
    this.cart = cart;
    this.item = item;
  }
  execute() { this.cart.push(this.item); }
  undo() { this.cart.pop(); }
}

// Usage with command history
const cart = [];
const history = [];

const cmd = new AddItemCommand(cart, 'Book');
cmd.execute();      // cart: ['Book']
history.push(cmd);

cmd.undo();         // cart: []
```

**Pros:** Queueable, undoable, decouples sender from receiver
**Cons:** More boilerplate for simple operations

> **Interview Tip:** Job queues like Bull use this pattern - each job is a command object with execute logic, retry policies, and failure handling.

---

## Async Patterns

### 11. Callback

**What:** Pass a function as an argument to be executed later (typically after an async operation).

**When to use:** Legacy Node.js APIs, event handlers. Prefer Promises/async-await for new code.

```js
// Node.js error-first callback convention
const fs = require('fs');

fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) return console.error(err);
  console.log(data);
});

// Converting callback to Promise
const readFileAsync = (path) => new Promise((resolve, reject) => {
  fs.readFile(path, 'utf8', (err, data) => err ? reject(err) : resolve(data));
});

// Or simply: const { readFile } = require('fs').promises;
```

**Pros:** Simple, native to Node.js
**Cons:** Callback hell (deeply nested), difficult error propagation

> **Interview Tip:** Be ready to convert a callback-based function to a Promise. Know about `util.promisify()` as a shortcut.

---

## Quick Reference

| Pattern | Category | Use Case | Key Concept |
|---------|----------|----------|-------------|
| Singleton | Creational | DB connections, config | One instance |
| Factory | Creational | Object creation by type | Returns new instance |
| Builder | Creational | Complex object construction | Method chaining |
| Module | Structural | Code organization | Exports/imports |
| Proxy | Structural | Access control, validation | `new Proxy()` |
| Decorator | Structural | Add behavior (logging, caching) | Function wrappers |
| Observer | Behavioral | Events, pub/sub | `EventEmitter` |
| Middleware | Behavioral | Request pipeline | `req, res, next()` |
| Strategy | Behavioral | Runtime behavior selection | Object lookup |
| Command | Behavioral | Job queues, undo/redo | Encapsulated actions |
| Callback | Async | Legacy async operations | Error-first convention |
