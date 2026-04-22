# Merge

**Topic:** [[git/topics/branching-and-merging]]
**Related:** [[git/concepts/rebase]], [[git/concepts/branch]], [[git/concepts/cherry-pick]]

## What it is
Merge combines the changes from two branches, preserving the full commit history of both. It produces a **merge commit** with two parent pointers (except in fast-forward cases).

## How it works

### Fast-Forward Merge
When the current branch has no commits the other branch doesn't have — git just moves the pointer. No merge commit.

```bash
git merge feature/login      # fast-forwards if possible
git merge --no-ff feature/login   # force a merge commit even if FF possible
```

### Three-Way Merge
Git finds the **common ancestor**, then computes two diffs: ancestor→ours and ancestor→theirs. Changes that don't conflict are auto-merged. Conflicts require manual resolution.

```
  base (ancestor)
  /            \
ours           theirs
  \            /
   merge commit
```

## Merge Strategies (advanced)

```bash
git merge -s ours feature    # keep our version entirely (theirs discarded)
git merge -s recursive -X theirs feature   # prefer their changes on conflict
git merge -s recursive -X ours feature     # prefer our changes on conflict
```

## Complexity
O(n) where n = changed files. Conflict detection is O(file size).

## When to use
- Merging feature branches into main (especially for audit trail)
- Preserving contributor history in open-source projects

## Common interview angles
- "Fast-forward vs. 3-way merge — what determines which one happens?" (whether branches have diverged)
- "How do you merge without fast-forwarding?" (`--no-ff` flag)
- "What does `git merge --squash` do?" (stages all changes as one commit; you then commit it; no parent link to feature)
- "How do you abort a merge in progress?" (`git merge --abort`)
- "What is ORIG_HEAD?" (saved before a merge/rebase; lets you undo with `git reset --hard ORIG_HEAD`)

## Examples

```bash
git merge feature/login                 # standard merge
git merge --no-ff feature/login         # always create merge commit
git merge --squash feature/login        # collapse changes, no history link
git merge --abort                       # abort ongoing merge

# Undo a bad merge
git reset --hard ORIG_HEAD              # before pushing
git revert -m 1 <merge-commit-sha>     # after pushing (safe)
```

## Sources
- [[git/sources/pro-git]]
