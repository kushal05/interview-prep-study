# Kubernetes Fundamentals

> **TL;DR:** K8s is a desired-state engine. You declare objects in YAML; controllers reconcile reality toward your spec. Know the core objects, the control loop, and how to debug a stuck Pod.

## Architecture in One Diagram

```
+--------- Control Plane ---------+      +-------- Worker Node ---------+
| kube-apiserver   <-> etcd       |      | kubelet  (talks to API)      |
| controller-manager              | <--> | kube-proxy (Service routing) |
| scheduler                       |      | container runtime (containerd)|
| cloud-controller-manager        |      | Pods (your containers)        |
+---------------------------------+      +-------------------------------+
```

- **kube-apiserver:** the only thing that talks to etcd. Everything else (kubectl, controllers, kubelet) goes through it.
- **etcd:** the source of truth. Strongly-consistent KV. **Back this up.**
- **scheduler:** decides which node a new Pod runs on.
- **controller-manager:** runs the reconciliation loops (Deployment, ReplicaSet, Node controllers).
- **kubelet:** on every node — ensures Pods declared on its node are actually running.
- **kube-proxy:** maintains iptables/IPVS rules so Service VIPs route to Pod IPs.

## Core Objects — The Family Tree

```
Namespace
  Pod (smallest unit; 1+ containers, shared net/storage)
    ReplicaSet (manages N Pod replicas)
      Deployment (manages ReplicaSet rollouts)
    StatefulSet (ordered, stable identities)
    DaemonSet (one Pod per node)
    Job / CronJob (run-to-completion)
  Service (stable VIP + DNS for Pods)
    Ingress (HTTP routing into Services)
  ConfigMap / Secret (config injection)
  PersistentVolumeClaim (storage request)
```

## Pod — the Atom

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api
  namespace: prod
  labels:
    app: api
    tier: backend
spec:
  containers:
    - name: api
      image: myapp/api:1.4.0
      ports:
        - containerPort: 3000
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
      resources:
        requests: { cpu: "100m", memory: "128Mi" }
        limits:   { cpu: "500m", memory: "512Mi" }
      readinessProbe:
        httpGet: { path: /health, port: 3000 }
        initialDelaySeconds: 5
        periodSeconds: 10
      livenessProbe:
        httpGet: { path: /alive, port: 3000 }
        initialDelaySeconds: 30
        periodSeconds: 30
```

**Probes are critical:**
- **readiness** — fail it → removed from Service endpoints (no traffic). Pass it → traffic.
- **liveness** — fail it → kubelet restarts the container.
- **startup** — gives slow-starting apps grace before liveness applies.

Bad liveness probes are the #1 cause of crash loops. Don't make it `/health` that hits the DB.

## Requests vs Limits

- **requests:** what scheduler reserves on the node. Used for scheduling decisions.
- **limits:** hard cap. CPU = throttled. Memory = OOMKilled.

```yaml
resources:
  requests: { cpu: 100m, memory: 128Mi }    # guarantees
  limits:   { cpu: 500m, memory: 512Mi }    # cap
```

QoS classes:
- **Guaranteed:** requests == limits for both cpu+mem. Last to be evicted.
- **Burstable:** requests < limits. Mid-priority.
- **BestEffort:** no requests/limits. First evicted.

## Deployment — the One You'll Use Most

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1            # how many can be down during update
      maxSurge: 1                  # how many extra above replicas
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: api
          image: myapp/api:1.4.0
          # ... (same as Pod above)
```

```bash
kubectl apply -f deploy.yaml
kubectl rollout status deploy/api
kubectl rollout history deploy/api
kubectl rollout undo deploy/api              # to previous revision
kubectl rollout undo deploy/api --to-revision=3
kubectl set image deploy/api api=myapp/api:1.4.1
```

## Service — Stable Endpoint for Pods

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP                  # default: in-cluster VIP
  selector:
    app: api                       # picks Pods with this label
  ports:
    - port: 80                     # what the Service exposes
      targetPort: 3000             # the Pod port
```

`api.prod.svc.cluster.local` resolves to the Service VIP; kube-proxy load-balances to Pods. Detail in [07-kubernetes-networking.md](07-kubernetes-networking.md).

## ConfigMap & Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: api-config }
data:
  LOG_LEVEL: info
  FEATURE_X: "true"
---
apiVersion: v1
kind: Secret
metadata: { name: db-credentials }
type: Opaque
stringData:                        # plaintext input; stored b64 in etcd
  url: "postgres://app:pw@db:5432/app"
```

```yaml
# In Pod spec
envFrom:
  - configMapRef: { name: api-config }
  - secretRef:    { name: db-credentials }
```

