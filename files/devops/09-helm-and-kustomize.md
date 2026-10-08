# Helm & Kustomize

> **TL;DR:** Helm templates YAML with Go templating + a values file — great for shipping reusable apps. Kustomize patches base YAML with overlays — great for environment-specific tweaks of your own apps. Many teams use both.

## When to Use What

| | Helm | Kustomize |
|---|---|---|
| You're **packaging** an app for many users | Yes | Awkward |
| You're **deploying** your own app to dev/staging/prod | Either | Often simpler |
| Heavy templating / conditionals | Yes | No (intentionally) |
| Patch a few fields per env | Overkill | Sweet spot |
| Pure YAML, no templating language | No | Yes |
| Built into kubectl | No (separate binary) | `kubectl apply -k` |

Many production setups: **Helm for third-party charts** (Postgres, Prometheus, cert-manager) + **Kustomize for your own apps**.

---

## Helm

### Chart Structure

```
myapp/
├── Chart.yaml              # name, version, dependencies
├── values.yaml             # default values
├── values-prod.yaml        # prod overrides (optional)
├── charts/                 # sub-charts (dependencies)
├── templates/
│   ├── _helpers.tpl        # template helpers
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── NOTES.txt           # printed after install
└── .helmignore
```

### Chart.yaml

```yaml
apiVersion: v2
name: myapp
description: My web application
type: application
version: 0.3.1                 # chart version
appVersion: "1.4.0"            # the app's version
dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml — Sane Defaults

```yaml
replicaCount: 2

image:
  repository: ghcr.io/me/myapp
  tag: ""                       # default to .Chart.AppVersion
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  hosts:
    - host: myapp.example.com
      paths: [{ path: /, pathType: Prefix }]

resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 500m, memory: 512Mi }

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true
  auth:
    username: app
    database: app
```

### Template — Deployment

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels: {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels: {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "myapp.selectorLabels" . | nindent 8 }}
      annotations:
        # Roll Pods when config changes
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 3000
          resources: {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.extraEnv }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
```

### `_helpers.tpl` — The Standard Boilerplate

```gotemplate
{{- define "myapp.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{- define "myapp.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### CLI Workflow

```bash
helm create myapp                          # scaffold
helm lint ./myapp
helm template ./myapp -f values-prod.yaml  # render locally — diff before apply
helm install api ./myapp -n prod \
    --create-namespace -f values-prod.yaml
helm upgrade api ./myapp -n prod -f values-prod.yaml --atomic
helm history api -n prod
helm rollback api 3 -n prod                # to revision 3
helm uninstall api -n prod

# Public chart
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install pg bitnami/postgresql \
    --set auth.postgresPassword=$PG_PASS \
    -n data --create-namespace
```

### `--atomic` and `--wait`

Production deploys should be `helm upgrade --install api ./myapp --atomic --timeout 5m`. `--atomic` rolls back automatically on failure. `--wait` blocks until resources are ready. Combine for safe CD.

### Templating Tricks

```gotemplate
{{- if .Values.ingress.enabled }}
# include block conditionally
{{- end }}

{{ .Values.foo | default "bar" }}
{{ .Values.foo | quote }}
{{ .Values.foo | upper }}
{{ .Values.list | join "," }}

# Loop
{{- range .Values.hosts }}
- host: {{ .name }}
{{- end }}

# Looking up another resource
{{- $cm := lookup "v1" "ConfigMap" "kube-system" "cluster-info" }}

# Required value
{{ required "image.repository required" .Values.image.repository }}
```

### Common Helm Pitfalls

- Hand-editing rendered YAML in cluster — next `helm upgrade` overwrites it.
- Forgetting `--atomic` → broken release stuck mid-rollout.
- Storing secrets in `values.yaml` and committing to git — use [SOPS](https://github.com/getsops/sops), Sealed Secrets, or `--set` from a CI secret.
- Whitespace bugs in templates — Helm is whitespace-sensitive. Use `{{-` and `-}}`. `helm lint` and `helm template` catch most.
- Upgrading the CRDs in a chart — Helm 3 doesn't manage CRDs in `templates/`. Put them in `crds/` (installed once) or apply separately.

---

## Kustomize

Patches plain YAML — no templating, no Go syntax. Built into `kubectl` since 1.14.

### Directory Layout

```
manifests/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── replicas-patch.yaml
    │   └── configmap.yaml
    ├── staging/
    │   └── ...
    └── prod/
        └── ...
```

### Base

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app.kubernetes.io/name: api
  app.kubernetes.io/managed-by: kustomize

images:
  - name: myapp/api
    newTag: 1.4.0                  # override tag from the base
```

