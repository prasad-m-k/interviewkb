# Sliding Window Maximum

**Difficulty:** Hard
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/sliding-window]]
**Companies:** Google, Amazon, Meta

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an integer array `nums` and a window size `k`, return an array of the maximum value in each window of size `k` as it slides from left to right.

Example: `nums = [1,3,-1,-3,5,3,6,7]`, `k = 3` → `[3,3,5,5,6,7]`

## Approach
Naive approach scans each window: O(n*k). We need O(n).

Use a **monotonic decreasing deque** of indices:
- The deque always holds indices whose values are in decreasing order.
- `dq[0]` is always the index of the current window's maximum.
- Before appending index `i`:
  1. **Evict** any index from the front that is outside the current window (`dq[0] < i - k + 1`).
  2. **Pop** any index from the back whose value is ≤ `nums[i]` — they can never be a future maximum while `i` is in the window.
- Append `i`. When the window is full (`i >= k - 1`), record `nums[dq[0]]`.

The deque stores indices, not values, so step 1 (expiry check) can compare index positions.

## Solution (Python)
```python
from collections import deque
from typing import List

def maxSlidingWindow(nums: List[int], k: int) -> List[int]:
    dq = deque()   # indices, values decreasing
    result = []

    for i, n in enumerate(nums):
        # Remove indices outside the current window
        while dq and dq[0] < i - k + 1:
            dq.popleft()
        # Maintain decreasing order: remove smaller elements from back
        while dq and nums[dq[-1]] < n:
            dq.pop()
        dq.append(i)
        # Window is full starting at index k-1
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result
```

## Complexity
Time: O(n) — each index is pushed and popped at most once | Space: O(k)

## Key insight
Store **indices** in the deque, not values. If you stored values, you couldn't tell whether `dq[0]` is still inside the current window — you'd need to check the original position, which requires the index anyway.

## Variants / follow-ups
- **Sliding Window Minimum**: change `<` to `>` in the back-pop condition to maintain an increasing deque.
- **Sliding Window Median**: harder — requires two heaps + lazy deletion (O(n log k)).
- **Longest Subarray with Absolute Diff ≤ Limit**: maintain both a min-deque and max-deque simultaneously.
- **Interviewers ask**: "What if k > len(nums)?" → return `[max(nums)]`; handle gracefully.

## MLOps connection
Rolling-window feature computation: max CPU utilization over the last k minutes, peak request rate, max latency in a window. The deque approach is how streaming feature pipelines compute window aggregates efficiently without re-scanning history.

## Sources
- [[DSA overview]]
