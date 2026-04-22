# Infrastructure as Code (IaC)

**Topic:** [[devops/overview]]
**Related:** [[devops/concepts/infrastructure-as-code]], [[devops/concepts/gitops]]

## What it is

IaC = describing infrastructure (servers, networks, databases, IAM roles) in code files, which are version-controlled and applied automatically. The infrastructure state is a function of the code, not of manual clicks.

**Key properties:**
- **Declarative:** you describe *what* you want, not *how* to create it (Terraform, Kubernetes YAML)
- **Idempotent:** applying the same code twice produces the same result — no duplicates
- **Version-controlled:** change history, code review, rollback via git revert
- **Auditable:** every infrastructure change has a PR, approver, and timestamp

---

## Tool Landscape

| Tool | Type | Language | Manages |
|---|---|---|---|
| **Terraform** | Declarative, stateful | HCL | Cloud resources (AWS, GCP, Azure, K8s) |
| **Pulumi** | Declarative, stateful | Python/TS/Go | Same as Terraform, real code |
| **Ansible** | Imperative/procedural | YAML | Config mgmt, software install, state on existing VMs |
| **Chef / Puppet** | Declarative (older) | Ruby DSL | Config mgmt on VMs |
| **Packer** | Imperative | HCL | Build VM/container images |
| **CloudFormation** | Declarative | YAML/JSON | AWS-only |
| **CDK** | Declarative | Python/TS | AWS-only, real code, compiles to CFN |

---

## Terraform Deep-Dive

### State management

Terraform stores the mapping between your HCL and real cloud resources in a **state file** (`terraform.tfstate`).

```
Problem: Local state is dangerous — team members overwrite each other
Solution: Remote state backend (S3 + DynamoDB for AWS, GCS, Terraform Cloud)

# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"   # prevents concurrent applies
    encrypt        = true
  }
}
```

**State locking:** DynamoDB table prevents two `terraform apply` runs from corrupting state simultaneously.

### Drift detection

```bash
terraform plan          # shows diff between desired (code) and actual (state + real infra)
                        # if plan shows changes you didn't make → drift detected

terraform refresh       # updates state to match real infra (read-only, no changes)
```

**Drift** = real infrastructure diverged from Terraform state (someone manually changed a resource). Fix: import the resource or reapply.

### Module pattern

```hcl
# modules/vpc/main.tf — reusable module
variable "cidr_block" {}
variable "environment" {}

resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
  tags = { Environment = var.environment }
}

output "vpc_id" { value = aws_vpc.main.id }

# root/main.tf — consume the module
module "prod_vpc" {
  source      = "./modules/vpc"
  cidr_block  = "10.0.0.0/16"
  environment = "prod"
}
```

---

## Immutable Infrastructure

**Mutable:** deploy by SSHing in and running `apt update && systemctl restart app` — servers accumulate configuration drift over time; "snowflake servers."

**Immutable:** servers are never modified. To update: build a new image (Packer), launch new instances, drain old ones, terminate them. Identical to how Kubernetes rolling updates work.

```
Mutable path:  server → SSH in → apt upgrade → config change
Immutable path: base image → Packer bake → new AMI → blue-green swap
```

Benefits: no drift, reproducible, rollback = launch old AMI.

---

## Common Interview Q&A

**Q: How do you handle secrets in Terraform?**  
A: Never hardcode secrets in `.tf` files. Options:
1. Pass via environment variables (`TF_VAR_db_password`)
2. Read from secrets manager at apply time: `data "aws_secretsmanager_secret_version"`
3. Mark outputs as `sensitive = true` to prevent logging
4. Use Vault provider to fetch secrets dynamically

**Q: How would you refactor Terraform without destroying resources?**  
A: Use `terraform state mv` to rename resources in state without destroying/recreating. Use `moved` blocks (TF 1.1+) for declarative refactoring.

**Q: What's the difference between `terraform plan` and `terraform apply`?**  
A: `plan` is a dry run — shows what *would* change. `apply` executes the changes. Always review plan output before apply. In CI, run plan on PR and apply on merge to main.

## Sources
- [[devops/concepts/infrastructure-as-code]]
- [[devops/overview]]
