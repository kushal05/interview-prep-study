# Service Mesh — Istio

> **TL;DR:** A service mesh injects a sidecar proxy (Envoy) next to every Pod, giving you mTLS, traffic shifting, retries, and telemetry without changing app code. Istio is the most-deployed mesh; Linkerd is lighter; Cilium is doing eBPF-based meshing without sidecars.

## Why a Mesh?

Problems that recur across services in microservice architectures:
- **mTLS** between every service (zero-trust).
- **Retries, timeouts, circuit breaking** done consistently.
- **Traffic shifting / canary** without app changes.
- **Per-call telemetry** (RED + traces) without instrumenting every language.
- **Authorization policies** (which service may call which) centrally.

Without a mesh, every language SDK reimplements these (poorly, differently). A mesh = solve once at the network layer.

## Architecture

```
+----------------+   +----------------+
|     App        |   |     App        |
|  Container     |   |  Container     |
|                |   |                |
|   <-> Envoy    |   |   <-> Envoy    |   <-- data plane (sidecars)
|       proxy    |   |       proxy    |
+--------+-------+   +--------+-------+
         |                    |
         v                    v
        +----------------------+
        |   istiod             |   <-- control plane
        |   (config + certs)   |
        +----------------------+
```

- **Data plane:** Envoy sidecars run in every Pod. All traffic in/out goes through them (iptables redirect).
- **Control plane (istiod):** distributes config + issues short-lived mTLS certs (SDS).

## Sidecar Injection

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    istio-injection: enabled       # any new Pod gets a sidecar
```

Or per-Pod via annotation `sidecar.istio.io/inject: "true"`. The mutating webhook injects an `istio-proxy` container.

## mTLS — Mutual TLS for Free

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: prod
spec:
  mtls:
    mode: STRICT             # require mTLS for all in-namespace traffic
```

Modes:
- `STRICT`: only mTLS accepted. Reject plaintext.
- `PERMISSIVE`: accept both (good for migration).
- `DISABLE`: no mTLS.

Each sidecar has an SVID (SPIFFE-style identity) signed by istiod. Server proves identity, client verifies. Identity = K8s ServiceAccount.

## Traffic Management

### VirtualService — the routing rule

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: { name: api }
spec:
  hosts: [api]
  http:
    - match:
        - headers:
            x-canary: { exact: "true" }
      route:
        - destination: { host: api, subset: v2 }     # opt-in canary
    - route:
        - destination: { host: api, subset: v1 }
          weight: 95
        - destination: { host: api, subset: v2 }
          weight: 5
```

### DestinationRule — define subsets + policy

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: { name: api }
spec:
  host: api
  trafficPolicy:
    connectionPool:
      tcp: { maxConnections: 100 }
      http: { http2MaxRequests: 1000, maxRequestsPerConnection: 10 }
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
    tls: { mode: ISTIO_MUTUAL }
  subsets:
    - name: v1
      labels: { version: v1 }
    - name: v2
      labels: { version: v2 }
```

Canary deploys = update weights gradually. Combined with metrics gates (Flagger, Argo Rollouts) → automated progressive delivery.

### Retries & Timeouts

```yaml
http:
  - route: [{ destination: { host: api } }]
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 1s
      retryOn: 5xx,gateway-error,connect-failure,refused-stream
```

Beware retry storms — total time can amplify if everyone retries on a transient blip. Add jitter + budget.

### Fault Injection (Chaos)

```yaml
http:
  - fault:
      delay: { fixedDelay: 2s, percentage: { value: 10 } }
      abort: { httpStatus: 500, percentage: { value: 1 } }
    route: [{ destination: { host: api } }]
```

Test client resilience without touching the upstream.

## Authorization

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata: { name: api-allow-web, namespace: prod }
spec:
  selector:
    matchLabels: { app: api }
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/prod/sa/web"]
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
```

Identity comes from the SPIFFE cert, not network labels — survives Pod IP changes. Default-deny with an empty policy:

```yaml
spec:
  {}                              # selects all in ns, allows nothing
```

## Ingress / Egress Gateways

The mesh's edge. Replaces or complements Ingress.

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata: { name: app-gw, namespace: prod }
spec:
  selector:
    istio: ingressgateway
  servers:
    - port: { number: 443, name: https, protocol: HTTPS }
      tls:
        mode: SIMPLE
        credentialName: api-tls
      hosts: ["api.example.com"]
```

Combine with a `VirtualService` whose `gateways: [app-gw]` to wire external traffic into mesh services.

Egress gateways: force all outbound traffic through a controlled gateway (audit, logging, IP allowlist on third parties).

## Observability — for Free

