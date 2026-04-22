# Unique Paths

**Difficulty:** Medium
**Pattern:** [[dsa/grid/patterns/dynamic-programming-grid]]
**Topic:** [[Grid overview]]
**Companies:** Google, Amazon, Bloomberg, Uber

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
A robot is on a top-left corner of an m×n grid. It can only move right or down. Count the number of unique paths to the bottom-right corner.

## Approach
DP: `dp[r][c]` = number of ways to reach (r, c). Each cell can only be reached from the left (`dp[r][c-1]`) or above (`dp[r-1][c]`). Sum the two.

## Solution (Python)
```python
# Solution 1: 2D DP
def uniquePaths(m, n):
    dp = [[1] * n for _ in range(m)]
    for r in range(1, m):
        for c in range(1, n):
            dp[r][c] = dp[r-1][c] + dp[r][c-1]
    return dp[m-1][n-1]

# Solution 2: 1D DP (space optimized)
def uniquePaths_o1(m, n):
    dp = [1] * n
    for _ in range(1, m):
        for c in range(1, n):
            dp[c] += dp[c-1]
    return dp[n-1]

# Solution 3: Combinatorics (math)
# Must make (m-1) downs and (n-1) rights in any order
# Answer = C(m+n-2, m-1)
from math import comb
def uniquePaths_math(m, n):
    return comb(m + n - 2, m - 1)
```

## Complexity
Time: O(m×n) | Space: O(n) with 1D DP, O(1) with combinatorics

## Key insight
Every cell has exactly one path into it from the top row or left column (only one choice). For all other cells, the path count is the sum of paths from above and from the left. Base case: entire top row and left column = 1 (only one way to reach them: go straight).

## Variants / follow-ups

### Unique Paths II (with obstacles)
```python
def uniquePathsWithObstacles(grid):
    ROWS, COLS = len(grid), len(grid[0])
    if grid[0][0] == 1 or grid[ROWS-1][COLS-1] == 1:
        return 0
    dp = [[0]*COLS for _ in range(ROWS)]
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
def minPathSum(grid):
    ROWS, COLS = len(grid), len(grid[0])
    for r in range(ROWS):
        for c in range(COLS):
            if r == 0 and c == 0: continue
            elif r == 0: grid[r][c] += grid[r][c-1]
            elif c == 0: grid[r][c] += grid[r-1][c]
            else: grid[r][c] += min(grid[r-1][c], grid[r][c-1])
    return grid[ROWS-1][COLS-1]
```

## Sources
- [[dsa/grid/patterns/dynamic-programming-grid]]
