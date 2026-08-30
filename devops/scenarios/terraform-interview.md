# Terraform Interview Scenarios

**Difficulty:** Medium–Hard
**Topic:** [[devops/topics/infrastructure-as-code]]
**Related:** [[devops/concepts/infrastructure-as-code]], [[devops/concepts/terraform-state]], [[devops/concepts/terraform-patterns]], [[devops/patterns/terraform-module-design]]

---

## Scenario 1: Design Terraform Architecture for a New Company

**Prompt:** "You're the first infrastructure engineer at a startup with 50 engineers and 5 microservices on AWS. Design the Terraform architecture from scratch."

### Answer Framework

**Start by asking:**
- Monorepo or polyrepo for application code?
- What AWS services do you use today? (VPC, ECS, RDS, S3, CloudFront?)
- How many environments? (dev, staging, prod — or more?)
- Is there a dedicated platform team or do app teams own their infra?

**Proposed architecture:**

```
infra-repo/
├── modules/                      # reusable modules (internal library)
│   ├── vpc/
│   ├── ecs-service/
│   ├── rds-postgres/
│   └── iam-role/
│
├── environments/
│   ├── dev/
│   │   ├── network/              # state 1: VPC, subnets, route tables
│   │   │   ├── main.tf
│   │   │   └── backend.tf        # s3://tf-state-dev/network/
│   │   ├── platform/             # state 2: ECS cluster, RDS, ElastiCache
│   │   │   └── ...
│   │   └── apps/
│   │       ├── service-a/        # state 3: per-service deployment config
│   │       └── service-b/
│   ├── staging/
│   └── prod/
│
├── .github/workflows/
│   ├── terraform-plan.yml        # plan on PR
│   └── terraform-apply.yml       # apply on merge to main
│
└── README.md
```

**Why separate state per layer:**
- Network changes rarely; app changes often. Coupling them means every app deploy locks the network state.
- A mistake in service-a's Terraform cannot destroy the VPC.
- Separate state → separate blast radius → safer.

**CI/CD pipeline for IaC:**
```yaml
# On PR:
  terraform fmt -check
  tflint
  checkov -d .
  terraform init
  terraform plan -out=plan.tfplan   # save plan for apply
  # Post plan output as PR comment (Atlantis or GitHub Actions TF action)

# On merge to main:
  terraform apply plan.tfplan       # apply the exact plan that was reviewed
```

**State backend per environment:**
```
S3 bucket: myorg-terraform-state-{env}
  with: versioning enabled, MFA delete, KMS encryption, no public access
DynamoDB table: terraform-locks-{env}
  LockID (String) partition key
```

**IAM for CI/CD:**
- CI runner assumes an IAM role via OIDC (no long-lived access keys)
- Role has least-privilege permissions per environment (prod role is more restricted than dev)
- Separate roles per layer if needed (network-deploy-role, apps-deploy-role)

---

## Scenario 2: Terraform State Corruption

**Prompt:** "You wake up to an alert: the Terraform apply that ran in CI failed partway through. The state file shows resources that don't exist and doesn't show resources that do. How do you recover?"

### Diagnosis Steps

```bash
# Step 1: pull the current state, save a backup
terraform state pull > pre-recovery-backup.tfstate
cp pre-recovery-backup.tfstate pre-recovery-backup-$(date +%s).tfstate

# Step 2: check what state thinks exists
terraform state list

# Step 3: compare against what actually exists in AWS
aws ec2 describe-instances --filters Name=tag:ManagedBy,Values=terraform
aws rds describe-db-instances
# ... per resource type

# Step 4: run plan to see the full divergence
terraform plan -refresh=true   # forces provider to query real state
```

### Recovery Patterns

**Case 1: Partial creation (some resources created, state not updated)**
```bash
# Resource exists in cloud but not in state
# Option A: import
terraform import aws_instance.web i-0abc123

# Option B (TF 1.5+): import block in code
# After import, terraform plan should show no changes
```

**Case 2: Resource in state but deleted from cloud**
```bash
# Remove the dangling reference from state
terraform state rm aws_instance.old_web
# Next plan will show it as "to create" — decide if you want it back
```

**Case 3: Full state corruption (state is garbled JSON)**
```bash
# Restore from S3 versioning
aws s3api list-object-versions \
  --bucket myorg-terraform-state \
  --prefix prod/apps/service-a/terraform.tfstate

aws s3api get-object \
  --bucket myorg-terraform-state \
  --key prod/apps/service-a/terraform.tfstate \
  --version-id VERSION_ID \
  restored.tfstate

# Upload restored state (with locking)
terraform state push restored.tfstate
```

