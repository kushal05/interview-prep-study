# DevOps Engineer - Interview Study Plan

A structured roadmap for preparing for DevOps and infrastructure-related interviews.

---

## 1. Linux & System Administration

- [ ] **Shell basics**: File operations, permissions (`chmod`, `chown`), piping, redirection
- [ ] **Process management**: `ps`, `top`, `htop`, `kill`, `systemctl`, background processes
- [ ] **Networking**: `curl`, `netstat`/`ss`, `dig`, `traceroute`, `iptables` basics
- [ ] **File systems**: Disk usage (`df`, `du`), mounting, `/etc/fstab`
- [ ] **SSH**: Key-based auth, tunneling, config file, `scp`/`rsync`
- [ ] **Scripting**: Bash scripting (loops, conditionals, functions, exit codes)

> **Interview Tip:** You will likely get a troubleshooting scenario: "A server is slow. How do you diagnose?" Start with `top` (CPU/memory), `df` (disk), `netstat` (connections), then check logs.

---

## 2. Version Control (Git)

- [ ] **Core commands**: clone, branch, merge, rebase, cherry-pick, stash
- [ ] **Branching strategies**: GitFlow, trunk-based development, feature branches
- [ ] **Merge vs Rebase**: Merge preserves history; rebase creates linear history
- [ ] **Conflict resolution**: Manual merge, using diff tools
- [ ] **Git hooks**: Pre-commit, pre-push for linting and tests

---

## 3. CI/CD Pipelines

- [ ] **Concepts**: Build, test, deploy automation; fast feedback loops
- [ ] **Tools**: GitHub Actions, GitLab CI, Jenkins, CircleCI
- [ ] **Pipeline design**: Stages (lint -> test -> build -> deploy), parallelism, caching
- [ ] **Deployment strategies**: Blue-green, canary, rolling updates, feature flags
- [ ] **Artifact management**: Docker registries, npm registries, S3 storage

> **Key Interview Point:** Be able to design a CI/CD pipeline from scratch. Know the difference between continuous delivery (manual deploy gate) and continuous deployment (fully automated).

```yaml
# Example GitHub Actions pipeline
name: CI/CD
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install
      - run: npm test
  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy to production"
```

---

## 4. Containers (Docker)

- [ ] **Core concepts**: Images, containers, layers, Dockerfile
- [ ] **Dockerfile best practices**: Multi-stage builds, small base images (alpine), `.dockerignore`
- [ ] **Networking**: Bridge, host, overlay networks; port mapping
- [ ] **Volumes**: Persist data beyond container lifecycle
- [ ] **Docker Compose**: Multi-container apps, service dependencies, environment variables
- [ ] **Security**: Non-root users, image scanning, minimal images

```dockerfile
# Multi-stage build example
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
USER node
CMD ["node", "server.js"]
```

> **Interview Tip:** Explain why multi-stage builds matter: smaller final image, no build tools in production, reduced attack surface.

---

## 5. Container Orchestration (Kubernetes)

- [ ] **Core objects**: Pod, Deployment, Service, ConfigMap, Secret, Ingress
- [ ] **Architecture**: Control plane (API server, etcd, scheduler) + worker nodes (kubelet, kube-proxy)
- [ ] **Scaling**: HorizontalPodAutoscaler (HPA), resource requests/limits
- [ ] **Networking**: Services (ClusterIP, NodePort, LoadBalancer), Ingress controllers
- [ ] **Storage**: PersistentVolume, PersistentVolumeClaim, StorageClass
- [ ] **Troubleshooting**: `kubectl describe pod`, `kubectl logs`, `kubectl exec`
- [ ] **Helm**: Package manager for Kubernetes; charts, values, releases

> **Key Interview Point:** Know the difference between a Deployment and a StatefulSet. Deployment = stateless apps (web servers). StatefulSet = stateful apps (databases) with stable network identity and ordered scaling.

---

## 6. Cloud Platforms (AWS / GCP / Azure)

- [ ] **Compute**: EC2/GCE/VMs, Lambda/Cloud Functions (serverless), ECS/EKS/GKE (containers)
- [ ] **Storage**: S3/GCS/Blob Storage, EBS, EFS
- [ ] **Networking**: VPC, subnets, security groups, load balancers (ALB/NLB), Route53/Cloud DNS
- [ ] **Databases**: RDS/Cloud SQL, DynamoDB/Firestore, ElastiCache/Memorystore
- [ ] **IAM**: Users, roles, policies, least privilege principle
- [ ] **Monitoring**: CloudWatch, Cloud Monitoring, SNS/alerting

