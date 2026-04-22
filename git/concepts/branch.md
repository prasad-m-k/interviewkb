# Branch

**Topic:** [[git/topics/core-concepts]], [[git/topics/branching-and-merging]]
**Related:** [[git/concepts/commit]], [[git/concepts/merge]], [[git/concepts/rebase]]

## What it is
A branch is a lightweight, movable pointer to a commit. It is literally a file in `.git/refs/heads/` containing the 40-character SHA of the tip commit. Creating or deleting a branch is instantaneous and costs negligible storage.

## How it works

```
main ──→ A ──→ B ──→ C   (HEAD points to main)
                ↑
           feature/x ──→ D ──→ E
```

`HEAD` points to the current branch. When you commit, the current branch pointer advances to the new commit.

**Detached HEAD:** HEAD points directly to a SHA instead of a branch. Any commits made are "floating" — not referenced by any branch. They will be garbage collected unless you create a branch.

```bash
git checkout abc123      # enters detached HEAD state
git switch -c recovery   # rescue work by creating a branch
```

## Complexity
Create/delete: O(1). Listing: O(branches).

## When to use
- One branch per feature/bugfix/experiment
- Delete branches promptly after merging to reduce noise

## Common interview angles
- "What is the difference between HEAD and main?" (HEAD is where you are; main is a branch pointer)
- "What happens to commits on a deleted branch?" (become dangling; recoverable via reflog for ~30 days)
- "How do you rename a branch?" (`git branch -m old new` locally; delete + push new on remote)
- "How do you list all remote branches?" (`git branch -r` or `git branch -a`)
- "What is a tracking branch?" (local branch configured to follow a remote branch; enables `git pull` without args)

## Examples

```bash
git branch                          # list local branches
git branch -a                       # list local + remote-tracking
git branch feature/login            # create
git switch -c feature/login         # create + switch
git branch -d feature/login         # delete (safe — won't delete unmerged)
git branch -D feature/login         # force delete
git branch -m old-name new-name     # rename

# Set upstream tracking
git branch --set-upstream-to=origin/main main

# See which branches are merged into main
git branch --merged main
git branch --no-merged main
```

## Sources
- [[git/sources/pro-git]]
