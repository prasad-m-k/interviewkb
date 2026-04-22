# Grid Problems — Anki Deck

> Cards for [[dsa/grid/index]]. Sync with Obsidian_to_Anki plugin.
> Deck: `DSA::Grid`

---

## Foundations

START
Basic
What are the **4-directional** and **8-directional** direction vectors in Python? When do you use each?
Back:
```python
# 4-directional: up/down/left/right
DIRS_4 = [(0,1),(0,-1),(1,0),(-1,0)]

# 8-directional: + diagonals
DIRS_8 = [(0,1),(0,-1),(1,0),(-1,0),(1,1),(1,-1),(-1,1),(-1,-1)]
```
- **4-dir**: standard grid traversal, island counting, maze, BFS shortest path
- **8-dir**: diagonal movement allowed (word search in 8 directions, chess pieces, Game of Life neighbors, shortest path in binary matrix)
Tags: grid foundations direction-vectors
END

START
Basic
What are the **three visited-tracking strategies** for grid DFS/BFS? When do you use each?
Back:
1. **Separate boolean array** `visited = [[False]*C for _ in range(R)]`
   → When you can't modify the grid; O(R×C) extra space
2. **In-place marking** `grid[r][c] = '0'` (or sentinel value)
   → When modifying input is OK; O(1) extra; classic for flood fill
3. **Backtracking (unmark after recursion)** `temp = grid[r][c]; grid[r][c] = '#'; ...; grid[r][c] = temp`
   → When different paths need different visited states; required for word search
Tags: grid foundations visited-tracking
END

START
Basic
What is the **most common BFS bug** for grids? How do you fix it?
Back:
**Marking visited when dequeuing (too late) instead of when enqueuing.**

```python
# WRONG: same cell can be added to queue multiple times
queue.append((nr, nc))
# ... later ...
visited.add((nr, nc))  # too late

# CORRECT: mark before adding
if (nr, nc) not in visited:
    visited.add((nr, nc))   # mark NOW
    queue.append((nr, nc))
```
Without the fix: same cell is enqueued multiple times → O(R²C²) instead of O(RC), and BFS may not terminate on large grids.
Tags: grid bfs bugs
END

START
Basic
Write the **in-bounds check** for a grid. What do you check?
Back:
```python
ROWS, COLS = len(grid), len(grid[0])

def in_bounds(r, c):
    return 0 <= r < ROWS and 0 <= c < COLS

# Combined check (short-circuits left-to-right in Python):
if 0 <= nr < ROWS and 0 <= nc < COLS and grid[nr][nc] == target:
    ...
```
Always check bounds BEFORE accessing `grid[nr][nc]` — Python's `and` short-circuits, so if bounds fail, `grid[nr][nc]` is never evaluated (no IndexError).
Tags: grid foundations bounds-checking
END

---

## BFS vs DFS

START
Basic
Why must you use **BFS (not DFS)** for shortest path in a grid?
Back:
DFS finds **a** path, not the **shortest** path. DFS may go the long way around.

BFS expands level by level:
- All cells at distance 1 are processed before any cell at distance 2
- The first time BFS reaches the target, it's via the shortest path (in unweighted graphs)

**Rule**: need shortest path → BFS. Need connectivity/counting → DFS (simpler code).
Tags: grid bfs dfs shortest-path
END

START
Basic
What is **multi-source BFS**? Name three problems that use it.
Back:
Initialize the BFS queue with **all source cells simultaneously** at distance 0. BFS then finds the shortest distance from any source to every other cell.

```python
queue = deque()
for r in range(ROWS):
    for c in range(COLS):
        if is_source(grid[r][c]):
            queue.append((r, c))
            dist[r][c] = 0
# Then: standard BFS
```

Problems:
1. **01 Matrix**: distance to nearest 0 — all 0s are sources
2. **Rotting Oranges**: time until all fresh rot — all rotten oranges are sources
3. **Walls and Gates**: fill each room with distance to nearest gate — all gates are sources
Tags: grid bfs multi-source
END

---

## Patterns

START
Basic
Give the **flood fill DFS template** for counting islands. What is the time/space complexity?
Back:
```python
def numIslands(grid):
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    def dfs(r, c):
        if not (0 <= r < ROWS and 0 <= c < COLS) or grid[r][c] != '1':
            return
        grid[r][c] = '0'   # mark visited in-place
        for dr, dc in DIRS:
            dfs(r+dr, c+dc)

    count = 0
    for r in range(ROWS):
        for c in range(COLS):
            if grid[r][c] == '1':
                dfs(r, c)
                count += 1
    return count
```
Time: O(R×C) — each cell visited once
Space: O(R×C) — worst-case recursion depth (one big island)
Tags: grid dfs flood-fill islands
END

