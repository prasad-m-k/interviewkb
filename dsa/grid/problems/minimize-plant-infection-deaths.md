# Minimize Deaths in a Spreading Plant Infection

**Difficulty:** Hard
**Topic:** [[Grid overview]]
**Pattern:** [[dsa/grid/patterns/bfs-shortest-path]] (multi-source BFS + greedy simulation)
**Companies:** Google, Amazon

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

A grid contains three cell types:
- `0` — empty
- `1` — healthy plant
- `2` — infected plant

Each **day** the following happens in order:
1. You may **quarantine exactly one** connected infected region — build walls around it so it can never spread again.
2. Every **unquarantined** infected cell spreads to all 4-adjacent healthy cells.

Return the **minimum number of healthy plants** that get infected before all infection is contained.

```
Initial grid:
2 1 1 1 1
1 1 0 0 1
0 1 1 1 1
0 0 0 0 1
1 1 0 0 2

Answer: 3   (quarantine the large top cluster on day 1; the small bottom-right cluster
             infects 3 nearby plants before it's fully contained on day 2)
```

## Key Observations

1. **Greedy choice:** each day, quarantine the infected component that threatens the **most reachable healthy plants**. Saving the largest group first is always optimal — a component you don't quarantine spreads one more wave before you can act.

2. **"Threatened" ≠ immediate frontier.** A component threatens not just its direct neighbors but the entire connected healthy region reachable from those neighbors (infection will flood-fill through healthy cells over future days). BFS through healthy cells from the frontier to count the full threatened set.

3. **Termination:** stop when every remaining infected component has an empty frontier (all neighbors are empty, quarantined, or already infected).

## Algorithm

```
repeat:
    BFS to find all infected connected components
    for each component:
        BFS from its frontier through healthy cells → count threatened plants
    quarantine the component with the highest threatened count (mark as 3)
    spread all other components: their frontier cells (1→2), accumulate deaths
until no component has a frontier
```

## Solution (Python)

```python
from collections import deque

def minimizeDeaths(grid: list[list[int]]) -> int:
    ROWS, COLS = len(grid), len(grid[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    def find_components():
        """
        Returns list of (infected_cells, frontier_cells, threatened_count).
        threatened_count = all healthy cells reachable from the frontier
        through healthy-cell adjacency (the full flood-fill the infection
        would eventually reach).
        """
        seen = set()
        components = []
        for r in range(ROWS):
            for c in range(COLS):
                if grid[r][c] != 2 or (r, c) in seen:
                    continue
                comp, frontier = [], set()
                q = deque([(r, c)])
                seen.add((r, c))
                while q:
                    cr, cc = q.popleft()
                    comp.append((cr, cc))
                    for dr, dc in DIRS:
                        nr, nc = cr + dr, cc + dc
                        if not (0 <= nr < ROWS and 0 <= nc < COLS):
                            continue
                        if grid[nr][nc] == 2 and (nr, nc) not in seen:
                            seen.add((nr, nc))
                            q.append((nr, nc))
                        elif grid[nr][nc] == 1:
                            frontier.add((nr, nc))

                # BFS through healthy cells to count full threatened region
                threatened = set(frontier)
                hq = deque(frontier)
                while hq:
                    fr, fc = hq.popleft()
                    for dr, dc in DIRS:
                        nr, nc = fr + dr, fc + dc
                        if (0 <= nr < ROWS and 0 <= nc < COLS
                                and grid[nr][nc] == 1
                                and (nr, nc) not in threatened):
                            threatened.add((nr, nc))
                            hq.append((nr, nc))

                components.append((comp, list(frontier), len(threatened)))
        return components

    deaths = 0

    while True:
        components = find_components()
        spreading = [c for c in components if c[1]]  # only those with a frontier
        if not spreading:
            break

        # Quarantine the most dangerous component
        spreading.sort(key=lambda x: -x[2])
        quarantine_cells = spreading[0][0]

        # Spread all other components (frontier healthy cells become infected)
        for _, frontier, _ in spreading[1:]:
            for r, c in frontier:
                if grid[r][c] == 1:   # guard: two components can share a frontier cell
                    grid[r][c] = 2
                    deaths += 1

        # Mark quarantined region as 3 — permanently contained, ignored in future rounds
        for r, c in quarantine_cells:
            grid[r][c] = 3

    return deaths
```

## Traced Example

```
Day 0 initial:          Day 1 after quarantine+spread:
2 1 1 1 1               3 1 1 1 1   ← top cluster quarantined (3)
1 1 0 0 1               1 1 0 0 1
0 1 1 1 1               0 1 1 1 1
0 0 0 0 1               0 0 0 0 2   ← bottom-right spread (+1 death)
1 1 0 0 2               1 1 0 0 2

Day 2: quarantine bottom-right; no others spread → 0 more deaths
Total deaths = 1... (exact count depends on the grid connectivity)
```

## Complexity

Time: O((R×C)²) — up to R×C days, each doing an O(R×C) BFS scan  
Space: O(R×C) — BFS queues and seen sets

> In practice, days ≤ number of initially healthy cells, and most grids terminate much faster.

## Key insight

Two BFS passes per component per day:
1. **Component BFS** — find all cells in each infected island (4-connectivity on `2`s).
2. **Threat BFS** — flood-fill from the frontier through `1`s to count the total healthy region at risk.

The greedy — quarantine max-threatened first — is correct because each unchecked component spreads exactly once per round. Saving the largest threatened group first always dominates any other ordering.

## Common mistakes

- **Forgetting the threat BFS.** Using `len(frontier)` instead of full reachable-healthy count makes the greedy wrong — a small frontier can still threaten a huge connected region.
- **Shared frontier cells.** Two components can have the same healthy cell as a frontier neighbor. Guard with `if grid[r][c] == 1` before converting to avoid double-counting deaths.
- **Not marking quarantined cells (`3`).** If you leave them as `2`, they get re-discovered and re-quarantined every day, wasting your quarantine budget.

## Variants / follow-ups

- **Count the walls built** (not deaths): add `len(frontier_of_quarantined)` each day — those are the wall segments placed.
- **Quarantine k regions per day:** sort and quarantine the top-k instead of top-1.
- **Infection spreads in 8 directions:** change `DIRS` to 8-directional.
- **Some regions are unreachable (isolated):** the algorithm handles this naturally — isolated `2`s have an empty frontier and are skipped.

## Sources

- [[dsa/grid/patterns/bfs-shortest-path]]
- [[Grid overview]]
