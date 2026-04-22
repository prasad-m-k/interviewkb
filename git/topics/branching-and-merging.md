# Branching and Merging

**Related:** [[git/concepts/branch]], [[git/concepts/merge]], [[git/concepts/rebase]], [[git/topics/workflows]]

## Branch Mechanics

A branch is a file containing a 40-char SHA. Creating a branch is instant and uses no extra storage for the history.

```bash
git branch feature/login     # create
git switch feature/login     # move HEAD
git switch -c feature/login  # create + move (shorthand)
```

## Merge Strategies

### Fast-Forward
When the target branch has no new commits since the branch diverged — git just moves the pointer.

```
Before:         After:
main            main/feature
  ↓               ↓
  A ← B ← C      A ← B ← C
```

### 3-Way Merge
When both branches have diverged. Git finds the common ancestor and produces a merge commit with two parents.

```
      feature
        ↓
A ← B ← D
 ↖       ↖
  C ← E ← M  ← main (merge commit)
```

Force a merge commit even on fast-forward: `git merge --no-ff`

### Squash Merge
All feature commits collapsed into one before merging. Good for keeping main history clean.

```bash
git merge --squash feature/login && git commit
```

## Rebase

Rebase replays commits from the current branch on top of another. Each commit gets a new SHA.

```bash
git rebase main          # replay current branch on top of main
git rebase -i HEAD~3     # interactively rewrite last 3 commits
```

Interactive rebase actions:
| Action | Effect |
|--------|--------|
| `pick` | keep commit as-is |
| `reword` | change commit message |
| `edit` | pause and amend the commit |
| `squash` | melt into previous commit (keep both messages) |
| `fixup` | melt into previous commit (discard this message) |
| `drop` | delete the commit |

## The Golden Rule

> **Never rebase commits that have been pushed to a shared branch.**

Rebasing rewrites SHAs. Anyone who has fetched the old SHAs will have a diverged history that is painful to reconcile.

## Conflict Resolution

```bash
git merge feature/login
# CONFLICT (content): Merge conflict in src/auth.py
# Edit the file, remove conflict markers
git add src/auth.py
git merge --continue     # or: git commit
```

Conflict markers:
```
<<<<<<< HEAD
current branch version
=======
incoming branch version
>>>>>>> feature/login
```

Tools: `git mergetool`, `git checkout --ours/--theirs` for binary conflicts.

## rerere (Reuse Recorded Resolution)

```bash
git config rerere.enabled true
```

Git remembers how you resolved a conflict and replays that resolution automatically next time.

## Interview Angles
- When does fast-forward not apply? (diverged histories)
- Merge vs. rebase trade-off? (history clarity vs. true chronology)
- How do you resolve a merge conflict in a binary file?
- What is `git rerere` and when is it useful?

## Sources
- [[git/sources/pro-git]]
