# Spiral Matrix

**Difficulty:** Medium
**Pattern:** [[dsa/grid/patterns/matrix-traversal]]
**Topic:** [[Grid overview]]
**Companies:** Amazon, Microsoft, Google, Apple

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an m×n matrix, return all elements in spiral order (left→down→right→up, layer by layer).

## Approach
Use four pointers: `top`, `bottom`, `left`, `right`. Traverse the outermost ring, then shrink inward.

## Solution (Python)
```python
def spiralOrder(matrix):
    result = []
    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1

    while top <= bottom and left <= right:
        # → right along top row
        for c in range(left, right + 1):
            result.append(matrix[top][c])
        top += 1

        # ↓ down along right column
        for r in range(top, bottom + 1):
            result.append(matrix[r][right])
        right -= 1

        # ← left along bottom row
        if top <= bottom:
            for c in range(right, left - 1, -1):
                result.append(matrix[bottom][c])
            bottom -= 1

        # ↑ up along left column
        if left <= right:
            for r in range(bottom, top - 1, -1):
                result.append(matrix[r][left])
            left += 1

    return result
```

## Complexity
Time: O(R×C) | Space: O(1) extra (output not counted)

## Key insight
The two guards (`if top <= bottom` and `if left <= right`) prevent double-counting the center row or column when the matrix is not square.

**Trace for [[1,2,3],[4,5,6],[7,8,9]]:**
```
→: 1, 2, 3  (top row, top=0→1)
↓: 6, 9     (right col, right=2→1)
←: 8, 7     (bottom row, bottom=2→1, guarded)
↑: 4        (left col, left=0→1, guarded)
→: 5        (top=1, bottom=1, left=1, right=1 → one cell)
```

## Variants / follow-ups
- **Spiral Matrix II**: fill a matrix with 1..n² in spiral order → same pointers, write `num++` instead of reading
- **Generate spiral**: same skeleton, assign instead of append

## Sources
- [[dsa/grid/patterns/matrix-traversal]]