**Prevention:**
- S3 versioning + MFA delete on state bucket (non-negotiable)
- Never run `terraform apply` locally against prod — always via CI
- State locking (DynamoDB) prevents concurrent corruption
- Backup state before any `state mv`, `state rm`, or `state push` operation

---

## Scenario 3: Migrating from CloudFormation to Terraform

**Prompt:** "Your team is migrating 200 existing AWS resources from CloudFormation to Terraform. How do you do this safely?"

### The Core Problem

Resources managed by CloudFormation cannot be managed by Terraform without first removing them from CloudFormation's management. But you cannot delete a CFN stack without deleting the resources.

**Solution: Retain + Import pattern**

```bash
# Step 1: Retain resources on stack deletion
# In CFN template, add DeletionPolicy: Retain to each resource
# OR use aws cloudformation update-stack with UpdateReplacePolicy: Retain

# Step 2: Remove resource from CFN stack (it will be retained, not deleted)
aws cloudformation update-stack \
  --stack-name my-stack \
  --use-previous-template \
  --parameters ...
# Remove the resource from the template — CFN will "delete" it from the stack
# but DeletionPolicy: Retain means the actual resource is not deleted

# Step 3: Write Terraform code for the retained resource
resource "aws_s3_bucket" "data" {
  bucket = "my-existing-bucket"  # must match existing bucket name
}

# Step 4: Import
terraform import aws_s3_bucket.data my-existing-bucket

# Step 5: Validate
terraform plan   # should show no changes
```

### Migration Order

Migrate in dependency order: resources that have no dependencies first, then resources that others depend on.

1. IAM roles and policies (many resources depend on these, but they don't depend on others)
2. VPC, subnets, security groups
3. RDS, ElastiCache (depend on VPC)
4. ECS / EC2 / Lambda (depend on VPC, IAM, databases)
5. Load balancers, Route 53 (depend on compute)

### Parallel Operation Risk

During migration, you will have some resources in CFN and some in Terraform. If both try to manage the same resource:
- CFN cannot detect Terraform changes → CloudFormation will try to revert to its template values on next stack update
- Strict rule: once a resource is imported into Terraform, remove it from CloudFormation immediately (via `DeletionPolicy: Retain` + remove from template)

---

## Scenario 4: Terraform in a GitOps Pipeline

**Prompt:** "How would you integrate Terraform into a GitOps workflow where all infra changes must be code-reviewed and applied automatically?"

### Architecture

```
Developer
  │ opens PR with .tf changes
  ▼
GitHub / GitLab
  │ triggers CI
  ▼
Atlantis / GitHub Actions
  │
  ├── terraform fmt -check          (style)
  ├── tflint                        (lint)
  ├── checkov                       (security policy)
  ├── terraform init
  ├── terraform plan -out plan.tfplan
  │     → posts plan output as PR comment
  │
  │ (human reviews plan diff in PR comment)
  │ PR approved → merged to main
  │
  ▼
Atlantis / GitHub Actions
  ├── terraform apply plan.tfplan   (applies the exact reviewed plan)
  └── posts apply output as commit comment
```

**Key design decisions:**

1. **Planned artifact:** save `plan.tfplan` and apply the exact plan that was reviewed. Re-planning before apply risks the plan changing between review and apply (e.g., another PR merged).

2. **Workspace locking:** only one plan or apply runs per workspace at a time. Atlantis handles this automatically.

3. **Credentials:** CI assumes IAM role via OIDC (no stored access keys). Separate OIDC subjects for plan vs. apply if you want stronger permission separation.

4. **Breaking glass:** for emergency infra changes, a separate procedure with explicit manual override + post-hoc PR documenting the change.

---

## Scenario 5: Remove a Resource Without Destroying It

**Prompt:** "An EC2 instance was created by Terraform but we want to manage it manually from now on. How do you remove it from Terraform management without terminating it?"

```bash
# Remove from state — does NOT destroy the actual EC2 instance
terraform state rm aws_instance.legacy_web

# Remove the corresponding resource block from .tf files

# Run plan to confirm
terraform plan   # should show no reference to the removed resource
                 # if there are other resources that depended on it,
                 # those may now show changes — handle accordingly
```

This is also the pattern for intentionally "orphaning" a resource: it continues running, it's just no longer tracked or managed by Terraform. The risk: future Terraform runs have no knowledge of this resource; if its IP or ID was referenced by other resources via outputs, those references break.

---

## Sources
- [[devops/concepts/infrastructure-as-code]]
- [[devops/concepts/terraform-state]]
- [[devops/concepts/terraform-patterns]]
- [[devops/patterns/terraform-module-design]]
