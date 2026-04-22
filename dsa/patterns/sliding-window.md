# Sliding Window

**Topic:** [[dsa/topics/algorithms]]
**Related concepts:** [[dsa/concepts/deque]], [[dsa/concepts/hash-map]]

## What it solves
Problems involving a contiguous subarray or substring where you need to find an optimal (max, min, longest, shortest) range satisfying some constraint — without examining every possible subrange in O(n²).

## Template / skeleton

**Fixed-size window (size k):**
```python
def fixed_window(nums, k):
    window_sum = sum(nums[:k])
    result = window_sum
    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]   # add right, remove left
        result = max(result, window_sum)
    return result
```

**Variable-size window (shrink until valid):**
```python
def variable_window(nums):
    left = 0
    result = 0
    state = ...   # track window invariant (sum, count, etc.)
    for right in range(len(nums)):
        # expand window to include nums[right]
        state = update(state, nums[right])
        # shrink from left while invariant is violated
        while not valid(state):
            state = remove(state, nums[left])
            left += 1
        result = max(result, right - left + 1)
    return result
```

**Sliding window maximum (monotonic deque):**
```python
from collections import deque

def sliding_window_max(nums, k):
    dq = deque()   # indices, decreasing order of values
    result = []
    for i, n in enumerate(nums):
        while dq and dq[0] < i - k + 1:
            dq.popleft()
        while dq and nums[dq[-1]] < n:
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            result.append(nums[dq[0]])
    return result
```

## Signal phrases
- "maximum/minimum sum subarray of size k"
- "longest substring with at most K distinct characters"
- "minimum window substring"
- "sliding window maximum/minimum"
- "find all substrings that..."
- "maximum product subarray"

## Complexity
- Fixed window: O(n) time, O(1) space
- Variable window: O(n) time, O(k) space for the state
- Monotonic deque window: O(n) time, O(k) space

## Problems using this pattern
- [[dsa/problems/sliding-window-maximum]]
- [[dsa/problems/design-hit-counter]]

## Sources
- [[DSA overview]]
