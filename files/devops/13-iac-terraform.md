# Infrastructure as Code — Terraform

> **TL;DR:** Terraform declares cloud resources in HCL; the **state file** is the source of truth for "what exists." Plan → apply → review diff. Remote state with locking is mandatory for teams. Modules are how you scale.

## Mental Model

- You write **HCL** describing desired resources.
- Terraform reads **state** (last-known reality) and queries the **provider API** (current reality).
- `terraform plan` shows the diff between desired and reality.
- `terraform apply` makes API calls to reconcile.

The state file maps your HCL `aws_instance.web` to the actual `i-0abc...` ID.

## Project Layout

```
infra/
├── main.tf              # primary resources
├── variables.tf         # inputs
├── outputs.tf           # outputs
├── providers.tf         # provider config
├── backend.tf           # state backend
├── versions.tf          # required versions
├── terraform.tfvars     # value defaults (often gitignored if env-specific)
└── modules/
    └── vpc/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

## Providers

```hcl
# versions.tf
terraform {
  required_version = "~> 1.7"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}

# providers.tf
provider "aws" {
  region = var.region
  default_tags {
    tags = {
      Project   = "myapp"
      ManagedBy = "terraform"
      Env       = var.env
    }
  }
}
```

`~> 5.40` = any 5.40.x, **not** 5.41. Pin both terraform and provider — surprise upgrades will bite.

## Resources

```hcl
# main.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "${var.env}-vpc" }
}

resource "aws_subnet" "private" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = var.azs[count.index]
  tags = { Name = "${var.env}-private-${var.azs[count.index]}" }
}

resource "aws_instance" "web" {
  for_each      = toset(var.instance_names)
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.private[0].id
  tags = { Name = each.key }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
```

### `count` vs `for_each`

- `count = 3` → indexed list `aws_subnet.private[0..2]`. **Removing an item shifts indices → destroys & recreates many resources.**
- `for_each = toset([...])` or `for_each = { a = ..., b = ... }` → addressed by key, stable across changes. Prefer `for_each`.

## Variables and Outputs

```hcl
# variables.tf
variable "env" {
  type        = string
  description = "Environment (dev, staging, prod)"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.env)
    error_message = "env must be dev, staging, or prod"
  }
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "instance_names" {
  type    = list(string)
  default = ["web-a", "web-b"]
}

# outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "instance_ips" {
  value = { for k, v in aws_instance.web : k => v.private_ip }
}
```

Pass values:

```bash
terraform apply -var="env=prod" -var-file="prod.tfvars"
# Or set env: TF_VAR_env=prod terraform apply
```

## State — the Single Most Important Concept

`terraform.tfstate` is a JSON file mapping HCL addresses to real resource IDs. It is your record of truth.

```bash
terraform state list                       # all tracked resources
terraform state show aws_instance.web      # details
terraform state mv aws_instance.web aws_instance.api   # rename without recreate
terraform state rm aws_instance.web        # forget (does not delete cloud resource)
terraform import aws_instance.web i-0abc   # adopt an existing resource
```

### Local State is Wrong for Teams

Two engineers running `apply` simultaneously corrupt state. Solution: **remote backend with locking**.

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-tfstate"
    key            = "infra/prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tfstate-locks"           # for state locking
    encrypt        = true
  }
}
```

`apply` acquires a lock row in DynamoDB; concurrent runs fail fast with "state locked by …".

### **Don't Lose the Lock!**

If a run dies (kill -9, lost VPN), the lock can be orphaned. `terraform force-unlock <LOCK_ID>` — but only after you're sure no one's actually running.

### **Don't Edit State by Hand!**

Edit `tfstate` in an editor → corruption → tears. Use `state mv` / `state rm` / `import`.

### **Encrypt and Back Up State**

State files contain **secrets in plaintext** (DB passwords, tokens, keys). Encrypt the bucket (`encrypt = true`), restrict IAM access, enable versioning, restrict who can read it.

## Plan → Apply Workflow

```bash
terraform init                              # download providers, init backend
terraform fmt                               # canonical formatting
terraform validate                          # syntax + types
terraform plan -out=tfplan                  # save plan to file
terraform apply tfplan                      # apply EXACT plan (no surprise drift)

terraform destroy                           # apply with all-delete plan (CAREFUL)
terraform plan -destroy                     # see what destroy would do
```

In CI, always `terraform plan -out=tfplan` then `terraform apply tfplan` so the apply matches what the reviewer saw.

## Modules — Reusable Units

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block = var.cidr
  tags       = var.tags
}

# modules/vpc/variables.tf
variable "cidr" { type = string }
variable "tags" { type = map(string), default = {} }

# modules/vpc/outputs.tf
output "id" { value = aws_vpc.this.id }
```

Use:

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
  tags   = { Env = "prod" }
}

# Or from registry / git
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.8.4"
  # ...
}
```

