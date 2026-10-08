# Networking & Protocols

> **Learning goal:** Understand how data moves between clients and servers — the plumbing of every system.

---

## Before You Begin — Plain-English Setup

- **A protocol** is a set of rules for how computers talk. Like grammar for a language.
- **An IP address** is a number that identifies a computer on the internet (e.g., `142.250.80.46`). Two flavors:
  - **IPv4**: 4 numbers separated by dots (`142.250.80.46`).
  - **IPv6**: a longer hex format (`2607:f8b0:4005:0808::200e`).
- **A port** is a "door" on a computer (a number 0–65535). Web servers usually live on port 80 (HTTP) or 443 (HTTPS).
- **A packet** is a small chunk of data sent over the network. A big message is broken into many packets.
- **A header** = metadata attached to a packet/request (e.g., `Content-Type: application/json`).
- **A payload / body** = the actual data being sent.
- **A handshake** = a short back-and-forth two computers do to agree before sending real data.
- **TTL (Time To Live)** = how long a piece of cached info is considered fresh, in seconds.
- **CRUD** = the 4 things you do to data: **C**reate, **R**ead, **U**pdate, **D**elete.

> **Acronym warning:** Networking is acronym-heavy. Every acronym in this file is spelled out the first time it appears. If you forget, ⌘-F / Ctrl-F for the acronym.

---

## 1. DNS (Domain Name System)

### What It Does
Translates human-readable domain names → IP addresses

### How It Works (Step by Step)
```
You type "google.com" in browser:

1. Browser cache → "Do I already know this IP?"
2. OS cache → "Does my computer know?"
3. Router cache → "Does my router know?"
4. ISP's Recursive Resolver → Starts the full lookup:
   │
   ├── Root DNS Server (.): "Who handles .com?"
   │       → Returns: .com TLD server address
   │
   ├── TLD Server (.com): "Who handles google.com?"
   │       → Returns: Google's authoritative nameserver
   │
   └── Authoritative Server (ns1.google.com): "What's the IP for google.com?"
           → Returns: 142.250.80.46

5. IP cached at each level with a TTL (Time To Live)
```

### DNS Record Types
| Type | Purpose | Example |
|------|---------|---------|
| **A** | Domain → IPv4 | google.com → 142.250.80.46 |
| **AAAA** | Domain → IPv6 | google.com → 2607:f8b0:... |
| **CNAME** | Domain → another domain (alias) | www.google.com → google.com |
| **MX** | Mail server for domain | google.com → smtp.google.com |
| **NS** | Authoritative nameserver | google.com → ns1.google.com |
| **TXT** | Arbitrary text (verification, SPF) | v=spf1 include:... |

### DNS in System Design
- **DNS-based load balancing:** Return different IPs for the same domain (Round Robin)
- **GeoDNS:** Return different IPs based on user location (route to nearest data center)
- **DNS failover:** Health checks + automatic IP change on failure
- **Limitation:** TTL means changes propagate slowly (minutes to hours)

---

## 2. HTTP / HTTPS

### HTTP Request/Response Cycle
```
Client                          Server
  │                               │
  │── GET /api/users HTTP/1.1 ──→│
  │   Host: api.example.com       │
  │   Authorization: Bearer xyz   │
  │                               │
  │←── HTTP/1.1 200 OK ──────────│
  │    Content-Type: application/json
  │    {"users": [...]}           │
```

### HTTP Methods
| Method | Purpose | Idempotent? | Safe? |
|--------|---------|-------------|-------|
| GET | Read resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace entire resource | Yes | No |
| PATCH | Update part of resource | No | No |
| DELETE | Remove resource | Yes | No |

