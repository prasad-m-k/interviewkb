# Concept: CI/CD Pipeline

**Topic:** [[devops/topics/ci-cd]]
**Related:** [[devops/concepts/deployment-strategies]], [[devops/concepts/gitops]], [[devops/topics/security-devsecops]]

## What it is

A CI/CD pipeline is an automated sequence of stages that takes code from a developer's commit to a running production deployment. CI (Continuous Integration) validates and builds the code; CD (Continuous Delivery/Deployment) gets it to users.

```
Developer                CI                        CD
─────────                ──                        ──
git push     →    lint → test → build    →    deploy staging → verify → deploy prod
(triggers)        (per commit/PR)              (per merge to main)
```

**CI vs CD distinction:**
- **Continuous Integration:** merge frequently; automated tests run on every commit; find bugs early
- **Continuous Delivery:** every merge to main produces a releasable artifact; deploy to prod is a manual approval
- **Continuous Deployment:** every merge to main automatically deploys to prod; no human gate

## Pipeline stages

### Stage 1: Trigger
```yaml
# GitHub Actions — trigger on PR and merge to main
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

### Stage 2: Fast feedback (< 3 min)
```
Lint            → code style, unused imports, type errors
Unit tests      → fast, isolated, no external dependencies
SAST            → static application security testing (Semgrep, CodeQL)
Secret scan     → TruffleHog, detect-secrets
Build           → compile / build Docker image (don't push yet)
```

### Stage 3: Integration (< 10 min)
```
Integration tests   → test against real DB/cache (testcontainers or Docker Compose)
Contract tests      → Pact — verify API contracts between services
SCA                 → open source vulnerability scan (Snyk, Trivy)
Container scan      → scan the built image for CVEs
```

### Stage 4: Publish
```bash
# Tag image with git SHA (immutable, traceable)
docker build -t registry/app:${GITHUB_SHA} .
docker push registry/app:${GITHUB_SHA}

# Attach SBOM
syft registry/app:${GITHUB_SHA} -o spdx-json > sbom.json
cosign attach sbom --sbom sbom.json registry/app:${GITHUB_SHA}

# Sign the image
cosign sign registry/app:${GITHUB_SHA}
```

### Stage 5: Deploy
```
dev:       auto-deploy; no gates; fast feedback for developers
staging:   auto-deploy; smoke tests + E2E; gate before prod
prod:      canary (5% → 100%); auto-rollback on error rate; human approval optional
```

## GitHub Actions — full example

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.12"}
      - run: pip install -r requirements.txt
      - run: ruff check .
      - run: pytest --cov=app tests/unit/

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Update image tag in config repo
        run: |
          # Update helm values file → Argo CD detects and deploys
          gh repo clone org/config-repo
          cd config-repo
          sed -i "s|tag:.*|tag: ${{ github.sha }}|" helm/app/values-staging.yaml
          git commit -am "chore: bump app to ${{ github.sha }}"
          git push
```

## Artifact versioning

| Strategy | Format | Traceability |
|---|---|---|
| Git SHA | `app:abc1234f` | Exact commit; immutable |
| Semver | `app:1.4.2` | Human-readable; needs tagging discipline |
| Branch+build | `app:main-42` | Shows branch; not globally unique |

**Best practice:** git SHA as the primary tag; semver tag added at release cut. Never use `latest` in production.

## Pipeline metrics (DORA-aligned)

```
Lead time for changes   = time from first commit to prod deploy
CI duration             = time from push to artifact published
Deploy frequency        = how often you successfully deploy to prod
Change failure rate     = % of deploys requiring rollback/hotfix
```

## Branching strategies and CI impact

**Trunk-based development (recommended for CD):**
- All developers merge to `main` daily
- Short-lived feature branches (< 1 day)
- Feature flags hide incomplete features
- CI runs on every commit to main; every green build is deployable

**GitFlow (for release management):**
- `main`, `develop`, `feature/*`, `release/*`, `hotfix/*`
- More complex; longer-lived branches → harder to integrate
- Better for: scheduled releases, multiple supported versions

## Common failures and fixes

| Failure | Cause | Fix |
|---|---|---|
| Flaky tests | Non-deterministic tests polluting results | Quarantine flaky tests; fix root cause |
| Slow pipeline | Sequential stages, cold build agents | Parallelize; cache dependencies; warm runners |
| Build succeeds, deploy fails | Environment config mismatch | Smoke tests in staging; parity between envs |
| Secret in logs | Secret echoed in a CI step | Use masked variables; never echo secrets |
| Image not immutable | Using `latest` tag | Always tag with git SHA |

## Sources
- [[devops/topics/ci-cd]]
- [[devops/concepts/deployment-strategies]]
- [[devops/concepts/gitops]]
- [[devops/scenarios/ci-cd-design-interview]]
