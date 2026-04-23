# Permutations

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/backtracking]]
**Companies:** [[dsa/companies/meta]], [[dsa/companies/google]], [[dsa/companies/apple]]

## Problem

Given an array `nums` of **distinct** integers, return all possible permutations.

```
Input:  [1, 2, 3]
Output: [[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]
```

## Approach

**Backtracking with a `used` set:**

At each position in the permutation, try every unused number. Mark it used before recursing, unmark it after.

The recursion tree has n levels. At level k, there are `n - k` choices. Total leaves = n! (all permutations).

```
choose 1 → choose 2 → choose 3 → [1,2,3] ✓
         → choose 3 → choose 2 → [1,3,2] ✓
choose 2 → choose 1 → choose 3 → [2,1,3] ✓
...
```

**Why backtracking works here:** We want *all* valid configurations (not just one), and each configuration is a full-length path in the decision tree. The "backtrack" (unchoose) step is what lets us re-use the same element in a different position.

## Solution (Python)

```python
def permute(nums: list[int]) -> list[list[int]]:
    result = []
    used = [False] * len(nums)

    def backtrack(current: list[int]):
        if len(current) == len(nums):
            result.append(current[:])  # copy — not a reference
            return
        for i, num in enumerate(nums):
            if used[i]:
                continue
            used[i] = True
            current.append(num)
            backtrack(current)
            current.pop()         # unchoose
            used[i] = False       # unchoose

    backtrack([])
    return result
```

## Alternative: Swap-in-place

More memory-efficient — avoids the `used` array by swapping elements to build permutations in-place:

```python
def permute_swap(nums: list[int]) -> list[list[int]]:
    result = []

    def backtrack(start: int):
        if start == len(nums):
            result.append(nums[:])
            return
        for i in range(start, len(nums)):
            nums[start], nums[i] = nums[i], nums[start]  # choose
            backtrack(start + 1)
            nums[start], nums[i] = nums[i], nums[start]  # unchoose

    backtrack(0)
    return result
```

## Complexity

| | Value |
|---|---|
| Time | O(n × n!) — n! permutations, each takes O(n) to copy |
| Space | O(n) recursion depth + O(n × n!) output |

## Key Insight

The `used` boolean array enforces "each element appears exactly once per permutation." Without it, you'd generate multisets with repetition.

The **copy** (`current[:]` or `nums[:]`) at the leaf is mandatory. If you append `current` directly, all stored permutations will point to the same list, which will be empty after backtracking completes.

## Variants / Follow-ups

### Permutations II (with duplicates)
If `nums` can have duplicates, sort first and skip duplicates at the same level:
```python
def permute_unique(nums):
    result = []
    nums.sort()
    used = [False] * len(nums)

    def backtrack(current):
        if len(current) == len(nums):
            result.append(current[:])
            return
        for i in range(len(nums)):
            if used[i]:
                continue
            # Skip: same value as previous, and previous was just unselected
            # This prevents duplicate permutations at the same tree level
            if i > 0 and nums[i] == nums[i-1] and not used[i-1]:
                continue
            used[i] = True
            current.append(nums[i])
            backtrack(current)
            current.pop()
            used[i] = False

    backtrack([])
    return result
```

- "What if you want permutations of length k (not all n elements)?" → Stop recursion when `len(current) == k`, not `== len(nums)`.
- "Generate permutations without recursion?" → Use `itertools.permutations(nums)` in Python, or implement Heap's algorithm iteratively.
- "Count permutations without generating them?" → n! for distinct, n!/(c1! × c2! × ...) for duplicates (multinomial coefficient).

## Sources
- [[dsa/companies/meta]]
- [[dsa/patterns/backtracking]]
