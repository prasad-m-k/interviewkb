# Scenario: Hotfix Production

**Pattern:** [[git/patterns/git-flow]]
**Related:** [[git/concepts/cherry-pick]], [[git/concepts/tags]], [[git/scenarios/interview-questions]]

## The Scenario
Production is broken. You have unfinished feature work in progress on `develop` or a feature branch. You need to fix production immediately without shipping the half-done features.

## Approach (GitHub Flow variant)

```bash
# 1. Identify the last production release tag
git log --oneline --tags --no-walk

# 2. Create hotfix branch from main (not develop)
git switch main
git pull origin main
git switch -c hotfix/fix-payment-crash

# 3. Fix the bug
# ... edit files ...
git add -p
git commit -m "fix: prevent null pointer in payment processor

Payment processor crashed when user had no saved cards.
Closes #217"

# 4. Test thoroughly on this branch (CI runs)
git push origin hotfix/fix-payment-crash

# 5. Merge to main and tag
git switch main
git merge --no-ff hotfix/fix-payment-crash
git tag -a v1.2.1 -m "Hotfix: prevent payment crash"
git push origin main --tags

# 6. Bring the fix into develop/feature branches
git switch develop
git merge --no-ff hotfix/fix-payment-crash
# or cherry-pick if develop has diverged significantly:
git cherry-pick <hotfix-commit-sha>

# 7. Delete hotfix branch
git push origin --delete hotfix/fix-payment-crash
git branch -d hotfix/fix-payment-crash
```

## Key Decisions

- **Always branch from main** — not develop. Develop may contain unreleased features you don't want to ship.
- **Tag the release** — creates an auditable marker and triggers the deploy pipeline.
- **Backport to develop** — if you forget this step, the fix will be overwritten on the next release from develop.
- **Use cherry-pick when develop has diverged** — safer than a merge that might bring unintended changes.

## CI/CD Integration

```yaml
# .github/workflows/deploy.yml
on:
  push:
    tags:
      - 'v*'
jobs:
  deploy-prod:
    steps:
      - uses: actions/checkout@v4
      - run: make deploy-prod
```

Tag push automatically triggers the production deployment.

## Common Mistakes
- Branching hotfix from `develop` — ships unreleased features
- Forgetting to merge hotfix back into develop — fix regresses on next release
- Not tagging — pipeline doesn't trigger; audit trail broken
- Not communicating with team — others may rebase on stale main

## Interview Answer Framework
> "I'd branch from main — specifically from the tag we deployed — fix only the bug, merge back to main, tag a new release, deploy, then cherry-pick or merge the fix into develop so it's not lost."

## Sources
- [[git/sources/pro-git]]