### Overlay

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
namePrefix: prod-
resources:
  - ../../base
patches:
  - path: replicas-patch.yaml
  - patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: 1Gi
    target:
      kind: Deployment
      name: api
configMapGenerator:
  - name: api-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
secretGenerator:
  - name: api-secrets
    envs:
      - secret.env                 # NOT committed
images:
  - name: myapp/api
    newTag: 1.4.0                  # immutable tag for prod
```

```yaml
# overlays/prod/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 5
```

### Apply

```bash
kubectl apply -k overlays/prod
kubectl kustomize overlays/prod | less   # render only, no apply
kubectl diff -k overlays/prod
```

### Strategic-Merge vs JSON Patch

- **Strategic merge** (default): smart merge respecting `patchMergeKey` on maps/lists. Best when matching by name.
- **JSON patch (RFC 6902)**: explicit ops (`add`/`remove`/`replace`) — needed when targeting list indices or arbitrary paths.

### configMapGenerator — Hashes for the Win

```yaml
configMapGenerator:
  - name: api-config
    literals:
      - LOG_LEVEL=info
```

This produces `api-config-bcdf76hgkt` (suffix is a hash of the content). When you change the value, the suffix changes, the Deployment's reference updates, and Pods roll — **no manual rollout-restart needed**.

### Common Kustomize Pitfalls

- Confusing strategic vs JSON patches — wrong format silently no-ops.
- Forgetting `behavior: merge` on configMapGenerator — overlay creates a *second* ConfigMap with the same name, conflict.
- Multiple bases in an overlay sharing the same resource name — conflicts.
- `namePrefix` + secret refs by name — patches in overlays must also include the prefix.

---

## Helmfile / ArgoCD CD Patterns

For multi-app, multi-cluster deploys you'll see:

- **Helmfile:** declarative wrapper over helm install — many releases in one file.
- **ArgoCD / Flux:** GitOps — Git is the source of truth; controller reconciles cluster to match.

ArgoCD `Application` manifest pointing at a Kustomize overlay:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-prod
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  source:
    repoURL: git@github.com:me/infra.git
    path: manifests/overlays/prod
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Interview Questions

**Q: Helm vs Kustomize — when which?**
A: Helm when distributing a reusable, parameterized app to many users (charts have semantic versions, dependency management, hooks). Kustomize when you own the manifests and only need env-specific tweaks — no templating language, just patches. Many teams use Kustomize for their apps + Helm for third-party charts.

**Q: How does Helm track what it deployed?**
A: Each `helm install`/`upgrade` creates a release. Release manifests are stored as Secrets (default) in the release's namespace (`sh.helm.release.v1.<name>.v<rev>`). `helm history` / `helm rollback` use these.

**Q: How do you roll Pods when a ConfigMap changes (Helm)?**
A: Add `checksum/config: {{ include ... | sha256sum }}` annotation in the Pod template. When the ConfigMap renders differently, the checksum changes → Pod spec changes → Deployment rolls.

**Q: How does Kustomize avoid that problem?**
A: `configMapGenerator` produces a name with a content hash suffix (`api-config-bcdf76hgkt`). Change the value → new name → Deployment ref updates → roll.

**Q: How do you secure secrets in a Helm/Kustomize workflow?**
A: Don't commit plaintext. Options: (1) [SOPS](https://github.com/getsops/sops) with age/PGP/KMS — encrypted at rest, decrypted in CI. (2) Sealed Secrets controller — public-key encrypted, cluster-decryptable. (3) External Secrets Operator — sync from AWS/Vault/GSM at runtime. (4) `--set` from CI secrets.

**Q: What's `helm upgrade --atomic` do?**
A: If the upgrade fails (timeout, failed hook, unhealthy Pods), Helm automatically rolls back to the previous release. Combine with `--wait` and `--timeout`.

## Common Pitfalls

- Editing live K8s objects that Helm/Kustomize manages — next deploy overwrites your fix.
- Helm chart with no `_helpers.tpl` labels → upgrading the chart changes selectors → immutable selector error → recreate required.
- Storing prod values + secrets in the same `values.yaml` in git.
- Using `latest` image tag in a Helm chart — no rollback by image, only by chart revision.
- Misusing JSON patch in Kustomize where strategic merge would do — fragile, breaks on YAML shape changes.

## Related

- [04-docker-basics.md](04-docker-basics.md)
- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [10-ci-cd-fundamentals.md](10-ci-cd-fundamentals.md)
- [21-security-devsecops.md](21-security-devsecops.md)
