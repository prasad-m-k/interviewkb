# CI/CD

**Topic:** [[devops/overview]]
**Related:** [[devops/concepts/ci-cd-pipeline]], [[devops/concepts/gitops]], [[devops/concepts/deployment-strategies]]

## What it is

**CI (Continuous Integration):** every code commit triggers an automated build and test. Goal: detect integration problems immediately.

**CD (Continuous Delivery):** every passing build produces a deployable artifact. Deployment to production requires manual approval.

**CD (Continuous Deployment):** every passing build deploys to production automatically — no manual gate.

> Most companies practice Continuous Delivery, not Deployment. Know which one you mean when you say "CD."

---

## The Pipeline Stages

```
Commit
  │
  ▼
[Lint + Static Analysis]   ← fast, < 1 min — catch style + obvious bugs
  │
  ▼
[Unit Tests]               ← fast, isolated, no I/O
  │
  ▼
[Build + Package]          ← compile, Docker build, tag with git SHA
  │
  ▼
[Integration Tests]        ← real DB, real queues — slower
  │
  ▼
[Security Scan]            ← SAST (Snyk, Trivy), dependency check
  │
  ▼
[Staging Deploy]           ← deploy to staging env, smoke tests
  │
  ▼
[E2E / Contract Tests]     ← full user journeys, API contract verification
  │
  ▼
[Manual Gate / Approval]   ← optional for CD (required for Delivery)
  │
  ▼
[Production Deploy]        ← canary or blue-green
  │
  ▼
[Post-deploy Monitoring]   ← watch error rate, latency for N minutes; auto-rollback
```

---

## Branching Strategies

| Strategy | How | Best for |
|---|---|---|
| **Trunk-based development** | All devs commit to `main`; use feature flags; short-lived branches (< 1 day) | High-velocity teams, FAANG |
| **GitFlow** | `main` + `develop` + `feature/*` + `release/*` + `hotfix/*` | Scheduled releases, slower cadence |
| **GitHub Flow** | `main` + short feature branches; merge via PR; deploy on merge | Most common; good balance |
| **Feature flags** | Code ships disabled; flag enables for % of users | Decouples deploy from release |

> Interview Q: *"What's trunk-based development and why do elite teams prefer it?"*  
> Answer: Eliminates long-lived merge hell; forces small changes; feature flags replace branch isolation; faster DORA metrics.

---

## Artifact Management

- Every build produces an **immutable artifact** tagged with the git commit SHA
- Artifacts stored in a registry (Docker Hub, ECR, Artifactory, GCS)
- Never rebuild — promote the same artifact from staging to prod
- **Why immutable?** Guarantees what you tested is exactly what you deploy

```
build tag:   my-app:abc1234   (git SHA)
staging tag: my-app:abc1234   (same image, different env config)
prod tag:    my-app:abc1234   (same image — promoted, not rebuilt)
```

---

## Pipeline Tools Comparison

| Tool | Model | Best for |
|---|---|---|
| **GitHub Actions** | YAML in `.github/workflows/`; GitHub-native | Most teams using GitHub; good ecosystem |
| **GitLab CI** | `.gitlab-ci.yml`; built-in registry, environments | GitLab users; powerful environments |
| **Jenkins** | Groovy Pipelines; self-hosted; huge plugin ecosystem | Legacy enterprise; highly customizable |
| **CircleCI** | YAML; fast, good caching | Startups; fast setup |
| **Tekton** | Kubernetes-native; CRDs | K8s-first platforms; GitOps pipelines |
| **Argo Workflows** | K8s-native DAG; good for ML pipelines | Data / ML CI/CD |

---

## Key CI/CD Metrics

| Metric | What it measures | Target |
|---|---|---|
| **Pipeline duration** | Total time from commit to deployable artifact | < 10 min for CI |
| **Test flakiness rate** | % of pipeline failures not caused by real bugs | < 1% |
| **Mean time to merge** | How long PRs sit open | < 1 day |
| **Deployment frequency** | How often you ship to prod | Daily or more |
| **Change failure rate** | % of deploys causing incidents | < 5% |

---

## Common Interview Q&A

**Q: How do you handle database migrations in CI/CD?**  
A: Run migrations before deploying new code (expand-contract pattern). New code must be backward-compatible with the old schema. Never drop columns in the same deploy that removes their code usage.

**Q: How do you test a pipeline without running the full thing?**  
A: Local runners (act for GitHub Actions), dry-run modes, mock the deploy step, run against a feature branch environment.

**Q: What if a test suite takes 30 minutes?**  
A: Parallelize across shards; split by test type; cache dependencies; run fast tests first; fail fast on linting before running slow tests.

## Sources
- [[devops/overview]]
