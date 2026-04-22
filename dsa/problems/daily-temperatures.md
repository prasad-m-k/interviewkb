# Daily Temperatures

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/monotonic-stack]] (decreasing — next greater element)
**Companies:** Amazon, Google, Facebook, Uber

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given an array `temperatures`, return an array `answer` where `answer[i]` is the number of days until a warmer temperature. If no future day is warmer, `answer[i] = 0`.

```
temperatures = [73,74,75,71,69,72,76,73]
answer       = [ 1, 1, 4, 2, 1, 1, 0, 0]
```

## Approach

This is "next greater element" — for each index, find the nearest index to the right with a larger value. Classic decreasing monotonic stack.

Store **indices** (not values) in the stack so you can compute the distance `i - j`.

Stack invariant: indices in the stack always have decreasing temperatures (largest at bottom, smallest at top).

When temperature `i` is warmer than the temperature at `stack[-1]`, pop — `i` is the answer for the popped index.

## Solution (Python)

```python
def dailyTemperatures(temperatures: list[int]) -> list[int]:
    n = len(temperatures)
    answer = [0] * n
    stack = []           # indices with decreasing temperatures (top = coldest)

    for i in range(n):
        while stack and temperatures[stack[-1]] < temperatures[i]:
            j = stack.pop()
            answer[j] = i - j        # days to wait
        stack.append(i)

    # remaining indices in stack have no warmer future day → answer stays 0
    return answer
```

## Traced execution

```
temps:  73  74  75  71  69  72  76  73
index:   0   1   2   3   4   5   6   7

i=0: stack=[]      → push 0        stack=[0]
i=1: temps[0]=73 < 74 → pop 0, ans[0]=1-0=1; push 1    stack=[1]
i=2: temps[1]=74 < 75 → pop 1, ans[1]=2-1=1; push 2    stack=[2]
i=3: 75 > 71 → just push 3         stack=[2,3]
i=4: 71 > 69 → just push 4         stack=[2,3,4]
i=5: 69 < 72 → pop 4, ans[4]=5-4=1; 71 < 72 → pop 3, ans[3]=5-3=2; push 5
                                    stack=[2,5]
i=6: 72 < 76 → pop 5, ans[5]=6-5=1; 75 < 76 → pop 2, ans[2]=6-2=4; push 6
                                    stack=[6]
i=7: 76 > 73 → push 7              stack=[6,7]

Remaining: ans[6]=0, ans[7]=0  (no warmer day)
Final: [1,1,4,2,1,1,0,0]  ✓
```

## Complexity

Time: O(n) — each index pushed and popped at most once  
Space: O(n) — stack

## Key insight

Store indices, not values. The answer for index `j` is `i - j` (distance), not `temperatures[i]` (value). You need the index to compute distance.

The decreasing invariant means the stack's top always points to the "most recent unsatisfied query" — the nearest day waiting for a warmer temperature.

## Variants / follow-ups

- **Next Greater Element I (LC 496):** same pattern; target array is a subset of a larger array
- **Next Greater Element II (LC 503):** circular array → run two passes with `i % n`
- **Online Stock Span (LC 901):** "consecutive days ≤ today" = decreasing stack tracking spans
- **What if you want the previous warmer day?** Traverse right-to-left with the same stack

## Sources

- [[dsa/patterns/monotonic-stack]]
- [[dsa/concepts/stack]]
