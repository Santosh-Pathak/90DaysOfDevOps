# Day 61 – Introduction to Terraform and Your First AWS Infrastructure

---

## Task 1: Infrastructure as Code — In My Own Words

### What is IaC and why does it matter?

Infrastructure as Code means describing your servers, networks, databases, and cloud resources in text files — the same way you describe an application in source code. Instead of clicking through a cloud console to create an EC2 instance, you write a `.tf` file that says "I want an EC2 instance with these properties," run a command, and the infrastructure appears. Delete the file and run another command — the infrastructure disappears.

**Why it matters in DevOps:** Manual infrastructure doesn't scale. Clicking through AWS consoles creates infrastructure that nobody can reproduce exactly, can't be reviewed, can't be version-controlled, and breaks the moment the person who built it leaves the team. IaC makes infrastructure as reviewable, testable, and repeatable as application code.

### Problems IaC solves vs manual console work

| Problem with manual console | IaC solution |
|---|---|
| Can't reproduce exactly (was it t2.micro or t3.micro?) | Manifest is the exact spec — always reproducible |
| No audit trail of who changed what | Git history shows every change, when, and by whom |
| Disaster recovery requires memory | `terraform apply` recreates everything from code |
| Multiple environments drift over time (dev ≠ prod) | Same code, different variable values → identical environments |
| No peer review for infrastructure changes | PRs on `.tf` files get reviewed like application code |
| Provisioning is manual and slow | `terraform apply` runs in minutes, unattended |

### Terraform vs other IaC tools

| Tool | Approach | Language | Cloud-agnostic? | Best for |
|---|---|---|---|---|
| **Terraform** | Declarative | HCL | ✅ Yes | Multi-cloud infra provisioning |
| **AWS CloudFormation** | Declarative | JSON/YAML | ❌ AWS only | AWS-native shops, deep service integration |
| **Ansible** | Procedural | YAML | ✅ Yes | Configuration management, OS-level tasks |
| **Pulumi** | Declarative + Imperative | Python/JS/Go/etc. | ✅ Yes | Devs who prefer real programming languages |

**Terraform vs CloudFormation:** Terraform works across AWS, GCP, Azure, Kubernetes, and 1000+ providers with one tool. CloudFormation is AWS-only but has tighter integration (native stack rollbacks, drift detection). In multi-cloud or cloud-agnostic environments, Terraform wins. In deeply AWS-integrated enterprises, CloudFormation is a valid choice.

**Terraform vs Ansible:** Terraform provisions infrastructure (creates VPCs, EC2s, RDS instances). Ansible configures what's already running (installs nginx, edits config files, restarts services). They complement each other — Terraform brings the server up, Ansible configures it.

### Declarative and cloud-agnostic

**Declarative:** You describe *what* you want, not *how* to get there. You write "I want 3 EC2 instances" — Terraform figures out the API calls needed. Compare to Ansible (procedural): "Run this script, then install this package, then restart this service" — you describe the steps.

**Cloud-agnostic:** The same Terraform workflow (`init` → `plan` → `apply`) works whether you're creating AWS S3 buckets, GCP Cloud Storage, Azure Blob Storage, or a Kubernetes namespace. You swap the provider, not the workflow.

---

## Task 2: Install Terraform and Configure AWS

```bash
# Linux (amd64) — HashiCorp apt repo
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor \
  -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform

terraform -version
# Terraform v1.8.x
# on linux_amd64
```

```bash
# Configure AWS credentials
aws configure
# AWS Access Key ID [None]: AKIAXXXXXXXXXXXXXXXX
# AWS Secret Access Key [None]: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
# Default region name [None]: ap-south-1
# Default output format [None]: json

# Verify access
aws sts get-caller-identity
# {
#   "UserId": "AIDAXXXXXXXXXXXXXXXX",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/devops-user"
# }
```

---

## Task 3: First Terraform Config — S3 Bucket

### `main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.0"
}

provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "my_bucket" {
  bucket = "terraweek-myname-2026"   # Must be globally unique

  tags = {
    Name        = "TerraWeek S3 Bucket"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

### The Terraform lifecycle

```bash
mkdir terraform-basics && cd terraform-basics
# (create main.tf as above)

terraform init
# Initializing the backend...
# Initializing provider plugins...
# - Finding hashicorp/aws versions matching "~> 5.0"...
# - Installing hashicorp/aws v5.x.x...
# - Installed hashicorp/aws v5.x.x
# Terraform has been successfully initialized!

terraform plan
# Terraform will perform the following actions:
#
#   # aws_s3_bucket.my_bucket will be created
#   + resource "aws_s3_bucket" "my_bucket" {
#       + bucket = "terraweek-myname-2026"
#       + tags   = {
#           + "Environment" = "dev"
#           + "ManagedBy"   = "Terraform"
#           + "Name"        = "TerraWeek S3 Bucket"
#         }
#       ...
#     }
#
# Plan: 1 to add, 0 to change, 0 to destroy.

terraform apply
# Plan: 1 to add, 0 to change, 0 to destroy.
# Do you want to perform these actions? yes
# aws_s3_bucket.my_bucket: Creating...
# aws_s3_bucket.my_bucket: Creation complete after 3s
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**What `terraform init` downloaded:**

The `.terraform/` directory contains:
- `providers/registry.terraform.io/hashicorp/aws/5.x.x/linux_amd64/terraform-provider-aws_v5.x.x` — the compiled AWS provider binary
- `.terraform.lock.hcl` — records exact provider versions for reproducibility (commit this to git)

The provider is a Go binary that translates Terraform HCL into AWS API calls.

---

## Task 4: Add an EC2 Instance

```hcl
# Add to main.tf
resource "aws_instance" "web" {
  ami           = "ami-0f5ee92e2d63afc18"   # Amazon Linux 2, ap-south-1
  instance_type = "t2.micro"

  tags = {
    Name = "TerraWeek-Day1"
  }
}
```

```bash
terraform plan
# aws_s3_bucket.my_bucket: Refreshing state...   ← S3 bucket already exists
#
# Terraform will perform the following actions:
#   # aws_instance.web will be created
#   + resource "aws_instance" "web" { ... }
#
# Plan: 1 to add, 0 to change, 0 to destroy.   ← only EC2 is new

terraform apply
# aws_instance.web: Creating...
# aws_instance.web: Still creating... (10s elapsed)
# aws_instance.web: Creation complete after 22s
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**How Terraform knows the S3 bucket already exists:**

Before planning, Terraform runs a **refresh** — it reads the current state from `terraform.tfstate` and compares it to the actual AWS API. The state file records that `aws_s3_bucket.my_bucket` was already created. The AWS provider confirms it still exists with the same attributes. Since there's no difference between desired and actual state for the bucket, Terraform only plans the EC2 instance that's new in the config.

This is the reconciliation loop — desired state (`.tf` files) vs actual state (state file + AWS API).

---

## Task 5: Understand the State File

```bash
# Human-readable state view
terraform show
# # aws_s3_bucket.my_bucket:
# resource "aws_s3_bucket" "my_bucket" {
#     bucket                      = "terraweek-myname-2026"
#     bucket_domain_name          = "terraweek-myname-2026.s3.amazonaws.com"
#     id                          = "terraweek-myname-2026"
#     region                      = "ap-south-1"
#     arn                         = "arn:aws:s3:::terraweek-myname-2026"
#     ...
# }
# # aws_instance.web:
# resource "aws_instance" "web" {
#     id                           = "i-0abc123def456"
#     ami                          = "ami-0f5ee92e2d63afc18"
#     instance_type                = "t2.micro"
#     public_ip                    = "13.x.x.x"
#     ...
# }

# List all managed resources
terraform state list
# aws_s3_bucket.my_bucket
# aws_instance.web

# Detailed view of one resource
terraform state show aws_s3_bucket.my_bucket
terraform state show aws_instance.web
```

### What the state file stores

`terraform.tfstate` is a JSON file containing:
- The resource type, name, and provider
- Every attribute of every managed resource (IDs, ARNs, IPs, creation times)
- Dependencies between resources
- Terraform and provider version metadata

**Why you should never manually edit the state file:** It is the source of truth for what Terraform manages. Manual edits corrupt the mapping between Terraform resources and real cloud resources. If you edit it wrong, `terraform apply` might destroy things it shouldn't or fail to create things it should. Use `terraform state mv`, `terraform state rm`, and `terraform import` for state manipulation.

**Why the state file should not be committed to Git:** It contains sensitive values in plaintext — database passwords, API keys, private IP addresses. It also causes merge conflicts on every team member's `apply`. In team environments, use **remote state** (S3 + DynamoDB for AWS, or Terraform Cloud) so everyone shares one authoritative state file.

---

## Task 6: Modify, Plan, and Destroy

```hcl
# Edit main.tf — change the EC2 tag
resource "aws_instance" "web" {
  ami           = "ami-0f5ee92e2d63afc18"
  instance_type = "t2.micro"

  tags = {
    Name = "TerraWeek-Modified"   # Changed from "TerraWeek-Day1"
  }
}
```

```bash
terraform plan
# aws_instance.web will be updated in-place
#   ~ resource "aws_instance" "web" {
#         id            = "i-0abc123def456"
#       ~ tags          = {
#           ~ "Name" = "TerraWeek-Day1" -> "TerraWeek-Modified"
#         }
#     }
#
# Plan: 0 to add, 1 to change, 0 to destroy.
```

### Plan symbols

| Symbol | Meaning |
|---|---|
| `+` | Resource will be **created** |
| `-` | Resource will be **destroyed** |
| `~` | Resource will be **updated in-place** |
| `-/+` | Resource will be **destroyed then recreated** (some changes require this) |

Tags are metadata that AWS can update without replacing the instance. This is an **in-place update** (`~`). Contrast with changing `instance_type` — some changes require destroy-and-recreate (`-/+`), which causes brief downtime.

```bash
terraform apply
# aws_instance.web: Modifying...
# aws_instance.web: Modifications complete
```

### Destroy everything

```bash
terraform destroy
# Plan:
#   - aws_s3_bucket.my_bucket will be destroyed
#   - aws_instance.web will be destroyed
#
# Plan: 0 to add, 0 to change, 2 to destroy.
# Do you really want to destroy all resources? yes
#
# aws_instance.web: Destroying...
# aws_s3_bucket.my_bucket: Destroying...
# Destroy complete! Resources: 2 destroyed.
```

Both resources confirmed gone in the AWS console.

---

## Terraform Command Reference

| Command | What it does |
|---|---|
| `terraform init` | Downloads provider plugins, initializes backend, creates `.terraform/` |
| `terraform validate` | Checks HCL syntax without connecting to cloud |
| `terraform fmt` | Auto-formats `.tf` files to canonical style |
| `terraform plan` | Shows what would change — dry run (never modifies anything) |
| `terraform apply` | Creates/updates/deletes resources to match config |
| `terraform destroy` | Destroys all resources managed by this config |
| `terraform show` | Human-readable view of current state |
| `terraform state list` | Lists all resources in state |
| `terraform state show <resource>` | Details of one resource in state |
| `terraform output` | Prints declared output values |
| `terraform graph` | Generates dependency graph (pipe to `dot` for visualization) |

---

## `.gitignore` for Terraform Projects

```gitignore
# Local .terraform directory (provider binaries — large, reproducible)
.terraform/

# State files (contain secrets, cause merge conflicts)
*.tfstate
*.tfstate.backup

# Variable files (may contain secrets)
*.tfvars
*.tfvars.json

# Crash logs
crash.log
crash.*.log

# Override files (local experiments)
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Commit this (locks provider versions for reproducibility)
# .terraform.lock.hcl → DO commit this
```