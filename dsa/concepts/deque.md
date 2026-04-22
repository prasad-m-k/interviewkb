# Deque (Double-Ended Queue)

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/patterns/sliding-window]], [[dsa/problems/sliding-window-maximum]]

## What it is
A deque (double-ended queue) supports O(1) append and pop from both the left and right ends. Python's `collections.deque` is implemented as a doubly-linked list of fixed-size blocks.

## How it works
- `appendleft(x)` / `popleft()` — O(1) operations at the front
- `append(x)` / `pop()` — O(1) operations at the back
- Random access `dq[i]` — O(n); don't use it in performance-critical code

**Monotonic deque pattern:** Maintain the deque in increasing or decreasing order of values by popping elements from the back that violate the invariant before each append. This gives O(1) amortized access to the current window's maximum (or minimum).

## Complexity
| Operation | Time |
|---|---|
| appendleft / popleft | O(1) |
| append / pop | O(1) |
| peek front/back | O(1) |
| Space | O(n) |

## When to use
- **Sliding window maximum/minimum**: monotonic deque tracks the best element in the current window
- **BFS**: standard queue for level-order traversal (can use deque as queue)
- **Design problems requiring O(1) both-end access**: Hit Counter, Deque-based LRU

## Common interview angles
- Sliding Window Maximum (Hard): store *indices* in the deque, not values, so you can evict out-of-window elements by comparing index vs. `i - k`
- The invariant is: elements in the deque are always in decreasing order → `dq[0]` is always the window max

## Examples
```python
from collections import deque

dq = deque()

# Monotonic decreasing deque for sliding window max
def maxSlidingWindow(nums, k):
    dq = deque()   # stores indices
    result = []
    for i, n in enumerate(nums):
        # evict indices outside the window
        while dq and dq[0] < i - k + 1:
            dq.popleft()
        # maintain decreasing order — evict smaller elements from back
        while dq and nums[dq[-1]] < n:
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            result.append(nums[dq[0]])
    return result

# deque as a plain queue (BFS)
queue = deque([start_node])
while queue:
    node = queue.popleft()
    for neighbor in graph[node]:
        queue.append(neighbor)
```

## Sources
- [[DSA overview]]
