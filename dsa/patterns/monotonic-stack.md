# Pattern: Monotonic Stack

**Topic:** [[dsa/topics/algorithms]]
**Related concepts:** [[dsa/concepts/stack]], [[dsa/concepts/deque]]

## What it solves

Any problem that asks, for each element, about the **nearest element** to its left or right that is larger, smaller, or equal. Also used for "span" (how many consecutive elements behind are smaller), "histogram rectangles", and range queries that depend on local extrema.

---

## The Core Insight

A brute-force "nearest greater/smaller" check scans O(n) per element = O(n²) total. The monotonic stack does it in **O(n)** by observing:

> When we push element `i` onto the stack and must first pop element `j` to maintain order, `i` is the **answer for `j`** (j's next-greater = i, or i's previous-smaller = j, depending on orientation).

Each element is pushed once and popped once → O(n) total.

---

## Two Orientations

### Decreasing stack → "Next Greater Element"

Stack stays decreasing (top is smallest). Before pushing `i`, pop anything smaller. Every popped element `j` has found its next-greater: `i`.

```python
def next_greater(nums: list[int]) -> list[int]:
    n = len(nums)
    result = [-1] * n        # default: no greater element
    stack = []               # stores indices; values are decreasing top→bottom

    for i in range(n):
        # pop all indices whose value is less than nums[i]
        while stack and nums[stack[-1]] < nums[i]:
            j = stack.pop()
            result[j] = nums[i]   # nums[i] is j's next greater
        stack.append(i)

    return result
```

**Trace:** `nums = [2, 1, 5, 6, 2, 3]`
```
i=0: push 0         stack=[0]         (val=2)
i=1: push 1         stack=[0,1]       (val=1, 2>1 so don't pop)
i=2: pop 1→result[1]=5, pop 0→result[0]=5, push 2
                    stack=[2]         (val=5)
i=3: pop 2→result[2]=6, push 3
                    stack=[3]         (val=6)
i=4: push 4         stack=[3,4]       (val=2, 6>2)
i=5: pop 4→result[4]=3, push 5
                    stack=[3,5]       (val=3, 6>3)
end: stack=[3,5] → result[3]=result[5]=-1

result = [5, 5, 6, -1, 3, -1]
```

---

### Increasing stack → "Next Smaller Element" / "Previous Smaller"

Stack stays increasing (top is largest). Before pushing `i`, pop anything larger.

```python
def next_smaller(nums):
    n = len(nums)
    result = [-1] * n
    stack = []               # values are increasing top→bottom

    for i in range(n):
        while stack and nums[stack[-1]] > nums[i]:
            j = stack.pop()
            result[j] = nums[i]
        stack.append(i)

    return result
```

**Previous smaller** = same stack, but record the answer at push time instead of pop time:

```python
def previous_smaller(nums):
    result = [-1] * len(nums)
    stack = []
    for i in range(len(nums)):
        while stack and nums[stack[-1]] >= nums[i]:
            stack.pop()
        result[i] = nums[stack[-1]] if stack else -1
        stack.append(i)
    return result
```

---

## Pattern Table

| Question | Stack order | Record answer | When |
|---|---|---|---|
| Next greater element | Decreasing | at pop | `nums[i] > nums[top]` |
| Next smaller element | Increasing | at pop | `nums[i] < nums[top]` |
| Previous greater | Decreasing | at push | `stack[-1]` before push |
| Previous smaller | Increasing | at push | `stack[-1]` before push |
| Span (consecutive ≤ top) | Decreasing | `i - stack[-1]` at push | — |

---

## Circular Array Variant

For circular arrays, run two passes (or `range(2*n)` with `i % n`):

```python
def nextGreaterCircular(nums):
    n = len(nums)
    result = [-1] * n
    stack = []
    for i in range(2 * n):
        while stack and nums[stack[-1]] < nums[i % n]:
            result[stack.pop()] = nums[i % n]
        if i < n:
            stack.append(i)
    return result
```

---

## Signal phrases

"next greater element" / "next warmer day" / "daily temperatures" / "stock span" / "largest rectangle" / "trap water" / "buildings with ocean view" / "remove k digits"

---

## Complexity

Time: **O(n)** — each element pushed and popped at most once  
Space: **O(n)** — stack holds at most n elements

---

## Problems using this pattern

- [[dsa/problems/daily-temperatures]] — next warmer day; decreasing stack; canonical template
- [[dsa/problems/largest-rectangle-histogram]] — max area rectangle; increasing stack; left+right boundary
- [[dsa/problems/trapping-rain-water]] — stack alternative to two-pointers; decreasing stack
- [[dsa/problems/asteroid-collision]] — simulation stack (values cancel); related mode
- [[dsa/problems/valid-parentheses]] — matching stack (different mode, same data structure)

## Sources

- [[dsa/concepts/stack]]
- [[dsa/topics/algorithms]]
