# Scenario: Revert a Bad Commit

**Related:** [[git/concepts/reflog]], [[git/concepts/merge]], [[git/scenarios/interview-questions]]

## The Core Question
> "You deployed a bad commit to production. How do you roll it back?"

The answer depends on whether the commit is local-only or already shared/deployed.

---

## Case 1: Local-only (not yet pushed)

```bash
# Undo last commit, keep changes in working tree
git reset --mixed HEAD~1

# Undo last commit, keep changes staged
git reset --soft HEAD~1

# Undo last commit, discard all changes
git reset --hard HEAD~1
```

## Case 2: Already pushed to a shared branch

**Use `git revert`** — it creates a new commit that undoes the bad commit. History is preserved; no force-push needed.

```bash
git revert abc123              # creates new "revert" commit
git push origin main
```

For a merge commit:
```bash
git revert -m 1 <merge-commit-sha>
# -m 1 = "keep mainline parent 1" (the branch you merged into)
```

## Case 3: Revert a range of commits

```bash
git revert --no-commit abc123..def456    # revert range, stage changes
git commit -m "revert: roll back broken login changes"
```

`--no-commit` is important for ranges — avoids one revert commit per commit.

## Case 4: Hard rollback of main (dangerous — requires force-push)

Only when the bad commits contain secrets, illegal content, or you have explicit team agreement:

```bash
git reset --hard <good-commit-sha>
git push --force-with-lease origin main
# All team members must: git fetch && git reset --hard origin/main
```

---

## Decision Tree

```
Is the commit pushed to a shared branch?
    │
    ├── No → git reset (--soft/--mixed/--hard)
    │
    └── Yes
          │
          ├── Is it safe to rewrite history? (no one else has it, or team agrees)
          │       └── Yes → git reset + --force-with-lease (coordinate with team)
          │
          └── No (others have pulled it)
                  └── git revert (safe, preserves history)
```

## revert vs. reset — Summary

| | `git revert` | `git reset` |
|--|-------------|------------|
| Creates new commit | Yes | No |
| Rewrites history | No | Yes |
| Safe for shared branches | Yes | No |
| Leaves history clean | No (shows revert) | Yes |
| Recoverable | Always | Via reflog (local only) |

## Key Interview Point
> "`git revert` is always the right answer for shared branches. It's explicit, auditable, and doesn't require a force-push."

## Sources
- [[git/sources/pro-git]]
