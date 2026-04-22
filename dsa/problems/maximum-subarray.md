# Maximum Subarray

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** Kadane's Algorithm (DP on arrays)
**Companies:** Apple, Amazon, Google, Microsoft, Adobe

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an integer array `nums`, find the contiguous subarray with the largest sum and return that sum.

Example: `nums = [-2,1,-3,4,-1,2,1,-5,4]` → `6` (subarray `[4,-1,2,1]`)

## Approach
**Kadane's algorithm** is the canonical solution. At each position `i`, we have a choice: extend the current subarray by including `nums[i]`, or start a fresh subarray at `nums[i]`. We pick whichever gives a larger current sum.

`curr_sum = max(nums[i], curr_sum + nums[i])`

Equivalently: if `curr_sum` went negative, starting fresh is always better (a negative prefix only hurts any future sum). Track the global maximum as we go.

This is a one-pass O(n) DP where `dp[i] = max subarray sum ending at i`, space-optimized to O(1) since we only need the previous state.

## Solution (Python)
```python
from typing import List

def maxSubArray(nums: List[int]) -> int:
    max_sum = curr_sum = nums[0]
    for n in nums[1:]:
        curr_sum = max(n, curr_sum + n)
        max_sum = max(max_sum, curr_sum)
    return max_sum
```

## Complexity
Time: O(n) | Space: O(1)

## Key insight
`curr_sum = max(n, curr_sum + n)` is equivalent to: "if extending the window makes it worse than starting fresh (`curr_sum < 0`), start fresh." The condition `curr_sum + n < n` simplifies to `curr_sum < 0`. So the rule is: if the running sum becomes negative, reset to 0 before adding the next element. Kadane's elegantly expresses this as a single `max`.

## Variants / follow-ups
- **Return the subarray indices (not just sum)**: track `start`, `end`, and `temp_start`. When `curr_sum` resets, update `temp_start = i`; when a new max is found, `start = temp_start`, `end = i`.
- **Maximum Product Subarray**: same structure but track both `curr_max` and `curr_min` (because a negative × negative = positive can flip the optimum). Swap `curr_max` and `curr_min` when the current element is negative.
- **Maximum Circular Subarray**: answer is either the normal max subarray OR total_sum − (minimum subarray) — because the "wraparound" subarray is everything except the minimum subarray.
- **Interviewers ask**: "What if all numbers are negative?" → The answer is the least-negative single element. Kadane's handles this correctly because we initialize with `nums[0]`, not 0.
- **Divide and conquer version**: O(n log n); not optimal but demonstrates a different thinking pattern. Max subarray either lies entirely in the left half, right half, or crosses the midpoint.

## MLOps connection
Maximum gain window in a time series: given a sequence of daily metric deltas (positive = improvement, negative = regression), find the contiguous time window with the greatest cumulative improvement. Direct application of Kadane's.

## Sources
- [[dsa/companies/apple]]
- [[DSA overview]]
