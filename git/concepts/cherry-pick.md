# Cherry-Pick

**Topic:** [[git/topics/branching-and-merging]]
**Related:** [[git/concepts/commit]], [[git/concepts/rebase]], [[git/concepts/merge]]

## What it is
`git cherry-pick` applies the changes introduced by one or more specific commits onto the current branch, creating new commits with new SHAs. Unlike merge/rebase, it lets you selectively port individual commits.

## How it works

```
main:    A ── B ── C ── D
                         ↑ HEAD

release/1.x:  A ── X ── Y

# Cherry-pick C (a bugfix) onto release/1.x
git cherry-pick C

release/1.x:  A ── X ── Y ── C'   (new SHA, same diff as C)
```

Git applies the diff from C's parent to C on top of Y.

## Complexity
O(changed files in selected commit). Conflicts possible, same as merge.

## When to use
- Backporting a bugfix to a maintenance/release branch
- Porting a single fix from one feature branch to another without merging everything
- Salvaging a good commit from an abandoned branch

## When NOT to use
- Moving large groups of commits (use rebase instead)
- If the full branch history matters (cherry-pick loses parent context)

## Common interview angles
- "How is cherry-pick different from merge?" (cherry-pick copies individual commits; merge brings entire diverged history)
- "Does cherry-pick create a duplicate commit?" (yes — new SHA, same change; can cause confusion if both branches later merge)
- "How do you cherry-pick multiple commits?" (`git cherry-pick A..B` for a range, or `git cherry-pick A B C` for specific SHAs)
- "How do you resolve a cherry-pick conflict?" (resolve files, `git add`, `git cherry-pick --continue`)

## Examples

```bash
git cherry-pick abc123                  # apply one commit
git cherry-pick abc123 def456           # apply two specific commits
git cherry-pick abc123..def456          # apply range (exclusive of abc123)
git cherry-pick abc123^..def456         # apply range (inclusive of abc123)

# Don't auto-commit (stage only)
git cherry-pick --no-commit abc123

# Abort in-progress cherry-pick
git cherry-pick --abort

# Continue after resolving conflict
git add resolved.py
git cherry-pick --continue
```

## Sources
- [[git/sources/pro-git]]