START
Basic
What are the **4 steps of spiral matrix traversal** and what guards prevent double-counting?
Back:
Four steps per layer:
1. → right along `top` row; then `top += 1`
2. ↓ down along `right` col; then `right -= 1`
3. ← left along `bottom` row; then `bottom -= 1` **(guard: `if top <= bottom`)**
4. ↑ up along `left` col; then `left += 1` **(guard: `if left <= right`)**

The two guards prevent re-traversing a single remaining row or column when the matrix isn't square (e.g., a 1×5 matrix — step 3 would re-process the top row without the guard).
Tags: grid spiral traversal
END

START
Basic
What is the **grid DP recurrence** for counting unique paths and minimum path sum?
Back:
```python
# Unique paths (count): each cell = ways from left + ways from above
dp[r][c] = dp[r-1][c] + dp[r][c-1]
# Base: dp[0][*] = 1, dp[*][0] = 1

# Minimum path sum: each cell = min of above/left + cell cost
dp[r][c] = min(dp[r-1][c], dp[r][c-1]) + grid[r][c]
# Base: dp[0][0] = grid[0][0]; fill first row and col separately
```
Both can be optimized to O(C) space by using a single 1D dp array (update left-to-right; `dp[c] += dp[c-1]` for counting).
Tags: grid dp dynamic-programming unique-paths
END

START
Basic
**Word Search**: why do you need to unmark cells after DFS, and when would NOT unmarking break the solution?
Back:
Unmarking (backtracking) is required because different search paths starting from other cells may need to pass through this cell.

Example:
```
board = [['A','B'],['C','A']]
word = "ABCA"
```
Path: (0,0)→(0,1)→(1,1)→... needs to revisit 'A' at a different position, but not the same cell.

**If you don't unmark**: a cell marked as visited in one path stays marked for ALL paths. The search would miss valid paths that share cells with dead-end paths.

**Contrast with flood fill**: in flood fill you never want to revisit, so you don't unmark. In word search, you want to reuse cells across different paths, so you must unmark.
Tags: grid dfs backtracking word-search
END

---

## Harder Problems

START
Basic
**Pacific Atlantic Water Flow** — what is the key insight that makes this solvable efficiently?
Back:
**Reverse the direction of flow.**

Instead of asking "can water at cell X flow to the Pacific?" (requires simulating downhill flow, expensive), ask "can the Pacific reach cell X going uphill?".

BFS from all Pacific border cells, moving to neighbors with height ≥ current (water flowing uphill).
Repeat from all Atlantic border cells.
Answer = intersection of the two reachable sets.

This converts two expensive forward-simulations into two cheap backward-BFS traversals.
Time: O(R×C) total.
Tags: grid bfs reversal-trick pacific-atlantic
END

START
Basic
**Swim in Rising Water** — what algorithm do you use and what does the "cost" represent?
Back:
**Dijkstra** (min-heap BFS). The cost of reaching a cell is the **maximum elevation** encountered on the path so far (not the sum).

```python
heap = [(grid[0][0], 0, 0)]   # (max_elevation, row, col)
while heap:
    t, r, c = heapq.heappop(heap)
    if r == n-1 and c == n-1: return t
    for dr, dc in DIRS:
        nr, nc = r+dr, c+dc
        if in_bounds and (nr,nc) not in visited:
            new_t = max(t, grid[nr][nc])
            heapq.heappush(heap, (new_t, nr, nc))
```

This is "minimum bottleneck path" — a classic variant of Dijkstra where cost = max edge weight on path, not sum.
Tags: grid dijkstra swim-in-water weighted
END

START
Basic
**Dungeon Game** — why must you solve it backwards? What does `dp[r][c]` represent?
Back:
`dp[r][c]` = **minimum HP needed to enter cell (r,c) and survive to the exit**.

Going forward is wrong: at each cell you'd need to track both minimum and maximum HP (too much state). Going backward works because you know exactly what's needed from the destination.

```python
# At exit: need max(1, 1 - dungeon[r][c])
# At other cells: need max(1, min(dp[r+1][c], dp[r][c+1]) - dungeon[r][c])
# min() because knight chooses the better next cell
# max(1, ...) because HP must be at least 1 to be alive
```

Key: the `max(1, ...)` ensures you always need at least 1 HP to be alive when entering the cell.
Tags: grid dp dungeon-game reverse
END

---

## Complexity Reference

START
Basic
What is the time and space complexity for the five main grid patterns?
Back:
| Pattern | Time | Space |
|---|---|---|
| DFS (flood fill, connectivity) | O(R×C) | O(R×C) stack |
| BFS (shortest path, multi-source) | O(R×C) | O(R×C) queue |
| DP (counting, min/max path) | O(R×C) | O(C) with optimization |
| Spiral traversal | O(R×C) | O(1) |
| Dijkstra on grid | O(R×C log RC) | O(R×C) |

All standard grid traversals are O(R×C) time — each cell visited at most once.
Tags: grid complexity
END
