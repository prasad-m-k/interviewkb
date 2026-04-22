# Branching Strategies for CI/CD

**Related:** [[git/cicd/overview]], [[git/topics/workflows]], [[git/patterns/git-flow]], [[git/patterns/trunk-based-development]]

## Aligning Branch Model to Deployment Strategy

Your branching model must match how and when you deploy. Mismatch causes either instability (deploying unreviewed work) or slowness (long merge queues).

## Strategy Comparison

| Strategy | Branch model | Deploy trigger | Environments |
|----------|-------------|---------------|-------------|
| Continuous Deployment | GitHub Flow / TBD | Every push to main | dev → staging → prod (auto) |
| Continuous Delivery | GitHub Flow | Push to main | dev → staging (auto), prod (manual) |
| Scheduled Release | Git Flow | Tag push | dev → staging → prod (manual promotion) |
| Multi-version Support | Release branches | Per-branch tag | Separate pipelines per version |

## Anti-Pattern: Environment Branches

```
# AVOID:
main → staging → prod

# Why it's bad:
# - main and prod diverge immediately after a prod-only fix
# - Forces a "merge ceremony" to promote: staging → prod → back-merge
# - Merge conflicts between environment branches are common
# - Audit trail is confusing: "what's in prod?" requires inspecting branch, not tags
```

## Recommended: Promote Artifacts, Not Branches

```
main (branch)
    │ CI builds image: myapp:abc1234
    │
    ├── Auto-deploy abc1234 → dev
    │
    ├── Auto-deploy abc1234 → staging
    │        │ integration tests pass
    │
    └── Manual approval → deploy abc1234 → prod
                              (same image, no rebuild)
```

The artifact (container, binary, package) is the unit of promotion. Git branch just triggers the initial build.

## Release Branches for Multi-Version

When you maintain multiple versions simultaneously:

```
main          (v3.x development)
release/2.x   (v2.x maintenance: bugfixes only)
release/1.x   (v1.x LTS: security fixes only)
```

```bash
# Fix in main, backport to release/2.x
git switch release/2.x
git cherry-pick <fix-sha>
git tag -a v2.3.1 -m "Bugfix release"
git push origin release/2.x --tags
```

Each release branch has its own deployment pipeline triggered by its own tags.

## Merge Queues (GitHub)

For high-velocity teams: the merge queue serializes merges to main, running CI on the combined state before merging.

```
PR #1 passes CI
PR #2 passes CI
             │
             ▼
Merge queue: combines PR#1 + PR#2, runs CI on combined diff
             │
             ▼
Both merge atomically — or fail without touching main
```

Prevents the "all green individually, broken together" problem.

## Deploy Freeze / Lock

During production incidents or release freeze periods:

```yaml
# GitHub environment with required reviewers
# Settings → Environments → prod → Required reviewers
# All deployments to prod require explicit approval from listed users
```

Or a simpler convention: branch protection rule preventing pushes to `main` tags except from a specific team.

## Sources
- [[git/sources/pro-git]]
- [[git/cicd/github-actions]]
