# Node.js Backend Developer - Interview Study Plan

A structured roadmap covering everything you need for a Node.js backend developer interview.

---

## 1. Core JavaScript (ES6+)

### Language Fundamentals
- [ ] **Variables**: `let` (block-scoped), `const` (block-scoped, immutable binding), `var` (function-scoped, hoisted)
- [ ] **Data Types**: Primitives (string, number, boolean, null, undefined, symbol, bigint) vs Reference types (objects, arrays, functions)
- [ ] **Type coercion**: `1 + "1"` = `"11"` (string wins), `1 - "1"` = `0` (number wins)
- [ ] **Scope**: Global > Function (`var`) > Block (`let`, `const`)
- [ ] **Closures**: A function that remembers its outer scope even after the outer function returns
- [ ] **`this` keyword**: Determined by call site; arrow functions inherit from lexical scope
- [ ] **Prototypes**: Objects delegate property lookup via the prototype chain

> **Interview Tip:** `typeof null === "object"` is a classic trick question. Know why (legacy JS bug).

```js
// Closure example - very commonly asked
function counter() {
  let count = 0;
  return function() { return ++count; };
}
const inc = counter();
inc(); // 1
inc(); // 2
```

### Modern JS Features
- [ ] **Destructuring**: `const {name, age} = person;` / `const [a, b] = arr;`
- [ ] **Spread/Rest**: `[...arr, 4]` / `function sum(...nums) {}`
- [ ] **Optional chaining**: `user?.profile?.email`
- [ ] **Nullish coalescing**: `value ?? "default"` (only null/undefined, not falsy)

### Async Programming
- [ ] **Event loop**: Call stack, microtask queue (Promises), macrotask queue (setTimeout)
- [ ] **Promises**: pending -> fulfilled/rejected (immutable once settled)
- [ ] **async/await**: Syntactic sugar over Promises; use try/catch for errors
- [ ] **Promise utilities**:
  - `Promise.all` - fail-fast, all must succeed
  - `Promise.allSettled` - waits for all, never rejects
  - `Promise.race` - first to settle (fulfill or reject)
  - `Promise.any` - first to fulfill (ignores rejections)

> **Key Interview Point:** Know the event loop order: call stack -> microtasks (Promise.then) -> macrotasks (setTimeout). This determines execution order.

### Advanced JS
- [ ] **Functional programming**: Pure functions, `map`, `reduce`, `filter` (immutable)
- [ ] **Currying**: `f(a, b)` becomes `f(a)(b)` - useful for partial application
- [ ] **Debounce vs Throttle**: Debounce waits for pause; Throttle limits frequency
- [ ] **Memory leaks**: Unremoved event listeners, global variables, forgotten closures

---

## 2. TypeScript for Node.js

- [ ] **Interfaces vs Types**: Interfaces are extendable (`extends`), types are more flexible (unions, intersections)
- [ ] **Generics**: `function wrap<T>(value: T): T[] { return [value]; }`
- [ ] **Utility types**: `Partial<T>`, `Pick<T, K>`, `Omit<T, K>`, `Record<K, V>`
- [ ] **Type guards**: Narrow types with `typeof`, `in`, `instanceof`, or custom guards
- [ ] **Decorators**: Metadata annotations (used in NestJS)

> **Interview Tip:** Be ready to write type-safe Express handlers with proper request/response types.

---

## 3. Core Node.js

### Event Loop & Libuv
- [ ] Single-threaded JS execution, but libuv uses a thread pool (default 4) for FS, crypto, DNS
- [ ] Phases: timers -> pending callbacks -> idle/prepare -> poll -> check -> close

### Core Modules
- [ ] **fs** - File operations (prefer async: `fs.promises`)
- [ ] **http** - Build servers (`http.createServer`)
- [ ] **events** - Custom EventEmitters
- [ ] **stream** - Process large data in chunks (Readable, Writable, Transform, Duplex)
- [ ] **child_process** - Spawn external processes (`exec`, `spawn`, `fork`)
- [ ] **cluster** - Utilize multiple CPU cores

### Error Handling
- [ ] Sync: `try/catch` with `throw new Error()`
- [ ] Async: `.catch()` on promises or `try/catch` in async functions
- [ ] Global safety nets: `process.on('uncaughtException')`, `process.on('unhandledRejection')`

> **Key Interview Point:** Never silently swallow errors. Always log and handle gracefully. `uncaughtException` should log and exit - do not try to recover.

---

## 4. Building APIs

- [ ] **Express.js**: Routing, middleware chain, error-handling middleware
- [ ] **Middleware pattern**: `(req, res, next) => {}` - order matters!
- [ ] **REST best practices**: Proper HTTP verbs, status codes (200, 201, 400, 401, 403, 404, 500), resource naming
- [ ] **GraphQL**: Schema-first design, resolvers, flexible querying (reduces over-fetching)
- [ ] **Input validation**: Use libraries like Joi or Zod
- [ ] **API versioning**: URL (`/v1/`) or header-based

