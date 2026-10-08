# Kubernetes Networking

> **TL;DR:** Every Pod gets its own IP, all Pods can talk to all Pods (flat network), Services give you a stable VIP and DNS, Ingress fronts HTTP, NetworkPolicies say who's allowed.

## The Four Rules (per the K8s network model)

1. Every Pod gets a unique cluster-internal IP.
2. Pod-to-Pod communication works without NAT.
3. Pod-to-Node and Node-to-Pod works without NAT.
4. The container's view of its own IP == what others see.

This is implemented by a **CNI plugin** (Calico, Cilium, Flannel, AWS VPC CNI, GKE's, etc.).

## Service Types

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 3000
```

### ClusterIP (default)

- Virtual IP, in-cluster only.
- DNS: `api.<ns>.svc.cluster.local`.
- kube-proxy programs iptables/IPVS rules to DNAT VIP → Pod IPs (random).
- Cheap, fast, the default for service-to-service traffic.

### NodePort

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080            # 30000–32767
```

- Opens the same port on **every node** in the cluster.
- Traffic to `<any-node-ip>:30080` → Service → Pod.
- Used for bare-metal clusters without a cloud LB. Don't expose to the internet directly — use Ingress.

### LoadBalancer

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 3000
```

- Provisions a cloud LB (AWS NLB/ALB, GCP LB, Azure LB) and routes external traffic to NodePorts.
- One LB per Service = $$$. Prefer Ingress for many HTTP services.

### Headless Service (`clusterIP: None`)

```yaml
spec:
  clusterIP: None
  selector: { app: kafka }
```

- DNS returns A records for each Pod IP — no VIP.
- Used for StatefulSets where clients need to talk to specific Pods (Kafka brokers, Cassandra nodes).

### ExternalName

```yaml
spec:
  type: ExternalName
  externalName: db.aws.example.com
```

- CNAME alias in cluster DNS. Lets in-cluster apps use `db.prod.svc.cluster.local` for an external service. No proxying.

## Endpoints / EndpointSlices

A Service has no clue what a Pod is. The **Endpoints controller** watches the Service's selector and writes the matching Pod IPs into an `Endpoints` (or `EndpointSlice`, more scalable) object. kube-proxy reads that and programs the data plane.

```bash
kubectl get endpoints api
kubectl get endpointslices -l kubernetes.io/service-name=api
```

If a Pod's readiness probe fails, its IP is removed from Endpoints → it stops receiving traffic. **That's how rolling updates avoid serving from a not-yet-ready Pod.**

## DNS in the Cluster (CoreDNS)

Every Pod's `/etc/resolv.conf` points to the CoreDNS Service.

- `api` (same namespace) → `api.<my-ns>.svc.cluster.local`
- `api.prod` → `api.prod.svc.cluster.local`
- FQDN ending in `.` → resolved directly

Set `ndots: 2` to skip wasted lookups for external names (default is 5, leading to many NXDOMAIN queries for `example.com.svc.cluster.local` etc.).

```yaml
spec:
  dnsConfig:
    options:
      - { name: ndots, value: "2" }
```

## Ingress — HTTP Routing

Ingress is just a manifest. An **Ingress controller** (nginx-ingress, Traefik, HAProxy, AWS ALB controller) watches them and configures a real reverse proxy.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: api, port: { number: 80 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: web, port: { number: 80 } }
```

- One LB IP, many HTTP routes — cheap and standard.
- TLS via cert-manager + Let's Encrypt is the default pattern.

### Gateway API — Ingress's Successor

Ingress couldn't express enough (header routing, multi-tenancy, traffic splitting). Gateway API splits the model:

- `GatewayClass` (the controller type)
- `Gateway` (the listener — ports, certs)
- `HTTPRoute` / `TCPRoute` / `GRPCRoute` (the rules)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: api }
spec:
  parentRefs: [{ name: shared-gateway }]
  hostnames: ["api.example.com"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /v1 } }]
      backendRefs: [{ name: api, port: 80, weight: 90 }]
    - matches: [{ path: { type: PathPrefix, value: /v1 } }]
      backendRefs: [{ name: api-v2, port: 80, weight: 10 }]   # 10% canary
```

## NetworkPolicy — Default-Deny

By default, every Pod can talk to every Pod. Lock it down:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: api-allow }
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: web } }
        - namespaceSelector: { matchLabels: { name: gateway } }
      ports:
        - { port: 3000, protocol: TCP }
  egress:
    - to:
        - podSelector: { matchLabels: { app: db } }
      ports: [{ port: 5432, protocol: TCP }]
    - to:                                # DNS
        - namespaceSelector: { matchLabels: { name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
      ports: [{ port: 53, protocol: UDP }]
```