> **Interview Tip:** You do not need to know every service. Focus on compute, storage, networking, and IAM for your primary cloud. Be able to design a basic architecture on a whiteboard.

---

## 7. Infrastructure as Code (IaC)

- [ ] **Terraform**: Providers, resources, state management, modules, plan/apply workflow
- [ ] **State management**: Remote state (S3 + DynamoDB lock), state locking, workspaces
- [ ] **Best practices**: Modular code, variable files, outputs, `terraform plan` before apply
- [ ] **Alternatives**: CloudFormation (AWS), Pulumi (code-based), Ansible (configuration management)

```hcl
# Terraform example
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags = { Name = "web-server" }
}
```

---

## 8. Monitoring, Logging & Observability

- [ ] **Three pillars**: Metrics (Prometheus), Logs (ELK/Loki), Traces (Jaeger/OpenTelemetry)
- [ ] **Prometheus + Grafana**: Metrics collection, PromQL queries, dashboards, alerting
- [ ] **Logging**: Structured JSON logs, centralized logging (ELK stack, Loki, CloudWatch)
- [ ] **Alerting**: Alert fatigue, severity levels, runbooks, on-call rotation (PagerDuty, OpsGenie)
- [ ] **SLIs/SLOs/SLAs**: Service Level Indicators (latency p99), Objectives (99.9% uptime), Agreements

> **Key Interview Point:** SLI = what you measure, SLO = your target, SLA = your contract. Example: SLI is p99 latency, SLO is under 200ms, SLA promises 99.9% uptime with penalty if breached.

---

## 9. Networking Fundamentals

- [ ] **OSI model**: Focus on layers 3 (Network/IP), 4 (Transport/TCP/UDP), 7 (Application/HTTP)
- [ ] **DNS**: Resolution process, record types (A, CNAME, MX, TXT), TTL
- [ ] **HTTP/HTTPS**: Methods, status codes, TLS handshake, certificates
- [ ] **Load balancing**: L4 (TCP) vs L7 (HTTP), algorithms (round-robin, least connections, IP hash)
- [ ] **Firewalls & Security Groups**: Inbound/outbound rules, principle of least privilege
- [ ] **CDN**: Edge caching, cache invalidation (CloudFront, Cloudflare)

---

## 10. Security

- [ ] **Secrets management**: HashiCorp Vault, AWS Secrets Manager, sealed secrets (K8s)
- [ ] **TLS/SSL**: Certificate management, Let's Encrypt, cert-manager in K8s
- [ ] **Image security**: Scan images (Trivy, Snyk), use minimal base images, pin versions
- [ ] **RBAC**: Role-based access control in K8s and cloud platforms
- [ ] **Network policies**: Restrict pod-to-pod traffic in Kubernetes
- [ ] **Compliance**: SOC2, GDPR awareness, audit logging

---

## 11. Site Reliability Engineering (SRE)

- [ ] **Error budgets**: If SLO is 99.9%, error budget is 0.1% (about 43 min/month)
- [ ] **Incident management**: Detection -> triage -> mitigation -> resolution -> postmortem
- [ ] **Postmortems**: Blameless, focus on systemic fixes, action items with owners
- [ ] **Chaos engineering**: Intentionally inject failures (Chaos Monkey, Litmus)
- [ ] **Capacity planning**: Forecast growth, load testing, right-sizing resources
- [ ] **Toil reduction**: Automate repetitive manual work

---

## 12. Scripting & Automation

- [ ] **Bash**: Variables, loops, conditionals, functions, exit codes, `set -euo pipefail`
- [ ] **Python**: Common for automation scripts, boto3 (AWS SDK), API interactions
- [ ] **Configuration management**: Ansible (agentless, YAML playbooks), Chef, Puppet
- [ ] **Cron jobs**: Scheduling, syntax (`* * * * *` = min hour day month weekday)

---

## Quick Reference Mindmap

```
DevOps Interview
|-- Linux & Sysadmin (shell, processes, networking, SSH)
|-- Git (branching, merge vs rebase, hooks)
|-- CI/CD (pipelines, deployment strategies, artifact mgmt)
|-- Docker (Dockerfile, compose, networking, security)
|-- Kubernetes (pods, deployments, services, scaling, Helm)
|-- Cloud (compute, storage, networking, IAM, monitoring)
|-- IaC (Terraform, state management, modules)
|-- Observability (metrics, logs, traces, alerting, SLOs)
|-- Networking (DNS, HTTP, load balancing, CDN)
|-- Security (secrets, TLS, RBAC, image scanning)
|-- SRE (error budgets, incidents, postmortems, chaos)
|-- Automation (Bash, Python, Ansible, cron)
```