---

## 5. Databases & Persistence

- [ ] **SQL**: ACID properties, transactions, indexes (B-tree), joins, normalization
- [ ] **NoSQL**: Document (MongoDB), Key-value (Redis), flexible schema, horizontal scaling
- [ ] **ORMs**: Prisma (type-safe), Sequelize, TypeORM
- [ ] **Best practices**: Migrations, avoid N+1 queries, pagination (cursor > offset for large datasets)

> **Interview Tip:** Be able to explain when you would choose SQL vs NoSQL for a given use case.

---

## 6. Authentication & Security

- [ ] **JWT**: Stateless auth, access + refresh token pattern, store in httpOnly cookies
- [ ] **OAuth2**: Authorization delegation (Google, GitHub login)
- [ ] **bcrypt**: Hash passwords with salt (never store plaintext)
- [ ] **CORS**: Whitelist trusted origins
- [ ] **Helmet.js**: Security headers (X-Frame-Options, CSP, etc.)
- [ ] **Common attacks**: SQL injection (parameterize queries), XSS (sanitize output), CSRF (tokens)

---

## 7. Performance & Scaling

- [ ] **Clustering**: `cluster` module or PM2 for multi-core utilization
- [ ] **Caching**: Redis for frequently accessed data; cache invalidation strategies
- [ ] **Message queues**: Kafka (high throughput, ordered) / RabbitMQ (flexible routing) for async workloads
- [ ] **Profiling**: `clinic flame` for CPU, `--inspect` with Chrome DevTools, `perf_hooks`
- [ ] **Load balancing**: Nginx reverse proxy, round-robin, least connections

---

## 8. Testing

- [ ] **Unit tests**: Jest or Mocha - test isolated functions
- [ ] **Integration tests**: Supertest with Express - test API endpoints
- [ ] **E2E tests**: Simulate real user flows against running server
- [ ] **Mocks & Stubs**: Isolate dependencies (DB, external APIs)
- [ ] **Coverage**: Aim for meaningful coverage, not 100%

---

## 9. DevOps & Deployment

- [ ] **Docker**: Multi-stage builds, `.dockerignore`, small images (alpine)
- [ ] **PM2**: Process manager with clustering, auto-restart, log management
- [ ] **Logging**: Structured JSON logs with Pino (fast) or Winston
- [ ] **Monitoring**: Prometheus + Grafana for metrics, OpenTelemetry for tracing
- [ ] **CI/CD**: GitHub Actions, automated testing and deployment

---

## 10. System Design

- [ ] **Monolith vs Microservices**: Start monolith, split when team/scale demands it
- [ ] **Pub/Sub**: Kafka, Redis Pub/Sub for event-driven architecture
- [ ] **Caching strategies**: Cache-aside, write-through, write-behind
- [ ] **Database scaling**: Replication (read replicas), sharding (horizontal partitioning)
- [ ] **CAP theorem**: In network partition, choose Consistency (CP) or Availability (AP)
- [ ] **Rate limiting**: Token bucket, sliding window

---

## 11. Advanced Topics

- [ ] **WebSockets**: Real-time bidirectional communication (Socket.IO or `ws`)
- [ ] **Streams**: Process files larger than available memory without loading fully
- [ ] **gRPC**: Binary protocol, faster than REST, great for service-to-service
- [ ] **Multi-tenancy**: Database-per-tenant vs shared DB with tenant column
- [ ] **Serverless**: AWS Lambda, cold starts, stateless design

---

## 12. Behavioral & Soft Skills

- [ ] Explain trade-offs clearly (e.g., "We chose X because...")
- [ ] Prepare 2-3 short stories: scaling challenge, debugging a tough bug, improving performance
- [ ] Agile/Scrum experience: sprints, standups, retrospectives
- [ ] Conflict resolution and mentoring examples

---

## Quick Reference Mindmap

```
Node.js Backend Prep
|-- JavaScript (ES6+)
|   |-- Fundamentals (scope, closures, this, prototypes)
|   |-- Modern JS (destructuring, spread, async/await)
|   |-- Advanced (FP, currying, memory leaks, event loop)
|
|-- TypeScript (types, generics, decorators, type guards)
|
|-- Node.js Core
|   |-- Event Loop & Libuv
|   |-- Core Modules (fs, http, events, streams, cluster)
|   |-- Error Handling
|
|-- API Development (Express, REST, GraphQL)
|-- Databases (SQL, NoSQL, ORMs)
|-- Security (JWT, OAuth2, bcrypt, CORS)
|-- Performance (clustering, caching, queues, profiling)
|-- Testing (unit, integration, E2E, mocks)
|-- DevOps (Docker, PM2, logging, monitoring, CI/CD)
|-- System Design (microservices, CAP, caching, sharding)
|-- Advanced (WebSockets, streams, gRPC, serverless)
```
