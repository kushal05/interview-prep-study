# Load Balancing & Proxies

> **Learning goal:** Understand how traffic is distributed, what proxies do, and how to design for high throughput.

---

## Before You Begin — Plain-English Setup

- **A load balancer (LB)** is a server whose only job is to receive requests and forward them to one of several backend servers, sharing the work evenly.
- **A proxy** is any server in the middle that forwards a request. The LB is one kind of proxy.
- **A backend / origin server** = the real server doing the work (your app, your API).
- **An OSI layer** = a way to describe what level of network info you're working with. Two layers matter here:
  - **Layer 4 (L4)**: the IP + port level. The LB sees "TCP connection to port 443" but not what's inside.
  - **Layer 7 (L7)**: the HTTP level. The LB can see the URL path, headers, cookies, etc.
- **SSL / TLS** = the encryption used by HTTPS. "**SSL termination**" means the LB decrypts the request, then forwards plain HTTP to the backend (so backends don't waste CPU on encryption).
- **A health check** = the LB periodically pings each backend with `GET /health`. If a backend stops responding, the LB stops sending it traffic.
- **A VIP (Virtual IP)** = a single IP address that points to whichever LB is currently active. Used for LB failover.
- **Sticky session** = the LB always sends one user's requests to the same backend (so the backend can keep session state in memory).

---

## 1. Load Balancer Basics

### What It Does
Distributes incoming traffic across multiple servers to ensure no single server is overwhelmed.

```
                    ┌──────────────┐
  Clients ────────→ │Load Balancer │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         [Server 1]   [Server 2]   [Server 3]
```

### Load Balancing Algorithms

**1. Round Robin**
```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (cycles back)
```
- ✓ Simple, even distribution
- ✗ Ignores server capacity/load

**2. Weighted Round Robin**
```
Server A (weight 3): gets 3 out of every 6 requests
Server B (weight 2): gets 2 out of every 6 requests
Server C (weight 1): gets 1 out of every 6 requests
```
- ✓ Accounts for different server capacities

**3. Least Connections**
```
Server A: 15 active connections
Server B: 8 active connections  ← next request goes here
Server C: 12 active connections
```
- ✓ Best for long-lived connections (WebSockets, databases)

**4. Least Response Time**
```
Server A: avg 50ms response
Server B: avg 20ms response  ← next request goes here
Server C: avg 35ms response
```
- ✓ Routes to fastest server

**5. IP Hash**
```
hash(client_ip) % num_servers = target server
Same client always goes to same server
```
- ✓ Session persistence without sticky sessions
- ✗ Uneven distribution if some IPs send more traffic

**6. Consistent Hashing**
- See Database sharding section — same concept
- Minimal redistribution when servers are added/removed

---

## 2. Layer 4 vs Layer 7 Load Balancing

### Layer 4 (Transport Layer — TCP/UDP)
```
Operates on: IP address + port number
Decision based on: Source/destination IP, port
Does NOT inspect: HTTP headers, URLs, cookies, body

Client ──[TCP packet]──→ L4 LB ──[TCP packet forwarded]──→ Server
```
- ✓ Very fast (simple packet forwarding)
- ✓ Protocol agnostic
- ✗ Can't route based on content (URL, headers)

### Layer 7 (Application Layer — HTTP)
```
Operates on: Full HTTP request
Decision based on: URL path, headers, cookies, query params

Client ──[GET /api/users]──→ L7 LB → Routes to User Service
Client ──[GET /api/orders]──→ L7 LB → Routes to Order Service
Client ──[GET /images/cat.jpg]──→ L7 LB → Routes to CDN/Static Server
```
- ✓ Smart routing (content-based, A/B testing, canary)
- ✓ Can terminate SSL (offload encryption from backends)
- ✓ Can cache responses
- ✗ Slower (must parse HTTP)
- ✗ More resource intensive

### When to Use Each
| Layer 4 | Layer 7 |
|---------|---------|
| Raw TCP/UDP traffic | HTTP/HTTPS traffic |
| Database connections | Microservice routing |
| Maximum performance | Content-based routing |
| Simple load distribution | A/B testing, canary deploys |

---

## 3. Reverse Proxy

### Forward Proxy vs Reverse Proxy
```
Forward Proxy (client-side):
Client → [Forward Proxy] → Internet → Server
- Hides client identity
- Used for: corporate firewalls, VPN, caching
- Client knows about proxy

Reverse Proxy (server-side):
Client → Internet → [Reverse Proxy] → Server
- Hides server identity
- Used for: load balancing, SSL, caching, security
- Client doesn't know about proxy
```

### Reverse Proxy Responsibilities
```
1. Load balancing      → Distribute traffic
2. SSL termination     → Decrypt HTTPS, forward HTTP internally
3. Caching             → Cache static content
4. Compression         → gzip responses
5. Security            → Hide backend, DDoS protection, WAF
6. Rate limiting       → Throttle abusive clients
7. Request routing     → Route by URL path to different backends
```

### Common Tools
| Tool | Type | Best For |
|------|------|----------|
| **Nginx** | Reverse proxy + web server | General purpose, high performance |
| **HAProxy** | Load balancer | Pure load balancing, TCP and HTTP |
| **Envoy** | Service proxy | Microservices, service mesh |
| **Traefik** | Cloud-native proxy | Kubernetes, Docker, auto-discovery |
| **AWS ALB** | Managed L7 LB | AWS HTTP load balancing |
| **AWS NLB** | Managed L4 LB | AWS TCP/UDP load balancing |

---

## 4. API Gateway vs Load Balancer vs Reverse Proxy

```
                          Reverse Proxy
                         ┌────────────────────────────────────┐
                         │  Load Balancer                     │
                         │  ┌─────────────────────────────┐   │
                         │  │  API Gateway                │   │
                         │  │  ┌──────────────────────┐   │   │
                         │  │  │ Auth, Rate Limit,    │   │   │
                         │  │  │ Routing, Transform   │   │   │
                         │  │  └──────────────────────┘   │   │
                         │  │  + Traffic Distribution     │   │
                         │  └─────────────────────────────┘   │
                         │  + SSL, Caching, Compression       │
                         └────────────────────────────────────┘
```

| Feature | Reverse Proxy | Load Balancer | API Gateway |
|---------|:------------:|:------------:|:-----------:|
| SSL termination | ✓ | ✓ | ✓ |
| Traffic distribution | ✓ | ✓ | ✓ |
| Caching | ✓ | Sometimes | ✓ |
| Authentication | Sometimes | ✗ | ✓ |
| Rate limiting | Sometimes | ✗ | ✓ |
| Request transformation | ✗ | ✗ | ✓ |
| API versioning | ✗ | ✗ | ✓ |
| Analytics/monitoring | Basic | Basic | ✓ |

---

## 5. Health Checks & Failover

### Health Check Types
```
1. Active Health Check:
   LB periodically pings servers: GET /health
   Server responds 200 OK → healthy
   Server responds 5xx or timeout → unhealthy → removed from pool

2. Passive Health Check:
   LB monitors actual traffic
   Too many errors from a server → mark unhealthy
```

### Failover Patterns
```
Active-Passive:
[Server A: ACTIVE] ←→ [Server B: STANDBY]
If A fails → B becomes active (automatic failover)
✓ Simple
✗ Standby server is wasted capacity

Active-Active:
[Server A: ACTIVE] ←→ [Server B: ACTIVE]
Both serve traffic. If one fails, other handles all.
✓ Better resource utilization
✗ More complex (data sync needed)
```

---

## 6. Global Server Load Balancing (GSLB)

```
User in Tokyo                    User in New York
     │                                │
     ▼                                ▼
  [DNS Query]                    [DNS Query]
     │                                │
     ▼                                ▼
  [GSLB/GeoDNS]                 [GSLB/GeoDNS]
     │                                │
     ▼                                ▼
  Tokyo DC                       Virginia DC
  [LB → Servers]                [LB → Servers]
```

**Routing strategies:**
- **Geography-based:** Route to nearest data center
- **Latency-based:** Route to lowest latency DC
- **Failover:** Route to healthy DC
- **Weighted:** Split traffic by percentage (e.g., 90% prod, 10% canary)

---

## 7. Load Balancing in System Design

### Single Point of Failure?
```
Problem: The load balancer itself is a SPOF!

Solution: Redundant load balancers

Client → [DNS returns VIP]
              │
         ┌────┴────┐
         ▼         ▼
    [LB Active] [LB Passive]  ← Heartbeat between them
         │                      If active dies, passive takes over VIP
    ┌────┼────┐
    ▼    ▼    ▼
  [S1] [S2] [S3]
```

### Capacity Math
```
If each server handles 1,000 QPS:
- 10,000 QPS needed → 10 servers + LB
- Peak = 3x average → 30,000 QPS → 30 servers
- With 20% headroom → 36 servers

LB itself: Modern LBs handle millions of QPS
- Nginx: ~1M concurrent connections
- HAProxy: ~500K req/s per instance
```

---

## Active Recall Questions

1. Explain Round Robin vs Least Connections. When would you use each?
2. What's the difference between L4 and L7 load balancing? Give examples.
3. What's the difference between a forward proxy and a reverse proxy?
4. How do you prevent the load balancer from being a single point of failure?
5. Explain the difference between active-passive and active-active failover.
6. When would you use an API Gateway vs a simple load balancer?
7. What is SSL termination? Why do we do it at the load balancer?
8. Explain GeoDNS/GSLB. How does it route users to the nearest data center?
9. Your system needs to handle 50,000 QPS. Each server handles 2,000 QPS. How many servers + what LB setup?
10. Name 3 differences between Nginx and HAProxy.
