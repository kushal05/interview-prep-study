# Load Balancers & Proxies

> **TL;DR:** L7 LB / reverse proxy = nginx, HAProxy, Envoy, AWS ALB. L4 = NLB, HAProxy mode tcp, IPVS. Use L7 for HTTP routing + TLS + WAF; L4 for raw TCP / non-HTTP / extreme throughput. Know stickiness, health checks, keep-alives, and X-Forwarded-* headers.

## What Sits Where

```
[client]
   |
   v
[CDN edge]              (CloudFront / Cloudflare / Fastly)
   |
   v
[global LB]             (Route 53 latency routing, Cloud Front Door, GCP global LB)
   |
   v
[regional L7 LB]        (ALB, Application Gateway, Cloud LB, nginx, Envoy)
   |
   v
[L4 LB]                 (NLB, Cloud LB TCP) — optional layer
   |
   v
[service mesh sidecar]  (Envoy in Istio/Linkerd) — optional
   |
   v
[app pods/containers]
```

Not every stack has every layer — many real stacks are CDN → ALB → app.

## L4 vs L7 in Practice

### L4 — Network/Transport Layer

- Decides routing on IP + port only.
- Doesn't terminate TLS (or does **TLS passthrough**).
- Best throughput, lowest latency.
- Used for: databases, gRPC streaming (sometimes), non-HTTP TCP services, ultra-low latency paths.

```
AWS NLB
HAProxy (mode tcp)
IPVS / LVS
F5 BIG-IP (operates at L4-L7)
```

### L7 — Application Layer

- Parses HTTP (method, path, headers, query, body).
- Usually terminates TLS.
- Path/host routing, cookie-based stickiness, header rewrites, WAF.
- More CPU per request — usually fine.

```
AWS ALB
nginx (most popular reverse proxy)
HAProxy (mode http)
Envoy (used by Istio, AWS App Mesh, GCP Traffic Director)
Traefik (popular for K8s ingress)
Caddy (auto-HTTPS, simple)
```

## Nginx — the Workhorse Reverse Proxy

```nginx
# /etc/nginx/conf.d/api.conf
upstream api_backend {
    least_conn;
    server 10.0.1.10:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:3000 backup;          # only used if all others down
    keepalive 32;                          # keep 32 idle upstream conns per worker
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    # Body / proxy timeouts
    client_max_body_size 10m;
    proxy_connect_timeout 5s;
    proxy_send_timeout    30s;
    proxy_read_timeout    30s;

    location /healthz {
        access_log off;
        return 200 'ok';
    }

    location /api/ {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection        "";    # required for upstream keepalive

        # Rate limit (defined in http block)
        limit_req zone=api burst=20 nodelay;
    }

    # Redirect HTTP -> HTTPS done at a separate :80 server block
}

# http {} block elsewhere
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
```

### Reload Without Dropping Connections

```bash
nginx -t                          # validate config
nginx -s reload                   # graceful: spawn new workers, drain old ones
```

## HAProxy — Mature, Fast L4/L7

```haproxy
global
    maxconn 50000
    log 127.0.0.1 local0
    tune.ssl.default-dh-param 2048

defaults
    mode http
    log global
    option httplog
    option dontlognull
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend fe_https
    bind :443 ssl crt /etc/haproxy/certs/site.pem alpn h2,http/1.1
    http-request redirect scheme https unless { ssl_fc }
    acl host_api hdr(host) -i api.example.com
    use_backend be_api if host_api
    default_backend be_web

backend be_api
    balance leastconn
    option httpchk GET /healthz
    http-check expect status 200
    server api1 10.0.1.10:3000 check inter 2s
    server api2 10.0.1.11:3000 check inter 2s
    server api3 10.0.1.12:3000 check inter 2s
```

HAProxy is famed for stats and observability — built-in dashboard at `/haproxy?stats`.

## Envoy — the Modern Edge / Mesh Proxy

Used in Istio, AWS App Mesh, GCP Traffic Director. xDS-driven (configured by a control plane).

Strengths:
- gRPC support is first-class.
- Rich filters (rate limit, JWT auth, RBAC, ext_authz).
- Outlier detection (eject misbehaving upstreams automatically).
- Tracing + metrics emitted natively.
- Hot-reload, no connection drops.

Generally you don't write Envoy config by hand — you run something that generates it (Istio, Contour, Gloo).

## AWS ALB / NLB

| | ALB | NLB |
|---|---|---|
| Layer | L7 | L4 |
| Path/host routing | Yes | No |
| TLS termination | Yes | Yes (passthrough or term) |
| Source IP preservation | Via `X-Forwarded-For` | Native (preserves source IP) |
| Throughput | High | Extreme (millions of conn/s) |
| WebSockets | Yes | Yes |
| gRPC | Yes (HTTP/2) | Yes (TCP) |
| Static IP | No (DNS only) | Yes (one per AZ, or BYOIP) |

Use ALB for HTTP services, NLB when you need a static IP, extreme throughput, or you need source IP preserved without `X-Forwarded-For` parsing.

## Health Checks — Get Them Right

```yaml
# ALB
HealthCheckProtocol: HTTP
HealthCheckPath: /healthz
HealthCheckIntervalSeconds: 10
UnhealthyThresholdCount: 2
HealthyThresholdCount: 2
HealthCheckTimeoutSeconds: 5
Matcher: { HttpCode: '200' }
```

- Don't health-check `/` (might be 200 from a frontend even when the backend is dead).
- `/healthz` returns 200 if the process is up.
- `/readyz` (K8s) — 200 if process can serve traffic (deps reachable).
- Avoid putting DB calls in `/healthz` (cascade failure: DB blip → all instances marked down).

## Stickiness (Session Affinity)

