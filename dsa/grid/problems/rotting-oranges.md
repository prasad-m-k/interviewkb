# Rotting Oranges

**Difficulty:** Medium
**Pattern:** [[dsa/grid/patterns/bfs-shortest-path]] (multi-source BFS + time simulation)
**Topic:** [[Grid overview]]
**Companies:** Amazon, Facebook, Microsoft

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Grid cells contain: 0 (empty), 1 (fresh orange), 2 (rotten orange). Every minute, any fresh orange adjacent to a rotten orange becomes rotten. Return the minimum minutes until no fresh orange remains, or -1 if impossible.

## Approach
Multi-source BFS starting from all initially rotten oranges. BFS naturally simulates time — each level = one minute. Count fresh oranges before; decrement as they rot. If any fresh remain after BFS, return -1.

## Solution (Python)
```python
from collections import deque

def orangesRotting(grid):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    queue = deque()
    fresh = 0

    for r in range(ROWS):
        for c in range(COLS):
            if grid[r][c] == 2:
                queue.append((r, c))
            elif grid[r][c] == 1:
                fresh += 1

    if fresh == 0:
        return 0

    minutes = 0
    while queue:
        # Process all oranges that rotted in the current minute
        for _ in range(len(queue)):
            r, c = queue.popleft()
            for dr, dc in DIRS:
                nr, nc = r + dr, c + dc
                if (0 <= nr < ROWS and 0 <= nc < COLS
                        and grid[nr][nc] == 1):
                    grid[nr][nc] = 2
                    fresh -= 1
                    queue.append((nr, nc))
        minutes += 1

    return minutes - 1 if fresh == 0 else -1
```

## Complexity
Time: O(R×C) | Space: O(R×C)

## Key insight
BFS processes level by level. Each "level" = one minute. Process all rotten oranges from the current minute before moving to the next. Use `len(queue)` at the start of each BFS level to separate levels cleanly.

The `minutes - 1` correction: after the last batch of oranges rot, we increment `minutes` one extra time for the loop iteration where no new oranges rot.

**Alternative**: store time explicitly: `queue.append((r, c, time))` and track `max_time`.

## Variants / follow-ups
- What if some fresh oranges are isolated? (return -1 — check `fresh == 0` at the end)
- What if oranges rot in 8 directions? (change DIRS to 8-directional)

## Sources
- [[dsa/grid/patterns/bfs-shortest-path]]
- [[dsa/grid/patterns/simulation]]
