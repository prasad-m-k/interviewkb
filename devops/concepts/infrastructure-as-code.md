# Infrastructure as Code (IaC)

**Topic:** [[devops/topics/infrastructure-as-code]]
**Related:** [[devops/concepts/terraform-state]], [[devops/concepts/terraform-patterns]], [[devops/concepts/ansible]], [[devops/concepts/gitops]], [[devops/concepts/secrets-management]]

---

## What it is

Infrastructure as Code is the practice of managing infrastructure (compute, networking, storage, IAM, DNS, databases) through machine-readable configuration files rather than manual processes. The configuration is version-controlled, peer-reviewed, and applied automatically — making infrastructure changes indistinguishable from application code changes in terms of process discipline.

---

## The Four Properties That Matter

**Declarative:** you describe the *desired end state*, not the sequence of steps to reach it. The tool figures out the how. `resource "aws_instance" "web" { ami = "ami-123" }` says "there should be an EC2 instance with this AMI" — not "run these 12 AWS API calls."

**Idempotent:** running the same configuration multiple times produces the same result. Apply twice → no change on the second apply. This enables safe re-runs after partial failures.

**Version-controlled:** every infrastructure change has a commit, an author, a timestamp, and a PR. Rollback is `git revert`. Audit trail is `git log`.

**Testable:** configurations can be validated before apply (dry-run / plan), linted (tflint, checkov), and tested against real infrastructure (Terratest, `terraform test`).

---

## Tool Decision Matrix

| Tool | Paradigm | Scope | When to Use |
|---|---|---|---|
| **Terraform** | Declarative, stateful | Cloud resources (any provider) | Provisioning cloud infra: VPCs, VMs, databases, IAM, DNS |
| **Pulumi** | Declarative, stateful | Same as Terraform | Same as Terraform but prefer real programming languages (TypeScript, Python, Go) |
| **Ansible** | Imperative/procedural, stateless | OS-level config on existing machines | Installing software, configuring services, patching VMs, ad-hoc fleet management |
| **Chef / Puppet** | Declarative, agent-based | OS config on long-lived VMs | Large enterprises with existing investment; largely superseded by Ansible + immutable |
| **CloudFormation** | Declarative, stateful | AWS only | AWS-only shops; required for some AWS services not yet in TF |
| **CDK** | Declarative (compiles to CFN) | AWS only | AWS-only shops that prefer Python/TypeScript over YAML |
| **Packer** | Imperative | VM / container image builds | Build golden AMIs / VM images; sits before Terraform (bake image → TF uses AMI) |
| **Crossplane** | Declarative, K8s-native | Cloud resources via K8s CRDs | Kubernetes-native IaC; GitOps for cloud resources |

### Terraform vs. Ansible — Which When?

This is the most common interview question on this topic:

```
Terraform provisions:  "Create an EC2 instance with these specs"
Ansible configures:    "On this existing machine, install nginx and write this config file"

Typical workflow:
  Terraform → provisions EC2 instance
  Ansible   → configures the installed OS (installs packages, writes configs)

Modern alternative (immutable infrastructure):
  Packer    → bakes AMI with all software pre-installed
  Terraform → provisions EC2 with that AMI (Ansible no longer needed at runtime)
```

Use Terraform for lifecycle management of cloud resources. Use Ansible when you must configure state on existing machines or when immutable infrastructure is not feasible (bare metal, legacy VMs). For new greenfield cloud infrastructure: Packer + Terraform eliminates Ansible at runtime entirely.

---

## IaC Team Organization Patterns

### Centralized Platform Team
A platform team owns all Terraform and publishes internal modules. Application teams consume modules but do not write Terraform directly.

- Pros: consistency, DRY, governance enforced at module level
- Cons: bottleneck; application teams blocked on platform team

### Federated with Shared Modules
Application teams own their own Terraform in their repos. A shared module library (internal registry or git-tagged modules) provides building blocks.

- Pros: team autonomy, scales with org size
- Cons: requires governance tooling (policy-as-code, code review standards)

### GitOps for IaC
Terraform is applied via CI/CD (not locally from a developer's laptop). `plan` runs on PR; `apply` runs on merge to main.

- Pros: audit trail for every apply; no "worked on my machine" state drift
- Cons: requires careful handling of plan/apply split; secrets in CI
- Tools: Atlantis (open-source); Terraform Cloud / Spacelift (managed)

---

## Drift

**Drift** occurs when the real infrastructure state diverges from what Terraform's state file (and code) describes. Common causes:
- Manual changes via AWS console or CLI ("just this once")
- Another automation tool modified the same resource
- Resource was created outside Terraform and not imported
- Cloud provider made an automatic change (auto-scaling, auto-patching)

**Detection:** `terraform plan` compares code + state to real infra. Changes shown that you did not make = drift.

**Remediation options:**
1. Re-apply to bring infra back to declared state (if manual change was wrong)
2. Update code to reflect the legitimate change (if manual change should be kept) → `terraform import` + update `.tf` file
3. `terraform refresh` updates state to match reality (for non-Terraform-managed properties like tags auto-added by the cloud provider)

**Prevention:**
- SCPs / IAM policies that prevent console changes to Terraform-managed resources
- Drift detection scans (Driftctl, cloud custodian, Terraform Cloud drift detection)
- Cultural: "all infra changes go through PR" as an enforced rule

---

## Common Interview Q&A

**Q: What happens if two engineers run `terraform apply` simultaneously?**
A: Without state locking, both reads will see the same state, both will plan changes, and both will apply — resulting in partial duplicate resources, resource conflicts, or state corruption. With DynamoDB locking (for S3 backend), the second `apply` blocks until the first finishes. Terraform acquires the lock before writing state, releases it after. Terraform Cloud handles this automatically.

**Q: How do you bring existing resources under Terraform management?**
A: Two approaches. (1) `terraform import`: run `terraform import resource_type.name resource_id` to pull the existing resource into state, then write the corresponding `.tf` block manually. (2) `import` blocks (TF 1.5+): declare the import declaratively in HCL and run `terraform plan`/`apply` to execute it. After either approach, `terraform plan` should show no changes.

**Q: How do you refactor Terraform code without destroying and recreating resources?**
A: `moved` blocks (Terraform 1.1+): declare that a resource previously at address `A` is now at address `B`. Terraform updates state without destroying anything. For module-level renames, `moved` blocks work the same way. For anything older than 1.1, use `terraform state mv old_address new_address` directly.

**Q: What is the difference between `count` and `for_each`?**
A: See [[devops/concepts/terraform-patterns#count vs for_each]]. Short answer: `for_each` is almost always preferred because resources are addressed by key (stable) rather than index (unstable — inserting a new item shifts all indices and destroys/recreates downstream resources).

---

## Sources
- [[devops/topics/infrastructure-as-code]]
- [[devops/concepts/terraform-state]]
- [[devops/concepts/terraform-patterns]]
- [[devops/concepts/ansible]]
