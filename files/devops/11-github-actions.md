# GitHub Actions

> **TL;DR:** YAML workflows in `.github/workflows/`. Jobs run on runners, steps run in jobs. Use composite/reusable workflows to DRY, OIDC for cloud creds, and **never use `${{ ... }}` interpolation around untrusted input**.

## File Anatomy

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:                    # manual trigger
  schedule:
    - cron: "0 6 * * *"                 # nightly 06:00 UTC

permissions:
  contents: read                        # least privilege
  id-token: write                       # for OIDC

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true              # cancel older runs on same branch

env:
  NODE_VERSION: "20"
  REGISTRY: ghcr.io

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage
      - uses: codecov/codecov-action@v4
```

## Triggers (`on:`) — The Important Ones

```yaml
on:
  push:
    branches: [main]
    paths: ['src/**', 'package*.json']  # only when these change
    paths-ignore: ['docs/**']

  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

  workflow_call:                         # this workflow is callable by others
  workflow_run:                          # triggered by another workflow finishing
    workflows: [CI]
    types: [completed]

  release:
    types: [published]

  issues:
    types: [opened]

  schedule:
    - cron: "*/15 * * * *"               # every 15 min
```

## Runners

| Runner | When |
|--------|------|
| `ubuntu-latest` | Default for Linux builds |
| `windows-latest` | .NET builds, Windows-specific |
| `macos-latest` | iOS/macOS builds (expensive) |
| Self-hosted | Your own VMs/k8s — for private network access, GPUs, larger machines |
| Larger runners | Paid GitHub-hosted runners (4/8/16+ cores) |

```yaml
runs-on: [self-hosted, linux, x64, gpu]   # array = match all labels
```

## Steps

```yaml
steps:
  # Run a shell command
  - name: Build
    run: |
      npm ci
      npm run build
    working-directory: ./packages/api
    env:
      NODE_ENV: production
    shell: bash                          # pwsh, python, sh, cmd

  # Use a marketplace / local action
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: npm-${{ hashFiles('**/package-lock.json') }}

  # Conditional
  - name: Publish
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    run: npm publish

  # Continue on error
  - name: Optional check
    continue-on-error: true
    run: ./optional.sh
```

### Pin Actions by SHA in Production

```yaml
# Risky — tag can be moved
- uses: actions/checkout@v4

# Safer — immutable SHA
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11   # v4.1.1
```

Why: a malicious or compromised tag (`v4`) repointed to a bad commit owns your secrets. SHA pinning is required for SLSA-grade pipelines. Dependabot can keep them current.

## Matrix Builds

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false                   # don't kill siblings on one failure
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
        include:                         # extra combo
          - os: ubuntu-latest
            node: 22
            experimental: true
        exclude:                         # remove a combo
          - os: macos-latest
            node: 18
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

## Jobs, Dependencies, and Outputs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tag }}
    steps:
      - id: meta
        run: echo "tag=ghcr.io/me/app:${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.image-tag }}"
```

`needs:` creates a dependency edge. Multiple deps: `needs: [build, test]`. Jobs without `needs` run in parallel.

## Caching

```yaml
- name: Cache Go modules
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-
```

The `restore-keys` fallback gives a partial cache on first build with new deps.

For Docker, use BuildKit caching to the registry:

```yaml
- uses: docker/setup-buildx-action@v3
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/me/api:${{ github.sha }}
    cache-from: type=registry,ref=ghcr.io/me/api:buildcache
    cache-to:   type=registry,ref=ghcr.io/me/api:buildcache,mode=max
```

## Secrets and OIDC

### Secrets

```yaml
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}     # repo or org secret
```

- `secrets.GITHUB_TOKEN` is auto-generated per run; scope its permissions per workflow.
- **`secrets` are not available on `pull_request` from forks** — by design.
- Use Environments for prod secrets + required reviewers.

### Environments

```yaml
jobs:
  deploy-prod:
    environment:
      name: production
      url: https://api.example.com
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

Configure in repo settings: required reviewers, wait timer, environment secrets.

### OIDC — Stop Using Long-Lived Cloud Keys

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gh-ci-deploy
          aws-region: us-east-1
      - run: aws s3 ls
```

GitHub's OIDC provider issues a short-lived JWT; the cloud role's trust policy validates the `sub` (e.g., `repo:me/app:ref:refs/heads/main`) and grants temporary creds. No static keys to leak.

### Untrusted Input — the `pull_request_target` Trap

```yaml
# DANGEROUS
- run: |
    echo "${{ github.event.pull_request.title }}"   # PR author controls this
```

