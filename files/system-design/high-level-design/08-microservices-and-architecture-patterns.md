# Microservices & Architecture Patterns

> **Learning goal:** Understand architectural styles, when to use each, and the patterns that make microservices work.

---

## Before You Begin — Plain-English Setup

- **A service** = a program that runs on a server and offers an API (a set of endpoints other programs can call).
- **A monolith** = ONE big service that contains all the features (users, orders, payments, emails) in one codebase, one deployment.
- **Microservices** = a style where each feature is its own small service with its own codebase and database. They talk to each other over the network.
- **Decoupling** = reducing how tightly two things depend on each other. Loosely coupled services can change independently.
- **Coupling** = the opposite — strong dependency. Tight coupling is hard to maintain at scale.
- **API contract** = the agreed-upon shape of requests and responses between two services (e.g., a `.proto` file or an OpenAPI spec).
- **A sidecar** = a small helper container that runs alongside your service, taking care of logging, encryption, etc., so your code doesn't have to.
- **A service mesh** = a network layer (Istio, Linkerd) that handles service-to-service communication (mTLS, retries, observability) automatically using sidecars.
- **mTLS (mutual TLS)** = both client AND server prove their identity with certificates. Standard for service-to-service security.
- **gRPC** = Google's high-performance RPC framework. Uses HTTP/2 + binary "Protocol Buffers" (`.proto` files) instead of JSON.
- **Protobuf (Protocol Buffers)** = a compact binary format for sending structured data. Smaller and faster than JSON, but not human-readable.
- **CQRS** = Command Query Responsibility Segregation. Separate code paths for "write" (commands) and "read" (queries). Defined in section 4.
- **Saga** = a way to do "transactions" that span multiple services. Defined in section 5 of file 09 too.

---

## 1. Monolith vs Microservices

### Monolith
```
┌─────────────────────────────────────────┐
│              Single Application          │
│  ┌──────┐ ┌───────┐ ┌────────┐ ┌─────┐ │
│  │ Auth │ │ Users │ │ Orders │ │ Pay │ │
│  └──────┘ └───────┘ └────────┘ └─────┘ │
│          Shared Database                 │
│         ┌──────────────┐                │
│         │  PostgreSQL  │                │
│         └──────────────┘                │
└─────────────────────────────────────────┘
One codebase, one deployment, one database
```

### Microservices
```
┌────────┐  ┌────────┐  ┌─────────┐  ┌─────────┐
│  Auth  │  │ Users  │  │ Orders  │  │ Payment │
│Service │  │Service │  │ Service │  │ Service │
└───┬────┘  └───┬────┘  └────┬────┘  └────┬────┘
    │           │            │            │
┌───┴──┐   ┌───┴──┐   ┌────┴───┐   ┌────┴───┐
│Redis │   │Postgres│  │MongoDB │   │Postgres│
└──────┘   └───────┘   └────────┘   └────────┘
Each service: own codebase, own deployment, own database
```

### Comparison
| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Complexity | Simple at start | Complex from start |
| Deployment | All-or-nothing | Independent per service |
| Scaling | Scale entire app | Scale individual services |
| Technology | One tech stack | Different tech per service |
| Data consistency | ACID (easy) | Distributed transactions (hard) |
| Team structure | One team | Team per service |
| Debugging | Easy (one process) | Hard (distributed tracing) |
| Latency | In-process calls (ns) | Network calls (ms) |
| Best for | Small teams, early stage | Large teams, proven domain |

### The Migration Path
```
Start: Monolith (understand your domain first)
  │
  ▼
Modular Monolith (separate modules, shared DB)
  │
  ▼
Strangler Fig Pattern (extract modules one-by-one)
  │
  ▼
Microservices (when team size & scale justify it)
```

---

## 2. Service Communication

### Synchronous (Request-Response)
```
REST (HTTP/JSON):
Order Service → GET http://user-service/users/123 → Response

gRPC (HTTP/2 + Protobuf):
Order Service → gRPC call → User Service → Response
✓ 10x faster than REST (binary protocol, HTTP/2 multiplexing)
✓ Strongly typed contracts (.proto files)
✓ Bidirectional streaming
Best for: Service-to-service communication
```

