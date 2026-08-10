---
title: Infrastructure as Code
type: concept
created: 2026-08-10
tags: [infrastructure, iac, terraform, gitops, platform-engineering, devops]
---

# Infrastructure as Code

## What

Describe servers, networks, databases, and cloud resources in **version-controlled text files**. The files are the source of truth. Tools read them and call cloud APIs to build the actual infrastructure.

The opposite is **click-ops** — someone SSH-ing into a server or clicking through a cloud console. Click-ops produces **snowflake servers**: no two alike, no audit trail, no way to reproduce.

IaC treats infrastructure with the same engineering discipline as application code: review it, test it, version it, roll it back.

---

## Core properties

- **Declarative** — say what you want, not how to build it. "I need a Postgres 15 instance with 2 CPU and 8GB RAM." The tool figures out the API calls.
- **Idempotent** — running the same definition ten times produces the same result. Safe to re-apply.
- **Reproducible** — staging is defined identically to production. "Works on my machine" applies to servers too.
- **Auditable** — every change is a git commit with author, timestamp, and diff. Cloud console clicks leave nothing.

---

## Terraform

The dominant IaC tool. Reads `.tf` files (HCL syntax), talks to cloud provider APIs, and reconciles desired state with actual state.

### Providers and resources

**Providers** are plugins — one per cloud or service. The AWS provider wraps the AWS API. The Cloudflare provider wraps Cloudflare's. One tool, any cloud.

**Resources** are things you want to exist:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags = {
    Env = "prod"
  }
}

resource "aws_db_instance" "postgres" {
  engine         = "postgres"
  engine_version = "15"
  instance_class = "db.t3.medium"
  username       = "app"
  password       = var.db_password
}
```

### Workflow

```
terraform init    # download provider plugins
terraform plan    # diff current state vs desired — shows what will change
terraform apply   # execute the changes
terraform destroy # tear everything down
```

`plan` is the key step: it's a safe, read-only preview. Review it like a PR diff before applying.

### State file

Terraform keeps a `terraform.tfstate` file tracking what it built. Without it, Terraform can't know "that EC2 instance already exists." State is the memory of the system.

**Remote state** is mandatory for teams — store in S3 with DynamoDB for locking (prevents two engineers applying simultaneously):

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-locks"
  }
}
```

### Modules

Reusable IaC components — same concept as functions or libraries:

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
  env    = "prod"
}

module "web_cluster" {
  source        = "./modules/ecs-cluster"
  vpc_id        = module.vpc.id
  instance_type = "t3.medium"
  min_capacity  = 2
  max_capacity  = 10
}
```

### Drift detection

Someone manually changed prod via the console? `terraform plan` detects the drift — it compares desired state (`.tf` files) against actual state (cloud API). Re-apply restores desired state.

> Discipline: never click the console in anger. If you change infra manually, you must update the `.tf` files immediately — or Terraform will overwrite the manual change on the next apply.

---

## GitOps

GitOps applies Git as the single source of truth for **both application code and infrastructure**. The deployment system continuously reconciles actual state toward what git says.

```
Developer → PR → Code Review → Merge → Automated Apply → Cloud
```

For Terraform, the GitOps workflow is:

1. Engineer opens PR with infra change
2. CI runs `terraform plan`, posts the diff as a PR comment
3. Team reviews the plan — what will be created, changed, destroyed
4. Merge triggers `terraform apply` automatically
5. Every infra change is now auditable, reviewable, and revertable via `git revert`

Key properties of GitOps:
- **Pull-based** — the automation system watches git and pulls changes. Not push-based scripts.
- **Convergence** — if actual state drifts from git, the system corrects it automatically.
- **Observability** — git log is the complete audit trail.

Kubernetes uses GitOps natively: tools like ArgoCD or Flux watch a git repo and continuously reconcile cluster state. Same principle as Terraform — desired state in files, operator reconciles.

```
┌──────────────┐   PR/merge   ┌──────────────┐   apply   ┌──────────────┐
│   Git repo   │ ────────────▶│  CI/CD (e.g. │ ─────────▶│  Cloud infra │
│  (.tf files) │              │   GitHub     │           │  (AWS/GCP/   │
│              │◀─────────────│   Actions)   │◀──────────│   Azure)     │
└──────────────┘   plan diff  └──────────────┘  drift    └──────────────┘
                   as PR comment               detection
