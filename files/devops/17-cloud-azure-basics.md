# Cloud — Azure Basics

> **TL;DR:** Azure's strength is enterprise / Microsoft ecosystem integration. Know VMs, Blob Storage, Azure SQL, App Service, AKS, Functions, and Azure AD / Entra ID for identity.

## The 80/20 List (vs AWS Equivalents)

| Azure | AWS | Category |
|-------|-----|----------|
| Virtual Machine | EC2 | VMs |
| Virtual Machine Scale Sets (VMSS) | Auto Scaling Group | VM autoscaling |
| App Service | Elastic Beanstalk | Managed PaaS |
| Azure Container Apps | Cloud Run / Fargate | Serverless containers |
| AKS | EKS | Managed K8s |
| Azure Functions | Lambda | Event-driven serverless |
| Blob Storage | S3 | Object storage |
| Managed Disk | EBS | Block storage |
| Azure Files | EFS | SMB/NFS file share |
| Azure SQL | RDS (SQL Server) | Managed DB (TSQL) |
| Cosmos DB | DynamoDB + more | Multi-model NoSQL global |
| Azure Database for Postgres/MySQL | RDS Postgres/MySQL | Managed Postgres/MySQL |
| Azure Cache for Redis | ElastiCache | Redis |
| Virtual Network (VNet) | VPC | Network |
| Application Gateway / Front Door | ALB / CloudFront | L7 LB + WAF / global edge |
| Load Balancer | NLB | L4 LB |
| Azure DNS | Route 53 | DNS |
| Microsoft Entra ID (Azure AD) | IAM Identity Center + IAM users | Identity |
| Managed Identities | IAM Roles | Service identity |
| Key Vault | Secrets Manager + KMS | Secrets + keys |
| Azure Monitor / Log Analytics | CloudWatch | Ops |
| Service Bus | SNS + SQS | Messaging |
| Event Grid | EventBridge | Event routing |

## Subscription / Resource Group / Resource

```
Tenant (Entra ID directory)
├── Management Group: engineering
│   ├── Subscription: myapp-prod
│   │   ├── Resource Group: myapp-prod-web
│   │   │   ├── VM, Disk, NIC, ...
│   │   └── Resource Group: myapp-prod-data
│   └── Subscription: myapp-staging
```

- **Subscription** = billing + RBAC boundary.
- **Resource Group** = lifecycle unit (deploy/delete together). Pick wisely — moving resources between RGs is allowed but painful.

## VMs

```bash
az group create -n rg-web -l eastus

az vm create -n web1 -g rg-web \
    --image Ubuntu2204 \
    --size Standard_B2s \
    --admin-username azureuser \
    --ssh-key-values ~/.ssh/id_ed25519.pub \
    --assign-identity \
    --vnet-name vnet-prod --subnet snet-app
```

VM sizes:
- `B-series` — burstable, cheap, dev.
- `D-series` — general purpose.
- `F-series` — compute-optimized.
- `E-series` — memory-optimized.
- `L-series` — storage-optimized.
- `N-series` — GPU.

**Spot VMs** — like EC2 Spot, big discount with eviction risk.

## Managed Identity — the Equivalent of IAM Roles

```bash
az vm identity assign -g rg-web -n web1                    # system-assigned
# or:
az identity create -g rg-id -n mi-web                      # user-assigned
az vm identity assign -g rg-web -n web1 --identities mi-web
```

The VM (or App Service, Container App, AKS Pod via Workload Identity) calls Instance Metadata Service for short-lived tokens. **Never** store Entra ID credentials on a VM.

## Blob Storage

```bash
az storage account create -n myappstor1234 -g rg-data \
    -l eastus --sku Standard_LRS --kind StorageV2 \
    --min-tls-version TLS1_2 --allow-blob-public-access false

az storage container create -n uploads --account-name myappstor1234 --auth-mode login
az storage blob upload -f file.txt -c uploads --account-name myappstor1234 \
    --auth-mode login -n file.txt
```

Tiers: **Hot**, **Cool** (30d min), **Cold** (90d min), **Archive** (180d min). Lifecycle Management auto-tiers and expires.

Redundancy options:
- **LRS** — 3 copies in one DC.
- **ZRS** — 3 copies across AZs.
- **GRS** — LRS + async copy to paired region.
- **GZRS** — ZRS + async to paired region (best).
- **RA-GRS / RA-GZRS** — readable secondary.

