# Word Search

**Difficulty:** Medium
**Pattern:** DFS + Backtracking on grid
**Topic:** [[Grid overview]]
**Companies:** Amazon, Microsoft, Facebook, Bloomberg

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given an m×n board of characters and a string word, return true if the word can be found by sequentially adjacent cells (horizontally or vertically). The same cell may not be used more than once.

## Approach
DFS with backtracking. At each step, check if the current cell matches the current character. Mark the cell visited, recurse to all neighbors for the next character, then UNMARK (backtrack) so other paths can use this cell.

**Key**: you MUST unmark after recursion. This is what makes it backtracking, not plain DFS.

## Solution (Python)
```python
def exist(board, word):
    ROWS, COLS = len(board), len(board[0])
    DIRS = [(0,1),(0,-1),(1,0),(-1,0)]

    def dfs(r, c, idx):
        if idx == len(word):
            return True  # found all characters
        if not (0 <= r < ROWS and 0 <= c < COLS):
            return False
        if board[r][c] != word[idx]:
            return False

        temp = board[r][c]
        board[r][c] = '#'  # mark visited for this path

        found = any(dfs(r+dr, c+dc, idx+1) for dr, dc in DIRS)

        board[r][c] = temp  # UNMARK — critical

        return found

    for r in range(ROWS):
        for c in range(COLS):
            if dfs(r, c, 0):
                return True

    return False
```

## Complexity
Time: O(R×C × 4^L) where L = len(word) | Space: O(L) recursion stack

## Key insight
- Mark the cell as visited using a special character (`'#'`) before recursing — prevents using the same cell twice in one path
- Unmark after returning — allows this cell to be used in different search paths starting from other cells
- Early termination: return True as soon as any path succeeds

## Optimization: Prune early with frequency count
```python
# If word has more of some char than the board, immediately return False
from collections import Counter
board_count = Counter(c for row in board for c in row)
word_count = Counter(word)
if any(word_count[c] > board_count[c] for c in word_count):
    return False

# Also: if more of the first char is rare than the last char, reverse the word
# (reduces branching at start)
if board_count[word[0]] > board_count[word[-1]]:
    word = word[::-1]
```

## Variants / follow-ups
- **Word Search II** (find all words from a list): use Trie + DFS to prune — [[dsa/grid/problems/word-search-ii]]
- **Word Search in diagonal directions**: change DIRS to 8-directional

## Sources
- [[dsa/grid/patterns/flood-fill-dfs]]
- [[dsa/patterns/backtracking]]
- [[Grid overview]]
