# Reflog

**Topic:** [[git/topics/core-concepts]]
**Related:** [[git/concepts/branch]], [[git/concepts/commit]], [[git/scenarios/revert-bad-commit]]

## What it is
The reflog (reference log) is a local, append-only journal that records every time HEAD or a branch pointer moves — including commits, checkouts, resets, rebases, merges, and cherry-picks. It is your primary safety net for recovering from "oops" moments.

## How it works

```
git reflog              # HEAD's reflog
git reflog show main    # main branch's reflog
```

Output:
```
abc123 (HEAD -> main) HEAD@{0}: commit: fix: resolve login bug
def456 HEAD@{1}: reset: moving to def456
789abc HEAD@{2}: commit: feat: add login form
...
```

Each `HEAD@{n}` is a reference you can use in any git command.

**Retention:** By default 90 days for reachable commits, 30 days for unreachable. Controlled by `gc.reflogExpire`.

## Complexity
O(log entries) for lookup; log itself is append-only.

## When to use
- Recovering commits lost after `git reset --hard`
- Finding the state before a bad rebase
- Recovering a deleted branch
- Understanding exactly what happened to your repo

## Common interview angles
- "You accidentally ran `git reset --hard HEAD~3`. How do you recover?" (git reflog to find the SHA, git reset --hard to it)
- "Is git reflog available on the remote?" (No — it's purely local, per-clone)
- "How long does git keep reflog entries?" (90 days by default, configurable)
- "What is the difference between reflog and git log?" (log shows commit DAG; reflog shows HEAD movement history including resets/rebases)

## Examples

```bash
git reflog                           # show HEAD movement history
git reflog show feature/login        # show branch movement history
git reflog --date=iso                # show with timestamps

# Recover commits lost after hard reset
git reflog                           # find the SHA before the reset
git reset --hard HEAD@{3}            # jump back

# Restore a deleted branch
git reflog                           # find last commit on deleted branch
git branch recovered-branch abc123   # recreate the branch

# Recover from bad rebase
git reflog                           # find pre-rebase HEAD
git reset --hard HEAD@{5}
```

## Sources
- [[git/sources/pro-git]]
