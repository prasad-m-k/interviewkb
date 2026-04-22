# Pattern: Flood Fill / DFS Connected Components

**Topic:** [[Grid overview]]
**Related concepts:** [[dsa/grid/concepts/direction-vectors]], [[dsa/grid/concepts/boundary-conditions]]

## What it solves
- Counting the number of connected regions ("islands") in a grid
- Marking/converting all cells in a connected region (paint bucket tool)
- Finding the size of each region
- Any "how many connected groups?" question on a 2D grid

## Template / Skeleton

```python
def flood_fill_dfs(grid):
    if not grid or not grid[0]:
        return 0

    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    def dfs(r, c):
        # Base case: out of bounds or not a valid cell to visit
        if not (0 <= r < ROWS and 0 <= c < COLS):
            return 0
        if grid[r][c] != '1':    # already visited (marked '0') or water
            return 0

        grid[r][c] = '0'         # mark visited IN-PLACE

        size = 1
        for dr, dc in DIRS:
            size += dfs(r + dr, c + dc)
        return size

    count = 0
    for r in range(ROWS):
        for c in range(COLS):
            if grid[r][c] == '1':
                dfs(r, c)          # or: count += 1; dfs also counts size
                count += 1

    return count
```

## BFS Variant (avoids recursion depth limit)
```python
from collections import deque

def flood_fill_bfs(grid, start_r, start_c, target_val, new_val):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    if grid[start_r][start_c] == new_val:
        return grid  # avoid infinite loop if already new_val

    queue = deque([(start_r, start_c)])
    grid[start_r][start_c] = new_val

    while queue:
        r, c = queue.popleft()
        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if (0 <= nr < ROWS and 0 <= nc < COLS
                    and grid[nr][nc] == target_val):
                grid[nr][nc] = new_val
                queue.append((nr, nc))

    return grid
```

## Signal phrases
"count islands" / "number of connected regions" / "flood fill" / "how many X groups" / "largest connected component" / "max area of island"

## Complexity
- Time: O(R×C) — each cell visited at most once
- Space: O(R×C) — DFS recursion stack depth in worst case (one big island)

## Problems using this pattern
- [[dsa/grid/problems/number-of-islands]] — count connected '1' components
- [[dsa/grid/problems/flood-fill]] — paint all connected cells of same color
- [[dsa/grid/problems/max-area-of-island]] — DFS + return size; track max
- [[dsa/grid/problems/surrounded-regions]] — mark regions from border; flip unmarked

## Key variations

### Max area: return size from DFS
```python
def dfs(r, c):
    if not valid(r, c): return 0
    grid[r][c] = '0'
    return 1 + sum(dfs(r+dr, c+dc) for dr, dc in DIRS)

max_area = max(dfs(r, c) for r in range(ROWS) for c in range(COLS) if grid[r][c] == '1')
```

### Mark from boundary (surrounded regions trick)
```python
# Mark all 'O' connected to border as '#' (safe)
# Then: 'O' remaining = surrounded → flip to 'X'
# Then: '#' → restore to 'O'
for r in range(ROWS):
    for c in range(COLS):
        if (r in [0, ROWS-1] or c in [0, COLS-1]) and grid[r][c] == 'O':
            dfs_mark(r, c)  # mark as '#'
```

## Sources
- [[Grid overview]]
