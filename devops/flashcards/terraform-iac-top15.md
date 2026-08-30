# Terraform & IaC — Top 15 Flashcards

**Topic:** [[devops/topics/infrastructure-as-code]]
**Related:** [[devops/concepts/infrastructure-as-code]], [[devops/concepts/terraform-state]], [[devops/concepts/terraform-patterns]], [[devops/concepts/ansible]]

---

**Q1. What is the Terraform state file and why is it necessary?**
A: The state file maps HCL resource declarations to real infrastructure objects (e.g., `aws_instance.web` → `i-0abc123`). Without it, Terraform cannot determine whether a resource already exists or needs to be created. It also stores the current values of all resource attributes, which are used to compute diffs during `terraform plan`. Always store state remotely (S3+DynamoDB, GCS, Terraform Cloud) — local state breaks in team environments because two engineers applying simultaneously will corrupt it.

---

**Q2. What happens when two engineers run `terraform apply` at the same time?**
A: With state locking enabled (DynamoDB for S3 backend), the second apply blocks with a "state locked" error until the first apply completes and releases the lock. Without locking, both reads see the same state, both plan changes against it, and both write back — resulting in duplicate resources, conflicting attribute values, or outright state corruption. Locking is why DynamoDB is required alongside S3; the DynamoDB table's conditional write ensures only one writer at a time.

---

**Q3. What is the difference between `terraform plan` and `terraform apply`?**
A: `terraform plan` is a dry run: it reads the current state, queries real infrastructure for drift, computes the diff against your desired configuration, and shows what would change — without making any changes. `terraform apply` executes those changes. In CI/CD, the correct pattern is: plan on PR (output posted as comment for human review), apply on merge. Optionally, save the plan with `-out=plan.tfplan` and apply the exact saved plan to guarantee that what was reviewed is what gets applied.

---

**Q4. What is Terraform drift and how do you detect and fix it?**
A: Drift occurs when real infrastructure diverges from what Terraform's state describes — typically from manual console changes, another automation tool touching the same resource, or cloud-provider auto-modifications. Detection: `terraform plan` compares code + state against real infra; changes you didn't make signal drift. Fix options: (1) re-apply Terraform to restore declared state (if the manual change was wrong), (2) update `.tf` code to reflect the legitimate change and use `terraform import` + `terraform refresh`, (3) add the attribute to `lifecycle { ignore_changes = [...] }` if the change is legitimately managed externally. Prevention: restrict console access to Terraform-managed resources via IAM SCPs.

---

**Q5. Why is `for_each` preferred over `count` for creating multiple resources?**
A: `count` addresses resources by integer index: `aws_instance.web[0]`, `[1]`, `[2]`. If you remove the middle element, all indices shift — Terraform destroys `web[2]` and plans to modify `web[1]` (which is now `web[1]` the old `web[2]`). For dependent resources (like EIPs attached to instances), this causes unnecessary destroys and recreates. `for_each` addresses by key: `aws_instance.web["web-a"]`, `["web-b"]`. Removing `"web-b"` leaves `"web-a"` and `"web-c"` completely untouched. Rule: use `count` only for 0/1 (enable/disable a resource); use `for_each` for N > 1.

---

**Q6. What is a Terraform module and what problem does it solve?**
A: A module is a directory of `.tf` files that can be called from other Terraform configurations via a `module` block. Modules solve DRY: instead of copy-pasting a VPC definition into dev, staging, and prod environments, you write it once as a module and call it three times with different variable values. Modules also enforce abstraction: callers express intent (I want a production VPC), not mechanism (I want these specific route table entries). Standard module structure: `main.tf` (resources), `variables.tf` (inputs), `outputs.tf` (exposed values), `versions.tf` (provider constraints).

---

**Q7. How do you refactor Terraform resources without destroying and recreating them?**
A: `moved` blocks (Terraform 1.1+): declare the rename declaratively in HCL. Terraform updates state to reflect the new address without destroying the resource. Example: `moved { from = aws_instance.web; to = module.webserver.aws_instance.web }`. The `moved` block is tracked in git, appears in `terraform plan` output, and can be code-reviewed. For pre-1.1: use `terraform state mv old_address new_address` (imperative command; no git history). Never rename a resource by deleting the old block and adding a new one — Terraform will destroy and recreate the real resource.

---

**Q8. What is the `lifecycle` meta-argument and when would you use `prevent_destroy`?**
A: The `lifecycle` block customizes how Terraform manages a resource's lifecycle: `prevent_destroy = true` causes `terraform apply` to error if the plan would destroy this resource (protects production databases from accidental deletion); `create_before_destroy = true` creates the replacement before destroying the old one (required for resources that cannot be absent even briefly, like TLS certificates); `ignore_changes = [...]` tells Terraform to stop tracking specific attributes (used when an external process legitimately modifies them, like auto-scaling modifying `desired_capacity`). `precondition`/`postcondition` blocks (1.2+) add validation before/after creation.

