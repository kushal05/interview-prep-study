# CI/CD Fundamentals

> **TL;DR:** CI verifies every change builds + tests. CD ships that verified artifact to environments — with gates, observability, and reversible deploys (canary/blue-green/rolling).

## Vocabulary — Get These Right in an Interview

- **CI (Continuous Integration):** Every commit triggers build + test against `main`. Catches breaks early.
- **Continuous Delivery:** Every passing build is *deployable* (artifact ready, deploy is a manual button press).
- **Continuous Deployment:** Every passing build is *deployed* automatically to prod.
- **Pipeline:** the orchestrated steps.
- **Stage:** a logical phase (build, test, deploy).
- **Job:** a unit of work within a stage (often on its own runner).
- **Artifact:** the binary/container/package produced by the pipeline.
- **Environment:** dev/staging/prod (or any named target).
- **Gate:** approval or automated check that controls promotion between environments.

## A Sensible Pipeline

```
[ commit ]
    |
    v
[ lint + typecheck + unit tests ]   <-- fast feedback (<5 min)
    |
    v
[ build artifact / docker image ]   <-- cache layers
    |
    v
[ integration tests ]                <-- spin up deps with compose
    |
    v
[ security scans ]                   <-- SAST, dep scan, image scan
    |
    v
[ publish to registry ]
    |
    v
[ deploy to dev ]   --> [ smoke tests ]
    |
    v
[ deploy to staging ]  --> [ e2e tests ]
    |
    v
[ manual approval gate ]
    |
    v
[ deploy to prod (canary) ]
    |
    v
[ progressive rollout / observability checks ]
    |
    v
[ full prod ]
```

## Build Once, Promote Many

The artifact built in stage 1 should be the artifact deployed in prod. Never rebuild per environment — config differs, code doesn't.

- Tag images by commit SHA (immutable) **and** semantic version (human-friendly).
- Config via env vars, ConfigMaps, Secrets — not baked into the image.
- Helm/Kustomize values differ per env; the image does not.

```
ghcr.io/me/api:1.4.0
ghcr.io/me/api:1.4.0-rc.3
ghcr.io/me/api:sha-abc1234     <-- pin to this in prod
ghcr.io/me/api:main             <-- ONLY for dev/staging "always latest"
```

## Deployment Strategies

### Rolling Update (the default for K8s Deployments)

- Replace old Pods incrementally (e.g., 25% at a time).
- Zero downtime if readiness probes are honest.
- Both versions serve traffic simultaneously during the rollout — schema/contract must be backward compatible.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```

### Blue-Green

- **Blue** = current prod, **Green** = new version on parallel infra.
- Cut traffic over once green is verified (DNS swap, LB target group switch, K8s Service selector flip).
- Instant rollback by swapping back.
- Double resources during cutover. Database compatibility still required.

```bash
# K8s service points to "blue" — flip to green
kubectl patch svc api -p '{"spec":{"selector":{"version":"green"}}}'
```

### Canary

- Send 1% → 5% → 25% → 100% of traffic to the new version, monitoring error rate / latency at each step.
- Cap the blast radius.
- Needs traffic-splitting capability — service mesh, weighted ingress, ALB target group weights, or Argo Rollouts.

```yaml
# Argo Rollouts canary
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - analysis: { templates: [{ templateName: success-rate }] }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
```

### Feature Flags

- Decouple deploy from release. Ship code dark; flip the flag when ready.
- Per-user / per-cohort targeting.
- LaunchDarkly, Unleash, OpenFeature, ConfigCat, or roll your own with a config service.

### Which to Use?

| Strategy | Risk | Cost | Speed of rollback |
|----------|------|------|-------------------|
| Rolling | Medium | $ | Manual revert |
| Blue-green | Low | $$ (2x infra) | Instant (swap back) |
| Canary | Lowest | $$ (mesh / tooling) | Reduce weight to 0 |
| Feature flag | Lowest | $ | Toggle off |

## Artifacts & Registries

| Type | Registry options |
|------|------------------|
| Docker images | Docker Hub, GHCR, ECR, GCR, Artifactory, Harbor |
| npm | npmjs.org, GitHub Packages, Verdaccio |
| Maven | Maven Central, Artifactory, Nexus |
| Helm charts | OCI registries (since Helm 3.8), ChartMuseum |
| Generic | S3, GCS, Artifactory |

Tag everything. Sign critical artifacts (`cosign`). Scan before promoting.

## Environments & Promotion

Common pattern: `dev → staging → prod`. Each environment:

- Has its own config (DB host, secrets, scale).
- Receives **the same artifact**, not a rebuild.
- Has automated tests gating promotion.
- Has its own monitoring + on-call paging (prod paging only).

Promotion via:
- Tag push (CI on tag deploys to staging; further tag → prod).
- Branch push (`develop` → staging, `main` → prod).
- Manual approval in pipeline UI.
- GitOps (commit to a `prod` overlay path; ArgoCD reconciles).

## GitOps in 60 Seconds

- **Git is the source of truth** for what runs in the cluster.
- A controller (Argo CD, Flux) reconciles the cluster state to match git.
- Deployments = git commits. PRs become deploys.
- Rollback = `git revert`.
- Audit trail is free (git history).

```
[ developer ] --PR--> [ app code repo ]
                          | CI builds image, bumps tag in infra repo
                          v
                      [ infra repo (manifests) ]
                          |
                          v
                      [ ArgoCD watches infra repo ]
                          |
                          v
                      [ cluster syncs ]
