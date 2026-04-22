# Git in CI/CD — Overview

**Related:** [[git/topics/ci-cd]], [[git/cicd/github-actions]], [[git/cicd/branching-strategies]]

## The Core Idea

CI/CD systems are Git event consumers. Every meaningful Git event (push, PR, tag) triggers a pipeline stage.

```
Developer pushes code
        │
        ▼
Git Server (GitHub)
        │ webhook
        ▼
CI System (GitHub Actions, Jenkins, CircleCI, GitLab CI...)
        │
        ├── Build
        ├── Test
        ├── Static Analysis / SAST
        ├── Publish Artifact (with Git SHA tag)
        └── Deploy (based on branch/tag rules)
```

## Git Events → Pipeline Mapping

| Git Event | Typical Stage | Purpose |
|-----------|--------------|---------|
| Push to feature branch | Fast CI | Lint + unit tests only |
| PR opened/updated | Full CI | All tests + coverage + preview deploy |
| Push to main | Build + deploy to staging | Integration validation |
| Annotated tag push | Release pipeline | Build artifact + prod deploy |
| PR closed (merged) | Cleanup | Delete preview environment |
| Manual dispatch | On-demand | Release to prod with approval |

## Artifact Immutability

Build once, promote through environments:

```
Git SHA abc123
    │
    ▼
Docker image: myapp:abc123  ← built from this exact commit
    │
    ├── deployed to dev     (automatic)
    ├── deployed to staging (automatic or triggered)
    └── deployed to prod    (manual approval gate)
```

Never rebuild the artifact per environment — promotes exactly what was tested.

## GitOps (Git as Source of Truth for Deployments)

In GitOps, the desired state of infrastructure/deployments is stored in Git. A reconciliation agent (ArgoCD, Flux) ensures the live environment matches the Git state.

```
Application Repo      Config/Infra Repo       Cluster
     │                       │                    │
     │ image build →         │                    │
     │ update image tag →    │                    │
     └──────────────────────▶│ PR: bump to abc123 │
                             │ merge →             │
                             └─── ArgoCD syncs ──▶│
```

Benefits: full audit trail, rollback = `git revert`, PR review for infra changes.

## Key Git Operations in CI Systems

```bash
# Clone (shallow for speed)
git clone --depth=1 --branch main https://github.com/org/repo.git

# Get current SHA for artifact tagging
git rev-parse HEAD
git rev-parse --short HEAD   # 8-char short SHA

# Get tag for semantic version
git describe --tags --always

# Check what changed (for monorepo selective testing)
git diff --name-only origin/main...HEAD

# Fetch full history when needed (git describe, bisect)
git fetch --unshallow
git fetch --tags
```

## Sources
- [[git/sources/pro-git]]
- [[git/cicd/github-actions]]