Module rules:
- **No `provider` blocks inside modules** (well, mostly — pass them in if multi-region).
- One module = one concern (network, app stack, IAM).
- Pin versions when consuming registry modules.
- Outputs are the public API — break them carefully (semver).

## Workspaces vs Folders vs `terragrunt` for Multi-Env

```bash
# Workspaces — same code, separate state per workspace
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform apply
```

**Workspaces are weak** for env separation (same backend, no per-env provider config, easy to apply to wrong workspace). Prefer:

- **Folder-per-env** (`envs/dev`, `envs/prod`), each with its own backend config + tfvars.
- **Terragrunt** — wrapper that keeps DRY config across many envs with stronger guardrails.

## Drift Detection

```bash
terraform plan                              # any non-empty diff is drift
```

Drift = someone clicked in the cloud console. Schedule a daily `terraform plan` in CI and alert on diff.

## Common Functions

```hcl
locals {
  app_name = lower(replace(var.app, " ", "-"))
  azs      = slice(data.aws_availability_zones.available.names, 0, 3)
  subnets  = [for i, az in local.azs : cidrsubnet(var.cidr, 8, i)]
  tags = merge(
    var.default_tags,
    { Name = "${var.env}-${local.app_name}" }
  )
}
```

Useful: `lookup`, `try`, `coalesce`, `concat`, `flatten`, `format`, `jsonencode`, `templatefile`, `cidrsubnet`.

## Lifecycle Tricks

```hcl
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy        = true         # block accidental destroy
    create_before_destroy  = true         # zero-downtime replace
    ignore_changes         = [tags["LastBackup"]]
  }
}
```

`prevent_destroy` is great for production DBs and state buckets. To actually delete, remove the lifecycle block first.

## Sensitive Outputs

```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true                          # hidden in terraform output
}
```

It's still in state. Store DB passwords in Secrets Manager / Vault and reference via `data` source.

## CI Pattern

```yaml
- run: terraform fmt -check
- run: terraform init -backend=false        # validate without contacting backend
- run: terraform validate
- run: tflint --recursive
- run: tfsec .                              # or checkov, trivy iac
- run: terraform init
- run: terraform plan -out=tfplan -no-color | tee plan.txt
- run: gh pr comment $PR --body-file plan.txt
# Require approval gate, then:
- run: terraform apply tfplan
```

## Interview Questions

**Q: What's in the Terraform state file and why does it matter?**
A: A JSON mapping of HCL addresses (`aws_instance.web`) to cloud resource IDs (`i-0abc`), plus stored attributes. It's the source of truth for "what does Terraform manage." Lose it → Terraform thinks nothing exists, tries to recreate everything → outage. Always store remote, encrypted, with locking, with versioning.

**Q: `count` vs `for_each`?**
A: `count` indexes by integer — removing item N shifts items N+1..end → destroys & recreates them. `for_each` keys by string/map key — stable across changes. Prefer `for_each` except for trivial cases.

**Q: How do you import an existing resource into Terraform?**
A: Write the HCL block to match, then `terraform import aws_instance.web i-0abc1234`. (Terraform 1.5+ supports `import` blocks in HCL itself.) Then `terraform plan` should show no diff. Reorganize state with `terraform state mv` afterward.

**Q: How do you handle multiple environments?**
A: Folder-per-env (`envs/dev`, `envs/prod`), each with its own backend config + tfvars, sharing the same modules. Workspaces are fine for tiny side projects but fragile in real teams. Terragrunt for many envs / many accounts.

**Q: Two engineers `apply` at the same time. What happens?**
A: With remote state + locking (S3 + DynamoDB), the second apply fails immediately with "state locked." Without locking, both write to the state file and corrupt it. Always use locking.

**Q: What's `terraform refresh` and when do you need it?**
A: Updates state file from real-world resources (drift detection) without applying changes. `plan` and `apply` do it automatically by default (unless `-refresh=false`). You rarely run `refresh` standalone.

## Common Pitfalls

- Committing `terraform.tfstate` to git — passwords + tokens leak.
- Using `count` + dynamic list — order changes recreate everything.
- Manual changes in the cloud console — silent drift, next apply reverts your fix or fights it.
- Editing state by hand — corruption that's hours to fix.
- No `prevent_destroy` on prod DB → `terraform destroy` wipes prod.
- Pinning provider with `>=` instead of `~>` — silent major version upgrade.
- Storing secrets in `tfvars` and committing — same problem as state. Use SOPS or env vars.

## Related

- [14-iac-pulumi-vs-cloudformation.md](14-iac-pulumi-vs-cloudformation.md)
- [15-cloud-aws-basics.md](15-cloud-aws-basics.md)
- [21-security-devsecops.md](21-security-devsecops.md)
- [10-ci-cd-fundamentals.md](10-ci-cd-fundamentals.md)
