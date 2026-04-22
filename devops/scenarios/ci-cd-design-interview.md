# Scenario: Design a CI/CD Pipeline for a Microservices App

**Type:** System Design
**Difficulty:** Medium–High
**Frequency:** Very high — tests architectural thinking, not just tool knowledge

## The Question

*"Walk me through how you'd design a CI/CD pipeline for a microservices application with 15 services, deployed to Kubernetes. The team ships 20 times a day."*

---

## What the Interviewer Is Testing

- Do you think end-to-end (code → prod) or just "run some tests"?
- Do you understand the tradeoffs of monorepo vs polyrepo?
- Do you know how to prevent one broken service from blocking all others?
- Do you understand deployment strategies (canary, blue-green)?
- Do you think about security, secrets, and compliance in the pipeline?

---

## Design Walk-Through

### Step 1: Clarify the requirements

Before designing, ask:
```
- Monorepo or polyrepo?
- Languages / runtimes per service?
- Deployment target: K8s on-prem, EKS, GKE?
- Compliance requirements? (PCI, SOC2, HIPAA)
- Rollback expectations: auto or manual?
- SLO for deploy time: < 10 min? < 5 min?
```

### Step 2: Sketch the pipeline stages

```
┌─────────────┐   ┌──────────────┐   ┌───────────────┐   ┌──────────────┐
│  Source      │──▶│  Build/Test  │──▶│  Publish      │──▶│  Deploy      │
│  (Git push)  │   │  (per svc)   │   │  (container   │   │  (K8s)       │
└─────────────┘   └──────────────┘   │   registry)   │   └──────────────┘
                                      └───────────────┘
        │                   │                  │                 │
    Trigger             Fast tests         Image tag          Canary →
    on PR +             Lint, unit,         = git SHA          Rollout →
    main push           SAST, build         SBOM attached      Verify →
                                                               Full deploy
```

### Step 3: CI per service (not per repo commit)

In a monorepo with 15 services, you must run CI **only for changed services**. Running all 15 on every commit kills build time and destroys developer experience.

```yaml
# GitHub Actions — detect changed services
- name: Get changed services
  id: changed
  run: |
    CHANGED=$(git diff --name-only HEAD~1 HEAD \
      | grep '^services/' \
      | cut -d'/' -f2 \
      | sort -u \
      | jq -R -s -c 'split("\n")[:-1]')
    echo "services=$CHANGED" >> $GITHUB_OUTPUT

# Use matrix strategy to run CI for each changed service in parallel
- name: Run CI for changed services
  uses: ./.github/workflows/service-ci.yml
  strategy:
    matrix:
      service: ${{ fromJson(steps.changed.outputs.services) }}
```

### Step 4: What goes in each stage

#### Stage 1 — Fast Feedback (< 3 min)
```
- Lint (ruff, eslint, golangci-lint)
- Unit tests
- SAST (semgrep, CodeQL, Snyk static)
- Build Docker image (multi-stage, no push yet)
```

#### Stage 2 — Integration / Contract (< 8 min)
```
- Integration tests (spin up dependencies via Docker Compose or testcontainers)
- Contract testing (Pact — verify API contracts between services)
- Dependency scanning (OWASP, Snyk OSS)
- Secret scanning (TruffleHog, detect-secrets)
```

#### Stage 3 — Publish (< 2 min)
```
- Tag image as: registry/service:git-sha (immutable)
- Push to ECR / GCR / GHCR
- Attach SBOM (Software Bill of Materials) to image
- Sign image (cosign + Sigstore)
```

#### Stage 4 — Deploy (environment-specific)
```
dev:    auto-deploy on merge to main; no gates
staging: auto-deploy; run smoke tests + E2E subset
prod:   canary deploy (5% → 25% → 100%); auto-rollback on error rate spike
```

---

## Deployment Strategy: Canary with Auto-Rollback

```yaml
# Argo Rollouts — canary with automated analysis
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5        # 5% of traffic
        - pause: {duration: 5m}
        - analysis:
            templates:
              - templateName: error-rate-check
        - setWeight: 25
        - pause: {duration: 10m}
        - analysis:
            templates:
              - templateName: error-rate-check
        - setWeight: 100
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
    - name: error-rate
      successCondition: result[0] < 0.01   # < 1% error rate
      failureLimit: 1
      provider:
        prometheus:
          query: |
            sum(rate(http_requests_total{status=~"5.."}[2m]))
            / sum(rate(http_requests_total[2m]))
```

---

## Monorepo vs Polyrepo Trade-offs

| | Monorepo | Polyrepo |
|---|---|---|
| **Atomic cross-service changes** | Easy — one PR | Hard — coordinate multiple PRs |
| **CI isolation** | Requires changed-path detection | Natural per-repo |
| **Dependency management** | Shared lockfile, easier to enforce | Each service pins its own |
| **Team autonomy** | Lower — shared pipeline ownership | Higher — teams own their pipelines |
| **Tooling** | Nx, Bazel, Turborepo for smart builds | Simpler tools work |
| **Scale** | Works to ~100 services | Better at 100+ services |

**Recommendation for 15 services:** monorepo with path-based CI triggers.

---

## GitOps Layer (Continuous Deployment)

The CI pipeline produces an image tag. The CD layer (separate from CI) deploys it:

```
CI pipeline:  builds image → pushes  registry/payment-svc:abc1234
              → opens a PR in config repo: update image tag in helm/payment-svc/values.yaml

Config repo PR:
  - Auto-merged if staging, reviewed if prod
  - Argo CD detects the change in config repo
  - Argo CD reconciles: applies the new Deployment to K8s
  - No kubectl in CI pipeline — CI never touches K8s directly
```

This is the **GitOps pull model**: K8s state is always derivable from the config repo. Rollback = revert the image tag commit.

---

## Security Checkpoints in the Pipeline

```
Stage           Control                         Tool
────────────────────────────────────────────────────────────────
Code            SAST                            Semgrep, CodeQL
Code            Secret scanning                 TruffleHog, detect-secrets
Dependencies    SCA (open source vulns)         Snyk, Trivy
Container build Container image scanning        Trivy, Grype
Container build Image signing                   Cosign + Sigstore
Deploy          Admission controller (block      OPA/Gatekeeper,
                unsigned/vulnerable images)      Kyverno
Runtime         Runtime threat detection         Falco
```

---

## Common Follow-Up Questions

**Q: "How do you handle shared libraries between services?"**
A: Publish as versioned internal packages (npm private registry, internal PyPI, Go module proxy). Services pin to explicit versions. Dependabot/Renovate sends automated PRs when new versions drop.

**Q: "How do you ensure a broken service doesn't block others?"**
A: Path-based CI (only affected services build), independent deployment pipelines per service, contract tests instead of shared E2E suites for cross-service correctness.

**Q: "How do you manage environment-specific config?"**
A: Config lives in the GitOps config repo, not in the application repo. Sensitive config comes from Vault / External Secrets Operator — never stored in Git. Environment differences are Helm values files (`values-staging.yaml`, `values-prod.yaml`).

**Q: "What's your deploy time SLO and how do you hit it?"**
A: < 10 minutes from merge to prod canary start. Achieved by: parallel service builds, cached Docker layers, test parallelization, pre-warmed build agents (no cold-start on runners).

## Sources
- [[devops/topics/ci-cd]]
- [[devops/concepts/deployment-strategies]]
- [[devops/concepts/gitops]]
- [[devops/topics/security-devsecops]]
