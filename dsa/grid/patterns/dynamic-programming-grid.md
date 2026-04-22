# Pattern: Dynamic Programming on Grids

**Topic:** [[Grid overview]]
**Related:** [[dsa/concepts/dynamic-programming]], [[dsa/grid/concepts/boundary-conditions]]

## What it solves
- Count the number of paths from (0,0) to (R-1,C-1)
- Find the minimum cost path
- Find the maximum value path
- Any "how many ways" or "min/max value" question on a grid where you can only move in limited directions

## When to use DP vs. BFS
- **BFS**: shortest path in terms of **steps** (unweighted; just counting hops)
- **DP**: counting paths, or min/max cost where cost = sum of values along path
- **Dijkstra**: shortest path when cells have different costs (weighted)

## Template / Skeleton

```python
def grid_dp(grid):
    ROWS, COLS = len(grid), len(grid[0])
    dp = [[0] * COLS for _ in range(ROWS)]
    dp[0][0] = grid[0][0]

    # Fill first row (can only come from left)
    for c in range(1, COLS):
        dp[0][c] = dp[0][c-1] + grid[0][c]

    # Fill first column (can only come from above)
    for r in range(1, ROWS):
        dp[r][0] = dp[r-1][0] + grid[r][0]

    # Fill rest: dp[r][c] depends on dp[r-1][c] and dp[r][c-1]
    for r in range(1, ROWS):
        for c in range(1, COLS):
            dp[r][c] = min(dp[r-1][c], dp[r][c-1]) + grid[r][c]
            # For count: dp[r][c] = dp[r-1][c] + dp[r][c-1]
            # For max:   dp[r][c] = max(dp[r-1][c], dp[r][c-1]) + grid[r][c]

    return dp[ROWS-1][COLS-1]
```

## Common Grid DP Problems

### Unique Paths (count paths, no obstacles)
```python
def unique_paths(m, n):
    dp = [[1] * n for _ in range(m)]
    for r in range(1, m):
        for c in range(1, n):
            dp[r][c] = dp[r-1][c] + dp[r][c-1]
    return dp[m-1][n-1]

# Space optimization: only need previous row
def unique_paths_o1(m, n):
    dp = [1] * n
    for _ in range(1, m):
        for c in range(1, n):
            dp[c] += dp[c-1]
    return dp[n-1]
```

### Unique Paths II (with obstacles)
```python
def unique_paths_with_obstacles(grid):
    ROWS, COLS = len(grid), len(grid[0])
    if grid[0][0] == 1 or grid[ROWS-1][COLS-1] == 1:
        return 0
    dp = [[0] * COLS for _ in range(ROWS)]
    dp[0][0] = 1
    for r in range(ROWS):
        for c in range(COLS):
            if grid[r][c] == 1:
                dp[r][c] = 0
            else:
                if r > 0: dp[r][c] += dp[r-1][c]
                if c > 0: dp[r][c] += dp[r][c-1]
    return dp[ROWS-1][COLS-1]
```

### Minimum Path Sum
```python
def min_path_sum(grid):
    ROWS, COLS = len(grid), len(grid[0])
    for r in range(ROWS):
        for c in range(COLS):
            if r == 0 and c == 0: continue
            elif r == 0: grid[r][c] += grid[r][c-1]
            elif c == 0: grid[r][c] += grid[r-1][c]
            else: grid[r][c] += min(grid[r-1][c], grid[r][c-1])
    return grid[ROWS-1][COLS-1]
```

### Dungeon Game (reverse DP — tricky!)
```python
def calculate_minimum_hp(dungeon):
    ROWS, COLS = len(dungeon), len(dungeon[0])
    # dp[r][c] = minimum HP needed to enter cell (r,c) and survive to end
    dp = [[0]*COLS for _ in range(ROWS)]

    for r in range(ROWS-1, -1, -1):
        for c in range(COLS-1, -1, -1):
            if r == ROWS-1 and c == COLS-1:
                # Need at least 1 HP after picking up dungeon[r][c]
                dp[r][c] = max(1, 1 - dungeon[r][c])
            elif r == ROWS-1:
                dp[r][c] = max(1, dp[r][c+1] - dungeon[r][c])
            elif c == COLS-1:
                dp[r][c] = max(1, dp[r+1][c] - dungeon[r][c])
            else:
                min_next = min(dp[r+1][c], dp[r][c+1])
                dp[r][c] = max(1, min_next - dungeon[r][c])

    return dp[0][0]
```
**Key insight for Dungeon**: go backward from end. At each cell, determine the minimum HP needed to enter that cell and survive. The answer propagates from the exit to the entrance.

### Triangle (bottom-up DP)
```python
def minimum_total(triangle):
    dp = triangle[-1][:]  # start from bottom row
    for row in range(len(triangle)-2, -1, -1):
        for c in range(len(triangle[row])):
            dp[c] = triangle[row][c] + min(dp[c], dp[c+1])
    return dp[0]
```

## Signal phrases
"number of ways to reach" / "minimum cost path" / "maximum sum path" / "count paths from top-left to bottom-right" / "can only move right or down"

## Complexity
- Time: O(R×C)
- Space: O(R×C) → optimizable to O(C) with rolling row

## Sources
- [[Grid overview]]
- [[dsa/concepts/dynamic-programming]]