### Asynchronous (Event-Driven)
```
Order Service → [Kafka: order-created] → Payment Service
                                       → Notification Service
                                       → Analytics Service

✓ Loose coupling (services don't know about each other)
✓ Better resilience (consumer can be temporarily down)
✓ Better scalability (consumers process at their own pace)
✗ Harder to debug, eventual consistency
```

### When to Use Each
```
Sync (REST/gRPC): User-facing requests that need immediate response
  "Get user profile" → must return data NOW

Async (Events): Background processing, fan-out, eventual consistency
  "Order placed" → payment, email, inventory can process async
```

---

## 3. Key Architecture Patterns

### API Gateway Pattern
```
Mobile App ──→ ┌────────────┐ ──→ User Service
Web App   ──→ │ API Gateway │ ──→ Order Service
3rd Party ──→ └────────────┘ ──→ Payment Service

Gateway handles: auth, rate limiting, routing, request aggregation
```

### Backend for Frontend (BFF)
```
Mobile App ──→ [Mobile BFF] ──→ Services
Web App    ──→ [Web BFF]    ──→ Services

Why? Mobile needs compact responses, web needs richer data.
Each BFF tailors API responses for its client.
```

### Service Mesh
```
┌─────────────────┐        ┌─────────────────┐
│  Service A       │        │  Service B       │
│  ┌────────────┐ │        │  ┌────────────┐ │
│  │ App Code   │ │        │  │ App Code   │ │
│  └─────┬──────┘ │        │  └─────┬──────┘ │
│  ┌─────▼──────┐ │◄─────►│  ┌─────▼──────┐ │
│  │Sidecar Proxy│ │ mTLS  │  │Sidecar Proxy│ │
│  │ (Envoy)    │ │        │  │ (Envoy)    │ │
│  └────────────┘ │        │  └────────────┘ │
└─────────────────┘        └─────────────────┘
         │                          │
         └────── Control Plane ─────┘
                  (Istio/Linkerd)

What the mesh handles (so your code doesn't have to):
- mTLS encryption between services
- Load balancing
- Circuit breaking
- Retries with backoff
- Observability (metrics, traces, logs)
- Traffic splitting (canary, A/B)
```

---

## 4. CQRS (Command Query Responsibility Segregation)

### The Pattern
```
Traditional: Same model for reads AND writes

CQRS: Separate models optimized for each

                    ┌──────────────┐
  Commands ────────→│ Write Model  │──→ [Event Store / Write DB]
  (Create, Update,  │ (Domain logic)│          │
   Delete)          └──────────────┘          │ Events/Sync
                                               ▼
  Queries  ────────→┌──────────────┐    [Read DB / View Store]
  (Read, List,      │ Read Model   │◄──(Denormalized, optimized)
   Search)          │ (Simple query)│
                    └──────────────┘
```

### Why Use CQRS?
```
Problem: A social media feed
- Writes: User posts a status (1 write/day average)
- Reads: Feed loads (100 reads/day average, 100:1 ratio)
- Read model needs JOINs across posts, users, likes, comments
- These JOINs are SLOW at scale

Solution with CQRS:
- Write side: Normalized tables, validates business rules
- Read side: Pre-computed denormalized feed (fast reads)
- Sync via events (write → event → update read model)

Trade-off:
✓ Read and write sides scale independently
✓ Each optimized for its workload
✗ Eventual consistency between read/write
✗ More infrastructure complexity
```

---

## 5. Event Sourcing

### Traditional CRUD vs Event Sourcing
```
CRUD (stores current state):
Account: { id: 1, balance: 750 }  ← That's all you know

Event Sourcing (stores all changes):
Account 1 events:
  [AccountCreated: balance=0]
  [Deposited: amount=1000]
  [Withdrawn: amount=200]
  [Deposited: amount=50]
  [Withdrawn: amount=100]
  → Current state: balance = 0 + 1000 - 200 + 50 - 100 = 750

You can reconstruct state at ANY point in time!
```