```

## Caching — The Main Speedup Lever

- **Dependency caches:** `~/.npm`, `~/.m2`, `~/.cargo`, `go mod` cache.
- **Docker layer cache:** `--cache-from=type=registry,ref=ghcr.io/me/api:buildcache`.
- **Test result cache:** Nx, Turborepo, Bazel — skip running tests for unchanged code.

```yaml
# GitHub Actions — cache npm
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: npm
```

## Parallelization

- Run lint + unit tests + type-check in parallel jobs.
- Matrix builds for multiple Node/Python versions.
- Split test suite by file count or by historical timing.

```yaml
strategy:
  matrix:
    node: [18, 20, 22]
    os: [ubuntu-latest, macos-latest]
```

## Secrets in Pipelines

- Never echo secrets — they may show in logs (most CI providers redact, but don't trust).
- Inject via environment variable from CI's secret store.
- Use short-lived OIDC tokens to assume cloud roles, not static `AWS_ACCESS_KEY_ID`.
- Scope secrets to the minimum (per-repo, per-environment).
- Rotate aggressively.

```yaml
# GitHub Actions: assume AWS role via OIDC — no static keys
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123:role/ci-deploy
    aws-region: us-east-1
```

## Pipeline Anti-Patterns

- **Snowflake runners** — manually configured agents that no one can reproduce. Pin to ephemeral, image-based runners.
- **CI as deploy script** — long bash scripts that ssh + scp. Use proper tooling (Helm, ArgoCD, Terraform).
- **No staging** — production is your test environment.
- **No rollback plan** — "we'll roll forward." Sure, until you can't.
- **Tests in CI that aren't in dev** — devs can't reproduce CI failures.

## Observability of the Pipeline Itself

Track pipeline health like a service:

- **DORA metrics:** Deployment frequency, Lead time for changes, Change failure rate, Mean time to restore.
- Build success rate per branch.
- Flaky test rate.
- Average pipeline duration; p99 duration.

A pipeline that's red 30% of the time is broken — fix it before adding more checks.

## Interview Questions

**Q: Continuous delivery vs continuous deployment?**
A: Both pipeline runs are automated, but Delivery requires a *manual approval* before prod; Deployment goes straight to prod on every green build.

**Q: Walk me through a CI/CD pipeline for a microservice.**
A: Lint + unit test (fast feedback) → build container (cache layers) → integration tests against compose stack → security scans (Trivy/Snyk) → push to registry tagged by SHA → deploy to dev → smoke tests → deploy to staging → e2e tests → manual approval (or automated DORA gate) → canary to 5% prod → analyze metrics → progressive ramp to 100%. Rollback = previous image tag or feature-flag off.

**Q: Blue-green vs canary?**
A: Blue-green: 100% switch between two complete environments; instant rollback but doubles infra cost and gives no opportunity to detect issues before they affect everyone. Canary: gradual traffic shift (1→5→25→100%); catches issues with a small blast radius, but requires traffic-splitting infra and a way to compare versions.

**Q: How do you deploy a backward-incompatible DB migration?**
A: Expand-contract (two deploys). (1) Expand: deploy schema change additive — new columns / new tables; old code still works. (2) Switch reads/writes in app code, deploy. (3) Contract: drop old columns. Never combine code + breaking migration in one deploy.

**Q: How do you handle secrets in CI?**
A: Use the CI provider's secret store (GitHub Actions secrets, GitLab variables). Prefer OIDC federation to assume cloud roles — no long-lived keys. Scope per-environment. Audit access. Rotate on every employee departure.

**Q: Pipeline takes 40 minutes. How do you cut it?**
A: Profile first. Common wins: parallelize jobs, cache deps + Docker layers, split tests by timing, skip unchanged packages (monorepo tools), run e2e tests on schedule rather than every PR, use bigger runners for build, prebuild base images.

## Common Pitfalls

- Rebuilding the artifact per environment — staging passes, prod fails because it was a different binary.
- Long-lived static cloud creds in CI — rotate them or use OIDC.
- Tests that depend on prod data / external APIs — flaky CI, slow signal.
- No promotion gates — a bug in PR #432 ships to prod 8 minutes later.
- Deploying outside the pipeline ("just SSH in and fix it") — drift from declared state, no audit.
- DB migrations that lock tables in prod — long migrations should run offline or in async batches.

## Related

- [11-github-actions.md](11-github-actions.md)
- [12-jenkins-basics.md](12-jenkins-basics.md)
- [13-iac-terraform.md](13-iac-terraform.md)
- [21-security-devsecops.md](21-security-devsecops.md)
- [20-incident-response.md](20-incident-response.md)
