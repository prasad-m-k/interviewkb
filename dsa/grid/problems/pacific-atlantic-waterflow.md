# Pacific Atlantic Water Flow

**Difficulty:** Medium
**Pattern:** Multi-source BFS/DFS from boundaries
**Topic:** [[Grid overview]]
**Companies:** Facebook, Google, Amazon

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given a grid of heights, rain water flows from a cell to adjacent cells (4-directional) if the adjacent cell's height is ≤ current cell's height. The Pacific Ocean borders the top/left; the Atlantic borders bottom/right. Return all cells from which water can flow to BOTH oceans.

## Approach
**Key insight (reverse thinking):** Instead of simulating water flowing DOWN from each cell (expensive), simulate water flowing UP from the oceans. Start BFS/DFS from all Pacific border cells and all Atlantic border cells. Mark which cells can reach each ocean. Answer = intersection.

```
Pacific: (r=0, any c) and (any r, c=0)
Atlantic: (r=ROWS-1, any c) and (any r, c=COLS-1)
```

Water flows up = move to neighbors with height ≥ current height.

## Solution (Python)
```python
from collections import deque

def pacificAtlantic(heights):
    ROWS, COLS = len(heights), len(heights[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    def bfs(sources):
        queue = deque(sources)
        reachable = set(sources)
        while queue:
            r, c = queue.popleft()
            for dr, dc in DIRS:
                nr, nc = r + dr, c + dc
                if (0 <= nr < ROWS and 0 <= nc < COLS
                        and (nr, nc) not in reachable
                        and heights[nr][nc] >= heights[r][c]):  # water flows UP
                    reachable.add((nr, nc))
                    queue.append((nr, nc))
        return reachable

    pacific_sources = (
        [(0, c) for c in range(COLS)] +
        [(r, 0) for r in range(1, ROWS)]
    )
    atlantic_sources = (
        [(ROWS-1, c) for c in range(COLS)] +
        [(r, COLS-1) for r in range(ROWS-1)]
    )

    pacific = bfs(pacific_sources)
    atlantic = bfs(atlantic_sources)

    return [list(cell) for cell in pacific & atlantic]
```

## Complexity
Time: O(R×C) | Space: O(R×C)

## Key insight
The reversal trick is the whole problem. Asking "which cells drain to the Pacific?" is hard (you'd need to simulate from every cell). Asking "which cells can the Pacific reach going uphill?" is easy — BFS from the Pacific border going only uphill.

## Variants / follow-ups
- "Water flows to exactly one ocean": cells in pacific XOR atlantic
- "Cells that flow to neither ocean": all cells minus (pacific union atlantic)

## Sources
- [[dsa/grid/patterns/bfs-shortest-path]]
- [[Grid overview]]
