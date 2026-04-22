# Stash

**Topic:** [[git/topics/core-concepts]]
**Related:** [[git/concepts/staging-area]], [[git/concepts/branch]]

## What it is
`git stash` saves your dirty working tree (modified tracked files and optionally untracked/ignored files) onto a stack, reverting the working tree to a clean state matching HEAD. You can pop the stash later to restore the work.

## How it works

Stash stores a special commit in `.git/refs/stash`. Each stash entry is a pair of commits: one for the index, one for the working tree.

```
Before stash:            After stash:
working tree: modified   working tree: clean (matches HEAD)
index: staged changes    stash stack: [stash@{0}: WIP on feature/login]
```

## Complexity
O(changed files).

## When to use
- Need to switch branches but aren't ready to commit
- Quick context switch for an urgent bugfix
- Pulling latest changes when you have uncommitted work

## Common interview angles
- "How is `git stash` different from committing?" (stash doesn't advance branch history; stash entries are outside normal DAG)
- "How do you stash untracked files?" (`git stash -u` or `--include-untracked`)
- "How do you apply a stash without removing it from the stack?" (`git stash apply` vs. `git stash pop`)
- "What happens to a stash if you delete the branch?" (stash is branch-independent; safe)
- "Can two people share a stash?" (No — stash is local to your repo)

## Examples

```bash
git stash                          # stash tracked changes
git stash -u                       # include untracked files
git stash push -m "WIP: login form" # named stash

git stash list                     # show all stash entries
git stash show stash@{1}           # show diff of specific stash
git stash show -p stash@{0}        # show patch

git stash pop                      # apply most recent + remove from stack
git stash apply stash@{2}          # apply specific stash (keep on stack)
git stash drop stash@{0}           # remove specific stash
git stash clear                    # remove all stashes

# Create a branch from a stash
git stash branch feature/rescued stash@{0}
```

## Sources
- [[git/sources/pro-git]]