### HTTP Status Codes (The Important Ones)
```
2xx Success:
  200 OK          — Standard success
  201 Created     — Resource created (POST)
  204 No Content  — Success, empty body (DELETE)

3xx Redirect:
  301 Moved Permanently — URL changed forever (SEO important)
  302 Found             — Temporary redirect
  304 Not Modified      — Use cached version

4xx Client Error:
  400 Bad Request   — Malformed request
  401 Unauthorized  — Not authenticated (who are you?)
  403 Forbidden     — Authenticated but not authorized (you can't do this)
  404 Not Found     — Resource doesn't exist
  429 Too Many Reqs — Rate limited

5xx Server Error:
  500 Internal Error   — Generic server failure
  502 Bad Gateway      — Upstream server failed
  503 Service Unavail  — Server overloaded/maintenance
  504 Gateway Timeout  — Upstream server timed out
```

### HTTP/1.1 vs HTTP/2 vs HTTP/3
```
HTTP/1.1:
- One request per TCP connection (or keep-alive reuse)
- Head-of-line blocking (requests queue up)
- Text-based headers (verbose)

HTTP/2:
- Multiplexing: many requests on ONE TCP connection
- Header compression (HPACK)
- Server push (send resources before client asks)
- Binary protocol (more efficient)
- Still has TCP head-of-line blocking

HTTP/3:
- Uses QUIC (over UDP) instead of TCP
- Eliminates TCP head-of-line blocking
- Faster connection setup (0-RTT)
- Built-in encryption
```

### HTTPS
```
TLS Handshake (simplified):
Client → Server: "Hello, I support these encryption methods"
Server → Client: "Let's use this one. Here's my certificate."
Client: Verifies certificate with Certificate Authority
Client → Server: "Here's a shared secret, encrypted with your public key"
Both: Derive symmetric encryption keys from shared secret
Result: All further communication is encrypted
```

---

## 3. TCP vs UDP

### TCP (Transmission Control Protocol)
```
Features:
✓ Connection-oriented (3-way handshake)
✓ Reliable delivery (retransmission on loss)
✓ Ordered delivery (packets reassembled in order)
✓ Flow control (sender adjusts to receiver speed)
✓ Congestion control (adjusts to network capacity)

3-Way Handshake:
Client → Server:  SYN (I want to connect)
Server → Client:  SYN-ACK (OK, let's connect)
Client → Server:  ACK (Confirmed, we're connected)

Use when: Correctness matters (web, email, file transfer, APIs)
```

### UDP (User Datagram Protocol)
```
Features:
✓ Connectionless (just send packets)
✓ No delivery guarantee
✓ No ordering guarantee
✓ No flow/congestion control
✓ Very low overhead → faster

Use when: Speed > reliability (video streaming, gaming, DNS, VoIP)
```

### Comparison
```
         TCP                    UDP
    ┌──────────┐          ┌──────────┐
    │ Reliable │          │   Fast   │
    │ Ordered  │          │ Lossy OK │
    │ Slower   │          │ Faster   │
    │ Heavy    │          │  Light   │
    └──────────┘          └──────────┘
    HTTP, SSH, DB          Video, DNS,
    connections            Gaming, IoT
```

---

## 4. WebSockets

### What Problem Does It Solve?
HTTP is request-response: client asks, server answers. What if the server needs to push data to the client?

### Polling vs Long Polling vs WebSockets
```
Regular Polling:
Client: "Any new messages?" → Server: "No"
Client: "Any new messages?" → Server: "No"
Client: "Any new messages?" → Server: "Yes! Here."
⚠ Wasteful — many empty responses, high server load

Long Polling:
Client: "Any new messages?" → Server: [holds connection open...]
                              Server: "Yes! Here." (when data arrives)
Client: "Any new messages?" → Server: [holds again...]
✓ Less wasteful, but still creates new connections repeatedly

WebSockets:
Client → Server: HTTP Upgrade request
Server → Client: "101 Switching Protocols"
[Now both can send messages at any time over persistent connection]
Client ←→ Server: "Hi" / "Hello" / "New data!" / "Got it"
✓ Full-duplex, persistent, low overhead
```

### When to Use WebSockets
- Real-time chat (Slack, WhatsApp)
- Live notifications
- Collaborative editing (Google Docs)
- Live sports scores / stock tickers
- Online gaming
- Live dashboards

