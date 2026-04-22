# Staging Area (Index)

**Topic:** [[git/topics/core-concepts]]
**Related:** [[git/concepts/commit]], [[git/concepts/branch]]

## What it is
The staging area (also called the **index**) is a binary file (`.git/index`) that tracks the proposed next commit. It sits between the working tree and the repository, giving you fine-grained control over what goes into each commit.

## How it works

The three-tree model in practice:

```
Working Tree    ──git add──▶    Index    ──git commit──▶    HEAD
     ↑                            ↑                           ↑
 (files on disk)           (next commit)              (last commit)

git restore <file>          ← discard working tree changes
git restore --staged <file> ← unstage (keep working tree change)
git reset HEAD~1            ← uncommit (move HEAD back, keep index)
git reset --hard HEAD~1     ← uncommit + discard everything
```

## Partial Staging

```bash
git add -p                  # interactively stage hunks
# Options per hunk:
# y = stage this hunk
# n = skip this hunk
# s = split into smaller hunks
# e = manually edit the hunk
```

This is the killer feature of the index: you can commit part of a file's changes while keeping other changes unstaged.

## Common interview angles
- "What does `git add -p` do?" (interactively stage parts of a file — lets you craft precise commits)
- "Difference between `git reset` and `git restore`?" (reset moves branch/HEAD; restore changes files in working tree or index)
- "How do you unstage a file without losing the changes?" (`git restore --staged <file>`)
- "What is the difference between `git reset --soft`, `--mixed`, `--hard`?" 

  | Mode | HEAD | Index | Working Tree |
  |------|------|-------|--------------|
  | `--soft` | moves | unchanged | unchanged |
  | `--mixed` (default) | moves | reset | unchanged |
  | `--hard` | moves | reset | reset |

## Examples

```bash
git add file.py             # stage a file
git add -p                  # interactively stage hunks
git add -A                  # stage all changes (new + modified + deleted)
git add .                   # stage all in current directory

git status                  # see staged vs. unstaged changes
git diff                    # working tree vs. index
git diff --cached           # index vs. HEAD (what will be committed)

git restore --staged file.py      # unstage (modern syntax)
git reset HEAD file.py            # unstage (older syntax)

git restore file.py               # discard working tree changes
git checkout -- file.py           # same (older syntax)
```

## Sources
- [[git/sources/pro-git]]
