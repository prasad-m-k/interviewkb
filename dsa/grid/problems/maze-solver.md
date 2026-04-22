# Maze Solver

**Difficulty:** Medium
**Pattern:** [[dsa/grid/patterns/bfs-shortest-path]]
**Topic:** [[Grid overview]]
**Companies:** Google, Amazon, Uber (system design rounds)

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given a maze represented as a 2D grid where 0 = open path and 1 = wall, find the shortest path from the start to the end. Return the minimum number of steps, or -1 if no path exists.

## Approach
BFS. Never use DFS for shortest path — DFS finds A path, not THE shortest. BFS guarantees shortest path in unweighted grids by expanding level by level.

## Solution (Python)
```python
from collections import deque

def solve_maze(grid, start, end):
    """
    grid: List[List[int]] — 0=open, 1=wall
    start: (row, col)
    end: (row, col)
    Returns: minimum steps, or -1 if unreachable
    """
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]
    sr, sc = start
    er, ec = end

    if grid[sr][sc] == 1 or grid[er][ec] == 1:
        return -1

    queue = deque([(sr, sc, 0)])    # (row, col, steps)
    visited = {(sr, sc)}

    while queue:
        r, c, steps = queue.popleft()

        if r == er and c == ec:
            return steps

        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if (0 <= nr < ROWS and 0 <= nc < COLS
                    and (nr, nc) not in visited
                    and grid[nr][nc] == 0):
                visited.add((nr, nc))
                queue.append((nr, nc, steps + 1))

    return -1  # no path found
```

## Path reconstruction (if you need the actual path, not just the length)
```python
from collections import deque

def solve_maze_with_path(grid, start, end):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]
    sr, sc = start
    er, ec = end

    queue = deque([(sr, sc)])
    visited = {(sr, sc): None}   # cell → previous cell (for path reconstruction)

    while queue:
        r, c = queue.popleft()
        if r == er and c == ec:
            # Reconstruct path
            path = []
            curr = (er, ec)
            while curr is not None:
                path.append(curr)
                curr = visited[curr]
            return path[::-1]  # reverse: start → end

        for dr, dc in DIRS:
            nr, nc = r + dr, c + dc
            if (0 <= nr < ROWS and 0 <= nc < COLS
                    and (nr, nc) not in visited
                    and grid[nr][nc] == 0):
                visited[(nr, nc)] = (r, c)
                queue.append((nr, nc))

    return []  # no path
```

## Complexity
Time: O(R×C) | Space: O(R×C)

## Key insight
BFS explores all cells at distance d before any cell at distance d+1. The first time BFS reaches the end cell is guaranteed to be via the shortest path.

For path reconstruction: store the parent of each cell in the visited dict. Trace back from end to start using parent pointers.

## Variants / follow-ups

### Maze with keys and doors (state includes keys held)
```python
# State = (r, c, frozenset of keys collected)
queue = deque([(sr, sc, frozenset())])
visited = set([(sr, sc, frozenset())])
```

### Maze with limited wall removals (LeetCode 1293)
```python
# State = (r, c, remaining_removals)
# More removals remaining = different state even at same position
queue = deque([(sr, sc, k, 0)])   # k = max walls removable
visited = set([(sr, sc, k)])
```

### Moving through a maze with teleporters
```python
# Add teleporter exits as neighbors with 0 additional cost → BFS still works
```

## Sources
- [[dsa/grid/patterns/bfs-shortest-path]]
- [[dsa/grid/concepts/grid-as-graph]]
