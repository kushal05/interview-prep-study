# DevOps Interview Prep — Index

> **TL;DR:** A structured, per-topic study path for DevOps / Platform / SRE interviews. The original monolithic plan lives in [`plan.md`](plan.md) (kept for reference); the files below split it into focused, runnable chunks with new content.

## How to Use This Folder

1. Skim **all** TL;DRs first (about 30 minutes) to map your gaps.
2. Deep-dive in the order below. Each file is roughly 150–300 lines: read, run the snippets, then answer the Interview Questions out loud.
3. After each group (Linux/Docker/K8s/Cloud), do a 20-minute mock — pick one Q from each file.

## Study Order

| # | File | Group | Priority |
|---|------|-------|----------|
| 01 | [Linux Essentials](01-linux-essentials.md) | Foundations | Must |
| 02 | [Bash & Shell Scripting](02-bash-and-shell-scripting.md) | Foundations | Must |
| 03 | [Git & VCS](03-git-and-vcs.md) | Foundations | Must |
| 04 | [Docker Basics](04-docker-basics.md) | Containers | Must |
| 05 | [Docker Compose](05-docker-compose.md) | Containers | Should |
| 06 | [Kubernetes Fundamentals](06-kubernetes-fundamentals.md) | Orchestration | Must |
| 07 | [Kubernetes Networking](07-kubernetes-networking.md) | Orchestration | Must |
| 08 | [Kubernetes Storage](08-kubernetes-storage.md) | Orchestration | Should |
| 09 | [Helm & Kustomize](09-helm-and-kustomize.md) | Orchestration | Should |
| 10 | [CI/CD Fundamentals](10-ci-cd-fundamentals.md) | Delivery | Must |
| 11 | [GitHub Actions](11-github-actions.md) | Delivery | Must |
| 12 | [Jenkins Basics](12-jenkins-basics.md) | Delivery | Nice |
| 13 | [IaC — Terraform](13-iac-terraform.md) | Infra | Must |
| 14 | [IaC — Pulumi vs CloudFormation](14-iac-pulumi-vs-cloudformation.md) | Infra | Nice |
| 15 | [Cloud — AWS Basics](15-cloud-aws-basics.md) | Cloud | Must |
| 16 | [Cloud — GCP Basics](16-cloud-gcp-basics.md) | Cloud | Should |
| 17 | [Cloud — Azure Basics](17-cloud-azure-basics.md) | Cloud | Should |
| 18 | [Monitoring & Observability](18-monitoring-and-observability.md) | Ops | Must |
| 19 | [Logging Best Practices](19-logging-best-practices.md) | Ops | Should |
| 20 | [Incident Response](20-incident-response.md) | Ops | Must |
| 21 | [Security & DevSecOps](21-security-devsecops.md) | Security | Must |
| 22 | [Networking Fundamentals](22-networking-fundamentals.md) | Foundations | Must |
| 23 | [Load Balancers & Proxies](23-load-balancers-and-proxies.md) | Networking | Should |
| 24 | [Databases in Production](24-databases-in-prod.md) | Data | Must |
| 25 | [Service Mesh — Istio](25-service-mesh-istio.md) | Advanced | Nice |

## Cross-References

- [../serious-prep/](../serious-prep/) — broader interview prep, behavioral templates, project stories
- [../nodejs/23-deployment-and-docker.md](../nodejs/23-deployment-and-docker.md) — Node.js specific Docker patterns
- [../system-design/](../system-design/) — high-level architecture context that DevOps operates within
- [../postgres/](../postgres/) — operational depth on the most common interview database
- [../mongodb/](../mongodb/) — operational notes for NoSQL workloads

## Suggested 2-Week Sprint

- **Day 1–2:** 01 → 03 (Linux, Bash, Git)
- **Day 3–4:** 04 → 05 (Docker)
- **Day 5–7:** 06 → 09 (Kubernetes + packaging)
- **Day 8:** 10 → 12 (CI/CD)
- **Day 9–10:** 13 → 17 (IaC + Cloud)
- **Day 11–12:** 18 → 21 (Observability, Incident, Security)
- **Day 13:** 22 → 25 (Networking, LB, DB, Mesh)
- **Day 14:** Mock interviews; revisit weak Qs.

## How Questions Are Structured

Every file ends with **Interview Questions** (4–6 Q&A) and **Common Pitfalls** so you can self-quiz. If you can answer all Q&A across the must-read files without notes, you're ready for most L4/L5 DevOps loops.
