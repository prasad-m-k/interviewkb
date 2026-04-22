# Search a 2D Matrix (LC 74)

**Difficulty:** Medium
**Topic:** [[Grid overview]]
**Pattern:** [[dsa/grid/patterns/binary-search-2d]]
**Companies:** Amazon, Google, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given an `m × n` matrix where:
- Each row is sorted in ascending order
- The first element of each row is greater than the last element of the previous row

Return `true` if `target` exists in the matrix, `false` otherwise. Must run in O(log(m×n)).

```
Input:  matrix = [[1,3,5,7],[10,11,16,20],[23,30,34,60]], target = 3
Output: true
```

## Approach

The matrix is one continuous sorted array wrapped into rows. A 1D index `i` maps to row `i // COLS`, column `i % COLS`. Run standard binary search over the virtual 1D range `[0, ROWS*COLS - 1]`.

## Solution (Python)

```python
def searchMatrix(matrix: list[list[int]], target: int) -> bool:
    ROWS, COLS = len(matrix), len(matrix[0])
    lo, hi = 0, ROWS * COLS - 1

    while lo <= hi:
        mid = (lo + hi) // 2
        val = matrix[mid // COLS][mid % COLS]
        if val == target:
            return True
        elif val < target:
            lo = mid + 1
        else:
            hi = mid - 1

    return False
```

## Complexity

Time: O(log(m×n)) | Space: O(1)

## Key insight

`mid // COLS` gives the row; `mid % COLS` gives the column. This index mapping is the entire trick — once you see it, the rest is vanilla binary search.

## Variants / follow-ups

- **What if the last element of a row can equal the first of the next?** Still works — the array is still sorted.
- **What if rows are sorted but don't stitch together?** Switch to the staircase approach → [[dsa/grid/problems/search-2d-matrix-ii]]
- **Find the insertion position** (first element ≥ target): replace the `return True` branch with tracking `lo` as the answer.

## Sources

- [[dsa/grid/patterns/binary-search-2d]]
