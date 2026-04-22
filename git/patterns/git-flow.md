# Git Flow

**Topic:** [[git/topics/workflows]]
**Related concepts:** [[git/concepts/branch]], [[git/concepts/merge]], [[git/concepts/cherry-pick]]

## What it solves
Provides a structured branching model for projects with scheduled, versioned releases. Separates feature work, integration, and release preparation into dedicated branch types.

## Branch Types

| Branch | Base | Merges into | Purpose |
|--------|------|------------|---------|
| `main` | — | — | Production code; only touched by release/hotfix |
| `develop` | main | — | Integration branch; next release |
| `feature/*` | develop | develop | New features |
| `release/*` | develop | main + develop | Release prep (bumping versions, final bugfixes) |
| `hotfix/*` | main | main + develop | Emergency production fixes |

## Template / Skeleton

```bash
# Feature
git switch develop
git switch -c feature/user-auth
# ... work ...
git switch develop
git merge --no-ff feature/user-auth
git branch -d feature/user-auth

# Release
git switch develop
git switch -c release/1.2.0
# bump version, final fixes
git switch main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release 1.2.0"
git switch develop
git merge --no-ff release/1.2.0
git branch -d release/1.2.0

# Hotfix
git switch main
git switch -c hotfix/fix-login-crash
# fix the bug
git switch main
git merge --no-ff hotfix/fix-login-crash
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git switch develop
git merge --no-ff hotfix/fix-login-crash
git branch -d hotfix/fix-login-crash
```

## Signal phrases
- "We have quarterly releases"
- "Multiple versions in production simultaneously"
- "We need a clear audit trail for each release"
- "Mobile / desktop app with app store review cycle"

## Complexity
High operational overhead. Many merge ceremonies. Suitable when the release cadence truly demands it.

## Problems using this pattern
- [[git/scenarios/hotfix-production]]

## Criticism
- Overkill for continuous deployment
- `develop` branch creates merge debt
- Trunk-based development is simpler and more compatible with CI/CD

## Sources
- [[git/sources/pro-git]]
