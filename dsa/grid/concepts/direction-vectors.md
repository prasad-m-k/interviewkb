# Direction Vectors

**Topic:** [[Grid overview]]
**Related:** [[dsa/grid/concepts/boundary-conditions]], [[dsa/grid/patterns/flood-fill-dfs]]

## What it is
The single most reusable building block for all grid problems. Direction vectors encode "which cells can I move to from cell (r, c)?".

## The Three Variants

### 4-Directional (most common)
```python
DIRS = [(0, 1), (0, -1), (1, 0), (-1, 0)]
#        right    left     down    up
```
Use when: movement is up/down/left/right only (standard grid traversal, island counting, maze, BFS shortest path).

### 8-Directional (includes diagonals)
```python
DIRS = [
    (0, 1), (0, -1), (1, 0), (-1, 0),   # cardinal
    (1, 1), (1, -1), (-1, 1), (-1, -1)   # diagonal
]
```
Use when: diagonal movement is allowed (chess problems, word search in 8 directions, Game of Life neighbors).

### Diagonal-Only (rare)
```python
DIRS = [(1, 1), (1, -1), (-1, 1), (-1, -1)]
```
Use when: only diagonal movement matters (anti-diagonal iteration, bishop in chess).

## How to Apply Direction Vectors

```python
ROWS, COLS = len(grid), len(grid[0])
DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

def get_neighbors(r, c):
    for dr, dc in DIRS:
        nr, nc = r + dr, c + dc
        if 0 <= nr < ROWS and 0 <= nc < COLS:
            yield nr, nc

# Usage in DFS
def dfs(r, c):
    visited[r][c] = True
    for nr, nc in get_neighbors(r, c):
        if not visited[nr][nc] and grid[nr][nc] == '1':
            dfs(nr, nc)
```

## Direction-Specific Traversal (for matrix traversal patterns)

### Row-by-row (left to right, top to bottom)
```python
for r in range(ROWS):
    for c in range(COLS):
        process(grid[r][c])
```

### Anti-diagonal iteration (useful for matrix problems)
```python
# All cells on anti-diagonal d: r + c == d
for d in range(ROWS + COLS - 1):
    for r in range(max(0, d-COLS+1), min(ROWS, d+1)):
        c = d - r
        process(grid[r][c])
```

### Spiral traversal direction cycle
```python
# Directions: right → down → left → up (cycle)
SPIRAL_DIRS = [(0,1),(1,0),(0,-1),(-1,0)]
dir_idx = 0
```

## Knight Moves (special case — chess)
```python
KNIGHT_MOVES = [
    (-2,-1),(-2,1),(-1,-2),(-1,2),
    (1,-2),(1,2),(2,-1),(2,1)
]
# Knight moves: (±1,±2) and (±2,±1) — 8 possible moves
```

## Common interview angles
- "Modify your solution to also allow diagonal movement" → swap `DIRS` to 8-directional
- "What if movement has a cost?" → use Dijkstra with priority queue instead of BFS
- "Can you do this in-place without a visited array?" → mark cells with special value (e.g., '0' for visited island cells)

## Sources
- [[Grid overview]]