## Azure SQL / Azure DB for Postgres

```bash
az sql server create -n myapp-sql -g rg-data \
    -l eastus -u sqladmin -p "$(openssl rand -base64 32)"

az sql db create -n appdb -g rg-data -s myapp-sql \
    --service-objective S1 --backup-storage-redundancy Zone

az sql server firewall-rule create -g rg-data -s myapp-sql \
    -n AllowAzureServices --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
```

**Three Azure SQL flavors:**
- **Single Database** — one DB, isolated.
- **Elastic Pool** — multiple DBs sharing pooled DTUs/vCores (cost-efficient for many small DBs).
- **Managed Instance** — near-100% SQL Server compatibility, in your VNet (great for lift-and-shift).

For Postgres: **Azure Database for PostgreSQL — Flexible Server** is the modern offering (VNet integration, high availability, point-in-time restore).

## App Service — Managed Web Hosting

```bash
az appservice plan create -n plan-web -g rg-web --sku P1v3 --is-linux
az webapp create -g rg-web -p plan-web -n myapp-prod --runtime "NODE:20-lts"
az webapp config set -g rg-web -n myapp-prod --always-on true

# Deploy from container
az webapp config container set -g rg-web -n myapp-prod \
    --docker-custom-image-name myreg.azurecr.io/api:1.4.0
```

Features:
- **Slots** — deployment slots ("staging", "prod") with swap-and-warm-up — Azure's built-in blue-green.
- **Easy Auth** — bolt-on OIDC/SAML for AAD, Google, Facebook, GitHub.
- Auto-scaling rules on metric/schedule.

```bash
az webapp deployment slot create -g rg-web -n myapp-prod --slot staging
# deploy to staging, validate
az webapp deployment slot swap -g rg-web -n myapp-prod --slot staging --target-slot production
```

## AKS — Managed Kubernetes

```bash
az aks create -g rg-k8s -n myapp \
    --node-count 3 --enable-cluster-autoscaler \
    --min-count 3 --max-count 10 \
    --enable-managed-identity \
    --enable-workload-identity --enable-oidc-issuer \
    --network-plugin azure \
    --enable-azure-rbac
az aks get-credentials -g rg-k8s -n myapp
```

AKS specifics:
- Control plane is free (you pay for nodes only).
- **Azure CNI** vs **Kubenet** — Azure CNI gives each Pod a VNet IP (subnet sizing matters!). Kubenet uses NAT.
- **Workload Identity** binds K8s SAs to Entra ID — same idea as GKE.
- AKS-managed Azure AD integration → use `kubectl` with `az login`.

## Functions

```bash
func init MyFunc --typescript
func new --name HttpHello --template "HTTP trigger"
func azure functionapp publish my-func-app
```

Triggers: HTTP, Timer, Blob, Queue, Service Bus, Event Hub, Cosmos DB change feed, more. Consumption / Premium / App Service plans (cold start vs always-warm).

## VNet — Azure's Networking

```
VNet: vnet-prod (10.0.0.0/16)
├── snet-app    10.0.1.0/24
├── snet-data   10.0.2.0/24
└── snet-bastion 10.0.250.0/27

NSGs (Network Security Groups) — stateful firewall at subnet or NIC level
Route tables — UDR (User-Defined Routes) for forced tunneling / NVAs
Private Endpoints — private IPs into PaaS (Storage, SQL, Key Vault)
Service Endpoints — VNet-level allowlist on PaaS (older mechanism)
```

**Private Endpoints** are the modern story: Storage / SQL / Key Vault gets a NIC in your VNet, traffic stays on Microsoft backbone, no public exposure.

## Entra ID (Azure AD) — Identity

- **Users / Groups** — humans, federated from on-prem AD via AD Connect.
- **Service Principals** — app identities (similar to IAM roles for apps registered against AAD).
- **Managed Identities** — system or user-assigned, attached to Azure resources. No secrets to rotate.
- **Conditional Access** — policy engine: require MFA, location-based, device compliance.
- **PIM (Privileged Identity Management)** — just-in-time role activation with approval.

```bash
# RBAC assignment
az role assignment create \
    --assignee kushal@example.com \
    --role "Storage Blob Data Reader" \
    --scope /subscriptions/.../resourceGroups/rg-data/providers/Microsoft.Storage/storageAccounts/myappstor1234
```

