# IaC — Pulumi vs CloudFormation (vs Terraform)

> **TL;DR:** Use Terraform for multi-cloud / portable infra. Use CloudFormation/CDK if you're all-in on AWS and want native integration. Use Pulumi if you want full programming-language power (TypeScript/Python/Go) over your infra.

## The Tradeoff Triangle

```
                Multi-cloud (Terraform)
                       /\
                      /  \
                     /    \
                    /      \
     Native-AWS    /        \   Language-native (Pulumi/CDK)
   (CloudFormation)--------- 
```

You pick **two**. Multi-cloud + language-native = Pulumi (multi-cloud) or CDKTF (Terraform's CDK). Native + language = CDK. Multi-cloud + DSL = Terraform.

## Comparison Table

| Aspect | Terraform | Pulumi | CloudFormation | AWS CDK |
|--------|-----------|--------|----------------|---------|
| Language | HCL (DSL) | TS / Py / Go / .NET / Java | YAML/JSON | TS / Py / Java / .NET / Go |
| Clouds | All (providers) | All (providers) | AWS only | AWS only |
| State | External file (S3, Terraform Cloud, etc.) | External (Pulumi Cloud, S3, etc.) | AWS-managed | AWS-managed (via CFN under the hood) |
| Diff before apply | `terraform plan` | `pulumi preview` | Change Sets | `cdk diff` |
| Modularity | Modules | Components (classes) | Nested stacks | Constructs (very strong) |
| Imperative loops | Limited (`for_each`) | Full language | Limited | Full language |
| Testing | Terratest (Go) | Standard test frameworks (jest/pytest) | Limited | jest/pytest + cdk synth snapshots |
| Secrets handling | Plaintext in state (encrypt at rest) | First-class (`pulumi.secret`) | SecureString params, manual | Mostly via Secrets Manager constructs |
| Open source | Yes (BSL since 1.6) → OpenTofu fork | Yes | No (proprietary) | Yes |
| Drift detection | Native (`plan`) | Native (`refresh`) | Built-in (in console) | Via underlying CFN |

## CloudFormation — AWS-Native, YAML-Heavy

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple S3 bucket

Parameters:
  Env:
    Type: String
    AllowedValues: [dev, staging, prod]

Resources:
  Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain                  # CRITICAL for data
    Properties:
      BucketName: !Sub myapp-${Env}-uploads
      VersioningConfiguration: { Status: Enabled }
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault: { SSEAlgorithm: AES256 }

Outputs:
  BucketArn:
    Value: !GetAtt Bucket.Arn
    Export: { Name: !Sub ${AWS::StackName}-BucketArn }
```

```bash
aws cloudformation deploy \
    --template-file template.yaml \
    --stack-name myapp-dev-storage \
    --parameter-overrides Env=dev \
    --capabilities CAPABILITY_NAMED_IAM
```

**CFN strengths:**
- Native — no extra tooling on AWS. IAM, change sets, drift detection in console.
- Rollback-on-failure built in.
- Service Catalog integration.
- No state file to manage (AWS owns it).

**CFN pain points:**
- YAML loops / conditionals are awkward (`!If`, `!FindInMap`, custom resources for the rest).
- Some resources lag the AWS API — new feature available days before CFN supports it.
- Updates can be slow; some failures stick the stack in `UPDATE_ROLLBACK_FAILED`.
- AWS-only.

### Change Sets

```bash
aws cloudformation create-change-set --stack-name X --change-set-name Y \
    --template-body file://t.yaml
aws cloudformation describe-change-set --stack-name X --change-set-name Y
aws cloudformation execute-change-set  --stack-name X --change-set-name Y
```

The CFN equivalent of `terraform plan`. Review diff before applying.

## AWS CDK — "CFN but in TypeScript"

CDK synthesizes CloudFormation under the hood. You write code, you get a CFN template, AWS deploys it.

```typescript
// lib/storage-stack.ts
import { Stack, StackProps, RemovalPolicy } from 'aws-cdk-lib';
import { Bucket, BucketEncryption } from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';

export class StorageStack extends Stack {
  public readonly uploads: Bucket;

  constructor(scope: Construct, id: string, props: StackProps & { env: string }) {
    super(scope, id, props);

    this.uploads = new Bucket(this, 'Uploads', {
      bucketName: `myapp-${props.env}-uploads`,
      versioned: true,
      encryption: BucketEncryption.S3_MANAGED,
      removalPolicy: props.env === 'prod' ? RemovalPolicy.RETAIN : RemovalPolicy.DESTROY,
    });
  }
}
```

```bash
cdk synth                              # render CFN template
cdk diff                               # diff against deployed stack
cdk deploy                             # deploy
cdk destroy                            # tear down
```

**Strengths:**
- L2/L3 constructs encode best practices (sensible defaults, IAM least privilege wiring).
- Reuse anything from npm/pypi.
- Loops, conditionals, types — all native.

**Caveats:**
- CFN is still underneath. Same rollback quirks, same race conditions, same drift behavior.
- Bootstrapping (`cdk bootstrap`) creates a one-time CDKToolkit stack per region.
- "What did this synth to?" — always run `cdk synth` and review the CFN diff for prod changes.

## Pulumi — Real Code, Any Cloud

```typescript
// index.ts
import * as aws from "@pulumi/aws";
import * as pulumi from "@pulumi/pulumi";

const cfg = new pulumi.Config();
const env = cfg.require("env");

const bucket = new aws.s3.BucketV2("uploads", {
  bucket: `myapp-${env}-uploads`,
  forceDestroy: env !== "prod",
});

new aws.s3.BucketVersioningV2("uploads-v", {
  bucket: bucket.id,
  versioningConfiguration: { status: "Enabled" },
});

new aws.s3.BucketServerSideEncryptionConfigurationV2("uploads-enc", {
  bucket: bucket.id,
  rules: [{
    applyServerSideEncryptionByDefault: { sseAlgorithm: "AES256" },
  }],
});

export const bucketArn = bucket.arn;
```

```bash
pulumi stack init prod
pulumi config set env prod
pulumi config set --secret dbPassword s3cr3t   # encrypted in state
pulumi preview
pulumi up
```

**Strengths:**
- Full language: tests, abstractions, package management.
- Secrets are first-class: `pulumi config set --secret`, `pulumi.secret(...)`, encrypted in state.
- Multi-cloud, multi-language.
- "Pulumi Cloud" SaaS for state, but you can self-host (S3 backend) or keep local.

**Caveats:**
- Newer ecosystem than Terraform — fewer community modules.
- "Just code" cuts both ways — easy to write spaghetti.
- Lock-in to the Pulumi runtime (less of an issue than CFN, more than Terraform).

## Decision Framework

Pick **CloudFormation/CDK** when:
- 100% AWS, no plans to change.
- You want AWS to own the state (no S3 + DynamoDB to bootstrap).
- Your team is heavy on AWS-certified engineers comfortable with CFN.
- You value AWS-native change sets and rollback.

Pick **Terraform** when:
- Multi-cloud or hybrid (AWS + GCP + Azure + Cloudflare + Datadog + GitHub).
- You want a strong, mature module ecosystem.
- You want a clean separation of config-from-code (HCL).
- Compliance / contractor portability matters.

Pick **Pulumi** when:
- You want unit tests, abstractions, reuse from your language ecosystem.
- Your team is strong in TS/Python and weak in DSLs.
- You need secrets handled natively (not "encrypt the bucket and hope").
- Multi-cloud + code beats Terraform + HCL for your team.

## Hybrid Patterns

Common real-world setup:

- **CloudFormation for landing zone** (AWS Org, Control Tower, baseline accounts).
- **Terraform for application infra** (VPC, EKS, RDS — multi-cloud-ready).
- **Helm + Kustomize for in-cluster** (app deployments, K8s addons).
- **CDK or Pulumi where teams demand language-native** (within bounds set by the platform team).

## Cost Modeling

All four can use [Infracost](https://www.infracost.io) to print monthly $$$ in your PR. Worth it.

```bash
infracost breakdown --path=. --terraform-plan-flags="-var-file=prod.tfvars"
```

## Interview Questions

**Q: When would you pick Terraform over CloudFormation?**
A: Multi-cloud, ecosystem of modules, contractor-friendly skill (HCL is a small DSL), open-source community. CFN if you're AWS-only and want AWS-native rollback, change sets, and zero state-storage burden.

**Q: What's the value of CDK if it just generates CloudFormation?**
A: You get real language features (loops, types, tests, npm packages) **plus** AWS-native deployment (change sets, drift, rollback). Best of both — for teams that want code, not YAML. Downside: CFN limits still apply (some races, slow ROLLBACK_FAILED states).

**Q: How does Pulumi handle secrets differently than Terraform?**
A: Pulumi encrypts secret config values with a per-stack key (or KMS / Vault) and stores ciphertext in state. Marked secrets are masked in CLI output and treated as `pulumi.Output<string>` so you can't accidentally leak them via `console.log`. Terraform stores them in plaintext in state — you must encrypt the backend bucket and restrict access.

**Q: How would you migrate from CloudFormation to Terraform?**
A: (1) `terraform import` each managed resource. (2) Run `terraform plan` until zero diff. (3) Delete the CFN stack with `DeletionPolicy: Retain` on each resource (so resources aren't dropped). (4) Going forward, manage in Terraform. Plan a phased migration per-stack rather than big-bang.

**Q: Pulumi vs CDK?**
A: CDK is AWS-only; Pulumi is multi-cloud. CDK synthesizes CFN (inherits its limitations); Pulumi calls cloud APIs directly. CDK has deeper AWS-native integration, Pulumi has more uniform multi-cloud DX. Pick CDK if you're committed to AWS; pick Pulumi if you cross clouds or want broader provider support.

## Common Pitfalls

- Using CFN's default `DeletionPolicy: Delete` on a stateful resource — destroying the stack wipes data.
- Mixing Terraform + CFN for the same logical stack without clear ownership — drift between tools.
- Running `cdk deploy` against a stack that's been modified in the console — undefined behavior.
- Pulumi state stored locally only — single point of loss.
- Trying to express logic in raw CFN YAML — use CDK or Terraform's `for_each`.

## Related

- [13-iac-terraform.md](13-iac-terraform.md)
- [15-cloud-aws-basics.md](15-cloud-aws-basics.md)
- [09-helm-and-kustomize.md](09-helm-and-kustomize.md)
