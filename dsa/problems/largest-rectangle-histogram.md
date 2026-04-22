# Largest Rectangle in Histogram

**Difficulty:** Hard
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/monotonic-stack]] (increasing — find left + right boundary for each bar)
**Companies:** Amazon, Google, Meta, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given an array `heights` representing the histogram's bar heights (width = 1 each), find the area of the largest rectangle that fits entirely within the histogram.

```
heights = [2, 1, 5, 6, 2, 3]
Answer  = 10   (rectangle using bars at index 2,3 with height 5: 5×2=10)
```

## Key Observation

For each bar `i`, the largest rectangle it can be part of has:
- Height = `heights[i]`
- Width = distance from its **left boundary** (first bar to the left that is shorter) to its **right boundary** (first bar to the right that is shorter)

`area[i] = heights[i] × (right_boundary[i] - left_boundary[i] - 1)`

This is two "next smaller element" queries simultaneously — exactly what an increasing monotonic stack computes.

## Approach

Use an increasing stack (bottom = tallest, top = shortest). When we encounter `heights[i] < heights[top]`, the top bar has found its right boundary (`i`) and its left boundary (the new top after popping). Pop and compute the area.

**Sentinel trick:** append `0` to `heights` so every bar gets popped (even bars at the end that never find a smaller right boundary).

## Solution (Python)

```python
def largestRectangleArea(heights: list[int]) -> int:
    heights = heights + [0]     # sentinel: forces all bars to be popped
    stack = [-1]                # sentinel: provides a valid left boundary for index 0
    max_area = 0

    for i in range(len(heights)):
        while stack[-1] != -1 and heights[stack[-1]] >= heights[i]:
            h = heights[stack.pop()]
            w = i - stack[-1] - 1    # width = right boundary - left boundary - 1
            max_area = max(max_area, h * w)
        stack.append(i)

    return max_area
```

## Why the sentinels?

**Right sentinel `0`:** without it, bars at the end of the array never find a shorter right neighbor and stay in the stack. Adding height `0` at the end forces them all to be popped and evaluated.

**Left sentinel `-1`:** after popping bar `j`, the width is `i - stack[-1] - 1`. If `j` was the last bar in the stack, `stack[-1]` would be empty. The `-1` index serves as a virtual "left wall" at position -1, so the width formula still works: `i - (-1) - 1 = i` (extends all the way to the left edge).

## Traced execution

```
heights = [2, 1, 5, 6, 2, 3, 0]  (0 appended)
stack   = [-1]

i=0: h=2 > heights[-1](sentinal) → push 0      stack=[-1, 0]
i=1: h=1 < heights[0]=2
     pop 0: h=2, w=1-(-1)-1=1, area=2
     push 1                                     stack=[-1, 1]
i=2: h=5 > 1 → push 2                          stack=[-1,1,2]
i=3: h=6 > 5 → push 3                          stack=[-1,1,2,3]
i=4: h=2 < 6 → pop 3: h=6, w=4-2-1=1, area=6
               h=2 < 5 → pop 2: h=5, w=4-1-1=2, area=10  ← max
               h=2 = 2, stop. push 4            stack=[-1,1,4]
i=5: h=3 > 2 → push 5                          stack=[-1,1,4,5]
i=6: h=0 < 3 → pop 5: h=3, w=6-4-1=1, area=3
               h=0 < 2 → pop 4: h=2, w=6-1-1=4, area=8
               h=0 < 1 → pop 1: h=1, w=6-(-1)-1=6, area=6
               stack=[-1], stop.

max_area = 10  ✓
```

## Complexity

Time: O(n) — each bar pushed and popped once  
Space: O(n) — stack

## Key insight

`width = right_boundary - left_boundary - 1` where:
- `right_boundary = i` (the current index that triggered the pop)
- `left_boundary = stack[-1]` (the index just below the popped element — the nearest shorter bar to the left)

The two sentinels eliminate edge-case code for "no left boundary" and "no right boundary."

## Connection to Trapping Rain Water

Both problems use a monotonic stack and the same left/right boundary concept, but:

| | Histogram | Trapping Rain Water |
|---|---|---|
| Stack type | Increasing (next smaller) | Decreasing (next greater) |
| Computes | max rectangle area | water trapped in valleys |
| Key formula | `h × (right - left - 1)` | `(min(left_max, right_max) - h) × width` |

## Extension: Maximal Rectangle in Binary Matrix (LC 85)

Convert each row to a histogram (count consecutive `1`s above, reset to 0 on `0`), then run this algorithm on each row's histogram.

```python
def maximalRectangle(matrix):
    if not matrix: return 0
    COLS = len(matrix[0])
    heights = [0] * COLS
    best = 0
    for row in matrix:
        for c in range(COLS):
            heights[c] = heights[c] + 1 if row[c] == '1' else 0
        best = max(best, largestRectangleArea(heights))
    return best
```

## Variants / follow-ups

- **Maximal Rectangle (LC 85):** binary matrix → run histogram algorithm row by row
- **Maximum Width Ramp (LC 962):** largest `j-i` where `nums[i] ≤ nums[j]` — decreasing stack + reverse scan
- **What if bars have variable widths?** Multiply by actual width instead of 1

## Sources

- [[dsa/patterns/monotonic-stack]]
- [[dsa/concepts/stack]]
