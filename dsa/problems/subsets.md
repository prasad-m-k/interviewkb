# Subsets

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/backtracking]]
**Companies:** [[dsa/companies/meta]], [[dsa/companies/google]], [[dsa/companies/apple]]

## Problem

Given an integer array `nums` of **unique** elements, return all possible subsets (the power set).

```
Input:  [1, 2, 3]
Output: [[], [1], [2], [3], [1,2], [1,3], [2,3], [1,2,3]]
```

The result must not contain duplicate subsets.

## Approach

Unlike permutations, order doesn't matter in subsets — `[1,2]` and `[2,1]` are the same subset. The classic trick: always move *forward* in the array by passing a `start` index. This prevents revisiting earlier elements and automatically avoids duplicates.

**Decision tree for `[1, 2, 3]`:**
```
[]
├── [1]
│   ├── [1, 2]
│   │   └── [1, 2, 3]
│   └── [1, 3]
├── [2]
│   └── [2, 3]
└── [3]
```

We record the current subset at **every node**, not just leaves. That's the key difference from permutations.

## Solution (Python — Backtracking)

```python
def subsets(nums: list[int]) -> list[list[int]]:
    result = []

    def backtrack(start: int, current: list[int]):
        result.append(current[:])   # record at every node, including []
        for i in range(start, len(nums)):
            current.append(nums[i])
            backtrack(i + 1, current)  # i+1: only use elements after index i
            current.pop()

    backtrack(0, [])
    return result
```

## Alternative: Bit Manipulation

Each subset corresponds to a bitmask of length n. Bit k is set if `nums[k]` is included.

```python
def subsets_bitmask(nums: list[int]) -> list[list[int]]:
    n = len(nums)
    result = []
    for mask in range(1 << n):   # 0 to 2^n - 1
        subset = [nums[i] for i in range(n) if mask & (1 << i)]
        result.append(subset)
    return result
```

Clean, iterative, no recursion. Downside: harder to extend to the "with duplicates" variant.

## Alternative: Iterative (build subsets one element at a time)

```python
def subsets_iterative(nums: list[int]) -> list[list[int]]:
    result = [[]]
    for num in nums:
        # For each existing subset, create a new subset with num added
        result += [subset + [num] for subset in result]
    return result
```

Trace for `[1, 2, 3]`:
- Start: `[[]]`
- Add 1: `[[], [1]]`
- Add 2: `[[], [1], [2], [1,2]]`
- Add 3: `[[], [1], [2], [1,2], [3], [1,3], [2,3], [1,2,3]]`

## Complexity

| | Value |
|---|---|
| Time | O(n × 2^n) — 2^n subsets, each takes O(n) to copy |
| Space | O(n) recursion depth + O(n × 2^n) output |

## Key Insight

The `start` index is the anti-duplicate mechanism. By only considering elements at indices `≥ start`, we enforce lexicographic ordering, which automatically prevents `[2,1]` being generated when `[1,2]` was already recorded.

This same `start` trick applies to **Combination Sum**, **Combinations**, and all "choose k from n" family problems.

## Variants / Follow-ups

### Subsets II (with duplicates)
If `nums` can have duplicates, sort first and skip duplicates at the same recursion level:
```python
def subsets_with_dup(nums):
    nums.sort()
    result = []

    def backtrack(start, current):
        result.append(current[:])
        for i in range(start, len(nums)):
            # Skip duplicate at the same tree level
            if i > start and nums[i] == nums[i-1]:
                continue
            current.append(nums[i])
            backtrack(i + 1, current)
            current.pop()

    backtrack(0, [])
    return result
```

The condition `i > start` (not `i > 0`) is crucial — it only skips duplicates *at the same recursion level*, not within a subset.

**Other follow-ups:**
- "Subsets of size exactly k" → Only record when `len(current) == k`.
- "How many subsets are there?" → `2^n` (without duplicates); `(c1+1)(c2+1)...(ck+1)` (with duplicates, where ci is the count of each distinct element).
- "Find all subsets that sum to a target" → Add a sum check before appending; prune early if `current_sum > target`.

## Interview Connection

Subsets is the foundation of many harder problems:
- **Combination Sum:** Subsets where sum == target, elements can repeat
- **Letter Combinations of Phone Number:** Subsets from a character set map
- **Partition Equal Subset Sum:** DP variant asking if any subset sums to total/2
- **Maximum XOR:** Bit manipulation on subsets

## Sources
- [[dsa/companies/meta]]
- [[dsa/patterns/backtracking]]
