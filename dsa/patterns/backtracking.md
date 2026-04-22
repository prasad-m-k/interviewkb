# Pattern: Backtracking

**Topic:** [[dsa/topics/algorithms]]
**Related concepts:** [[dsa/concepts/graph]], [[dsa/concepts/dynamic-programming]]

## What it solves

Problems that require exploring all (or some) combinations/arrangements of choices subject to constraints, where invalid branches can be pruned early. The output is usually one or all valid configurations.

Classic signals:
- "Find all arrangements / combinations / subsets..."
- "Place N pieces on a board such that..."
- "Fill in a grid satisfying rules..."
- "Generate all valid..."

---

## The Three-Step Template

Every backtracking solution has the same skeleton:

```python
def backtrack(state):
    if is_complete(state):
        record(state)          # base case: solution found
        return

    for choice in get_choices(state):
        if is_valid(state, choice):
            make(state, choice)        # 1. CHOOSE
            backtrack(state)           # 2. EXPLORE
            undo(state, choice)        # 3. UNCHOOSE  ← the defining step
```

**The undo step is what makes it backtracking.** Without it you have plain DFS that corrupts state.

---

## Constraint Tracking: The N-Queens Insight

Naive backtracking checks validity by scanning existing placements — O(n) per step. For grid/board problems, pre-track constraints in sets for O(1) checks:

| Constraint | Key | Example |
|---|---|---|
| Column conflict | `col` | N-Queens: one queen per column |
| `\` diagonal | `row - col` | constant on each `\` diagonal |
| `/` diagonal | `row + col` | constant on each `/` diagonal |
| Row conflict | `row` | Sudoku: one digit per row |
| Box conflict | `(row//3, col//3)` | Sudoku: one digit per 3×3 box |
| Path visited | `(row, col)` | Word Search: don't reuse a cell |

This pattern generalizes to any constraint that can be expressed as a hashable key.

---

## Backtracking vs. DP

| | Backtracking | DP |
|---|---|---|
| Goal | enumerate / find valid arrangements | compute optimal value |
| Subproblems | no overlapping subproblems | overlapping subproblems |
| Pruning | constraint pruning (invalid → skip) | memoization (repeat → lookup) |
| Output | configurations | a number / value |

If a backtracking problem asks "how many solutions?" and the state space has overlapping subproblems, you can often memoize it into a DP.

---

## Pruning Strategies

1. **Constraint sets** — O(1) conflict check (N-Queens, Sudoku)
2. **Forward checking** — after placing, immediately check if any remaining slot has zero valid choices; bail if so
3. **Most-constrained variable** — in Sudoku, fill the cell with fewest remaining candidates first (reduces branching factor dramatically)
4. **Frequency pruning** — in word search, if the board has fewer of a needed character than the target, return immediately
5. **Symmetry breaking** — in N-Queens, only explore columns `0..n//2` for row 0, then mirror (halves work)

---

## General Complexity

Backtracking is exponential in the worst case. The goal of pruning is to reduce the effective branching factor.

| Problem | Worst case | With pruning |
|---|---|---|
| N-Queens (n=8) | 8^8 ≈ 16M | ~92 solutions found in ~15k nodes |
| Sudoku | 9^81 | effectively polynomial with constraint propagation |
| Permutations of n | n! | — |
| Subsets of n | 2^n | — |

---

## Problems using this pattern

### Grid / board placement (N-Queens family)
- [[dsa/problems/n-queens]] — Place n queens; row+col+diagonal constraint sets
- [[dsa/problems/sudoku-solver]] — Fill 9×9; row+col+box constraint sets; most-constrained-cell heuristic
- [[dsa/grid/problems/word-search]] — DFS + path-visited set; unmark after recursion
- [[dsa/problems/knight-tour]] — Visit every cell with a knight; Warnsdorff heuristic

### Combinatorics
- [[dsa/problems/combination-sum]] — Pick numbers summing to target; sorted input + skip duplicates
- [[dsa/problems/permutations]] — All orderings; swap-in-place or used-set
- [[dsa/problems/subsets]] — Power set; include/exclude at each index
- [[dsa/problems/generate-parentheses]] — Track open/close counts; prune when invalid

## Sources

- [[dsa/topics/algorithms]]
