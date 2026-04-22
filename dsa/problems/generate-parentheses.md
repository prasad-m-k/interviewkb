# Generate Parentheses

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/backtracking]]
**Companies:** Google, Amazon, Bloomberg, Adobe

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem

Given `n` pairs of parentheses, generate all combinations of well-formed parentheses.

```
n = 3 → ["((()))", "(()())", "(())()", "()(())", "()()()"]
```

## Approach

Backtracking with two counters: `open` (open brackets placed so far) and `close` (close brackets placed). Rules:
- Add `(` if `open < n`
- Add `)` if `close < open` (can't close what isn't open)
- Base case: `open == close == n`

No explicit "undo" needed here — string concatenation creates new strings, so state is naturally isolated. This is a degenerate backtracking where the constraints eliminate invalid choices before recursing (no invalid placements to undo).

## Solution (Python)

```python
def generateParenthesis(n: int) -> list[str]:
    results = []

    def backtrack(s: str, open: int, close: int) -> None:
        if len(s) == 2 * n:
            results.append(s)
            return
        if open < n:
            backtrack(s + '(', open + 1, close)
        if close < open:
            backtrack(s + ')', open, close + 1)

    backtrack('', 0, 0)
    return results
```

### Space-optimized (in-place list)

```python
def generateParenthesis(n: int) -> list[str]:
    results = []
    path = []

    def backtrack(open: int, close: int) -> None:
        if open == close == n:
            results.append(''.join(path)); return
        if open < n:
            path.append('('); backtrack(open + 1, close); path.pop()
        if close < open:
            path.append(')'); backtrack(open, close + 1); path.pop()

    backtrack(0, 0)
    return results
```

## Complexity

Time: O(4^n / √n) — the nth Catalan number, which counts valid sequences  
Space: O(n) recursion depth

## Key insight

The constraint `close < open` is the entire validity check. It's stricter than "no more `)` than `(`" — it enforces validity at every prefix, not just at the end. This means every path that reaches the base case is automatically valid; no post-filtering needed.

## N-Queens connection

This is the same choose/explore/unchoose skeleton but the constraint is arithmetic rather than spatial. Instead of checking a set for column conflicts, you check `open < n` and `close < open`. The pruning philosophy is identical: reject invalid choices before recursing.

## Variants / follow-ups

- **Count only:** the answer is the nth Catalan number `C(n) = C(2n,n)/(n+1)` — no backtracking needed
- **Multiple bracket types `()[]{}`:** extend with a stack to track which type needs closing
- **Score of parentheses (LC 856):** compute a numeric score from a valid string — stack problem, not backtracking

## Sources

- [[dsa/patterns/backtracking]]
