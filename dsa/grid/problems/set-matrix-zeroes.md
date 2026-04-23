# Set Matrix Zeroes

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/grid/patterns/matrix-traversal]]
**Companies:** [[dsa/companies/apple]], [[dsa/companies/google]], [[dsa/companies/meta]]

## Problem

Given an `m x n` integer matrix, if an element is `0`, set its **entire row and column** to `0`s. Do it **in-place**.

```
Input:  [[1,1,1],[1,0,1],[1,1,1]]
Output: [[1,0,1],[0,0,0],[1,0,1]]

Input:  [[0,1,2,0],[3,4,5,2],[1,3,1,5]]
Output: [[0,0,0,0],[0,4,5,0],[0,3,1,0]]
```

## Why This Is Tricky

**Naive mistake:** As you scan the matrix, when you find a `0`, immediately set its entire row/column to `0`. Problem: newly created zeros will be treated as original zeros in subsequent passes, causing incorrect results.

**Correct approach:** Collect all positions of original zeros first, then do the row/column zeroing in a second pass.

## Approach 1: O(m+n) Space — Two Sets

```python
def set_zeroes_sets(matrix: list[list[int]]) -> None:
    rows_to_zero = set()
    cols_to_zero = set()

    # Pass 1: identify original zeros
    for r in range(len(matrix)):
        for c in range(len(matrix[0])):
            if matrix[r][c] == 0:
                rows_to_zero.add(r)
                cols_to_zero.add(c)

    # Pass 2: zero out rows and columns
    for r in range(len(matrix)):
        for c in range(len(matrix[0])):
            if r in rows_to_zero or c in cols_to_zero:
                matrix[r][c] = 0
```

## Approach 2: O(1) Space — Use First Row and Column as Flags

The elegant O(1) space solution uses the first row and first column of the matrix itself as the flag arrays. The only complication: the `(0,0)` cell is shared between "first row needs zeroing" and "first column needs zeroing," so we handle it separately with a boolean.

```python
def set_zeroes(matrix: list[list[int]]) -> None:
    rows, cols = len(matrix), len(matrix[0])
    first_col_zero = any(matrix[r][0] == 0 for r in range(rows))

    # Use first row and first column as flags for the rest
    for r in range(rows):
        for c in range(1, cols):   # start from col 1 to protect first column
            if matrix[r][c] == 0:
                matrix[r][0] = 0   # mark row r needs zeroing
                matrix[0][c] = 0   # mark col c needs zeroing

    # Zero out non-first cells based on flags
    for r in range(1, rows):       # start from row 1 to protect first row
        for c in range(1, cols):
            if matrix[r][0] == 0 or matrix[0][c] == 0:
                matrix[r][c] = 0

    # Zero out first row if (0,0) was originally 0
    if matrix[0][0] == 0:
        for c in range(cols):
            matrix[0][c] = 0

    # Zero out first column if any original zero was in first column
    if first_col_zero:
        for r in range(rows):
            matrix[r][0] = 0
```

## Complexity

| Approach | Time | Space |
|---|---|---|
| Two sets | O(m × n) | O(m + n) |
| First row/col flags | O(m × n) | O(1) |

## Key Insight

The O(1) approach works because the first row and column cells are going to be zeroed out anyway if they contain (or are in the same row/column as) a zero. So we repurpose them as our flag array *before* their values matter.

The ordering of operations is critical:
1. Save whether the first column needs zeroing (before we overwrite `matrix[r][0]`).
2. Process rows 1+ before rows 0 to avoid corrupting the flags.
3. Handle the first row and column last.

## Interview Tips

- Start with the O(m+n) approach — it's easy to explain and correct. Offer to optimize to O(1) as a follow-up.
- The two-pass pattern (collect → apply) appears in many "don't modify while scanning" problems.
- This problem tests whether you can reason about shared state — the common pitfall is self-contamination during a single pass.

## Variants / Follow-ups

- "What if you need to zero entire row + column + diagonals?" → Same two-pass approach; just expand what "zero out" means.
- "What if the matrix contains floats and you can't store a sentinel?" → Use a separate boolean matrix as flags (O(m × n) space).
- "Can you do it in one pass?" → No clean way to do it in one pass without either extra memory or a sentinel value.

## Sources
- [[dsa/companies/apple]]
- [[dsa/companies/google]]
