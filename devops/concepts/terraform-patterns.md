# Terraform Patterns and Advanced Features

**Topic:** [[devops/topics/infrastructure-as-code]]
**Related:** [[devops/concepts/infrastructure-as-code]], [[devops/concepts/terraform-state]], [[devops/patterns/terraform-module-design]]

---

## Variables: Input, Local, Output

```hcl
# Input variables — caller provides values
variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, prod)"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "instance_count" {
  type    = number
  default = 1
}

# Local values — computed within the module, not exposed
locals {
  name_prefix  = "${var.project}-${var.environment}"
  common_tags  = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Output values — expose resource attributes to callers
output "instance_id" {
  value       = aws_instance.web.id
  description = "EC2 instance ID for the web server"
  sensitive   = false
}

output "db_password" {
  value     = random_password.db.result
  sensitive = true   # hidden from logs and console output
}
```

**Variable precedence** (highest to lowest):
1. `-var "key=value"` CLI flag
2. `.tfvars` file via `-var-file=`
3. `terraform.tfvars` (auto-loaded)
4. `*.auto.tfvars` (auto-loaded, alphabetical)
5. Environment variable `TF_VAR_name`
6. Default value in variable block

---

## count vs. for_each

### `count` — index-based

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }
}
# Addresses: aws_instance.web[0], aws_instance.web[1], aws_instance.web[2]
```

**Problem with count:** if you remove the middle element (e.g., go from 3 to 2), Terraform destroys `web[2]` but what you actually wanted was to remove `web[1]`. All indices shift — downstream resources that depend on these may be destroyed and recreated.

### `for_each` — key-based (preferred)

```hcl
resource "aws_instance" "web" {
  for_each      = toset(["web-a", "web-b", "web-c"])
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  tags = { Name = each.key }
}
# Addresses: aws_instance.web["web-a"], aws_instance.web["web-b"], aws_instance.web["web-c"]

# With a map (more common):
variable "servers" {
  default = {
    web  = { type = "t3.micro",  az = "us-east-1a" }
    api  = { type = "t3.small",  az = "us-east-1b" }
    worker = { type = "t3.medium", az = "us-east-1c" }
  }
}

resource "aws_instance" "server" {
  for_each          = var.servers
  ami               = "ami-0c55b159cbfafe1f0"
  instance_type     = each.value.type
  availability_zone = each.value.az
  tags = { Name = each.key }
}
```

**Why for_each is preferred:** removing `web-b` from the set leaves `web-a` and `web-c` untouched. No index shifting. Stable addresses mean stable state.

**Rule:** use `count` only for binary create/don't-create (count = 0 or 1). Use `for_each` whenever you need N > 1 of something.

---

## Dynamic Blocks

Generate repeated nested blocks programmatically:

```hcl
variable "ingress_rules" {
  default = [
    { from_port = 80,  to_port = 80,  protocol = "tcp", cidr = "0.0.0.0/0" },
    { from_port = 443, to_port = 443, protocol = "tcp", cidr = "0.0.0.0/0" },
    { from_port = 22,  to_port = 22,  protocol = "tcp", cidr = "10.0.0.0/8" },
  ]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = [ingress.value.cidr]
    }
  }
}
```

Without `dynamic`, you would need a separate `ingress` block for each rule — not DRY, not parameterizable.

---

## Lifecycle Meta-Arguments

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  lifecycle {
    # Prevent accidental destruction (e.g., on database instances)
    prevent_destroy = true

    # Create new before destroying old (zero-downtime replacement)
    create_before_destroy = true

    # Ignore changes to these attributes (e.g., auto-scaling changes instance count)
    ignore_changes = [tags["LastUpdated"], user_data]

    # Validation before resource is created
    precondition {
      condition     = var.environment != "prod" || var.instance_type != "t3.micro"
      error_message = "Production must not use t3.micro."
    }

    # Validation after resource is created
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
}
```

**`prevent_destroy`:** blocks `terraform destroy` and any plan that would destroy this resource. Remove the meta-argument to allow destruction. Essential for production databases.

**`create_before_destroy`:** for resource types that cannot be updated in-place (e.g., launch templates, SSL certificates), create the replacement before destroying the old one. Required when other resources depend on this one and cannot tolerate a brief absence.

**`ignore_changes`:** tells Terraform to stop tracking changes to these attributes in the state. Useful when an external process legitimately modifies attributes Terraform manages (auto-scaling modifies `desired_capacity`; Kubernetes modifies node labels).

---

## Data Sources

Data sources read existing infrastructure without managing it:

```hcl
# Read an existing VPC by tag (not managed by this TF code)
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["prod-vpc"]
  }
}

# Read latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use in a resource
resource "aws_instance" "web" {
  ami    = data.aws_ami.amazon_linux.id
  vpc_id = data.aws_vpc.main.id
}
```

Data sources are the correct way to reference infrastructure managed by another team's Terraform (rather than `terraform_remote_state`, which creates tight coupling).

---

## `depends_on`

Terraform infers resource dependencies automatically from attribute references. Use `depends_on` only when the dependency is not expressed through attribute references:

```hcl
resource "aws_iam_role_policy_attachment" "s3_access" {
  role       = aws_iam_role.lambda.name
  policy_arn = aws_iam_policy.s3.arn
}

resource "aws_lambda_function" "processor" {
  function_name = "processor"
  role          = aws_iam_role.lambda.arn

  # Terraform cannot infer that the function needs the policy to be attached
  # before it can run successfully, even though there's no attribute reference
  depends_on = [aws_iam_role_policy_attachment.s3_access]
}
```