**Gotcha:** Secrets are base64-encoded, **not encrypted**, in etcd by default. Enable [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) or use [Sealed Secrets](https://sealed-secrets.netlify.app) / [External Secrets Operator](https://external-secrets.io) — see [21-security-devsecops.md](21-security-devsecops.md).

## Namespaces — Soft Tenancy

```bash
kubectl create namespace staging
kubectl apply -n staging -f .
kubectl config set-context --current --namespace=staging
```

Namespaces give DNS scoping (`api.prod.svc.cluster.local`), RBAC scoping, ResourceQuotas. They do **not** provide network isolation by default — use [NetworkPolicies](07-kubernetes-networking.md).

## Labels & Selectors — How Everything Connects

Labels are how Services find Pods, Deployments find ReplicaSets, monitoring scrapes targets. There's no foreign key — it's all label matching.

```bash
kubectl get pods -l app=api,tier=backend
kubectl get pods --selector='env in (staging,prod)'
kubectl label pod api-7d-x version=v2 --overwrite
```

Conventions (the [recommended labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)):
- `app.kubernetes.io/name`
- `app.kubernetes.io/instance`
- `app.kubernetes.io/version`
- `app.kubernetes.io/component`
- `app.kubernetes.io/part-of`
- `app.kubernetes.io/managed-by`

## Debugging Workflow

```bash
kubectl get pods -A                       # cluster-wide view
kubectl get pods -o wide                  # with node + IP
kubectl describe pod api-7d-xyz           # events at the bottom matter most
kubectl logs api-7d-xyz                   # logs
kubectl logs api-7d-xyz -c sidecar        # specific container
kubectl logs api-7d-xyz --previous        # last crashed instance's logs
kubectl exec -it api-7d-xyz -- sh         # shell in
kubectl port-forward svc/api 8080:80      # local 8080 -> Service
kubectl top pods                          # CPU/mem (needs metrics-server)
kubectl get events --sort-by='.lastTimestamp' -n prod
```

### `kubectl apply` vs `replace` vs `edit` vs `create`

| Verb | Behavior |
|------|----------|
| `apply -f` | declarative; merges with stored "last-applied" annotation. **Default choice.** |
| `create -f` | fails if object exists. Use once. |
| `replace -f` | imperative; replaces the entire object — drops fields not in your YAML. |
| `edit` | opens current spec in `$EDITOR`. Dangerous; not in source control. |
| `patch` | partial update; useful in scripts. |

**Gotcha — apply vs replace:** `apply` does a three-way merge between (last-applied, current, your file). `replace` is a full overwrite — if a controller injected fields (annotations, sidecar), `replace` removes them. Stick with `apply` and `kustomize`.

## Autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
```

Custom metrics need [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter) or [KEDA](https://keda.sh) (event-driven autoscaling — queue depth, Kafka lag).

## Workload Types Cheat Sheet

| Kind | Use case |
|------|----------|
| Deployment | Stateless web apps |
| StatefulSet | DBs, Kafka, anything needing stable identity + ordered scale |
| DaemonSet | One per node — log shippers, CNI agents, node-exporter |
| Job | Batch one-shot (DB migration) |
| CronJob | Scheduled batch (`0 2 * * *` backup) |
| ReplicaSet | Almost never used directly — Deployment manages it |

## Interview Questions

**Q: What's the difference between a Deployment and a StatefulSet?**
A: Deployment = stateless, Pods are interchangeable, random names, no stable identity. StatefulSet = stable network identity (`api-0`, `api-1`), stable storage, ordered create/scale/delete. Use StatefulSet for DBs, Kafka, anything needing predictable identity.

**Q: A Pod is stuck in `CrashLoopBackOff`. How do you debug?**
A: `kubectl describe pod` (events at bottom). `kubectl logs pod --previous` (last crash). Check resource limits (OOMKilled). Check readiness/liveness probes (bad URL? db dependency in liveness?). Check env vars / missing secrets.

**Q: Pod is `Pending` forever. Why?**
A: No node has enough resources for requests (check `kubectl describe pod` events — "Insufficient cpu/memory"). Or no node matches nodeSelector / taints / tolerations. Or PVC isn't binding.

**Q: How does a Service route traffic?**
A: kube-proxy programs iptables/IPVS rules: traffic to ServiceIP gets DNAT'd to one of the Pod IPs matching the Service selector. DNS gives you `svc-name.ns.svc.cluster.local`. CoreDNS resolves it to the Service ClusterIP.

**Q: When would you use a DaemonSet?**
A: When you need exactly one Pod per node — log shippers (fluentd), monitoring agents (node-exporter), CNI plugins, GPU drivers.

**Q: What's the difference between `requests` and `limits`?**
A: Requests = what the scheduler reserves. Limits = the cap. CPU over limit → throttled. Memory over limit → OOMKilled. Match them for Guaranteed QoS (best for critical workloads).

## Common Pitfalls

- Using `replace` (which drops controller-managed fields) instead of `apply`.
- Liveness probe that hits the DB — DB blip → all Pods restart → outage.
- No `resources.requests` — scheduler packs nodes blindly; bad neighbors cause throttling.
- Setting memory limit < actual usage → OOMKilled in a loop.
- Storing secrets in ConfigMaps — they're visible in `kubectl get cm -o yaml` and in describe.
- Editing live objects with `kubectl edit` instead of editing your YAML in git.
- Forgetting `imagePullPolicy: Always` when using `:latest` (which you shouldn't, but if you do).

## Related

- [07-kubernetes-networking.md](07-kubernetes-networking.md)
- [08-kubernetes-storage.md](08-kubernetes-storage.md)
- [09-helm-and-kustomize.md](09-helm-and-kustomize.md)
- [21-security-devsecops.md](21-security-devsecops.md)
- [../system-design/](../system-design/)