PR title → shell injection → exfiltrate secrets. Always:
- Use env vars: `env: { TITLE: ${{ github.event... }} }` then `echo "$TITLE"`.
- Avoid `pull_request_target` unless absolutely necessary (it runs in the base repo context with secrets — high-risk).

## Reusable Workflows

A workflow can call another with `workflow_call`:

```yaml
# .github/workflows/_deploy.yml
on:
  workflow_call:
    inputs:
      env:
        required: true
        type: string
    secrets:
      AWS_ROLE_ARN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to ${{ inputs.env }}"
```

```yaml
# .github/workflows/ci.yml
jobs:
  deploy-staging:
    uses: ./.github/workflows/_deploy.yml
    with:
      env: staging
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_STAGING }}
```

## Composite Actions

Bundle multiple steps as a single action your workflows can call.

```yaml
# .github/actions/setup/action.yml
name: Setup
description: Install Node and deps with cache
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with: { node-version: 20, cache: npm }
    - run: npm ci
      shell: bash
```

```yaml
# In a workflow
- uses: ./.github/actions/setup
```

## Artifacts & Releases

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: dist
    path: dist/
    retention-days: 7

# In another job
- uses: actions/download-artifact@v4
  with:
    name: dist
```

```yaml
- uses: softprops/action-gh-release@v2
  if: startsWith(github.ref, 'refs/tags/')
  with:
    files: dist/*.tar.gz
    generate_release_notes: true
```

## Common Expressions

```yaml
${{ github.ref }}                          # refs/heads/main, refs/tags/v1
${{ github.ref_name }}                     # main
${{ github.event_name }}                   # push, pull_request, ...
${{ github.actor }}                        # who triggered
${{ github.sha }}                          # commit SHA
${{ runner.os }}                           # Linux / Windows / macOS

${{ contains(github.event.head_commit.message, '[skip ci]') }}
${{ startsWith(github.ref, 'refs/tags/') }}
${{ steps.x.outputs.foo == 'bar' }}
${{ needs.build.result == 'success' }}
```

## Job-Level Conditions for Deploy Gating

```yaml
jobs:
  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: [test, scan]
    runs-on: ubuntu-latest
```

## Workflow Linting and Local Testing

- `actionlint` — static analysis (broken refs, shell quoting, etc.).
- `act` — run workflows locally in Docker (close-enough simulation).

## Interview Questions

**Q: How does `secrets` work for forks?**
A: On `pull_request` from a fork, `secrets` are **not** available — prevents fork PRs from stealing them. `pull_request_target` runs in the base repo's context with secrets, but executes the base repo's workflow code — using it to run untrusted PR code is the famous escalation vulnerability.

**Q: Static keys vs OIDC for AWS access in GH Actions?**
A: OIDC: short-lived (1 hour), no rotation needed, tied to a repo/branch/env claim, auditable. Static keys: long-lived, must rotate, leak if logs/screenshot/echo leak them. OIDC every time.

**Q: How do you cancel old runs on a busy PR?**
A: `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }`. Saves runner minutes and CI minutes.

**Q: How do you share steps across workflows?**
A: Composite actions (encapsulate steps) or reusable workflows (`workflow_call`, encapsulate entire jobs). Composite when you want to embed steps inline; reusable when you want a separate job with its own runner/permissions.

**Q: How do you secure a workflow that needs to push to ghcr.io?**
A: Set `permissions: { contents: read, packages: write }` at the workflow or job level (least privilege). Use the auto-generated `${{ secrets.GITHUB_TOKEN }}` rather than a PAT.

**Q: A workflow hangs forever. What's the typical cause and fix?**
A: Usually waiting on input (interactive command), or `apt-get` prompting, or `git pull` asking for creds. Fix: always set `timeout-minutes` per job; use `DEBIAN_FRONTEND=noninteractive` for apt; provide creds non-interactively.

## Common Pitfalls

- Using `actions/checkout@v4` (tag) — fine for trust, but pin to SHA for prod.
- Interpolating untrusted input directly into `run:` — shell injection.
- Forgetting `permissions:` — `GITHUB_TOKEN` defaults to a broader scope than you need.
- `pull_request_target` to "fix" the no-secrets-in-fork-PR issue — almost always wrong; introduces RCE risk.
- Job duration creep — no `timeout-minutes`, runaway test eats your free minutes.
- Caching binary builds without invalidating on dep change — stale builds.
- Putting `${{ secrets.X }}` in PR titles or commit messages — shows up in logs even after redaction in older runs.

## Related

- [10-ci-cd-fundamentals.md](10-ci-cd-fundamentals.md)
- [12-jenkins-basics.md](12-jenkins-basics.md)
- [21-security-devsecops.md](21-security-devsecops.md)
- [03-git-and-vcs.md](03-git-and-vcs.md)