Sidecars emit:
- **Metrics:** request count, duration histograms, response codes — labeled by source/destination workload. Scrape with Prometheus.
- **Access logs:** structured JSON per request.
- **Distributed traces:** if you propagate `traceparent` (B3 headers used to be required; W3C now standard), Envoy adds spans for every hop.

Kiali — the topology UI. Service-mesh-aware Grafana dashboards.

## When Not to Use a Mesh

Costs:
- **Resource overhead** — each Pod gains a sidecar (~50-100MB RSS, ~20-50ms p99 added latency).
- **Operational complexity** — yet another control plane to upgrade, debug, monitor.
- **Debugging:** "is my error from app or sidecar?" — needs new mental model.

Skip if:
- You have <10 services.
- mTLS not required (already in private network with strong perimeter).
- Telemetry from app SDKs (OpenTelemetry) is already in place.
- Single-language stack with good libraries.

A mesh is **operational debt traded for app-code simplicity**. Some teams prefer the libraries-in-app pattern (gRPC interceptors, OpenTelemetry, retry middleware).

## Linkerd — the Lightweight Alternative

- Rust-based "linkerd2-proxy" instead of Envoy — smaller, fewer features, faster.
- mTLS, traffic split, basic policy — covers 80% of needs.
- Easier to operate; less configurable.

For most teams, Linkerd is the sweet spot. Pick Istio when you need its advanced features (rich AuthZ, wasm filters, multi-cluster federation, multi-tenant gateways).

## Cilium Service Mesh — eBPF, Sidecarless

Uses Linux kernel eBPF to do L4 + L7 policy + mTLS without a sidecar — saves memory and latency. Younger, fewer features, but the direction the industry is moving.

## Multi-Cluster

Istio supports multi-cluster mesh (services discoverable across clusters). Useful for:
- Geographic distribution + transparent failover.
- Per-team-per-cluster isolation with shared service registry.

Operationally heavy — only pursue if the value is clear.

## Interview Questions

**Q: What's a service mesh and why use one?**
A: A network-layer abstraction (typically sidecar proxies) that provides mTLS, traffic management, retries/timeouts, telemetry, and authorization policies — *without* app code changes. Centralizes cross-cutting concerns. Use it when you have many polyglot services and want consistent, app-independent network behavior.

**Q: Istio vs Linkerd?**
A: Istio: Envoy-based, feature-rich (rich AuthZ, wasm, multi-cluster, ext_authz), heavier ops. Linkerd: Rust proxy, narrower feature set, lighter and easier. Most teams pick Linkerd; pick Istio when you need its advanced features.

**Q: How does mTLS work in Istio?**
A: istiod issues each workload a short-lived SVID cert tied to its K8s ServiceAccount. Both sidecars present certs; both verify. Identity is the SPIFFE URI, not IP — authorization policies reference SAs (`principals: ["cluster.local/ns/prod/sa/api"]`).

**Q: What's the cost of a sidecar?**
A: ~50-100 MB RSS per Pod, ~5-50ms p99 latency added (depends on workload and tuning), more complex debugging ("which layer dropped the request?"). At scale that adds up — 10k Pods × 100MB = 1TB just for sidecars.

**Q: How would you canary deploy with Istio?**
A: Define two `subsets` (v1, v2) in a `DestinationRule`. A `VirtualService` weights them 95/5 → 75/25 → 50/50 → 0/100 with automated analysis (Flagger watches Prometheus metrics, advances or rolls back). Combine with `match` rules to send opt-in canary traffic (header / cookie based) for staff testing.

**Q: Why might you choose libraries-in-app (gRPC interceptors + OTel) over a mesh?**
A: Lower latency / memory overhead, simpler debugging, fewer control planes. Costs: each language needs its own libraries, harder to enforce uniformly. Good fit for small teams, single-language stacks, and when latency budgets are tight.

## Common Pitfalls

- Adopting a mesh too early — solving for problems you don't have yet.
- mTLS `STRICT` rolled out cluster-wide without `PERMISSIVE` migration — breaks legacy non-mesh services.
- Default-allow AuthZ — mesh isn't doing zero-trust until you flip to default-deny.
- Retry storms — every sidecar retrying on transient errors amplifies load.
- Sidecar resource limits too low — proxy gets OOMKilled under load, mysterious 503s.
- Upgrading istiod without a strategy — a control-plane regression can break the data plane.

## Related

- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [07-kubernetes-networking.md](07-kubernetes-networking.md)
- [21-security-devsecops.md](21-security-devsecops.md)
- [18-monitoring-and-observability.md](18-monitoring-and-observability.md)
- [23-load-balancers-and-proxies.md](23-load-balancers-and-proxies.md)
