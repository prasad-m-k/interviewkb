# Security & DevSecOps

**Topic:** [[devops/overview]]
**Related:** [[devops/concepts/secrets-management]]

## What DevSecOps Is

"Shift security left" — integrate security checks into every stage of the CI/CD pipeline instead of a gate at the end. Developers own security as part of their workflow, not a separate team's problem.

```
Traditional: Dev → Dev → Dev → [Security Review] → Deploy   (bottleneck, late feedback)
DevSecOps:   [Scan on commit] → [Scan on build] → [Scan on deploy] → [Runtime monitoring]
```

---

## Security Scanning Types

| Type | What it scans | When to run | Tools |
|---|---|---|---|
| **SAST** (Static Analysis) | Source code for vulnerabilities | Pre-commit, CI | Semgrep, SonarQube, Bandit (Python) |
| **DAST** (Dynamic Analysis) | Running application for vulnerabilities | Staging | OWASP ZAP, Burp Suite |
| **SCA** (Software Composition Analysis) | Open-source dependencies for CVEs | CI on every PR | Snyk, Dependabot, OWASP Dependency-Check |
| **Container scanning** | Docker image layers for CVEs | CI after build | Trivy, Grype, Clair |
| **IaC scanning** | Terraform/K8s YAML for misconfigs | CI | Checkov, tfsec, kube-score |
| **Secret scanning** | Committed secrets in code | Pre-commit + CI | git-secrets, truffleHog, gitleaks |

---

## The Supply Chain (SLSA Framework)

Software supply chain attacks target the build process, not just the code (e.g., SolarWinds, Log4Shell).

**SLSA** (Supply-chain Levels for Software Artifacts) defines levels of supply chain integrity:

| Level | Requirement |
|---|---|
| 1 | Build process documented; provenance generated |
| 2 | Signed provenance; hosted build service |
| 3 | Isolated builds; auditable build process |
| 4 | Hermetic builds; two-party review |

**Practical steps:**
- Sign container images (Cosign + Sigstore)
- Generate SBOM (Software Bill of Materials) per build
- Pin dependency versions (no `latest` tags, no floating semver)
- Use a private registry mirror — don't pull from DockerHub in prod pipelines

---

## Secrets Management

Never store secrets in code, environment variables in Dockerfiles, or Kubernetes Secrets unencrypted. Full deep-dive: [[devops/concepts/secrets-management]]

**Common patterns:**
1. **HashiCorp Vault** — secrets as a service; dynamic secrets (DB credentials that expire)
2. **AWS Secrets Manager / SSM Parameter Store** — cloud-native; IAM-controlled access
3. **K8s External Secrets Operator** — sync secrets from Vault/AWS into K8s Secrets
4. **SOPS + Age/KMS** — encrypt secret files in Git; decrypt at deploy time
5. **Sealed Secrets** — encrypt K8s Secrets for safe Git storage; decrypt by in-cluster controller

---

## Kubernetes Security Checklist

```yaml
# Pod Security Context best practices
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

**Other K8s security checks:**
- [ ] RBAC: least-privilege; avoid `cluster-admin` for app service accounts
- [ ] NetworkPolicy: default deny all; allow only needed pod-to-pod traffic
- [ ] Pod Security Admission (replaces PodSecurityPolicy): enforce `restricted` profile
- [ ] Image signing + admission webhook to reject unsigned images
- [ ] Secrets from external store (not plain K8s Secrets in etcd)
- [ ] Audit logging enabled on the API server

---

## Common Interview Q&A

**Q: A secret was accidentally committed to Git. What do you do?**  
→ [[devops/scenarios/secret-exposed]]

**Q: How do you prevent secrets from being committed in the first place?**  
A: Pre-commit hooks (`git-secrets`, `detect-secrets`, `gitleaks`). Secret scanning in CI (fail the build). Developer education. Rotate aggressively if caught.

**Q: What's the difference between authentication and authorization in K8s?**  
A: **Authentication** = proving who you are (certificates, OIDC, service account tokens). **Authorization** = what you're allowed to do (RBAC: Role + RoleBinding, ClusterRole + ClusterRoleBinding). Always separate the two layers.

**Q: How do you handle a CVE in a base Docker image?**  
A: Rebuild all images using the patched base. Automate: Dependabot / Renovate can open PRs for base image updates. In the short term, check if the CVE is exploitable in your context (many low-severity CVEs in distro packages don't affect your attack surface).

## Sources
- [[devops/concepts/secrets-management]]
- [[devops/overview]]
