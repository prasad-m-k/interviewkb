# Combination Sum

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/backtracking]]
**Companies:** Amazon, Google, Apple, Bloomberg

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given an array of distinct integers `candidates` and a target integer `target`, return all unique combinations of `candidates` where the chosen numbers sum to `target`. The same number may be used an unlimited number of times. Order within a combination does not matter.

```
candidates = [2, 3, 6, 7], target = 7
→ [[2,2,3], [7]]
```

**Variant — Combination Sum II (LC 40):** each candidate may only be used once; input may have duplicates.

## Approach

Backtracking with a `start` index to avoid generating duplicate combinations (e.g., `[2,3]` and `[3,2]` are the same). At each step, include `candidates[i]` or skip it. Sort the input first — skip elements > remaining target early.

## Solution (Python)

```python
def combinationSum(candidates: list[int], target: int) -> list[list[int]]:
    candidates.sort()
    results = []

    def backtrack(start: int, path: list[int], remaining: int) -> None:
        if remaining == 0:
            results.append(path[:])
            return
        for i in range(start, len(candidates)):
            c = candidates[i]
            if c > remaining:
                break                      # sorted → no point continuing
            path.append(c)
            backtrack(i, path, remaining - c)   # i (not i+1): reuse allowed
            path.pop()

    backtrack(0, [], target)
    return results
```

### Variant: Combination Sum II (no reuse, skip duplicates)

```python
def combinationSum2(candidates, target):
    candidates.sort()
    results = []

    def backtrack(start, path, remaining):
        if remaining == 0:
            results.append(path[:]); return
        for i in range(start, len(candidates)):
            if candidates[i] > remaining: break
            if i > start and candidates[i] == candidates[i-1]:
                continue                   # skip duplicate at this level
            path.append(candidates[i])
            backtrack(i + 1, path, remaining - candidates[i])
            path.pop()

    backtrack(0, [], target)
    return results
```

## Complexity

Time: O(N^(T/M)) where T = target, M = min candidate — branching factor N, depth T/M  
Space: O(T/M) recursion depth

## Key insight

Passing `start=i` (not `i+1`) allows reuse of the same element. Sorting + breaking early when `c > remaining` prunes the tree significantly. In Combination Sum II, the `i > start and candidates[i] == candidates[i-1]` guard is the only extra line — it prevents choosing the same value twice at the same decision level.

## Variants / follow-ups

- **Combination Sum III (LC 216):** choose k numbers from 1–9 summing to target — add a depth limit
- **Combination Sum IV (LC 377):** count orderings (permutations) that sum to target — DP, not backtracking
- **Coin Change (LC 322):** minimum coins to hit target — DP (optimal value, not enumerate)

## Sources

- [[dsa/patterns/backtracking]]
