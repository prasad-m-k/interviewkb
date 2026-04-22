# Scenario: Clean Up Commit History Before Merging

**Patterns:** [[git/patterns/squash-and-merge]], [[git/patterns/conventional-commits]]
**Concepts:** [[git/concepts/rebase]], [[git/concepts/commit]]

## The Problem
Your feature branch has commits like:
```
wip
fix typo
more fixes
actually fix it this time
add test
```

Before opening a PR, clean this into meaningful, atomic commits.

## Approach 1: Interactive Rebase

```bash
# See how many commits ahead of main you are
git log --oneline origin/main..HEAD

# Rewrite the last N commits
git rebase -i origin/main     # rebase onto main's current tip

# Editor opens:
pick a1b2c3 feat: add login form
pick d4e5f6 wip
pick 7890ab fix typo
pick cdef12 more fixes
pick abcdef actually fix it this time
pick 123456 add test
```

Change to:
```
pick a1b2c3 feat: add login form
fixup d4e5f6 wip
fixup 7890ab fix typo
fixup cdef12 more fixes
fixup abcdef actually fix it this time
pick 123456 test: add login form tests
```

Result: two clean commits.

## Approach 2: Soft Reset + Single Recommit

Nuke all commits since branching and write one clean commit:

```bash
git fetch origin
git reset --soft origin/main    # stage all your changes
git commit -m "feat(auth): add login form with email + password validation

- Email validation with regex
- Password min-length enforcement  
- Rate limiting on failed attempts

Closes #42"
```

## Approach 3: Fixup Commits Inline (while working)

While developing, mark commits as fixups using `--fixup`:

```bash
git commit -m "feat: add login form"
# later, discover a bug
git commit --fixup HEAD    # creates: "fixup! feat: add login form"
# at PR time, autosquash
git rebase -i --autosquash origin/main
```

## Splitting a Commit into Multiple

If one commit does too much:

```bash
git rebase -i HEAD~3           # mark the commit as 'edit'
git reset HEAD~                 # unstage its changes
git add -p                      # stage first logical group
git commit -m "feat: add login form UI"
git add -p                      # stage second logical group
git commit -m "test: add login form validation tests"
git rebase --continue
```

## What Makes a Good Commit

- **One logical change** — reviewers can follow the reasoning
- **Tests and implementation together** — commit doesn't break the test suite at any point
- **Follows Conventional Commits** — `feat:`, `fix:`, etc.
- **Passes CI** — every commit on main should be deployable

## When to NOT clean history

- OSS contributions where individual commits are credited
- When auditing exact steps matters (e.g., security patches)
- When the PR is already merged — rewriting shared history is disruptive

## Sources
- [[git/sources/pro-git]]