When useful: stateful in-memory sessions (most real apps shouldn't have these), WebSocket connections.

| Mechanism | How |
|-----------|-----|
| **Source IP hash** | L4; works for any TCP. Coarse — many users behind one NAT all stick. |
| **Cookie-based** | L7. LB issues a cookie; same cookie → same backend. Precise. |
| **Application-managed cookie** | App sets a cookie LB reads. App owns rotation. |

Cleanest answer: **don't need stickiness** — store sessions in Redis / DB, scale stateless.

## TLS Termination Choices

- **Terminate at LB:** simpler, LB does cert management; backend speaks plain HTTP (inside VPC).
- **Re-encrypt:** terminate at LB, then re-encrypt to backend (e.g., backend wants HTTPS for compliance).
- **Passthrough:** LB does L4, backend terminates. Required if you need mTLS to backends.

## X-Forwarded-* Headers — Don't Lie

When an LB sits in front, the backend sees the LB's IP. To know the real client:

```
X-Forwarded-For: 203.0.113.45, 10.0.1.1
X-Forwarded-Proto: https
X-Forwarded-Host: api.example.com
X-Real-IP: 203.0.113.45            (nginx-style, only the first hop)
```

- Trust them only from your LB. A direct-to-app request can spoof `X-Forwarded-For` and your audit log is wrong.
- Set up a "trusted proxies" list in the app: trust headers only if the source IP is in the list.

```js
// Express
app.set('trust proxy', ['10.0.0.0/8', '172.16.0.0/12']);
```

## Rate Limiting

Layered approach:
- **Edge (CDN / WAF):** per-IP / per-region, very coarse, drops DDoS noise.
- **LB / reverse proxy:** per-IP / per-route. `limit_req` in nginx, rate-limit filter in Envoy.
- **App:** per-user / per-token / per-route — semantics-aware.

Algorithms:
- **Token bucket** — smooth, allows short bursts.
- **Leaky bucket** — fixed output rate.
- **Sliding window** — most accurate, simple to reason about.
- **Fixed window** — easiest, has edge-of-window bursts.

Return `429 Too Many Requests` + `Retry-After: <seconds>`.

## Connection Pooling Notes

Configure both ends:
- LB upstream `keepalive` (nginx) so the LB doesn't reopen TCP for every request.
- App-to-DB connection pool (`pg-pool`, `HikariCP`) — most prod outages start with "connection pool exhausted."

## Graceful Shutdown — Drain Before Kill

When a Pod / instance is going away, your stack must:
1. Stop the readiness probe / deregister from LB.
2. Drain in-flight requests (wait for them to finish).
3. Close upstream connections cleanly.
4. Exit.

In K8s: `terminationGracePeriodSeconds` + `preStop` hook + `SIGTERM` handler in the app. ALB / NLB has a "deregistration delay" (default 300s, often too long — set to 30s).

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 15"]   # let LB notice unready before kill
```

## Interview Questions

**Q: ALB vs NLB?**
A: ALB is L7 (HTTP) — path/host routing, WAF, AWS-managed certs. NLB is L4 — static IPs per AZ, source IP preservation, extreme throughput. Use ALB for HTTP apps, NLB for non-HTTP TCP, gRPC streaming with strict source-IP needs, or when you must have static IPs.

**Q: How do you load-balance WebSockets?**
A: L4 LB (NLB / HAProxy tcp) works easily because it's just a long-lived TCP connection. L7 LBs work too if they support WebSocket upgrade (ALB does). Avoid short upstream timeouts — set them generously. For sticky sessions, use cookie-based affinity (L7) or source IP hash (L4).

**Q: Why is source IP lost behind a reverse proxy and how do you recover it?**
A: The LB is the TCP peer, so backend sees its IP. Recover via `X-Forwarded-For` / `X-Real-IP` / `Forwarded` headers. Configure trusted proxy list in the app so spoofed headers from direct connections are ignored. NLB preserves source IP natively (no headers needed).

**Q: How does nginx do a zero-downtime reload?**
A: `nginx -s reload` sends SIGHUP. Master process spawns new workers with the new config; old workers finish in-flight requests then exit. New connections go to new workers. No drop. The new master keeps the same listening sockets — no port re-bind.

**Q: A backend pool of 5 hosts — one is slow. What happens?**
A: With round-robin, traffic stays evenly split — slow host's latency drags p95/p99. Switch to **least connections** or **EWMA / least time** — fewer requests go to the slow one. Best: enable **outlier detection** (Envoy) so it's temporarily ejected.

**Q: What's TLS passthrough and when do you use it?**
A: LB does pure L4 — does not decrypt traffic, just forwards encrypted bytes. Backend terminates TLS. Used when: you need mTLS to the backend, end-to-end encryption is required for compliance, or you can't share certs with the LB. Costs: no LB-level WAF, no path routing.

## Common Pitfalls

- Healthcheck that hits DB / dependencies — single dep blip → all instances marked unhealthy → 503.
- Trusting `X-Forwarded-For` without a trusted-proxies list — IP spoofing.
- No keepalive on upstream → connection thrashing under load.
- ALB deregistration delay too long → slow rolling deploys.
- Sticky sessions used to "fix" a bug → can't roll deploys without dropping users.
- L7 LB without HTTP/2 enabled → grpc-web bombs out.
- Aggressive client `keepalive` + restrictive LB idle timeout → mysterious 502s.

## Related

- [22-networking-fundamentals.md](22-networking-fundamentals.md)
- [07-kubernetes-networking.md](07-kubernetes-networking.md)
- [25-service-mesh-istio.md](25-service-mesh-istio.md)
- [15-cloud-aws-basics.md](15-cloud-aws-basics.md)
