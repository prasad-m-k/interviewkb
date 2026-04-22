# Boundary Conditions and Visited Tracking

**Topic:** [[Grid overview]]
**Related:** [[dsa/grid/concepts/direction-vectors]], [[dsa/grid/concepts/grid-as-graph]]

## What it is
The most common source of bugs in grid problems. Getting bounds checks, visited state management, and in-place modification right is the difference between a clean solution and an infinite loop or index error.

## Bounds Checking

### Standard pattern
```python
ROWS, COLS = len(grid), len(grid[0])

def in_bounds(r, c):
    return 0 <= r < ROWS and 0 <= c < COLS
```

### Combined bounds + condition check
```python
def valid(r, c):
    return 0 <= r < ROWS and 0 <= c < COLS and grid[r][c] == '1'

# Usage
for dr, dc in DIRS:
    nr, nc = r + dr, c + dc
    if valid(nr, nc):
        dfs(nr, nc)
```

### The short-circuit order matters
```python
# WRONG: grid[nr][nc] evaluated even when out of bounds
if 0 <= nr < ROWS and 0 <= nc < COLS and grid[nr][nc] == '1':  # this is fine actually

# The and operator short-circuits left-to-right in Python
# So bounds check MUST come before grid access
# (This is fine in Python due to short-circuit evaluation)
```

## Visited State: Three Strategies

### Strategy 1: Separate visited array
```python
visited = [[False] * COLS for _ in range(ROWS)]
# or: visited = set()

# Pros: original grid unchanged
# Cons: O(R×C) extra space
```

### Strategy 2: In-place marking (modify grid)
```python
# Mark visited by changing cell value
def dfs(r, c):
    grid[r][c] = '0'   # mark as visited (was '1')
    for dr, dc in DIRS:
        nr, nc = r + dr, c + dc
        if in_bounds(nr, nc) and grid[nr][nc] == '1':
            dfs(nr, nc)

# Pros: O(1) extra space
# Cons: modifies input (may not be acceptable); must use a different marker
# Classic: number-of-islands uses this; marks '1' → '0' or '1' → '#'
```

### Strategy 3: Backtracking (unmark after exploring)
```python
# For word search — need to re-use cells in different paths
def dfs(r, c, idx):
    if idx == len(word):
        return True
    if not in_bounds(r, c) or grid[r][c] != word[idx]:
        return False

    temp = grid[r][c]
    grid[r][c] = '#'   # mark visited for this path

    found = any(dfs(r+dr, c+dc, idx+1) for dr, dc in DIRS)

    grid[r][c] = temp  # UNMARK — critical for correctness

    return found

# Pros: allows re-exploring cells in different paths
# Cons: can't use a visited array (different paths need different visited state)
```

## Edge Cases to Always Check

```python
# 1. Empty grid
if not grid or not grid[0]:
    return []

# 2. Single cell grid
if ROWS == 1 and COLS == 1:
    return grid[0][0]

# 3. Start or end cell is blocked
if grid[0][0] == '#' or grid[ROWS-1][COLS-1] == '#':
    return -1

# 4. Grid with all same values
# (usually not a bug source, but affects base case)
```

## Spiral Traversal Boundary Tracking (separate pattern)
```python
def spiral_order(matrix):
    result = []
    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1

    while top <= bottom and left <= right:
        # Move right along top row
        for c in range(left, right + 1):
            result.append(matrix[top][c])
        top += 1

        # Move down along right col
        for r in range(top, bottom + 1):
            result.append(matrix[r][right])
        right -= 1

        # Move left along bottom row (if still valid)
        if top <= bottom:
            for c in range(right, left - 1, -1):
                result.append(matrix[bottom][c])
            bottom -= 1

        # Move up along left col (if still valid)
        if left <= right:
            for r in range(bottom, top - 1, -1):
                result.append(matrix[r][left])
            left += 1

    return result
```
The `if top <= bottom` and `if left <= right` guards prevent double-counting center row/column in non-square matrices.

## Common Bugs and Fixes

| Bug | Symptom | Fix |
|---|---|---|
| Missing bounds check | `IndexError: list index out of range` | Always call `in_bounds(nr, nc)` before `grid[nr][nc]` |
| Forgot to mark visited | Infinite loop / TLE | Mark immediately when adding to queue / entering DFS |
| Marking after adding to queue (BFS) | Same cell added multiple times | Mark **before** appending to queue |
| Not unmarking in backtrack | Incorrect paths found | Add `grid[r][c] = temp` after recursive call |
| Wrong grid dimensions | Off-by-one everywhere | Use `ROWS = len(grid)`, `COLS = len(grid[0])` consistently |

## BFS: Mark Visited When Enqueuing (not when dequeuing)

```python
# WRONG: same cell can be enqueued multiple times before processing
queue.append((nr, nc))
# later: visited[nr][nc] = True  ← too late

# CORRECT: mark when adding to queue
if (nr, nc) not in visited:
    visited.add((nr, nc))
    queue.append((nr, nc))
```

## Sources
- [[Grid overview]]
