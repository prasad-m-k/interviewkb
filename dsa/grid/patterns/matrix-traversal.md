# Pattern: Matrix Traversal (Spiral, Diagonal, Rotation)

**Topic:** [[Grid overview]]
**Related:** [[dsa/grid/concepts/boundary-conditions]]

## What it solves
Traversing a matrix in a specific order that isn't simple row-by-row. Classic interview questions: spiral order, diagonal order, matrix rotation.

## Spiral Traversal

**Idea:** Maintain four pointers (top, bottom, left, right). Peel the matrix layer by layer: go right along top, down along right side, left along bottom, up along left side. Shrink boundaries. Repeat.

```python
def spiral_order(matrix):
    if not matrix:
        return []

    result = []
    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1

    while top <= bottom and left <= right:
        # → move right along top row
        for c in range(left, right + 1):
            result.append(matrix[top][c])
        top += 1

        # ↓ move down along right column
        for r in range(top, bottom + 1):
            result.append(matrix[r][right])
        right -= 1

        # ← move left along bottom row (guard: avoid re-traversing single row)
        if top <= bottom:
            for c in range(right, left - 1, -1):
                result.append(matrix[bottom][c])
            bottom -= 1

        # ↑ move up along left column (guard: avoid re-traversing single col)
        if left <= right:
            for r in range(bottom, top - 1, -1):
                result.append(matrix[r][left])
            left += 1

    return result
```

**Trace for a 3×3 matrix:**
```
1 2 3
4 5 6   →  [1, 2, 3, 6, 9, 8, 7, 4, 5]
7 8 9
```

**Direction-cycle variant** (alternative implementation):
```python
def spiral_order_v2(matrix):
    ROWS, COLS = len(matrix), len(matrix[0])
    DIRS = [(0,1),(1,0),(0,-1),(-1,0)]  # right, down, left, up
    visited = [[False]*COLS for _ in range(ROWS)]
    result = []
    r = c = d = 0

    for _ in range(ROWS * COLS):
        result.append(matrix[r][c])
        visited[r][c] = True
        nr, nc = r + DIRS[d][0], c + DIRS[d][1]
        if (0 <= nr < ROWS and 0 <= nc < COLS and not visited[nr][nc]):
            r, c = nr, nc
        else:
            d = (d + 1) % 4          # turn
            r, c = r + DIRS[d][0], c + DIRS[d][1]

    return result
```

## Diagonal Traversal

### Main diagonal (top-left to bottom-right)
```python
# All cells on diagonal d: r - c == d - (COLS-1)
# Or iterate by diagonal index
for d in range(-(ROWS-1), COLS):
    for r in range(max(0, -d), min(ROWS, COLS-d)):
        c = r + d
        process(matrix[r][c])
```

### Anti-diagonal (top-right to bottom-left)
```python
# All cells on anti-diagonal d: r + c == d
for d in range(ROWS + COLS - 1):
    for r in range(max(0, d - COLS + 1), min(ROWS, d + 1)):
        c = d - r
        process(matrix[r][c])
```

### Diagonal zigzag (LeetCode 498)
```python
def find_diagonal_order(matrix):
    ROWS, COLS = len(matrix), len(matrix[0])
    result = []
    for d in range(ROWS + COLS - 1):
        if d % 2 == 0:  # going up-right
            r = min(d, ROWS-1)
            c = d - r
            while r >= 0 and c < COLS:
                result.append(matrix[r][c])
                r -= 1; c += 1
        else:           # going down-left
            c = min(d, COLS-1)
            r = d - c
            while c >= 0 and r < ROWS:
                result.append(matrix[r][c])
                r += 1; c -= 1
    return result
```

## Matrix Rotation

### Rotate 90° clockwise (in-place)
```python
def rotate_90_cw(matrix):
    n = len(matrix)
    # Step 1: Transpose (swap matrix[i][j] and matrix[j][i])
    for i in range(n):
        for j in range(i+1, n):
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
    # Step 2: Reverse each row
    for i in range(n):
        matrix[i].reverse()
```

### Rotate 90° counter-clockwise
```python
def rotate_90_ccw(matrix):
    n = len(matrix)
    # Step 1: Transpose
    for i in range(n):
        for j in range(i+1, n):
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
    # Step 2: Reverse each column (instead of row)
    for j in range(n):
        for i in range(n//2):
            matrix[i][j], matrix[n-1-i][j] = matrix[n-1-i][j], matrix[i][j]
```

## Set Matrix Zeroes (in-place)
```python
def set_zeroes(matrix):
    ROWS, COLS = len(matrix), len(matrix[0])
    zero_rows, zero_cols = set(), set()

    # Pass 1: find all zeros
    for r in range(ROWS):
        for c in range(COLS):
            if matrix[r][c] == 0:
                zero_rows.add(r)
                zero_cols.add(c)

    # Pass 2: zero out rows and columns
    for r in range(ROWS):
        for c in range(COLS):
            if r in zero_rows or c in zero_cols:
                matrix[r][c] = 0
```

## Signal phrases
"spiral order" / "rotate matrix" / "diagonal elements" / "layer-by-layer" / "set zeroes"

## Complexity
- All traversals: O(R×C) time, O(1) extra space (pointer-based)
- Spiral: O(1) extra if using pointer approach

## Sources
- [[Grid overview]]