**Gotchas:**
- A Pod with **no** NetworkPolicy = all traffic allowed.
- A Pod with **any** NetworkPolicy = only explicitly allowed traffic.
- A "default-deny" baseline is essential — apply an empty `podSelector: {}` policy that allows nothing.
- NetworkPolicies are **enforced by the CNI**. Flannel (default in many clusters) doesn't enforce them — use Calico or Cilium.

```yaml
# default-deny-all in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny, namespace: prod }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

## Service Mesh — When You Need More

A service mesh (Istio, Linkerd, Consul) adds a sidecar proxy (Envoy) to every Pod:

- **mTLS between every Pod** (zero-trust networking)
- **Traffic shifting** (10% to v2)
- **Circuit breaking, retries, timeouts** centrally configured
- **Detailed telemetry** without app changes
- **Authorization policies** at L7 (HTTP path, method)

Cost: complexity, CPU/memory per sidecar, debugging is harder. Don't reach for it until basics are in place. See [25-service-mesh-istio.md](25-service-mesh-istio.md).

## externalTrafficPolicy

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local       # vs Cluster (default)
```

- **Cluster (default):** LB can send traffic to any node; node forwards to any Pod. Source IP is lost (SNAT). Even distribution.
- **Local:** node only handles traffic if it has a Pod. Source IP preserved. If a node has 0 Pods, the LB still tries it and the health check fails — uneven load if Pods aren't spread evenly.

If you need real client IP, use `Local` + spread your Pods.

## Debugging Networking

```bash
# Can my pod reach a service?
kubectl run -it --rm test --image=nicolaka/netshoot -- bash
# inside the pod:
curl -v http://api/health
dig api.prod.svc.cluster.local
nslookup kubernetes.default
nc -zv api.prod 80

# Service to Pod
kubectl get endpoints api          # empty? selector doesn't match any ready Pod
kubectl get pods -l app=api -o wide

# DNS issues?
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl exec -n kube-system <coredns-pod> -- cat /etc/coredns/Corefile

# Is NetworkPolicy blocking?
kubectl get netpol -A
# Temporary: delete the policy, retry; if it works, fix the policy.
```

## Interview Questions

**Q: How does kube-proxy implement Services?**
A: Two modes. **iptables:** writes DNAT rules — fast but doesn't scale well past ~10k services. **IPVS:** kernel-level L4 LB, scales to 100k+ services, more algorithms (round-robin, least-conn). Both watch the Endpoints API and update rules.

**Q: What's the difference between ClusterIP and Headless Service?**
A: ClusterIP gives one VIP, kube-proxy load-balances to Pods. Headless (`clusterIP: None`) returns A records for each Pod — clients see all Pod IPs and balance themselves. Use headless for StatefulSets where identity matters.

**Q: How does Ingress differ from a Service of type LoadBalancer?**
A: LoadBalancer = one cloud LB per Service ($$$, L4 mostly). Ingress = one LB total, fan out to many Services via host/path rules (L7). Use Ingress for HTTP, LoadBalancer for TCP/UDP or unique IPs.

**Q: A Pod can't reach a Service. Where do you look?**
A: (1) Service selector matches Pod labels? `kubectl get endpoints svc-name` — should have IPs. (2) Pod readiness probe passing? Failing pods aren't in endpoints. (3) NetworkPolicy blocking? (4) DNS — `nslookup svc-name` inside the source pod. (5) Cross-namespace? Use FQDN.

**Q: How would you implement zero-trust networking in K8s?**
A: (1) Default-deny NetworkPolicies per namespace. (2) Allow only required ingress/egress per workload. (3) Service mesh for mTLS between Pods. (4) Authorization policies at the mesh level (which service may call which).

**Q: Why might `externalTrafficPolicy: Local` cause uneven load?**
A: The cloud LB still spreads connections round-robin across nodes. Nodes with 0 matching Pods fail health checks and are dropped — but among healthy nodes, those with more Pods carry more load only if Pod count varies. With even Pod spread (e.g., topology spread constraints), it works fine.

## Common Pitfalls

- Service selector typo — Endpoints empty, no traffic, no obvious error.
- Default-permit networking — no NetworkPolicies means a compromised Pod can scan the whole cluster.
- Using Flannel and writing NetworkPolicies — Flannel doesn't enforce them; install Calico/Cilium.
- Ingress controller missing — manifests apply fine but nothing routes (no controller, no proxy).
- DNS slowness due to `ndots: 5` — each external lookup hits 5 NXDOMAIN attempts. Set `ndots: 2`.

## Related

- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [08-kubernetes-storage.md](08-kubernetes-storage.md)
- [22-networking-fundamentals.md](22-networking-fundamentals.md)
- [23-load-balancers-and-proxies.md](23-load-balancers-and-proxies.md)
- [25-service-mesh-istio.md](25-service-mesh-istio.md)
