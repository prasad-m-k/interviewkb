# GitHub Actions for Git Workflows

**Related:** [[git/cicd/overview]], [[git/topics/ci-cd]], [[git/concepts/hooks]]

## Core Concepts

- **Workflow** — YAML file in `.github/workflows/`
- **Event (trigger)** — what causes the workflow to run
- **Job** — a unit of work running on a runner
- **Step** — individual command or Action within a job
- **Action** — reusable step from Marketplace or local `action.yml`
- **Runner** — VM executing jobs (`ubuntu-latest`, `macos-latest`, `windows-latest`)

## Common Triggers

```yaml
on:
  push:
    branches: [main, 'release/**']
    tags: ['v*']
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
  workflow_dispatch:          # manual trigger
    inputs:
      environment:
        type: choice
        options: [staging, prod]
  schedule:
    - cron: '0 2 * * 1'      # every Monday at 2am UTC
```

## CI Workflow — Full Example

```yaml
# .github/workflows/ci.yml
name: CI

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
          fetch-depth: 0          # full history for git describe

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'

      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: mypy src/
      - run: pytest --cov=src tests/

      - uses: codecov/codecov-action@v4

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: |
          SHA=$(git rev-parse --short HEAD)
          docker build -t myapp:${SHA} .
          echo "IMAGE_TAG=${SHA}" >> $GITHUB_OUTPUT
        id: build
      - run: docker push myapp:${{ steps.build.outputs.IMAGE_TAG }}
```

## Release Workflow (Tag-Based)

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: ['v*.*.*']

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write    # required to create GitHub Release
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: make build
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: dist/*
          generate_release_notes: true
```

## Branch Protection + Required Status Checks

In GitHub repo settings → Branches → Branch protection rules:

```
✅ Require status checks to pass before merging
   ✅ test (the job name from CI workflow)
   ✅ build
✅ Require pull request reviews before merging
   Number of reviewers: 1
✅ Require branches to be up to date before merging
✅ Do not allow bypassing the above settings
```

This creates the quality gate: CI must pass and reviews must be approved before the Merge button activates.

## Useful Patterns

### Conditional Steps

```yaml
- name: Deploy to prod
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: make deploy-prod
```

### Monorepo Selective Testing

```yaml
- name: Detect changed services
  id: changes
  run: |
    CHANGED=$(git diff --name-only origin/main...HEAD)
    echo "auth=$(echo "$CHANGED" | grep -q '^services/auth' && echo true || echo false)" >> $GITHUB_OUTPUT

- name: Test auth service
  if: steps.changes.outputs.auth == 'true'
  run: cd services/auth && pytest
```

### Reusable Workflows

```yaml
# .github/workflows/deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string

jobs:
  deploy:
    environment: ${{ inputs.environment }}
    runs-on: ubuntu-latest
    steps:
      - run: make deploy ENV=${{ inputs.environment }}
```

## Secrets Management

```yaml
steps:
  - run: docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
  - run: ./deploy.sh
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

Secrets are never printed to logs. Repository secrets → Organization secrets → Environment secrets (with approval gates).

## Sources
- [[git/sources/pro-git]]
- [[git/cicd/overview]]