---

**Q9. How do you pass secrets to Terraform without hardcoding them in `.tf` files?**
A: Four approaches: (1) Environment variables: `TF_VAR_db_password=secret` — Terraform reads them automatically, no file contains the secret. (2) `data` source from secrets manager at apply time: `data "aws_secretsmanager_secret_version" "db" {}` — fetched live during apply, never stored in code. (3) HashiCorp Vault provider: `data "vault_generic_secret" "db" {}` — fetched from Vault, supports dynamic credentials so the secret expires. (4) Mark outputs as `sensitive = true` to hide them from plan/apply logs. Never commit `.tfvars` files containing secrets to git; add them to `.gitignore`.

---

**Q10. What is the difference between Terraform workspaces and separate state files for multi-environment management?**
A: Workspaces are multiple state files within the same backend configuration, switched with `terraform workspace select`. They share the same backend S3 bucket and the same `.tf` code. Separate state files use completely different backend configurations (different S3 buckets, different keys, potentially different AWS accounts). Workspaces are appropriate for ephemeral dev environments (PR previews) where environments are structurally identical. For prod/staging separation, separate state files (or separate accounts) are strongly preferred: a `workspace select prod && terraform destroy` in the wrong terminal can destroy production. Separate backends remove this lateral movement risk entirely.

---

**Q11. When should you use Ansible instead of Terraform?**
A: Terraform manages cloud resource lifecycle (create, update, destroy) with state tracking. Ansible runs imperative tasks on existing machines without state. Use Ansible when you need to: configure an OS (install packages, write config files, manage services), run ad-hoc commands across a fleet (patch 500 servers), or manage state on bare metal or VMs where Terraform has no provider. Modern approach: use Packer + Ansible to bake VM images at build time (not at deploy time), then Terraform to deploy the pre-baked image. This eliminates runtime SSH access and removes Ansible from the critical deployment path.

---

**Q12. How do you import existing infrastructure into Terraform management?**
A: Two approaches: (1) CLI import: write the `.tf` resource block first, then run `terraform import resource_type.name cloud_resource_id`. Terraform adds the resource to state, but the `.tf` attributes must be filled in manually. Run `terraform plan` — no changes means the code matches reality. (2) Import blocks (Terraform 1.5+): declare `import { id = "bucket-name"; to = aws_s3_bucket.data }` in HCL, run `terraform plan -generate-config-out=generated.tf` to auto-generate the resource block. Review and clean up generated code, then `terraform apply` to execute the import. The import block approach is code-reviewed and tracked in git.

---

**Q13. How does Ansible ensure idempotency?**
A: Each Ansible module is written to check the current state before taking action. `package: state: present` checks if the package is installed; if yes, reports `ok` and does nothing; if no, installs and reports `changed`. `template` computes a hash of the rendered template and compares to the destination file; only writes if different. The exception: `command` and `shell` modules are not idempotent by default — they always run and always report `changed`. Make them idempotent with the `creates` argument (skip if a file exists) or a `when` conditional on a `stat` task.

---

**Q14. What is Ansible Vault and how does it compare to HashiCorp Vault?**
A: Ansible Vault is a built-in encryption mechanism for protecting sensitive data (passwords, API keys) in Ansible playbooks and variable files. It encrypts files with AES-256, and the vault password is provided at runtime. The encrypted file is safe to commit to git. HashiCorp Vault is a full secrets management platform: it stores, rotates, and audits secrets; generates dynamic credentials (database passwords that expire); enforces access policies; and integrates with Kubernetes, AWS, LDAP. For Ansible: Vault (Ansible's) is sufficient for small teams. Prefer HashiCorp Vault for production: use `lookup('community.hashi_vault.hashi_vault', ...)` in Ansible to fetch secrets at runtime without storing encrypted secrets in the repo at all.

---

**Q15. Design a Terraform structure for a company with 3 environments (dev/staging/prod) and 5 microservices. What state layout would you use?**
A: Separate state per environment per layer. Layer the stacks by rate of change and blast radius: (1) `network/{env}` — VPC, subnets, route tables; rarely changes; all services depend on it; separate state prevents an app change from touching the network. (2) `platform/{env}` — EKS/ECS cluster, RDS, ElastiCache, shared IAM roles; changes infrequently; consumed by all services. (3) `apps/{env}/{service-name}` — per-service deployment config, service-specific IAM, ALB listener rules; changes frequently; isolated blast radius. Cross-stack references via `data` sources (loose coupling) or `terraform_remote_state` (tight coupling — use only when the consuming stack is owned by the same team). CI/CD applies each stack independently; a failing app stack does not block the network stack apply.

---

## Sources
- [[devops/concepts/infrastructure-as-code]]
- [[devops/concepts/terraform-state]]
- [[devops/concepts/terraform-patterns]]
- [[devops/concepts/ansible]]
- [[devops/patterns/terraform-module-design]]
