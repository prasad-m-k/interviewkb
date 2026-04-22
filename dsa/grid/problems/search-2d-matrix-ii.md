# Search a 2D Matrix II (LC 240)

**Difficulty:** Medium
**Topic:** [[Grid overview]]
**Pattern:** [[dsa/grid/patterns/binary-search-2d]]
**Companies:** Amazon, Google, Apple

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given an `m × n` matrix where:
- Each row is sorted in ascending order (left → right)
- Each column is sorted in ascending order (top → bottom)
- Rows do NOT stitch together into one sorted array

Return `true` if `target` exists, `false` otherwise.

```
Input:
matrix = [
  [ 1,  4,  7, 11, 15],
  [ 2,  5,  8, 12, 19],
  [ 3,  6,  9, 16, 22],
  [10, 13, 14, 17, 24],
  [18, 21, 23, 26, 30]
]
target = 5 → true
target = 20 → false
```

## Approach

Start at the **top-right corner**. At each step exactly one dimension can be eliminated:
- `val == target` → found it
- `val > target` → this value is too big; every cell below it in this column is also too big (column sorted), so eliminate the column by moving left (`c -= 1`)
- `val < target` → this value is too small; every cell left of it in this row is also too small (row sorted), so eliminate the row by moving down (`r += 1`)

The pointer zigzags from top-right toward bottom-left, eliminating one row or one column per step.

**Why not flatten binary search?** The rows don't stitch — `matrix[0][-1]` can be greater than `matrix[1][0]`, so the 1D mapping produces a non-sorted array. Binary search would break.

## Solution (Python)

```python
def searchMatrix(matrix: list[list[int]], target: int) -> bool:
    ROWS, COLS = len(matrix), len(matrix[0])
    r, c = 0, COLS - 1          # top-right corner

    while r < ROWS and c >= 0:
        val = matrix[r][c]
        if val == target:
            return True
        elif val > target:
            c -= 1               # eliminate column
        else:
            r += 1               # eliminate row

    return False
```

## Complexity

Time: O(m + n) | Space: O(1)

## Key insight

The top-right (and bottom-left) corner is the only position where you can make a binary decision: one direction always increases the value, the other always decreases it. Any interior cell has all four options and you can't prune.

## Variants / follow-ups

- **Count occurrences of target:** same staircase, accumulate count when `val == target` and continue both ways — but you need to think carefully. Simpler: once found, expand outward. Or use the O(R log C) row-by-row approach.
- **Matrix has duplicate values:** the staircase still works — duplicates don't break the row/col sorted invariant.
- **Rows stitch together (LC 74):** use the O(log(m×n)) flatten approach → [[dsa/grid/problems/search-2d-matrix]]
- **What is the lower bound?** O(R + C) is optimal for the general row+col sorted case (proven by adversary argument).

## Sources

- [[dsa/grid/patterns/binary-search-2d]]
