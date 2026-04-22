# Pattern: Grid Simulation

**Topic:** [[Grid overview]]
**Related:** [[dsa/grid/concepts/boundary-conditions]], [[dsa/grid/concepts/direction-vectors]]

## What it solves
Problems where you apply repeated rules to a grid and observe the result. No shortest path, no counting — just follow the rules step by step.

## Signal phrases
"game of life" / "simulate" / "after N steps" / "what does the grid look like after" / "generate the next state"

## Template: Game of Life (in-place simulation)

**Key challenge:** all cells update simultaneously. Naive in-place modification is wrong (new state affects neighbors being computed).

**Solution:** Use bit manipulation to encode both old and new state in one value.

```python
def game_of_life(board):
    ROWS, COLS = len(board), len(board[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0),(1,1),(1,-1),(-1,1),(-1,-1)]

    def count_live_neighbors(r, c):
        count = 0
        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if 0 <= nr < ROWS and 0 <= nc < COLS:
                count += board[nr][nc] & 1   # check original state (bit 0)
        return count

    # Encode: bit 0 = current state; bit 1 = next state
    for r in range(ROWS):
        for c in range(COLS):
            live = count_live_neighbors(r, c)
            if board[r][c] == 1 and live in [2, 3]:
                board[r][c] |= 2   # next state = alive (bit 1 = 1)
            elif board[r][c] == 0 and live == 3:
                board[r][c] |= 2   # dead cell becomes alive

    # Shift: move next state to current state
    for r in range(ROWS):
        for c in range(COLS):
            board[r][c] >>= 1
```

## Template: Rotting Oranges (BFS simulation with time)

```python
from collections import deque

def oranges_rotting(grid):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    queue = deque()
    fresh = 0

    # Initialize: all rotten oranges at t=0
    for r in range(ROWS):
        for c in range(COLS):
            if grid[r][c] == 2:
                queue.append((r, c, 0))   # (row, col, time)
            elif grid[r][c] == 1:
                fresh += 1

    time = 0
    while queue:
        r, c, t = queue.popleft()
        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if 0 <= nr < ROWS and 0 <= nc < COLS and grid[nr][nc] == 1:
                grid[nr][nc] = 2
                fresh -= 1
                time = t + 1
                queue.append((nr, nc, t + 1))

    return time if fresh == 0 else -1
```

## Template: Spiral Matrix Generation

```python
def generate_matrix(n):
    matrix = [[0] * n for _ in range(n)]
    top, bottom, left, right = 0, n-1, 0, n-1
    num = 1

    while top <= bottom and left <= right:
        for c in range(left, right + 1):
            matrix[top][c] = num; num += 1
        top += 1
        for r in range(top, bottom + 1):
            matrix[r][right] = num; num += 1
        right -= 1
        if top <= bottom:
            for c in range(right, left - 1, -1):
                matrix[bottom][c] = num; num += 1
            bottom -= 1
        if left <= right:
            for r in range(bottom, top - 1, -1):
                matrix[r][left] = num; num += 1
            left += 1

    return matrix
```

## Problems using this pattern
- [[dsa/grid/problems/rotting-oranges]] — BFS simulation; multi-source; time tracking
- Game of Life — simultaneous update; bit trick for in-place
- [[dsa/grid/problems/spiral-matrix]] — pointer-based traversal / generation

## Complexity
- O(R×C) time for one simulation step
- Space: O(1) with in-place trick (bit encoding), O(R×C) for copy-based approach

## Sources
- [[Grid overview]]