### Server-Sent Events (SSE) — The Middle Ground
```
- One-way: server → client only
- Uses regular HTTP (simpler than WebSockets)
- Auto-reconnection built-in
- Good for: live feeds, notifications, event streams
- Not good for: bidirectional communication
```

---

## 5. REST API Design

### REST Principles
1. **Stateless:** Each request contains all info needed (no server-side sessions)
2. **Resource-based:** URLs represent nouns, not verbs
3. **Standard methods:** HTTP methods map to CRUD operations
4. **Uniform interface:** Consistent URL patterns

### Good API Design
```
Resources as nouns:
✓ GET    /users              — List users
✓ GET    /users/123          — Get user 123
✓ POST   /users              — Create user
✓ PUT    /users/123          — Replace user 123
✓ PATCH  /users/123          — Update user 123
✓ DELETE /users/123          — Delete user 123

✗ GET /getUser?id=123        — Verb in URL (bad)
✗ POST /createUser           — Verb in URL (bad)
✗ POST /deleteUser/123       — Wrong method + verb (bad)

Nested resources:
✓ GET /users/123/posts       — Posts by user 123
✓ GET /users/123/posts/456   — Post 456 by user 123

Filtering, sorting, pagination:
✓ GET /users?role=admin&sort=name&page=2&limit=20
```

### REST vs GraphQL vs gRPC
| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Protocol | HTTP | HTTP | HTTP/2 |
| Data format | JSON | JSON | Protobuf (binary) |
| Flexibility | Fixed endpoints | Client chooses fields | Defined by proto files |
| Over/Under fetching | Common problem | Solved | No (fixed contracts) |
| Best for | Public APIs, CRUD | Complex UIs, mobile | Microservice-to-microservice |
| Caching | Easy (HTTP caching) | Hard | Hard |
| Learning curve | Low | Medium | High |

---

## 6. API Gateway

### What It Does
Single entry point for all client requests → routes to appropriate microservice.

```
                    ┌─────────────┐
Clients ──────────→ │ API Gateway │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ User Svc │ │Order Svc │ │Payment   │
        └──────────┘ └──────────┘ └──────────┘
```

### Responsibilities
- **Routing:** Direct requests to correct service
- **Authentication:** Verify tokens before forwarding
- **Rate limiting:** Protect backend from abuse
- **Load balancing:** Distribute across service instances
- **Request/Response transformation:** Adapt formats
- **Caching:** Cache common responses
- **Circuit breaking:** Stop forwarding to failing services
- **Logging & monitoring:** Central observability point

### Examples
- AWS API Gateway, Kong, Nginx, Envoy, Zuul

---

## 7. CDN (Content Delivery Network) — Networking View

### How CDN Works
```
Without CDN:
User (Tokyo) ──── 200ms ────→ Origin Server (Virginia)

With CDN:
User (Tokyo) ── 20ms ──→ CDN Edge (Tokyo) ── cache hit ──→ Response
                              │
                         cache miss
                              │
                              ▼
                     Origin Server (Virginia)
                     (edge caches the response)
```

### Push vs Pull CDN
| Type | How it Works | Best For |
|------|-------------|----------|
| **Pull** | CDN fetches from origin on first request, caches it | Dynamic/large content catalogs |
| **Push** | You upload content to CDN proactively | Static assets, known content |

---

## Active Recall Questions

1. Walk through what happens when you type "google.com" in your browser (DNS → TCP → HTTP → render).
2. What's the difference between HTTP/1.1, HTTP/2, and HTTP/3? Why was each created?
3. When would you use TCP vs UDP? Give 3 examples of each.
4. Explain the difference between polling, long polling, and WebSockets. When would you use each?
5. Design a REST API for a blog platform (posts, comments, users). Include filtering and pagination.
6. What is an API Gateway? Name 5 responsibilities it handles.
7. How does a CDN reduce latency? What's the difference between push and pull CDN?
8. What does the 3-way handshake accomplish? Why is it necessary?
9. Explain the difference between 401 and 403 status codes.
10. When would you choose gRPC over REST? GraphQL over REST?
