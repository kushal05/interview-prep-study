# Cloud — AWS Basics

> **TL;DR:** EC2, S3, RDS, IAM, VPC are the must-knows. ECS/EKS for containers. Lambda for serverless. Build mental models of how they compose for "design a web app on AWS."

## The 80/20 List

| Service | Category | Use |
|---------|----------|-----|
| **EC2** | Compute | VMs |
| **ECS / Fargate** | Compute | Containers (no K8s overhead) |
| **EKS** | Compute | Managed Kubernetes |
| **Lambda** | Compute | Event-driven serverless |
| **S3** | Storage | Object store — backups, logs, static assets |
| **EBS** | Storage | Block storage for EC2 |
| **EFS** | Storage | Shared filesystem (NFS) |
| **RDS** | Database | Managed Postgres/MySQL/etc. |
| **DynamoDB** | Database | Managed key-value/doc, single-digit ms |
| **ElastiCache** | Database | Managed Redis/Memcached |
| **VPC** | Network | Virtual network |
| **ALB / NLB** | Network | Load balancers (L7/L4) |
| **Route 53** | Network | DNS + health checks |
| **CloudFront** | Network | CDN |
| **IAM** | Security | Users/roles/policies |
| **KMS** | Security | Key management |
| **Secrets Manager** | Security | Secret storage + rotation |
| **CloudWatch** | Ops | Metrics, logs, alarms |
| **SNS / SQS** | Messaging | Pub/sub / queue |
| **EventBridge** | Messaging | Event bus + scheduling |

## EC2 — VMs

```bash
# Launch
aws ec2 run-instances \
    --image-id ami-0abc \
    --instance-type t3.micro \
    --key-name my-key \
    --security-group-ids sg-0xyz \
    --subnet-id subnet-0def \
    --iam-instance-profile Name=webrole \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web1}]'
```

Instance families to know:
- `t3` / `t4g` — burstable, dev/staging, bursty workloads.
- `m6i` / `m7g` — general purpose, balanced.
- `c7g` — compute-optimized (web servers, batch).
- `r7g` — memory-optimized (in-memory DBs, caches).
- `i4i` — storage-optimized (high IOPS NVMe).
- `g5` — GPU.

**Graviton (ARM, `g` suffix):** ~20% cheaper than x86 equivalents. If your workload runs on ARM, use it.

### IAM Instance Profiles vs Access Keys

NEVER put `AWS_ACCESS_KEY_ID` on an EC2 instance. Attach an IAM role:

```bash
aws iam create-role --role-name web --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name web --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam create-instance-profile --instance-profile-name web
aws iam add-role-to-instance-profile --instance-profile-name web --role-name web
# Then attach to instance
```

EC2 calls IMDSv2 to get short-lived creds. Force IMDSv2 (`HttpTokens=required`) to prevent SSRF-based credential theft.

## S3 — Object Storage

```bash
aws s3 mb s3://my-bucket
aws s3 cp file.txt s3://my-bucket/
aws s3 sync ./dist s3://my-bucket/ --delete
aws s3 ls s3://my-bucket/ --recursive --human-readable --summarize
```

### Key Concepts

- **11 nines (99.999999999%) of durability**, multi-AZ by default.
- **Strong read-after-write consistency** (since 2020).
- **Storage classes:** Standard, Standard-IA (infrequent access), Intelligent-Tiering, Glacier (archival), Glacier Deep Archive. Move stale data via Lifecycle rules.
- **Versioning** — keeps all versions, soft-deletes (essential for ransomware protection + accidental deletes).
- **Object Lock** — WORM, compliance.
- **Encryption:** SSE-S3 (AES-256), SSE-KMS (KMS keys, audit log), SSE-C (customer keys).
- **Block Public Access** — turn it on at the account level. Default since 2023.

### Lifecycle Rule

```json
{
  "Rules": [{
    "Id": "archive-old-logs",
    "Status": "Enabled",
    "Filter": { "Prefix": "logs/" },
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER" }
    ],
    "Expiration": { "Days": 365 }
  }]
}
```

### Public Read of a Static Site — Use CloudFront

Don't make S3 public. Put CloudFront in front (with OAC — Origin Access Control), restrict bucket policy to only the CloudFront distribution.

## RDS — Managed Relational DB

