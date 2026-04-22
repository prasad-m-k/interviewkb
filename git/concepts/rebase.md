# Rebase

**Topic:** [[git/topics/branching-and-merging]]
**Related:** [[git/concepts/merge]], [[git/concepts/commit]], [[git/patterns/squash-and-merge]]

## What it is
Rebase takes a sequence of commits and replays them on top of a different base commit. Each replayed commit gets a **new SHA** because its parent changes. The result is a linear history as if the feature branch was started from the new base.

## How it works

```
Before rebase:
main:    A ── B ── C
                    ↑
feature: D ── E

git rebase main (from feature branch)

After rebase:
main:    A ── B ── C
                    ↑
feature:            D' ── E'   (new SHAs)
```

Git finds the common ancestor (B), temporarily removes D and E, fast-forwards feature to C, then replays D → D', E → E'.

## Interactive Rebase

```bash
git rebase -i HEAD~4    # rewrite last 4 commits interactively
```

Opens editor with:
```
pick a1b2c3 feat: add login form
pick d4e5f6 fix typo
pick 7890ab WIP: half-done validation
pick cdef12 feat: complete validation
```

Change `pick` to:
- `squash` / `s` — combine with previous, merge messages
- `fixup` / `f` — combine with previous, discard this message
- `reword` / `r` — keep commit, edit message
- `edit` / `e` — pause to amend
- `drop` / `d` — delete the commit

## Complexity
O(n) where n = number of commits being replayed.

## When to use
- Before opening a PR: `git rebase origin/main` to bring feature branch current
- Cleaning up local commit history before sharing
- Splitting a monolithic change into logical commits

## When NOT to use
- On branches others have already cloned/fetched (SHA rewrite breaks their history)
- On `main` or shared integration branches

## Common interview angles
- "Rebase vs. merge — when do you use each?" (rebase for local cleanup, merge for shared branches)
- "What is the golden rule of rebasing?" (never rebase public/shared history)
- "How do you abort a rebase in progress?" (`git rebase --abort`)
- "How do you continue after resolving a rebase conflict?" (`git add <file> && git rebase --continue`)
- "How do you squash the last N commits?" (`git rebase -i HEAD~N`, mark all but first as `fixup`)

## Examples

```bash
# Update feature branch with latest main
git fetch origin
git rebase origin/main

# Squash last 3 commits into one
git rebase -i HEAD~3

# Rebase onto a different branch entirely
git rebase --onto main feature/old-base feature/my-feature

# Abort a rebase
git rebase --abort

# Continue after conflict
git add resolved-file.py
git rebase --continue
```

## Sources
- [[git/sources/pro-git]]
