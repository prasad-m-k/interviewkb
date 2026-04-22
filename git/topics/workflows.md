# Git Workflows

**Related:** [[git/patterns/git-flow]], [[git/patterns/trunk-based-development]], [[git/topics/ci-cd]]

## GitHub Flow (recommended for most teams)

Simple workflow optimized for continuous deployment.

```
main ────────────────────────────────────▶
      ↖ feature/a ──── PR ──▶ merge
      ↖ feature/b ──── PR ──▶ merge
```

**Rules:**
1. `main` is always deployable.
2. Create a descriptive branch from main for every change.
3. Open a PR as early as possible (use Draft PRs for WIP).
4. CI must pass and get approval before merge.
5. Deploy immediately after merge.
6. Delete the feature branch after merge.

**Best for:** SaaS teams deploying multiple times a day.

## Git Flow

Structured model with dedicated branches per purpose. Popularized by Vincent Driessen (2010).

```
main     ─────────────────────────────●─────▶  (only release tags)
hotfix          ────▶ merge to main+develop
develop  ──────────────────────────────────▶
feature        ──▶ merge to develop
release                   ──▶ merge to main+develop
```

**Branches:**
| Branch | Purpose |
|--------|---------|
| `main` | Production-ready, tagged releases only |
| `develop` | Integration branch; what's in next release |
| `feature/*` | New features; branch from develop |
| `release/*` | Release prep; branch from develop, merge to main+develop |
| `hotfix/*` | Emergency production fixes; branch from main, merge to main+develop |

**Best for:** Libraries, versioned products with scheduled releases, mobile apps.

**Criticism:** Overhead is high for teams deploying continuously. Hotfix flow is complex.

## Trunk-Based Development (TBD)

Everyone commits to `main` (the "trunk") or uses very short-lived branches (< 1 day).

```
main ──●──●──●──●──●──▶   (every commit deployable)
         ↖short branch (< 24h)
```

**Requirements:**
- Strong CI (tests must run fast — under 10 min)
- Feature flags to hide incomplete work
- Pair programming or very small PRs
- High test coverage

**Best for:** High-velocity teams with mature CI/CD. Used by Google, Facebook.

## Release Branching

Used when multiple versions need maintenance simultaneously.

```
main        ──●──●──●──▶
release/1.x ──────────▶ (bugfixes only, cherry-picked from main)
release/2.x      ──────▶
```

```bash
git cherry-pick abc123   # port fix from main to release/1.x
```

## Choosing a Workflow

| Factor | GitHub Flow | Git Flow | TBD |
|--------|-------------|----------|-----|
| Deploy frequency | Multiple/day | Scheduled | Continuous |
| Team size | Any | Medium+ | Any |
| Versioned releases | No | Yes | No |
| CI maturity needed | Low | Low | High |
| Complexity | Low | High | Medium |

## Interview Angles
- Compare Git Flow vs. trunk-based development.
- How do feature flags enable trunk-based development?
- When would you keep a long-lived release branch?
- What branch would you cut a hotfix from in Git Flow?

## Sources
- [[git/sources/pro-git]]