Multi-AZ for HA: synchronous standby in another AZ, automatic failover.

```bash
aws rds create-db-instance \
    --db-instance-identifier prod-pg \
    --engine postgres --engine-version 16.3 \
    --db-instance-class db.r7g.large \
    --allocated-storage 100 --storage-type gp3 \
    --master-username admin --master-user-password "$(openssl rand -base64 32)" \
    --multi-az \
    --backup-retention-period 14 \
    --storage-encrypted \
    --vpc-security-group-ids sg-0xyz \
    --db-subnet-group-name private \
    --deletion-protection
```

**Aurora** = AWS-native MySQL/Postgres-compatible. Storage decoupled from compute (auto-grow), 6-way replication across 3 AZs, faster failover, **Serverless v2** scales compute by ACU. Costs more per hour but often wins on TCO and ops.

**Read replicas** — async read scaling. Up to 15 for RDS, 15 for Aurora. Watch replication lag.

## VPC — The Network Mental Model

```
VPC (10.0.0.0/16)
├── AZ a
│   ├── public subnet  10.0.0.0/24    -> IGW (Internet Gateway)
│   └── private subnet 10.0.10.0/24   -> NAT GW (outbound only)
├── AZ b
│   ├── public subnet  10.0.1.0/24
│   └── private subnet 10.0.11.0/24
└── AZ c
    ├── public subnet  10.0.2.0/24
    └── private subnet 10.0.12.0/24

Security Groups (stateful firewall, per-ENI)
NACLs (stateless, per-subnet — rarely tune in practice)
Route tables (per subnet: 0.0.0.0/0 -> IGW for public, -> NAT for private)
```

- **Public subnet:** route to IGW. Things with public IPs (ALBs, bastion).
- **Private subnet:** route to NAT GW for egress, no inbound from internet. Where your app + DB live.
- **NAT GW:** $$$ per AZ. Cost-conscious setups use one NAT GW (single AZ failure tolerated for outbound traffic), or VPC endpoints for AWS service calls.
- **VPC Endpoints:** route traffic to S3 / DDB / Secrets Manager etc. **without going to the internet** — security + cost.

### Security Groups

Stateful (return traffic implicitly allowed). Allow-only — no deny rules. Reference another SG as source:

```hcl
resource "aws_security_group" "app" {
  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]   # only ALB can reach app
  }
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## IAM — Least Privilege

- **Users** — for humans. Prefer SSO (AWS IAM Identity Center / Okta SAML).
- **Groups** — bundle policies for users.
- **Roles** — assumable identities for services and cross-account. Use these for EC2, Lambda, EKS, CI.
- **Policies** — JSON documents granting permissions.
- **Permission boundary** — max permissions a role can ever get (defense in depth).
- **Service Control Policies (SCPs)** — guardrails at the AWS Organization level.

### A Sane Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::myapp-uploads-*/*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/Env": "prod" }
      }
    }
  ]
}
```

Never `"Action": "*"` or `"Resource": "*"` together unless you really mean root.

### AssumeRole Pattern

```bash
aws sts assume-role \
    --role-arn arn:aws:iam::123:role/ops \
    --role-session-name kushal-debug
```

Use this for cross-account access and short-lived creds. CI uses OIDC → AssumeRole.

## ECS / Fargate vs EKS

| | ECS + Fargate | EKS |
|---|---|---|
| K8s? | No, AWS-proprietary | Yes |
| Ops burden | Lowest | Higher |
| Portability | Low | High |
| Ecosystem | AWS-only | Huge (CNCF) |
| Cost | Cheaper at small scale | Cluster fee + nodes |
| Best for | Single-cloud, simple containers | Multi-cloud, complex / many services |

**Fargate** = serverless compute for containers (no EC2 to manage). Works with both ECS and EKS.

## Lambda

```python
# handler.py
import json

def handler(event, context):
    body = json.loads(event.get("body") or "{}")
    return {
        "statusCode": 200,
        "body": json.dumps({"hello": body.get("name", "world")}),
    }
```

Triggers: API Gateway, S3 events, SQS, EventBridge, DynamoDB Streams, ALB, Kinesis, more.

Cold starts hurt for JVM/Node — keep code small, use [Lambda SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html) for Java, or provisioned concurrency.

## A Reference Architecture (interview whiteboard)