**When to use:** IAM policy attachments (IAM propagation lag means the role needs the policy before Lambda can execute, but there is no attribute reference), Kubernetes resources that depend on CRD installation (CRD must exist before the custom resource is created).

**When not to use:** if you can express the dependency through attribute references, do that instead. `depends_on` is a blunt tool that serializes the dependency — it can slow apply significantly if overused.

---

## Conditional Resources and Expressions

```hcl
# Create resource only in non-prod
resource "aws_cloudwatch_metric_alarm" "debug" {
  count = var.environment != "prod" ? 1 : 0
  # ...
}

# Conditional value
resource "aws_instance" "web" {
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
}

# String interpolation and functions
locals {
  s3_prefix  = lower(replace(var.project_name, " ", "-"))
  bucket_name = "${local.s3_prefix}-${var.environment}-${data.aws_caller_identity.current.account_id}"
}

# Common built-in functions
locals {
  merged_tags   = merge(local.common_tags, { Service = "web" })
  subnet_list   = flatten([aws_subnet.public[*].id, aws_subnet.private[*].id])
  env_is_prod   = contains(["prod", "production"], var.environment)
  cidr_list     = [for i in range(3) : cidrsubnet("10.0.0.0/16", 8, i)]
}
```

---

## Terraform Testing

### `terraform validate` and `tflint`

```bash
terraform validate      # syntax and type checking (no cloud calls)
tflint                  # additional linting: unused variables, deprecated syntax, AWS-specific rules
```

### `checkov` (policy-as-code)

```bash
checkov -d .            # scan all .tf files for security misconfigs
# Checks: S3 bucket not public, SGs not open to 0.0.0.0/0, encryption enabled, etc.
```

### `terraform test` (Terraform 1.6+)

Write tests in `.tftest.hcl` files:

```hcl
# tests/instance_type.tftest.hcl
run "prod_uses_large_instance" {
  variables {
    environment = "prod"
  }
  assert {
    condition     = aws_instance.web.instance_type == "t3.large"
    error_message = "Prod must use t3.large"
  }
}
```

### Terratest (Go-based integration tests)

```go
func TestVPCModule(t *testing.T) {
  opts := &terraform.Options{TerraformDir: "../modules/vpc"}
  defer terraform.Destroy(t, opts)
  terraform.InitAndApply(t, opts)
  vpcID := terraform.Output(t, opts, "vpc_id")
  assert.NotEmpty(t, vpcID)
  // Call AWS SDK to verify actual VPC attributes
}
```

Terratest deploys real infrastructure, runs assertions, then destroys — actual cloud costs, but catches issues that `terraform test` cannot.

---

## Providers and Version Pinning

```hcl
terraform {
  required_version = ">= 1.6.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # allow 5.x, block 6.x
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.20.0"
    }
  }
}

provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile
  default_tags {
    tags = local.common_tags   # applied to every resource by default
  }
  assume_role {
    role_arn = "arn:aws:iam::${var.account_id}:role/TerraformDeploy"
  }
}
```

**Always pin provider versions.** `hashicorp/aws` has had breaking changes between minor versions. `~> 5.0` allows patch and minor updates within 5.x but blocks 6.0. Run `terraform providers lock` to generate `.terraform.lock.hcl` — commit this file; it pins exact checksums.

**Multiple provider aliases** (deploy to multiple regions or accounts):

```hcl
provider "aws" { region = "us-east-1" }

provider "aws" {
  alias  = "eu"
  region = "eu-west-1"
}

resource "aws_s3_bucket" "replica" {
  provider = aws.eu
  bucket   = "my-replica-bucket"
}
```

---

## Common Interview Q&A

**Q: What is a Terraform provider and how does it work under the hood?**
A: A provider is a plugin that translates Terraform resource declarations into API calls for a specific service (AWS, GCP, Azure, Kubernetes, GitHub, etc.). Providers are downloaded by `terraform init` from the Terraform Registry and stored in `.terraform/providers/`. They implement a gRPC interface that Terraform core calls during plan and apply. Each provider exposes resource types, data sources, and their schemas. The provider handles authentication, retry logic, and mapping between Terraform's HCL representation and the service's API.

**Q: A `terraform apply` is running in CI and the pipeline fails mid-run. Resources were partially created. What do you do?**
A: First, do not run `terraform apply` again blindly — you need to understand what was created. Run `terraform plan` to see the current state vs. desired state. Terraform's state was partially updated for resources that were successfully created. The plan will show: already-created resources as no-change, partially-created resources as needing update or replacement, not-yet-created resources as add. Typically you can just re-run `apply` and Terraform will complete what was left. If a resource is in a broken partial state (like a half-configured RDS), you may need to `terraform state rm` it and `terraform import` the actual resource ID, or delete it manually and re-apply.

**Q: How do you use Terraform with multiple AWS accounts?**
A: Use the `assume_role` provider argument to assume cross-account IAM roles from a central CI/CD role: `assume_role { role_arn = "arn:aws:iam::ACCOUNT_ID:role/TerraformDeploy" }`. The CI/CD runner's role has `sts:AssumeRole` permission on each target account's TerraformDeploy role. Separate state files per account (different S3 keys or different state buckets). Separate provider aliases if deploying to multiple accounts in the same Terraform code.

---

## Sources
- [[devops/concepts/infrastructure-as-code]]
- [[devops/concepts/terraform-state]]
- [[devops/patterns/terraform-module-design]]
- [[devops/topics/infrastructure-as-code]]