```

---

## Layered infra architecture

At scale, a single Terraform root is dangerous — one mistake can destroy everything. Separate by blast radius:

| Layer | Contents | Who applies it | How often |
|---|---|---|---|
| Foundation | VPCs, accounts, DNS zones, IAM org | Platform team | Rarely |
| Services | ECS clusters, RDS, queues, caches | Platform or service teams | Per environment |
| App | Security groups, IAM roles, env-specific config | Dev teams per service | Per deploy |

Each layer has its own state file. `terraform destroy` on the app layer can't touch the VPC. Failure isolation mirrors state isolation.

> Conway's Law applies to infra: the state file boundaries will reflect your org chart. Two teams shouldn't share a state file any more than they'd share a database.

---

## Platform Engineering

GitOps + IaC at scale leads to a new organizational pattern: **Platform Engineering**.

The problem: if every dev team writes their own Terraform, you get inconsistency, security gaps, and duplication. The platform team becomes a bottleneck reviewing every infra PR.

Platform Engineering solution: the platform team builds **golden modules and self-service abstractions**. Dev teams consume them.

```
Platform team builds:                Dev teams use:

module "service" {                   module "order-service" {
  source = "platform/service"          source  = "platform/service"
  # encodes: ECS, ALB, IAM,           name    = "order-service"
  # security groups, observability,   image   = "my-ecr/orders:v1.2"
  # cost tags — all correct           cpu     = 512
}                                      memory  = 1024
                                     }
```

Dev teams get working, compliant infrastructure without knowing Terraform internals. Platform team sets guardrails once.

### Policy as Code

IaC enables **Policy as Code** — automated enforcement of org-wide rules:

- No public S3 buckets
- All EC2 must have cost-allocation tags
- No security groups open to `0.0.0.0/0` on port 22
- All databases must be encrypted at rest

Tools: **Checkov** (static analysis of `.tf` files), **OPA/Rego** (general policy language), **Sentinel** (Terraform Cloud's built-in). These run in CI, blocking non-compliant infra before it ever reaches apply.

Click-ops can never have policy enforcement. IaC makes it trivial.

---

## Trade-offs

| Pro | Con |
|---|---|
| Reproducible environments | State file is a critical dependency — lose it, lose control |
| Full audit trail in git | Terraform apply is destructive — plan review is mandatory |
| Testable, reviewable infra changes | Learning curve: HCL, provider quirks, state model |
| Policy enforcement possible | Cloud APIs change; providers lag behind new features |
| Self-service for dev teams (with platform modules) | Remote state + locking adds ops overhead for small teams |

---

## IaC tool landscape

| Tool | Model | Config language | Best for |
|---|---|---|---|
| **Terraform / OpenTofu** | Declarative | HCL | Multi-cloud, large teams, rich ecosystem |
| **Pulumi** | Declarative | TypeScript / Python / Go | Dev-heavy teams, complex conditional logic |
| **AWS CDK** | Declarative | TypeScript / Python | AWS-only, devs uncomfortable with HCL |
| **Ansible** | Imperative | YAML | Configuration management, not cloud provisioning |
| **Crossplane** | Declarative | YAML / CRDs | Kubernetes-native infra management |

OpenTofu is the open-source Terraform fork (after HashiCorp's BSL license change in 2023) — API-compatible, drop-in replacement.

---

## Key mental shift

Most developers think of infrastructure as someone else's problem. IaC says infra is **code** — subject to the same practices: review, test, version, refactor. The deployment layer of [[full-stack-layers]] is not ops folklore; it's a first-class engineering artifact.

GitOps extends this: the git repo is not just the history — it *is* the system. Actual state is merely a projection of what git says.

Platform Engineering extends this further: don't give teams raw tools, give them **abstractions**. The platform team is an internal product team; dev teams are their customers.

---

## Related

- [[full-stack-layers]] — where IaC sits in the overall stack
- [[ddd-bounded-context]] — Conway's Law applies to infra ownership too

## References

- Kief Morris, *Infrastructure as Code* (2021, O'Reilly) — comprehensive treatment
- Weave Works, [GitOps](https://www.weave.works/technologies/gitops/) — original GitOps definition
- Weaveworks, *The GitOps Handbook* — principles and patterns
- Terraform docs: [terraform.io/docs](https://developer.hashicorp.com/terraform/docs)
- Checkov: [checkov.io](https://www.checkov.io)
- Humanitec, [Platform Engineering](https://platformengineering.org/blog/what-is-platform-engineering) — community resource
