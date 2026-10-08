# Security & DevSecOps

> **TL;DR:** Shift security left into the pipeline. Don't store secrets in repos / images. Scan dependencies and containers. Sign artifacts. Run with least privilege. Track CVEs. Threat-model new features.

## The Pipeline-as-Security-Control Pattern

```
[code]   --> SAST + secret scan        (pre-commit + PR)
[deps]   --> SCA / dep scan            (PR)
[image]  --> image scan (Trivy/Snyk)   (build stage)
[image]  --> image sign (cosign)       (publish stage)
[IaC]    --> tfsec / checkov / KICS    (PR)
[runtime]--> RASP / runtime scanning   (in-cluster)
[runtime]--> CSPM / cloud config audit (continuous)
```

Each layer cheap to add, multiplicatively raises the bar.

## Secrets Management

### The Hierarchy of Bad

1. **Worst:** secrets in git (especially `.env` files committed).
2. Secrets in CI as long-lived plaintext variables.
3. Secrets in K8s ConfigMaps.
4. Secrets in K8s Secrets without encryption-at-rest.
5. **Acceptable:** Secrets in K8s Secrets with etcd encryption + RBAC + audit.
6. **Better:** External secrets manager + Pod fetches at runtime via Workload Identity.
7. **Best:** Short-lived dynamic secrets (Vault leases, AWS STS, GCP workload identity tokens).

### Managers

- **HashiCorp Vault** — most flexible; dynamic secrets, transit encryption, PKI.
- **AWS Secrets Manager / SSM Parameter Store** — native AWS, IAM-gated, rotation.
- **GCP Secret Manager** — IAM-gated, versioned.
- **Azure Key Vault** — secrets + keys + certs.
- **1Password / Doppler** — developer-friendly UX.

### External Secrets Operator (K8s)

Sync from a manager into native K8s Secrets:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-creds }
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-store
    kind: ClusterSecretStore
  target:
    name: db-creds                         # creates this K8s Secret
  data:
    - secretKey: url
      remoteRef:
        key: prod/app/db
        property: url
```

Now your app mounts `db-creds` like any normal Secret; rotation flows automatically.

### Sealed Secrets

Encrypt secret YAML with a public key; only the in-cluster controller can decrypt:

```bash
kubectl create secret generic db-creds --from-literal=password=s3cret --dry-run=client -o yaml \
  | kubeseal -o yaml > db-creds-sealed.yaml
# Commit db-creds-sealed.yaml to git safely
```

Great for GitOps workflows where you don't want to depend on a remote manager at apply time.

### Rotation

Static secrets must rotate. Cadence:
- Database passwords: 90 days minimum.
- API keys for third parties: per provider policy.
- Service account keys: avoid them entirely (use OIDC / Workload Identity).
- Compromise → rotate immediately + audit everywhere it was used.

## Image Security

### Scan Every Build

```bash
trivy image myapp:1.4.0 --severity HIGH,CRITICAL --exit-code 1
docker scout cves myapp:1.4.0
grype myapp:1.4.0
```

Wire into CI to fail builds on HIGH/CRITICAL CVEs. Add allowlist with expiry for known-tolerated issues.

### Sign and Verify

```bash
# Sign
cosign sign --key cosign.key ghcr.io/me/api:1.4.0

