# Sudoku Solver

**Difficulty:** Hard
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/backtracking]]
**Companies:** Google, Amazon, Microsoft, Uber

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Fill a partially complete 9×9 Sudoku board (`'.'` = empty, `'1'`–`'9'` = given) so that every row, column, and 3×3 box contains the digits 1–9 exactly once. The puzzle is guaranteed to have a unique solution.

## Why it's the N-Queens sibling

| N-Queens | Sudoku |
|---|---|
| One queen per row (row constraint free by construction) | Iterate over empty cells explicitly |
| Track `cols`, `diag1`, `diag2` sets | Track `rows[r]`, `cols[c]`, `boxes[r//3][c//3]` sets |
| Backtrack on column choice | Backtrack on digit choice |
| O(1) conflict check | O(1) conflict check |

The exact same idea — maintain constraint sets, check in O(1), add/remove on recurse/backtrack — scaled from 1D (column + two diagonals) to 2D (row + column + box).

## Approach

1. **Pre-populate** constraint sets from the given digits.
2. Collect all empty cells.
3. Recurse: for each empty cell, try digits 1–9; skip any already in that row/col/box; place, recurse, undo.
4. **Most-constrained-first heuristic:** sort empty cells by the number of valid candidates (fewest first). Cells with only 1 candidate should be filled immediately — this cuts the branching factor from ~5 on average to much lower.

## Solution (Python)

```python
def solveSudoku(board: list[list[str]]) -> None:
    rows  = [set() for _ in range(9)]
    cols  = [set() for _ in range(9)]
    boxes = [[set() for _ in range(3)] for _ in range(3)]
    empty = []

    # Pre-populate constraint sets
    for r in range(9):
        for c in range(9):
            d = board[r][c]
            if d != '.':
                rows[r].add(d); cols[c].add(d); boxes[r//3][c//3].add(d)
            else:
                empty.append((r, c))

    def candidates(r, c):
        used = rows[r] | cols[c] | boxes[r//3][c//3]
        return [str(d) for d in range(1, 10) if str(d) not in used]

    def backtrack(idx: int) -> bool:
        if idx == len(empty):
            return True                    # all cells filled

        r, c = empty[idx]
        for d in candidates(r, c):
            # CHOOSE
            board[r][c] = d
            rows[r].add(d); cols[c].add(d); boxes[r//3][c//3].add(d)
            # EXPLORE
            if backtrack(idx + 1):
                return True
            # UNCHOOSE
            board[r][c] = '.'
            rows[r].remove(d); cols[c].remove(d); boxes[r//3][c//3].remove(d)

        return False                       # no digit worked → trigger backtrack

    backtrack(0)
```

### With most-constrained-cell heuristic

```python
def solveSudokuFast(board: list[list[str]]) -> None:
    rows  = [set() for _ in range(9)]
    cols  = [set() for _ in range(9)]
    boxes = [[set() for _ in range(3)] for _ in range(3)]

    for r in range(9):
        for c in range(9):
            d = board[r][c]
            if d != '.':
                rows[r].add(d); cols[c].add(d); boxes[r//3][c//3].add(d)

    def candidates(r, c):
        used = rows[r] | cols[c] | boxes[r//3][c//3]
        return [str(d) for d in range(1, 10) if str(d) not in used]

    def backtrack() -> bool:
        # Pick the empty cell with the fewest candidates (most constrained first)
        best, best_cands = None, None
        for r in range(9):
            for c in range(9):
                if board[r][c] == '.':
                    cands = candidates(r, c)
                    if not cands:
                        return False       # dead end — no valid digit here
                    if best is None or len(cands) < len(best_cands):
                        best, best_cands = (r, c), cands

        if best is None:
            return True                    # no empty cells left → solved

        r, c = best
        for d in best_cands:
            board[r][c] = d
            rows[r].add(d); cols[c].add(d); boxes[r//3][c//3].add(d)
            if backtrack():
                return True
            board[r][c] = '.'
            rows[r].remove(d); cols[c].remove(d); boxes[r//3][c//3].remove(d)

        return False

    backtrack()
```

## Constraint Set Visualization

```
Row r:    all 9 cells in the same row
Col c:    all 9 cells in the same column
Box:      which 3×3 box? → (r//3, c//3)

For cell (4, 7):
  row  = rows[4]
  col  = cols[7]
  box  = boxes[4//3][7//3] = boxes[1][2]
```

## Complexity

Time: O(9^m) where m = number of empty cells, but with pruning this is drastically reduced — well-formed puzzles typically require near-zero backtracking with the most-constrained heuristic.  
Space: O(m) recursion depth + O(1) fixed constraint sets (81 cells total)

## Key insight

The `boxes[r//3][c//3]` indexing is the one thing to get right — integer division maps all 9 cells of each 3×3 box to the same `(box_r, box_c)` pair. Everything else is the same as N-Queens: add on choose, remove on unchoose.

## Variants / follow-ups

- **Check validity only (LC 36):** no backtracking needed — just build the three sets and check for duplicates
- **Count solutions:** change `return True` to `count += 1; return False` to keep searching
- **Larger boards (16×16, etc.):** same algorithm — parameterize grid size and box size
- **Why most-constrained first?** Cells forced to a single digit are essentially free (no branching). Filling them first shrinks the remaining problem before any branching happens. A puzzle with 17 given clues (minimum) is typically solved with < 100 node visits using this heuristic vs. millions without

## Sources

- [[dsa/patterns/backtracking]]
- [[dsa/problems/n-queens]]