```
                                  Route 53 (DNS)
                                       |
                                  CloudFront (CDN)
                                       |
                                       v
                                       ALB (in public subnets, multi-AZ)
                                       |
                          +------------+------------+
                          v                         v
                ECS Fargate (private subnets, multi-AZ)
                          |                         |
                          v                         v
                   ElastiCache (Redis)     RDS Aurora (multi-AZ, encrypted)
                                                    |
                                                  Backups -> S3 (KMS)

Cross-cutting:
- IAM Roles per service (no static creds)
- Secrets Manager for DB password (rotation enabled)
- CloudWatch Logs + Metrics + Alarms
- VPC Flow Logs -> S3
- WAF on ALB
```

## Cost Hygiene

- Tag everything (`Env`, `Project`, `Owner`, `CostCenter`).
- Cost Explorer to attribute spend.
- Reserved Instances / Savings Plans for steady-state EC2/Fargate (up to 72% off).
- Spot instances for fault-tolerant batch / dev clusters (up to 90% off).
- S3 Lifecycle rules.
- Delete unattached EBS, old snapshots, idle NAT GWs.
- Set a billing alarm at $X.

## Interview Questions

**Q: Design a highly-available web app on AWS.**
A: Route53 → CloudFront → ALB across ≥2 AZs in a VPC. ECS Fargate or EKS in private subnets. Aurora Multi-AZ. ElastiCache. Secrets Manager. S3 for static. IAM roles per service. CloudWatch for monitoring. Set up auto-scaling on ECS/EKS based on CPU + queue depth.

**Q: How do EC2 instances get AWS credentials securely?**
A: Attach an IAM role via Instance Profile. IMDSv2 (force `HttpTokens=required`) serves short-lived creds the SDK uses automatically. No static keys on the box.

**Q: What's the difference between Security Groups and NACLs?**
A: SGs are stateful (return traffic implicit), apply to ENIs (instance level), allow-only. NACLs are stateless (must allow both directions), apply to subnets, allow + deny rules. SGs cover 95% of needs; NACLs for subnet-level coarse rules.

**Q: VPC endpoint — why?**
A: Lets resources in a private subnet reach AWS services (S3, DDB, Secrets Manager) without going through a NAT GW or the internet. Faster, cheaper, more secure. Gateway endpoints for S3/DDB (free), Interface endpoints for everything else (per-hour cost).

**Q: When ECS vs EKS?**
A: ECS for AWS-only shops that want minimal ops — works great for monolith + few services. EKS when you have many teams / many services, multi-cloud aspirations, or need the broader K8s ecosystem (Helm, ArgoCD, mesh, operators). EKS has a $73/month per-cluster control plane fee — irrelevant at scale.

**Q: How do you handle secrets?**
A: Secrets Manager (rotates, audit log via CloudTrail). Or SSM Parameter Store (cheaper, no auto-rotate). Reference from ECS/EKS/Lambda via IAM. Never bake into AMIs/images. Encrypt with customer-managed KMS for high-value secrets.

**Q: How do you protect S3 from accidental deletion / ransomware?**
A: Versioning + MFA Delete + Object Lock for critical data. Replicate to another bucket / account / region. Block Public Access on by default. Bucket policies + IAM least-privilege. CloudTrail for audit. Backup with point-in-time recovery for stateful services.

## Common Pitfalls

- One NAT GW + one AZ for all private subnets — AZ outage = total egress outage.
- Long-lived IAM access keys in CI / on laptops — rotated rarely, leaked often.
- Public S3 buckets that "just need a quick test" — turn on Block Public Access account-wide.
- Forgetting `deletion_protection` on RDS / Aurora — `terraform destroy` wipes prod.
- Not enabling CloudTrail / VPC Flow Logs — no forensics after a breach.
- Over-provisioned EC2 — `t3.2xlarge` running at 5% CPU. Rightsize quarterly.

## Related

- [16-cloud-gcp-basics.md](16-cloud-gcp-basics.md)
- [17-cloud-azure-basics.md](17-cloud-azure-basics.md)
- [13-iac-terraform.md](13-iac-terraform.md)
- [22-networking-fundamentals.md](22-networking-fundamentals.md)
- [21-security-devsecops.md](21-security-devsecops.md)
- [../system-design/](../system-design/)
