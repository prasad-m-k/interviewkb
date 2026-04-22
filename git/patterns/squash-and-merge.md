# Squash and Merge

**Topic:** [[git/topics/branching-and-merging]]
**Related concepts:** [[git/concepts/merge]], [[git/concepts/rebase]], [[git/concepts/commit]]

## What it solves
Keeps `main` history clean and atomic — one commit per feature, regardless of how many intermediate "WIP" commits exist on the feature branch. Each commit on main represents a complete, deployable unit of work.

## Template / Skeleton

```bash
# Via GitHub UI: "Squash and merge" button on PR

# Manually:
git switch main
git merge --squash feature/login
git commit -m "feat(auth): add login with email + password

Closes #42"

# Or via GitHub CLI:
gh pr merge 123 --squash --delete-branch
```

What happens to the feature branch? It's deleted after merge (convention). The squashed commit on main is a new commit with no parent link to the feature branch.

## Signal phrases
- "I want one commit per PR on main"
- "Our feature branches have lots of WIP commits"
- "We want to be able to revert a feature by reverting one commit"
- "Clean, bisectable main history"

## Complexity
O(1) additional commit. Feature branch history is not preserved in main (only in the branch/PR).

## Trade-offs

| Pros | Cons |
|------|------|
| Clean, readable main history | Feature branch commits lost after branch deleted |
| Easy to revert a whole feature | `git blame` only shows squash commit, not original author |
| Each main commit is deployable | Can't trace individual change back to original commit |
| Simple bisect on main | Contributors' individual commits not visible in main |

## When to prefer merge commit instead
- Open-source projects where individual contributions should be credited
- When forensic auditability of each step matters
- When the feature branch history is already clean

## Problems using this pattern
- [[git/scenarios/clean-up-history]]

## Sources
- [[git/sources/pro-git]]
