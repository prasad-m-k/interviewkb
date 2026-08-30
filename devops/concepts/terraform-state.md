# Terraform State

**Topic:** [[devops/topics/infrastructure-as-code]]
**Related:** [[devops/concepts/infrastructure-as-code]], [[devops/concepts/terraform-patterns]], [[devops/concepts/secrets-management]]

---

## What it is

Terraform's state file (`terraform.tfstate`) is the mapping between the resources declared in your HCL and the real infrastructure objects they represent. It stores resource IDs, attributes, and metadata that Terraform uses to compute diffs and plan changes.

Without state, Terraform would have no way to know that `resource "aws_instance" "web" {}` already corresponds to `i-0abc123` in AWS — it would try to create a new one every time.

---

## What Is Inside the State File

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [{
        "schema_version": 1,
        "attributes": {
          "id": "i-0abc123def456",
          "ami": "ami-0c55b159cbfafe1f0",
          "instance_type": "t3.medium",
          "private_ip": "10.0.1.5",
          "tags": {"Name": "web-prod"}
          // ... all resource attributes
        }
      }]
    }
  ]
}
```

**Key characteristics:**
- Contains sensitive values (RDS passwords, private IPs, IAM key material) — treat as a secret
- JSON format; human-readable but not human-editable
- Includes the provider version used to create each resource
- Each resource has a `schema_version` used for state migration on provider upgrades

---

## Remote Backends

Never use local state (`terraform.tfstate` on disk) in a team environment. Two engineers with local state will diverge and corrupt each other's work.

### S3 + DynamoDB (AWS — most common)

```hcl
terraform {
  backend "s3" {
    bucket         = "myorg-tf-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"   # locking
    encrypt        = true                       # SSE at rest
    kms_key_id     = "arn:aws:kms:..."          # customer-managed KMS
  }
}
```

The DynamoDB table needs a `LockID` (String) partition key. Terraform writes a lock record before any write operation and deletes it after.

### GCS (Google Cloud)

```hcl
terraform {
  backend "gcs" {
    bucket = "myorg-tf-state"
    prefix = "prod/vpc"
    # Locking is built into GCS (object versioning + conditional updates)
  }
}
```

### Azure Blob Storage

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "myorgtfstate"
    container_name       = "tfstate"
    key                  = "prod/vpc/terraform.tfstate"
    # Blob leases provide locking
  }
}
```

### Terraform Cloud / HCP Terraform

```hcl
terraform {
  cloud {
    organization = "myorg"
    workspaces {
      name = "prod-vpc"
    }
  }
}
```

Terraform Cloud stores state, provides locking, runs plans/applies in a managed environment, and handles secrets. The managed option for teams without platform capacity to operate S3+DynamoDB.

---

## State Locking

State locking prevents concurrent writes that would corrupt state. The locking mechanism is backend-specific:
- **S3:** DynamoDB conditional write (put item with condition expression `attribute_not_exists(LockID)`)
- **GCS:** GCS object versioning + generation preconditions
- **Azure:** Azure Blob lease
- **Terraform Cloud:** built-in

If a Terraform run is interrupted (CTRL+C, process kill, network failure), the lock may not be released. To unlock:

```bash
terraform force-unlock LOCK_ID   # LOCK_ID from error message
```

Do not force-unlock if another apply is genuinely in progress — you will cause state corruption.

---

## State Manipulation Commands

These should be used rarely and carefully. Always take a backup first: `terraform state pull > backup.tfstate`.

| Command | Effect | When to Use |
|---|---|---|
| `terraform state list` | List all resources in state | Inspect what Terraform knows about |
| `terraform state show <address>` | Show all attributes for one resource | Debug unexpected plan behavior |
| `terraform state mv <old> <new>` | Rename resource in state | Renaming resources or moving into modules without destroy |
| `terraform state rm <address>` | Remove resource from state (does NOT destroy it) | Stop managing a resource without deleting it |
| `terraform state pull` | Download remote state to stdout | Backup; inspect; manipulation |
| `terraform state push` | Upload a local state file to remote | Recovery after state corruption (dangerous) |
| `terraform import <address> <id>` | Bring existing resource into state | Adopt pre-existing infrastructure |

### The `moved` Block (Preferred over `state mv`)

```hcl
# Moving a resource into a module
moved {
  from = aws_instance.web
  to   = module.webserver.aws_instance.web
}

# Renaming a resource
moved {
  from = aws_s3_bucket.old_name
  to   = aws_s3_bucket.new_name
}
```

`moved` blocks are declarative and tracked in git. `state mv` is an imperative command that leaves no record. Use `moved` for refactoring; it also generates a plan entry showing the move so it can be code-reviewed.

---

## Splitting State

As infrastructure grows, a single monolithic state file becomes a liability:
- `terraform plan` is slow (must check all resources)
- A bug in one team's code can block everyone's deploys
- Blast radius of a `terraform destroy` is the entire org

