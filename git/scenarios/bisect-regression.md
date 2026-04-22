# Scenario: Finding a Regression with git bisect

**Concepts:** [[git/concepts/bisect]]
**Related:** [[git/scenarios/interview-questions]]

## The Scenario
A bug appeared in production but you don't know which commit caused it. The history has hundreds of commits since the last known good state.

## Step-by-Step Walkthrough

```bash
# 1. Start bisect
git bisect start

# 2. Mark current state as bad
git bisect bad                    # HEAD is bad, OR:
git bisect bad abc123             # specify a known bad commit

# 3. Mark last known good state
git bisect good v2.0.0            # a tag
git bisect good def456            # or a specific SHA

# Git now checks out the midpoint commit automatically

# 4. Test the current checkout
make test                         # or: open the app, reproduce the bug

# 5. Mark the result and repeat
git bisect good    # if the bug is NOT present at this commit
git bisect bad     # if the bug IS present

# Git keeps halving the range until it identifies the first bad commit:
# "abc123 is the first bad commit"

# 6. Examine the culprit
git show abc123
git log --oneline abc123~1..abc123    # what changed

# 7. Return to original state
git bisect reset
```

## Automating Bisect

When you have a test that reliably reproduces the bug, automate it:

```bash
git bisect start
git bisect bad HEAD
git bisect good v2.0.0
git bisect run pytest tests/test_payment.py::test_checkout
```

Script contract:
- Exit **0** → good (bug not present)
- Exit **1–127** → bad (bug present)
- Exit **125** → skip (can't test this commit)

## Handling Untestable Commits

```bash
git bisect skip                    # skip current commit
git bisect skip abc123             # skip specific commit
git bisect skip abc123..def456     # skip a range
```

If the culprit might be in a skipped range, git reports: `first bad commit could be any of...`

## Efficiency

With N commits between good and bad: **⌈log₂(N)⌉ steps**.

| Range | Steps needed |
|-------|-------------|
| 100 commits | 7 |
| 1,000 commits | 10 |
| 10,000 commits | 14 |
| 1,000,000 commits | 20 |

## After Finding the Culprit

```bash
git show <culprit-sha>        # examine the change
git log --oneline <culprit-sha>^..<culprit-sha>  # what exactly changed

# Options:
# 1. Fix forward: understand what the commit broke and fix it
# 2. Revert: git revert <culprit-sha>
# 3. Cherry-pick the fix to a release branch
```

## Interview Answer Framework
> "I'd run `git bisect start`, mark HEAD as bad, mark the last known good tag as good, then either test manually at each midpoint or pipe it to `pytest` with `git bisect run`. With 1000 commits, I'd find the culprit in about 10 steps."

## Sources
- [[git/sources/pro-git]]