# Verify in cluster admission (with policy-controller, kyverno, etc.)
cosign verify --key cosign.pub ghcr.io/me/api:1.4.0
```

K8s admission controller (Kyverno / Connaisseur / sigstore policy-controller) blocks unsigned images.

### Slim Bases

- Distroless / scratch / chiseled Ubuntu → no shell, no apt, no curl → smaller attack surface and CVE pool.
- Pin by digest: `FROM node@sha256:...` — defeats tag repointing.

### SBOM (Software Bill of Materials)

Generate at build:

```bash
syft myapp:1.4.0 -o spdx-json > sbom.json
trivy image --format cyclonedx -o sbom.cdx.json myapp:1.4.0
```

Attach as an OCI artifact next to your image. Required by U.S. EO 14028 / many enterprise procurement teams. Lets you answer "are we vulnerable to log4shell?" instantly.

## Dependency Scanning (SCA)

| Tool | Languages |
|------|-----------|
| Dependabot (GitHub) | Most major languages |
| Snyk | Most major languages |
| GitHub Advanced Security | Code + deps + secrets |
| OWASP Dependency-Check | Java, .NET, Node, Python |
| `npm audit` / `pip audit` / `cargo audit` | Native ecosystem tools |
| `osv-scanner` | Cross-ecosystem, OSV DB |

PR-time alerts + a dashboard of open vulnerable deps + an SLA to remediate by severity.

## SAST — Static Application Security Testing

| Tool | Notes |
|------|-------|
| Semgrep | Customizable rules, language-agnostic, fast |
| CodeQL | GitHub-native, very deep analysis |
| SonarQube | Mature, broad ruleset |
| Bandit | Python-specific |
| Brakeman | Ruby/Rails |
| gosec | Go |

Add as a PR check. Tune rules — false positives kill adoption.

## IaC Scanning

- **tfsec / Checkov / KICS / Terrascan** for Terraform/CloudFormation/K8s YAML/Helm.
- **kube-linter / kubesec** for K8s manifests.

Catches: public S3, RDS without encryption, K8s containers running as root, security group 0.0.0.0/0 on SSH, etc.

```bash
checkov -d infra/ --framework terraform
kube-linter lint manifests/
```

## OWASP — the App-Sec Big Ten

The 2021 [OWASP Top 10](https://owasp.org/Top10/):

1. Broken Access Control
2. Cryptographic Failures
3. Injection
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable and Outdated Components
7. Identification and Authentication Failures
8. Software and Data Integrity Failures
9. Security Logging and Monitoring Failures
10. Server-Side Request Forgery (SSRF)

Know them. Bring up in interview when discussing app design.

## Kubernetes Security

### Pod Security

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    fsGroup: 10001
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: app
      securityContext:
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: [ALL]
```

Enforce cluster-wide with [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) (Baseline / Restricted) via `PodSecurity` admission, Kyverno, or OPA Gatekeeper.

### RBAC — Least Privilege

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: api-reader, namespace: prod }
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: api-reader, namespace: prod }
subjects:
  - kind: ServiceAccount
    name: api
    namespace: prod
roleRef:
  kind: Role
  name: api-reader
  apiGroup: rbac.authorization.k8s.io
```

Avoid `cluster-admin`. Avoid `*` in verbs/resources. Audit periodically.

### NetworkPolicies

Default-deny + explicit allows. See [07-kubernetes-networking.md](07-kubernetes-networking.md).

### Admission Controllers

Enforce cluster policy on create/update:
- **Kyverno** — Kubernetes-native YAML policies, easy to write.
- **OPA Gatekeeper** — Rego language, very flexible.
- **Connaisseur / sigstore policy-controller** — image signature verification.

Example Kyverno policy: "block containers running as root":

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: no-root }
spec:
  validationFailureAction: enforce
  rules:
    - name: require-non-root
      match: { resources: { kinds: [Pod] } }
      validate:
        message: "Containers must runAsNonRoot."
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
```

## Supply Chain — SLSA

