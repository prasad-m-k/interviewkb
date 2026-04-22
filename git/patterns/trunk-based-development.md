# Trunk-Based Development

**Topic:** [[git/topics/workflows]]
**Related concepts:** [[git/concepts/branch]], [[git/concepts/commit]], [[git/concepts/hooks]]

## What it solves
Eliminates long-lived branches and the associated merge pain. Enables true continuous integration by ensuring everyone's code integrates at least daily.

## Core Rules

1. Everyone commits to `main` (the trunk) — or uses branches that live < 1–2 days
2. Main is **always** releasable
3. Use feature flags to hide incomplete features in production
4. CI must be fast (< 10 minutes) and always green
5. Pair programming or very small PRs reduce the need for long review cycles

## Template / Skeleton

```bash
# Short-lived branch workflow (< 1 day)
git switch -c feature/add-header    # branch off main
# work for a few hours
git push origin feature/add-header
gh pr create                        # open PR, get quick review
# CI passes → merge (squash or rebase)
git switch main && git pull
git branch -d feature/add-header

# Feature flag pattern (incomplete work on main)
if feature_flags.is_enabled("new_checkout", user):
    return new_checkout_flow()
else:
    return old_checkout_flow()
```

## Signal phrases
- "We deploy multiple times a day"
- "We want to eliminate merge conflicts"
- "We practice continuous integration strictly"
- "Our CI suite runs in under 10 minutes"

## Complexity
Time per integration: O(1) — no long-lived divergence.

Requires investment in:
- Fast, reliable test suite
- Feature flag infrastructure
- Monitoring for quick rollback

## Comparison with Git Flow

| | TBD | Git Flow |
|--|-----|---------|
| Branch lifetime | Hours | Days–weeks |
| Release cadence | Continuous | Scheduled |
| Merge conflicts | Minimal | Frequent |
| CI requirement | High | Low |
| Feature flags needed | Yes | No |

## Sources
- [[git/sources/pro-git]]
