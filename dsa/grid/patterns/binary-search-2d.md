# Pattern: 2D Binary Search

**Topic:** [[Grid overview]]
**Related concepts:** [[dsa/grid/concepts/boundary-conditions]]

## What it solves

Searching for a target value in a 2D matrix that has some sorted structure. Two common variants require different strategies:

| Matrix type | Sorted property | Best approach |
|---|---|---|
| **Fully sorted** (LC 74) | Rows sorted; last element of row < first of next row — matrix is one continuous sorted array | Flatten index binary search |
| **Row+col sorted** (LC 240) | Each row sorted L→R; each column sorted T→B (rows do NOT stitch together) | Staircase search from top-right |

---

## Variant 1 — Fully Sorted Matrix (LC 74)

### Key insight
The matrix is conceptually a 1D sorted array laid out row by row. Map a 1D index `mid` to `(mid // COLS, mid % COLS)`.

### Template

```python
def search_matrix(matrix: list[list[int]], target: int) -> bool:
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

**Complexity:** O(log(R×C)) time, O(1) space

---

## Variant 2 — Row+Col Sorted Matrix (LC 240)

### Key insight
Start at the **top-right corner** (or bottom-left). From top-right:
- `val > target` → eliminate the column (move left)
- `val < target` → eliminate the row (move down)

This works because moving left decreases the value, moving down increases it — the pointer is always at a decision boundary.

### Template

```python
def search_matrix_ii(matrix: list[list[int]], target: int) -> bool:
    if not matrix or not matrix[0]:
        return False

    ROWS, COLS = len(matrix), len(matrix[0])
    r, c = 0, COLS - 1          # start top-right

    while r < ROWS and c >= 0:
        val = matrix[r][c]
        if val == target:
            return True
        elif val > target:
            c -= 1               # too big → go left
        else:
            r += 1               # too small → go down

    return False
```

**Complexity:** O(R + C) time, O(1) space

> **Note:** This is NOT O(log(R×C)) — it's linear in the sum of dimensions. You cannot do better than O(R + C) in the general row+col sorted case.

---

## Variant 3 — Binary Search Row by Row

When rows are independently sorted but rows don't relate to each other:

```python
import bisect

def search_row_by_row(matrix, target):
    for row in matrix:
        idx = bisect.bisect_left(row, target)
        if idx < len(row) and row[idx] == target:
            return True
    return False
```

**Complexity:** O(R log C) time, O(1) space. Use when R is small.

---

## Signal phrases

- "sorted matrix", "find target in matrix"
- "each row sorted, each column sorted" → staircase (Variant 2)
- "row values less than next row's first element" → flatten binary search (Variant 1)
- "matrix sorted in row-major order" → Variant 1

## Which variant to pick?

```
Is the matrix one continuous sorted sequence (last of row i < first of row i+1)?
  └─ YES → Variant 1 (flatten binary search, O(log RC))
  └─ NO
      Is each row sorted AND each col sorted?
        └─ YES → Variant 2 (staircase, O(R+C))
        └─ NO → Variant 3 (row-by-row, O(R log C)) or linear scan
```

## Complexity Summary

| Variant | Time | Space |
|---|---|---|
| Flatten binary search | O(log(R×C)) | O(1) |
| Staircase (top-right) | O(R + C) | O(1) |
| Row-by-row binary search | O(R log C) | O(1) |

## Problems using this pattern

- [[dsa/grid/problems/search-2d-matrix]] — LC 74; fully sorted; flatten binary search
- [[dsa/grid/problems/search-2d-matrix-ii]] — LC 240; row+col sorted; staircase

## Sources

- [[Grid overview]]
