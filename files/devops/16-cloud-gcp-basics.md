# Cloud — GCP Basics

> **TL;DR:** GCE for VMs, GCS for objects, Cloud SQL for managed DBs, GKE for K8s (best-in-class), Cloud Functions / Cloud Run for serverless. IAM differs from AWS — roles bind to **principals** at the **resource** level.

## The 80/20 List (vs AWS Equivalents)

| GCP | AWS | Category |
|-----|-----|----------|
| Compute Engine (GCE) | EC2 | VMs |
| Cloud Run | Fargate / App Runner | Serverless containers |
| GKE | EKS | Managed K8s |
| Cloud Functions | Lambda | Event-driven serverless |
| Cloud Storage (GCS) | S3 | Object storage |
| Persistent Disk | EBS | Block storage |
| Filestore | EFS | NFS |
| Cloud SQL | RDS | Managed Postgres/MySQL |
| Spanner | (none — closest: Aurora Global) | Global RDBMS |
| Firestore / Datastore | DynamoDB | NoSQL |
| BigQuery | Redshift / Athena | Analytics warehouse |
| Memorystore | ElastiCache | Redis/Memcached |
| VPC | VPC | Network |
| Cloud Load Balancing | ALB/NLB | LB |
| Cloud DNS | Route 53 | DNS |
| Cloud CDN | CloudFront | CDN |
| IAM | IAM | Auth |
| Secret Manager | Secrets Manager | Secrets |
| Cloud Logging / Monitoring | CloudWatch | Ops |
| Pub/Sub | SNS + SQS | Messaging |

## Project / Folder / Org Hierarchy

```
Organization
├── Folder: engineering
│   ├── Folder: prod
│   │   ├── Project: myapp-prod-web
│   │   └── Project: myapp-prod-data
│   └── Folder: staging
│       └── Project: myapp-staging
└── Folder: shared
    └── Project: shared-networking-host
```

**Project** = the unit of billing, IAM, and resource scoping. Far more granular than AWS Accounts.

## Compute Engine

```bash
gcloud compute instances create web1 \
    --machine-type=e2-medium \
    --zone=us-central1-a \
    --image-family=debian-12 --image-project=debian-cloud \
    --service-account=web@my-proj.iam.gserviceaccount.com \
    --scopes=cloud-platform \
    --tags=http-server,https-server
```

Instance families:
- `e2` — cost-optimized.
- `n2` / `n2d` — general purpose.
- `c3` — compute-optimized.
- `m3` — memory-optimized.
- `a3` — GPU.
- `t2a` — ARM (Ampere).

**Preemptible / Spot VMs** — up to 91% cheaper, can be reclaimed with 30s notice. Great for batch.

### Service Accounts — the GCP "IAM Role" Equivalent

```bash
gcloud iam service-accounts create web --display-name="Web app SA"
gcloud projects add-iam-policy-binding my-proj \
    --member="serviceAccount:web@my-proj.iam.gserviceaccount.com" \
    --role="roles/storage.objectViewer"
```

Attach the SA to a VM / Cloud Run / GKE Pod (via Workload Identity). VM calls the metadata server for short-lived tokens — equivalent to EC2 IMDSv2.

## Cloud Storage (GCS)

```bash
gcloud storage cp file.txt gs://my-bucket/
gcloud storage rsync ./dist gs://my-bucket/ --delete-unmatched-destination-objects
```

- Strong global consistency.
- Storage classes: Standard, Nearline (30d), Coldline (90d), Archive (365d).
- Object Lifecycle Management — same idea as S3 lifecycle rules.
- Object Versioning (enable explicitly).
- Uniform Bucket-Level Access — disables per-object ACLs (recommended; cleaner IAM).
- Public access prevention — enforce at project / org level.

## Cloud Run — Serverless Containers Done Right

Deploy a container, get HTTPS + autoscaling + scale-to-zero.

```bash
gcloud run deploy api \
    --image=us-central1-docker.pkg.dev/my-proj/repo/api:1.4.0 \
    --region=us-central1 \
    --platform=managed \
    --allow-unauthenticated \
    --memory=512Mi --cpu=1 \
    --min-instances=0 --max-instances=100 \
    --set-env-vars="NODE_ENV=production" \
    --set-secrets="DATABASE_URL=db-url:latest" \
    --service-account=api-runtime@my-proj.iam.gserviceaccount.com
```