### Split by Layer (recommended pattern)

```
state/
├── network/          # VPC, subnets, route tables, VPN
│   └── terraform.tfstate
├── platform/         # EKS cluster, RDS, ElastiCache
│   └── terraform.tfstate
├── apps/
│   ├── service-a/
│   │   └── terraform.tfstate
│   └── service-b/
│       └── terraform.tfstate
└── iam/              # IAM roles, policies
    └── terraform.tfstate
```

### Cross-Stack References with `terraform_remote_state`

Lower layers expose outputs consumed by higher layers:

```hcl
# In network/ output:
output "vpc_id" {
  value = aws_vpc.main.id
}

# In platform/ reference the network state:
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "myorg-tf-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = data.terraform_remote_state.network.outputs.private_subnet_ids
  }
}
```

**Pitfall:** tight coupling between stacks. If the network stack changes its output names, the platform stack breaks. Prefer data sources (`data "aws_vpc" "main" { tags = { Name = "prod" } }`) for loose coupling when possible.

---

## Workspaces

Workspaces allow multiple independent state files for the same configuration directory.

```bash
terraform workspace new staging
terraform workspace new prod
terraform workspace select prod
terraform workspace list   # shows all; * = current
```

Inside configuration, `terraform.workspace` gives the current workspace name:

```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
}
```

### When to Use Workspaces (and When Not to)

**Use workspaces for:** ephemeral environments (PR preview environments, developer sandboxes), where the infra is structurally identical and differs only by a few values.

**Do not use workspaces for prod/staging separation** in most cases. The problem: a single `terraform destroy` in the wrong workspace kills prod. Both environments share the same backend configuration. Isolating prod into a completely separate backend (separate AWS account, separate S3 bucket) is safer. Workspaces give isolation-per-state-file but not isolation-per-backend — the lateral movement risk is too high for prod.

**Preferred alternative for environments:** separate directories or separate backend configurations per environment, managed by Terragrunt or a wrapper script.

---

## `terraform import` and Import Blocks

### Legacy CLI import

```bash
# Adopt existing S3 bucket into state
terraform import aws_s3_bucket.mybucket my-existing-bucket-name

# You must also write the resource block manually in .tf files
resource "aws_s3_bucket" "mybucket" {
  bucket = "my-existing-bucket-name"
}
```

After import, run `terraform plan` — ideally shows no changes (attributes match). If plan shows changes, update `.tf` to match reality.

### Declarative Import Blocks (Terraform 1.5+)

```hcl
import {
  id = "my-existing-bucket-name"
  to = aws_s3_bucket.mybucket
}

resource "aws_s3_bucket" "mybucket" {
  bucket = "my-existing-bucket-name"
}
```

Run `terraform plan -generate-config-out=generated.tf` to have Terraform generate the resource block automatically. Review and clean up the generated file, then `terraform apply`.

---

## State Security

The state file contains sensitive data: database passwords, private keys, API tokens. Always:
- Enable encryption at rest on the backend (S3 KMS, GCS encryption, Azure storage encryption)
- Use IAM policies to restrict who can read the state bucket
- Never commit `terraform.tfstate` to git (add to `.gitignore`)
- Consider using Vault's dynamic credentials so passwords are never in state at all (`data "vault_generic_secret"` fetched at plan time)
- Mark outputs containing sensitive data with `sensitive = true` to prevent logging

---

## Common Interview Q&A

**Q: The state file was accidentally deleted. What do you do?**
A: If using a remote backend with versioning (S3 versioning enabled, GCS versioning), restore the previous version of the state object. If no backup exists: run `terraform import` for each critical resource, in dependency order (resources used by other resources first). For large environments this is hours of work — this is why versioning on the state bucket is non-negotiable. Also check: Terraform Cloud keeps state history by default.

**Q: You need to destroy one resource that Terraform wants to destroy along with 10 dependent resources. How do you target just the one?**
A: `terraform destroy -target=resource_type.resource_name`. This destroys only the targeted resource. But be careful: resources that depended on the destroyed resource may now be in an inconsistent state. Always review the plan output for `-target` operations — Terraform warns that target mode may leave state inconsistent. After the targeted operation, run a full `terraform plan` to check the overall state.

**Q: Two teams both use Terraform to manage resources in the same AWS account. How do you prevent them from stepping on each other?**
A: Separate state files per team/service (no shared state file). IAM policies restrict each team to their own resource namespace (resource name prefix or tag enforcement via SCP). Optionally: separate AWS accounts per team (strongest isolation). Code-review gates enforce that cross-team resources are accessed via `data` sources (read-only reference), never via direct `resource` blocks in another team's code.

---

## Sources
- [[devops/concepts/infrastructure-as-code]]
- [[devops/concepts/terraform-patterns]]
- [[devops/topics/infrastructure-as-code]]
