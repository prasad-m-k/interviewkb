# Bisect

**Topic:** [[git/topics/core-concepts]]
**Related:** [[git/concepts/commit]], [[git/scenarios/bisect-regression]]

## What it is
`git bisect` performs a binary search through the commit history to identify which commit introduced a bug or regression. Given a known-good and known-bad commit, it halves the search space each step.

## How it works

```
History: A ── B ── C ── D ── E ── F ── G  (bad)
                 ↑ (good)

Bisect tests D (midpoint): bad
Bisect tests B: good
→ Bug introduced in C
```

With N commits between good and bad, it takes ⌈log₂(N)⌉ steps.

## Complexity
O(log n) steps, where n = commits between good and bad.

## When to use
- Regression appeared sometime in the past but you don't know which commit
- Automated test can reliably reproduce the regression (perfect for scripted bisect)

## Common interview angles
- "You have 1000 commits and a regression appeared somewhere. How do you find it efficiently?" (git bisect — O(log n) ≈ 10 checks)
- "How do you automate git bisect?" (`git bisect run <script>` — script exits 0 for good, non-zero for bad)
- "How do you handle a commit that can't be tested?" (`git bisect skip`)
- "What does git bisect reset do?" (returns to original branch/HEAD)

## Examples

```bash
# Manual bisect
git bisect start
git bisect bad                    # current commit is bad
git bisect good v2.0.0            # last known good tag
# Git checks out midpoint; you test, then:
git bisect good                   # or: git bisect bad
# Repeat until git identifies the culprit commit
git bisect reset                  # return to original HEAD

# Automated bisect
git bisect start
git bisect bad HEAD
git bisect good v2.0.0
git bisect run python test_regression.py
# Script must exit 0=good, 1-127=bad, 125=skip

# Skip untestable commits
git bisect skip abc123
git bisect skip HEAD~3..HEAD~1   # skip a range
```

## Sources
- [[git/sources/pro-git]]