Roles are hierarchical: assign at subscription → inherits to all RGs and resources.

## Key Vault — Secrets + Keys + Certs

```bash
az keyvault create -n myapp-kv -g rg-sec -l eastus --enable-rbac-authorization

az keyvault secret set --vault-name myapp-kv -n db-password --value "$(openssl rand -base64 32)"

# App accesses by Managed Identity
az role assignment create \
    --assignee <managed-identity-objectid> \
    --role "Key Vault Secrets User" \
    --scope $(az keyvault show -n myapp-kv -g rg-sec --query id -o tsv)
```

App Service / Functions / Container Apps / AKS Pods all support **Key Vault references** in app settings:

```
@Microsoft.KeyVault(SecretUri=https://myapp-kv.vault.azure.net/secrets/db-password)
```

## Reference Architecture (whiteboard)

```
                 Azure Front Door (global edge + WAF)
                          |
                          v
                Application Gateway (regional WAF + L7 LB)
                          |
                          v
                 App Service / AKS / Container Apps
                          |
            +-------------+-------------+
            v                           v
      Azure Cache (Redis)         Azure DB for Postgres (Flexible Server, HA)
                                          |
                                  Backups -> Blob

Cross-cutting:
- Managed Identities (no creds in code)
- Key Vault for secrets (referenced by URL)
- Azure Monitor + Log Analytics + Application Insights
- Private Endpoints to PaaS (no public IPs on DBs / Storage)
- Defender for Cloud (security posture)
- Conditional Access on Entra ID
```

## Interview Questions

**Q: How does Managed Identity compare to AWS IAM Roles?**
A: Same idea — an identity attached to a resource (VM, App Service, AKS Pod) so it can call Azure APIs without stored credentials. System-assigned = lifecycle tied to the resource. User-assigned = standalone identity, can be assigned to many resources (preferred for stable, shared identities).

**Q: What's the difference between App Service and Container Apps?**
A: App Service = traditional PaaS (web apps, APIs, scheduled WebJobs), supports many runtimes natively, slots for blue-green, scale up/out by metrics. Container Apps = serverless containers built on KEDA + Dapr, scale-to-zero, micro-VMs. App Service for traditional web apps; Container Apps for event-driven / scale-to-zero microservices without managing K8s.

**Q: When AKS vs App Service vs Container Apps?**
A: AKS when you need full K8s flexibility (operators, mesh, complex deployments) or multi-cloud aspirations. App Service for traditional web hosting with minimal ops. Container Apps for serverless containers without K8s overhead.

**Q: How do Private Endpoints differ from Service Endpoints?**
A: Service Endpoints = VNet-level allowlist on a PaaS service (still uses public endpoint). Private Endpoints = a NIC with a private IP in your VNet representing the PaaS service; traffic stays internal. Private Endpoints are the modern, more secure choice.

**Q: What does PIM do?**
A: Privileged Identity Management — instead of permanent admin access, users activate elevated roles on-demand for a time-box (e.g., 4 hours), with approval workflow and audit log. Reduces standing privilege.

**Q: Cosmos DB — when?**
A: Need globally distributed, multi-master, tunable consistency, low single-digit ms reads anywhere. Pick the API that fits: SQL (document), MongoDB, Cassandra, Gremlin (graph), Table. Cost is high; Azure SQL or Postgres often fits better for typical workloads.

## Common Pitfalls

- Subscription = blast radius — prod and dev in the same sub means a misconfig can hit prod resources.
- App Service slot swap with sticky settings → settings flip when you didn't expect them to.
- Storage account names — must be globally unique, 3-24 chars, alphanumeric only. Plan a naming convention.
- AKS Azure CNI subnet exhaustion — every Pod takes a VNet IP. Size subnets generously.
- Forgetting `--enable-soft-delete` on Key Vault — deleted secrets are gone for good.
- Putting sensitive configs in App Service environment variables without using Key Vault references.

## Related

- [15-cloud-aws-basics.md](15-cloud-aws-basics.md)
- [16-cloud-gcp-basics.md](16-cloud-gcp-basics.md)
- [06-kubernetes-fundamentals.md](06-kubernetes-fundamentals.md)
- [21-security-devsecops.md](21-security-devsecops.md)