- Scale to zero between requests — pay only for request time + small overhead.
- Up to 60-minute timeouts (vs Lambda's 15).
- Bring any container — no special runtime adapter.
- Cloud Run Jobs (batch) for run-to-completion tasks.

**This is the easiest "ship a container to prod" experience across all clouds.**

## GKE — Managed Kubernetes

Two modes:
- **Standard** — you manage node pools.
- **Autopilot** — Google manages nodes; you only pay for Pod resources. Lower ops burden, slight markup.

```bash
gcloud container clusters create-auto myapp \
    --region=us-central1
gcloud container clusters get-credentials myapp --region=us-central1
kubectl get nodes
```

GKE features that beat raw K8s:
- Workload Identity (Pod ↔ Service Account binding without secrets).
- Multi-cluster Ingress / Multi-cluster Services.
- Cluster autoscaler + Node auto-provisioning.
- GKE Backup for stateful workloads.

### Workload Identity (the cleanest cloud auth model)

```bash
# Bind a K8s SA to a GCP SA
gcloud iam service-accounts add-iam-policy-binding \
    api-sa@my-proj.iam.gserviceaccount.com \
    --member="serviceAccount:my-proj.svc.id.goog[default/api]" \
    --role="roles/iam.workloadIdentityUser"

kubectl annotate sa api \
    iam.gke.io/gcp-service-account=api-sa@my-proj.iam.gserviceaccount.com
```

Pods using the K8s SA `api` automatically get tokens for the GCP SA. No mounted credentials, no secrets in env.

## Cloud SQL / AlloyDB / Spanner

- **Cloud SQL** — managed Postgres/MySQL/SQL Server. Like RDS.
- **AlloyDB** — Google's Postgres-compatible, with vector + ANN support, columnar engine for HTAP.
- **Spanner** — globally distributed RDBMS with external consistency. Strong-consistent multi-region. The "Google magic" service.

## VPC — Global by Default

Unlike AWS (VPC is regional), **GCP VPCs are global**. Subnets are per-region.

```
VPC: myapp-vpc            (global)
├── subnet: us-central1   (10.0.0.0/20)
├── subnet: europe-west1  (10.1.0.0/20)
└── subnet: asia-east1    (10.2.0.0/20)
```

Pods/VMs in different regions can talk over the internal backbone without VPC peering. Big advantage over AWS.

### Firewall Rules

Project-scoped, target by network tag or service account:

```bash
gcloud compute firewall-rules create allow-http \
    --network=default \
    --direction=INGRESS \
    --action=ALLOW \
    --rules=tcp:80 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=http-server
```

Default-deny inbound (good!), default-allow outbound (lock down for sensitive workloads).

## IAM — Different Model from AWS

- **Roles** are collections of permissions: `roles/storage.objectViewer`, `roles/editor`, etc.
- Bind a **principal** (user, group, service account) to a **role** at a **resource** scope (org, folder, project, resource).
- No "policy attached to a principal" — bindings live on the resource.

```bash
gcloud projects add-iam-policy-binding my-proj \
    --member=user:kushal@example.com \
    --role=roles/editor

gcloud storage buckets add-iam-policy-binding gs://my-bucket \
    --member=serviceAccount:web@my-proj.iam.gserviceaccount.com \
    --role=roles/storage.objectViewer
```

**Predefined roles** are usually too broad. Use **custom roles** for production:

```yaml
title: My App Runtime
description: Minimum perms for the api service
stage: GA
includedPermissions:
  - storage.objects.get
  - storage.objects.list
  - secretmanager.versions.access
```

## Operations Suite (formerly Stackdriver)

- **Cloud Logging** — all logs from GCE/GKE/Cloud Run land here.
- **Cloud Monitoring** — metrics, dashboards, alerting policies.
- **Cloud Trace** — distributed tracing.
- **Cloud Profiler** — sampled production profiling.
- **Error Reporting** — automatic grouping of exceptions.

Logs router → BigQuery / Pub/Sub / GCS for analysis or long-term storage.

## Reference Architecture (whiteboard)

```
                    Cloud DNS
                         |
                  Global HTTPS LB
                  (Cloud Armor WAF + Cloud CDN)
                         |
                         v
                  GKE Autopilot or Cloud Run
                         |
            +------------+-----------+
            v                        v
       Memorystore (Redis)       Cloud SQL / AlloyDB
                                     |
                                Backups -> GCS

Cross-cutting:
- Workload Identity (no secrets in Pods)
- Secret Manager for app secrets
- Cloud Logging + Monitoring + Alerts
- VPC Service Controls around data services
- Audit Logs to GCS / BigQuery
```

## Interview Questions

**Q: How does GCP IAM differ from AWS IAM?**
A: GCP binds principals to roles **at the resource scope** (org/folder/project/resource) — no policy "attached to" a principal. Predefined roles are broad; custom roles for least-privilege. Service accounts are the runtime identity (equivalent to AWS IAM Roles for services).

**Q: Cloud Run vs GKE vs GCE — when which?**
A: Cloud Run for stateless HTTP services / Jobs — easiest, scale-to-zero, no infra. GKE when you need full K8s (operators, mesh, complex networking, many services). GCE only when you need a specific kernel/OS or licensed binaries.

**Q: What is Workload Identity?**
A: A binding between a Kubernetes ServiceAccount and a GCP service account, so Pods authenticate to GCP APIs **without any mounted credentials**. GKE's metadata server hands out short-lived tokens. The cleanest cloud auth story.

**Q: Why are GCP VPCs global?**
A: Subnets are regional but the VPC spans all regions, with Google's backbone providing free inter-region routing within the VPC. Simplifies multi-region apps vs AWS where you peer regional VPCs.

**Q: What's Spanner and when would you reach for it?**
A: Globally distributed SQL database with external consistency (synchronous multi-region commits via TrueTime). Use for financial / multi-region transactional systems needing strong consistency across continents. Cost is significant — overkill for single-region apps.

**Q: How do you secure data-at-rest in GCS?**
A: Default Google-managed encryption (transparent). For higher control: CMEK (customer-managed keys via Cloud KMS) — bring-your-own per-bucket or per-object. CSEK for customer-supplied keys (rare). Uniform bucket-level access + IAM + Object Versioning for ransomware protection.

## Common Pitfalls

- Granting `roles/editor` (overly broad) instead of crafting custom roles.
- Forgetting Workload Identity — mounting service account JSON keys into Pods (the GCP equivalent of leaking IAM keys).
- Single-project blast radius — put prod / staging in **separate projects** for isolation.
- Public GCS buckets — turn on "Public access prevention" at the org level.
- Letting Cloud SQL run in the default VPC with public IP — use private IP only.
- Quota surprises — GCP quotas are project-scoped and many require pre-approval to raise.

## Related

- [15-cloud-aws-basics.md](15-cloud-aws-basics.md)
- [17-cloud-azure-basics.md](17-cloud-azure-basics.md)
- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [13-iac-terraform.md](13-iac-terraform.md)
