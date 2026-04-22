# Git in CI/CD Pipelines

**Related:** [[git/cicd/overview]], [[git/cicd/github-actions]], [[git/cicd/branching-strategies]], [[git/topics/workflows]]

## How Git Events Drive Pipelines

CI/CD systems listen to Git events (webhooks) to trigger automation:

| Git Event | Typical Pipeline Trigger |
|-----------|------------------------|
| `push` to feature branch | Run unit tests, lint |
| PR opened / updated | Full test suite, static analysis, preview deploy |
| PR merged to main | Build artifact, integration tests, staging deploy |
| Tag pushed (`v*`) | Build release artifact, production deploy |
| PR closed (not merged) | Cleanup preview environments |

## Branch Protection + Status Checks

The link between Git and CI/CD quality gates:

```
PR opened
    │
    ├── CI triggered (GitHub Actions, Jenkins, CircleCI...)
    │       ├── lint
    │       ├── unit tests
    │       ├── build
    │       └── report status back to GitHub via Commit Status API
    │
    ├── Branch protection rules enforce:
    │       ├── All status checks must be green ✓
    │       └── N approvals required ✓
    │
    └── Merge button unlocks
```

## Deployment Strategies Tied to Git

| Strategy | Git Trigger | Notes |
|----------|------------|-------|
| Continuous Deployment | Push to main | Every merge auto-deploys |
| Manual Promotion | Workflow dispatch or tag | Human approves prod |
| Environment Branches | Push to `staging`, `prod` branches | Anti-pattern; avoid |
| Tag-based Releases | `git tag v1.2.3 && git push --tags` | Clean and auditable |

## Artifact Versioning

Best practice: tag artifacts with the Git SHA.

```bash
IMAGE_TAG=$(git rev-parse --short HEAD)
docker build -t myapp:${IMAGE_TAG} .
docker push myapp:${IMAGE_TAG}
```

This makes every deployment fully traceable to source code.

## Git in GitHub Actions

```yaml
# .github/workflows/ci.yml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0        # full history for git describe / bisect
      - run: make test
      - run: make lint
```

## Shallow Clones in CI

CI systems often use shallow clones to speed up checkout:

```bash
git clone --depth=1 https://github.com/org/repo.git
```

Tradeoffs:
- Faster: no full history download
- Breaks: `git describe`, `git bisect`, `git log --follow`
- `fetch-depth: 0` in Actions restores full history when needed

## Secrets and Git

**Never commit secrets.** Mitigations:
- `.gitignore` — prevent accidental add
- `git-secrets` / `gitleaks` — pre-commit hook scanning
- GitHub Secret Scanning — alerts on pushed secrets
- If committed: rotate the secret, use `git filter-repo` to scrub history, force-push (coordinate with team)

```bash
# Scrub a file from all history
git filter-repo --path secrets.env --invert-paths
```

## Interview Angles
- How do you tie a Docker image back to the exact source code that built it?
- What is the difference between a status check and a branch protection rule?
- Why are shallow clones common in CI and what breaks with them?
- How would you handle an accidentally committed secret?
- What Git event triggers a production deployment in your ideal pipeline?

## Sources
- [[git/sources/pro-git]]
- [[git/cicd/github-actions]]
