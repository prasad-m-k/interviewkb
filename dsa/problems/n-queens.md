# N-Queens

**Difficulty:** Hard
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/backtracking]]
**Companies:** Google, Amazon, Microsoft, Meta

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Place `n` queens on an `n × n` chessboard so that no two queens attack each other (no two queens share a row, column, or diagonal). Return all distinct solutions. Each solution is a list of strings where `'Q'` marks a queen and `'.'` marks an empty cell.

```
n = 4 → two solutions:
. Q . .     . . Q .
. . . Q     Q . . .
Q . . .     . . . Q
. . Q .     . Q . .
```

**Variant — N-Queens II (LC 52):** return just the count of solutions (same algorithm, skip board construction).

## Approach

Place queens one row at a time. For each row, try every column. Skip any column where a conflict exists. Conflict = another queen in the same column, same `\` diagonal, or same `/` diagonal.

**Key insight:** since we place exactly one queen per row, row conflicts are impossible by construction. We only need to track three constraint sets:

| Set | Tracks | Key formula |
|---|---|---|
| `cols` | occupied columns | `col` |
| `diag1` | `\` diagonals (top-left → bottom-right) | `row - col` (constant along `\`) |
| `diag2` | `/` diagonals (top-right → bottom-left) | `row + col` (constant along `/`) |

These sets make conflict-checking O(1) per placement, and undo is just a `set.remove`.

## Solution (Python)

```python
def solveNQueens(n: int) -> list[list[str]]:
    solutions = []
    placement = [0] * n      # placement[row] = column of queen in that row

    cols  = set()
    diag1 = set()             # row - col
    diag2 = set()             # row + col

    def backtrack(row: int) -> None:
        if row == n:
            solutions.append(build(placement, n))
            return
        for col in range(n):
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue
            # CHOOSE
            cols.add(col); diag1.add(row - col); diag2.add(row + col)
            placement[row] = col
            # EXPLORE
            backtrack(row + 1)
            # UNCHOOSE
            cols.remove(col); diag1.remove(row - col); diag2.remove(row + col)

    backtrack(0)
    return solutions

def build(placement: list[int], n: int) -> list[str]:
    return ["." * c + "Q" + "." * (n - c - 1) for c in placement]
```

### N-Queens II (count only)

```python
def totalNQueens(n: int) -> int:
    count = [0]
    cols = set(); diag1 = set(); diag2 = set()

    def backtrack(row):
        if row == n:
            count[0] += 1
            return
        for col in range(n):
            if col in cols or (row-col) in diag1 or (row+col) in diag2:
                continue
            cols.add(col); diag1.add(row-col); diag2.add(row+col)
            backtrack(row + 1)
            cols.remove(col); diag1.remove(row-col); diag2.remove(row+col)

    backtrack(0)
    return count[0]
```

## Why `row - col` and `row + col`?

```
Board (row, col):
(0,0) (0,1) (0,2) (0,3)
(1,0) (1,1) (1,2) (1,3)
(2,0) (2,1) (2,2) (2,3)
(3,0) (3,1) (3,2) (3,3)

\ diagonal: (0,0),(1,1),(2,2),(3,3)  → row-col = 0,0,0,0  ✓ same value
/ diagonal: (0,3),(1,2),(2,1),(3,0)  → row+col = 3,3,3,3  ✓ same value
```

All cells on the same `\` share `row - col`. All cells on the same `/` share `row + col`. Storing these sums in a set is the O(1) conflict check.

## Traced execution (n=4)

```
row=0: try col=0 → place Q at (0,0); cols={0}, d1={0}, d2={0}
  row=1: col=0 → conflict (cols); col=1 → conflict (d2: 0+0=0, 1+1=2 — ok? 
         wait: d2={0+0=0}, 1+1=2 not in d2 → but d1: 0-0=0, 1-1=0 → conflict!)
         col=2 → place Q at (1,2); cols={0,2}, d1={0,-1}, d2={0,3}
    row=2: col=0→d2 conflict(2); col=1→d1 conflict(-1); col=2→cols conflict; col=3→place
           Q at (2,3); cols={0,2,3}, d1={0,-1,0}→col 0 d1=0, conflict!
           → backtrack
    no valid col for row=2 → backtrack from row=1 col=2
  row=1: col=3 → place Q at (1,3); cols={0,3}, d1={0,-2}, d2={0,4}
    row=2: col=1 → place Q at (2,1); cols={0,1,3}, d1={0,-2,1}, d2={0,4,3}
      row=3: col=2 → d1: 3-2=1 → conflict; col=0,1,3 → cols conflict
      → backtrack
    col=2 → d1: 2-2=0 → conflict ...
    → backtrack from row=1 col=3
→ backtrack from row=0 col=0
row=0: col=1 → ... (eventually finds solution: [1,3,0,2])
```

## Complexity

Time: O(n!) — upper bound; pruning reduces actual nodes explored significantly  
Space: O(n) — recursion depth n, three sets each holding ≤ n elements

## Key insight

One queen per row by construction eliminates row conflicts entirely. Diagonal conflicts reduce to checking two integer sums — no board scanning needed. This turns an O(n) validity check into O(1) with three sets.

## Variants / follow-ups

- **Count only** → N-Queens II (LC 52): skip `build()`, just increment a counter
- **How to print one solution fast?** Stop at the first valid placement instead of collecting all
- **Why does `row - col` work for `\` diagonals?** All squares on the same `\` diagonal have equal `row - col` (both increase by 1 as you move down-right)
- **Can you solve it iteratively?** Yes — explicit stack replacing the call stack, but recursion is cleaner and standard in interviews
- **Bit manipulation version:** represent `cols`, `diag1`, `diag2` as integers; find valid columns with `valid = ~(cols | diag1 | diag2) & ((1<<n)-1)`. Cuts constant factors but adds complexity
- **Generalize to N-Rooks:** remove diagonal constraints — trivially solved in O(n) (place on diagonal)

## Sources

- [[dsa/patterns/backtracking]]