### When to Use
```
✓ Audit trail is critical (banking, healthcare, legal)
✓ Need to replay/reprocess events (fix bugs, new features)
✓ Temporal queries ("what was the balance on March 1st?")
✓ Complex domain logic with many state transitions

✗ Simple CRUD apps (overkill)
✗ When current state is all that matters
```

---

## 6. Circuit Breaker Pattern

```
States:
                    ┌──────────┐
                    │  CLOSED  │  ← Normal operation, requests pass through
                    │(healthy) │
                    └────┬─────┘
                         │ Failure threshold reached
                         ▼
                    ┌──────────┐
                    │   OPEN   │  ← All requests fail immediately (no calls to service)
                    │ (tripped)│
                    └────┬─────┘
                         │ Timeout expires (try again?)
                         ▼
                    ┌──────────┐
                    │HALF-OPEN │  ← Allow limited requests through
                    │ (testing)│
                    └────┬─────┘
                        / \
                Success/   \Failure
                   ▼        ▼
              [CLOSED]   [OPEN]
```

### Why It Matters
```
Without circuit breaker:
Service A → Service B (down) → timeout (30s) → retry → timeout → retry
Meanwhile: Thread pool exhausted, Service A crashes too → cascading failure

With circuit breaker:
Service A → Service B (down) → 5 failures → OPEN
Service A → immediately returns fallback/error (no waiting)
After 60s → HALF-OPEN → try one request → success → CLOSED
```

---

## 7. Other Essential Patterns

### Bulkhead Pattern
```
Without bulkhead:
[Thread Pool: 100 threads]
Service B is slow → 90 threads stuck waiting → only 10 left for everything else

With bulkhead:
[Thread Pool A: 30 threads] → Service B (isolated)
[Thread Pool B: 30 threads] → Service C (unaffected)
[Thread Pool C: 40 threads] → Service D (unaffected)

Service B is slow → only Pool A is affected
```

### Sidecar Pattern
```
┌─────────────────────┐
│  Pod / Container     │
│  ┌────────────────┐ │
│  │  Main Service  │ │
│  └────────┬───────┘ │
│  ┌────────▼───────┐ │
│  │    Sidecar     │ │  ← Handles cross-cutting concerns:
│  │  (Logging,     │ │     logging, monitoring, proxy,
│  │   Proxy, etc.) │ │     config, service discovery
│  └────────────────┘ │
└─────────────────────┘
```

### Strangler Fig Pattern (Migration)
```
Phase 1: All traffic → Old Monolith
Phase 2: /users traffic → New User Service
         Everything else → Old Monolith
Phase 3: /users + /orders → New Services
         Everything else → Old Monolith
Phase N: All traffic → New Services
         Old Monolith → decommissioned

Use an API gateway/proxy to gradually route traffic to new services.
```

---

## 8. Database Per Service Pattern

### The Rule
Each microservice owns its data. No other service can directly access another's database.

```
✗ Bad: Order Service → directly queries User DB

✓ Good: Order Service → calls User Service API → User Service queries its own DB

Why?
- Loose coupling (can change DB schema without breaking others)
- Independent scaling
- Technology freedom (service A uses Postgres, B uses MongoDB)
```

### Cross-Service Data Patterns
```
1. API Composition:
   API Gateway → calls User Service + Order Service → merges responses

2. Event-Driven:
   User Service → [UserUpdated event] → Order Service updates its local copy

3. Shared Data via Events (CQRS):
   Each service maintains its own read model, updated via events
```

---

## Active Recall Questions

1. When would you choose a monolith over microservices? Give 3 scenarios.
2. Explain the strangler fig pattern. How would you migrate a monolith?
3. What is CQRS? Draw the architecture. When would you use it?
4. Explain the circuit breaker pattern. What happens in each state?
5. What is a service mesh? Name 3 things it handles.
6. Why should each microservice have its own database? What problems does this create?
7. Compare REST vs gRPC for service-to-service communication.
8. What is the bulkhead pattern? How does it prevent cascading failures?
9. Explain event sourcing vs CRUD. When is event sourcing worth the complexity?
10. Design the communication pattern for an e-commerce checkout (Order → Payment → Inventory → Notification).
