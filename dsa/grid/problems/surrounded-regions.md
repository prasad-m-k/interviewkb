# Surrounded Regions

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/grid/patterns/flood-fill-dfs]]
**Companies:** [[dsa/companies/google]], [[dsa/companies/meta]]

## Problem

Given an `m x n` matrix `board` containing `'X'` and `'O'`, capture all regions surrounded by `'X'`.

A region is **captured** by flipping all `'O'`s into `'X'`s in that surrounded region. A region is **not captured** if any `'O'` in it is on the **border** of the board, or is connected (4-directionally) to a border `'O'`.

```
Input:
X X X X
X O O X
X X O X
X O X X

Output:
X X X X
X X X X
X X X X
X O X X
```

The bottom-left `O` is not captured because it touches the border.

## Key Insight: Invert the Problem

**Wrong approach:** Try to find which `O` regions are *surrounded*. This requires checking whether a region touches the border, which is hard to determine during DFS without global knowledge.

**Right approach:** Find which `O` regions are *not* surrounded (i.e., touching the border) and protect them. Capture everything else.

Algorithm:
1. DFS/BFS from every border `'O'` — temporarily mark all connected `'O'`s as `'S'` (safe).
2. Flip all remaining `'O'` → `'X'` (they are surrounded).
3. Restore all `'S'` → `'O'` (they were safe all along).

## Solution (Python)

```python
def solve(board: list[list[str]]) -> None:
    """Modifies board in-place."""
    if not board:
        return

    rows, cols = len(board), len(board[0])

    def dfs(r: int, c: int) -> None:
        if r < 0 or r >= rows or c < 0 or c >= cols or board[r][c] != 'O':
            return
        board[r][c] = 'S'   # mark as safe
        dfs(r+1, c); dfs(r-1, c)
        dfs(r, c+1); dfs(r, c-1)

    # Step 1: Mark all border-connected O's as safe
    for r in range(rows):
        for c in range(cols):
            if (r == 0 or r == rows-1 or c == 0 or c == cols-1) and board[r][c] == 'O':
                dfs(r, c)

    # Step 2: Flip O → X (surrounded), S → O (safe)
    for r in range(rows):
        for c in range(cols):
            if board[r][c] == 'O':
                board[r][c] = 'X'
            elif board[r][c] == 'S':
                board[r][c] = 'O'
```

## Solution (BFS — avoids Python recursion limit for large boards)

```python
from collections import deque

def solve_bfs(board: list[list[str]]) -> None:
    if not board:
        return
    rows, cols = len(board), len(board[0])

    queue = deque()
    for r in range(rows):
        for c in range(cols):
            if (r == 0 or r == rows-1 or c == 0 or c == cols-1) and board[r][c] == 'O':
                queue.append((r, c))
                board[r][c] = 'S'

    while queue:
        r, c = queue.popleft()
        for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and board[nr][nc] == 'O':
                board[nr][nc] = 'S'
                queue.append((nr, nc))

    for r in range(rows):
        for c in range(cols):
            board[r][c] = 'X' if board[r][c] != 'S' else 'O'
```

## Complexity

| | Value |
|---|---|
| Time | O(rows × cols) |
| Space | O(rows × cols) — recursion depth or queue |

## Key Insight (Summary)

The problem can't be solved with a single inside-out pass because you don't know if an `O` region is surrounded until you've explored the whole region and checked borders. The inversion trick — start from borders and work inward — converts "which regions touch the border" from a global query into a simple outward flood fill.

This **"mark from boundary, then sweep"** pattern reappears in:
- Counting cells reachable from both sides (Pacific Atlantic Water Flow)
- Identify unreachable nodes in a graph
- Image segmentation problems

## Variants / Follow-ups

- **Pacific Atlantic Water Flow:** Two separate boundary flood fills (Pacific border, Atlantic border), then intersect. [[dsa/grid/problems/pacific-atlantic-waterflow]]
- "What if diagonals also form connections?" → Add 4 diagonal directions to DFS.
- "What if the board uses different characters?" → Same algorithm, parametrize the target character.
- "Return the count of captured regions?" → Count connected components of captured cells (DFS again after the solve pass).

## Sources
- [[dsa/companies/google]]
- [[dsa/companies/meta]]