[SLSA](https://slsa.dev) — Supply-chain Levels for Software Artifacts:

- **L1:** Build is documented.
- **L2:** Build is hosted, generates provenance.
- **L3:** Build is non-falsifiable, hermetic.
- **L4:** Two-party review + reproducible builds.

Practical wins to reach L2/L3: GitHub Actions OIDC + sigstore/cosign + provenance attestation + pinned action SHAs + Dependabot.

## Cloud Posture

CSPM (Cloud Security Posture Management) — continuous compliance scanning:
- **AWS:** Security Hub, Config Conformance Packs, Inspector.
- **GCP:** Security Command Center.
- **Azure:** Defender for Cloud.
- Cross-cloud: Wiz, Orca, Prisma Cloud.

Track: public buckets, overly-broad IAM, unused privileged roles, unencrypted disks, IMDSv1 enabled, no MFA on root.

## Defense in Depth — A Checklist

- [ ] Branch protection on `main` + signed commits + required reviewers.
- [ ] Secret scanning on commits and PRs.
- [ ] SAST + SCA on every PR.
- [ ] Container scan + sign at build.
- [ ] Image admission policy: only signed images.
- [ ] No root containers; read-only root FS; seccomp default.
- [ ] NetworkPolicies (default-deny + allow-lists).
- [ ] RBAC minimization; no cluster-admin for humans.
- [ ] Secrets via Workload Identity / external manager; never in env at rest in git.
- [ ] Audit logging + log retention; CloudTrail / Cloud Audit / Activity Logs on.
- [ ] CSPM scanning + remediation SLA.
- [ ] Penetration test / red team annually.
- [ ] Incident response plan with security on-call.

## Interview Questions

**Q: Walk me through how a deployed container's secret should reach it.**
A: Best: app fetches from a manager (Vault / Secrets Manager) using Workload Identity at startup — short-lived token, no static cred in K8s. Acceptable: External Secrets Operator syncs the manager into a K8s Secret; Pod mounts. Bad: secret hardcoded in image; secret as plaintext env in `Deployment` YAML committed to git.

**Q: How do you prevent supply-chain attacks like SolarWinds / event-stream?**
A: Pin dependencies (lockfiles, action SHAs). Verify signatures (cosign on images, sigstore on packages). Generate SBOMs. Scan dependencies continuously (Dependabot, Snyk). Provenance attestations (SLSA L2+). Restrict CI to a controlled runner pool. Review high-impact deps before bumping.

**Q: What's the principle of least privilege in K8s?**
A: ServiceAccount per workload (not the default SA). RBAC scoped to namespace + verbs needed. No cluster-admin. NetworkPolicies restrict pod-to-pod traffic. Pod-level securityContext drops capabilities + runs non-root. Combine: bare minimum to function, nothing more.

**Q: Image scan failed with HIGH CVEs. Now what?**
A: Triage: is the CVE reachable in your code path? If yes — patch (bump base image / dep). If no but easy patch — patch anyway. If patch unavailable — document allowlist with expiry + compensating control + revisit. Block prod deploys until resolved or risk-accepted.

**Q: How does cosign work?**
A: Sigstore-based image signing. cosign signs an image's digest with a key (or keyless via OIDC). The signature is stored next to the image in the registry. Admission controller verifies the signature before letting the image run. Provides image integrity + provenance.

**Q: What's the difference between SAST, DAST, and SCA?**
A: SAST — analyze source code (or compiled artifact) without running it. DAST — exercise the running app (web requests, fuzzing). SCA — analyze third-party dependencies for known CVEs. Layer all three.

## Common Pitfalls

- `.env` checked into git with prod creds — rotate everything, audit clones, accept history is poisoned (or rewrite with filter-repo).
- IMDSv1 on EC2 — SSRF in app gives full IAM role to attacker.
- Cluster-admin bindings for service accounts because "it was easier."
- `:latest` images + no admission policy — bad image promoted to prod silently.
- Static AWS keys in CI — rotate them or use OIDC.
- Logs of full JWTs / API tokens — leaks in incident logs, compliance fail.
- No CVE SLA — CVEs pile up until audit time, then a panic sprint.

## Related

- [11-github-actions.md](11-github-actions.md)
- [13-iac-terraform.md](13-iac-terraform.md)
- [15-cloud-aws-basics.md](15-cloud-aws-basics.md)
- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [19-logging-best-practices.md](19-logging-best-practices.md)
- [../serious-prep/](../serious-prep/)
