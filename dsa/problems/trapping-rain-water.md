# Trapping Rain Water

**Difficulty:** Hard
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** Two Pointers
**Companies:** Apple, Google, Amazon, Meta

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given `n` non-negative integers representing an elevation map where the width of each bar is 1, compute how much water can be trapped after raining.

Example: `height = [0,1,0,2,1,0,1,3,2,1,2,1]` → `6`

## Approach

**Key observation:** the water level above position `i` is `min(max_left[i], max_right[i]) - height[i]`, clamped at 0. The limiting wall is the shorter of the tallest bar to the left and right.

**Approach 1 — Prefix arrays O(n) time, O(n) space:**
Precompute `max_left[i]` and `max_right[i]`, then sum `max(0, min(max_left[i], max_right[i]) - height[i])`.

**Approach 2 — Two pointers O(n) time, O(1) space (preferred):**
Move two pointers inward. At each step, process the side with the smaller max boundary — because that's the side where we know the water level is determined by `max_left` (left pointer) or `max_right` (right pointer). We don't need the other side's exact max because the limiting wall is already known.

- If `height[left] <= height[right]`: left side is limited by `max_left`. If `height[left] >= max_left`, update `max_left`; else add `max_left - height[left]` to water. Advance `left`.
- Else: mirror logic on right side. Advance `right`.

## Solution (Python)
```python
from typing import List

def trap(height: List[int]) -> int:
    left, right = 0, len(height) - 1
    max_left = max_right = 0
    water = 0

    while left < right:
        if height[left] <= height[right]:
            if height[left] >= max_left:
                max_left = height[left]
            else:
                water += max_left - height[left]
            left += 1
        else:
            if height[right] >= max_right:
                max_right = height[right]
            else:
                water += max_right - height[right]
            right -= 1

    return water
```

## Complexity
Time: O(n) | Space: O(1)

## Key insight
The two-pointer approach works because of the asymmetry: when processing the left pointer, we only need `max_left` (not `max_right`) — because `height[left] <= height[right]` guarantees there's a wall on the right at least as tall as `max_left`. So `max_left` is the definitive ceiling. Mirror reasoning applies to the right pointer. This justifies processing each side independently.

## Variants / follow-ups
- **Container With Most Water**: find two lines that together with the x-axis form the container holding the most water — same two-pointer approach but simpler (no summing, just `min(h[l], h[r]) * (r - l)`).
- **Trapping Rain Water II (3D)**: given a 2D height map, compute trapped water — use a min-heap (priority queue) processing cells from the boundary inward (Dijkstra-like).
- **Interviewers ask**: "Can you solve it with a stack?" → Yes: maintain a monotonic stack; when a taller bar is encountered, pop and compute trapped water for the valley. Also O(n) time and space.

## Sources
- [[dsa/companies/apple]]
- [[DSA overview]]
