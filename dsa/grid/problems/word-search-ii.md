# Word Search II

**Difficulty:** Hard
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/grid/patterns/flood-fill-dfs]] + Trie
**Companies:** [[dsa/companies/apple]], [[dsa/companies/google]], [[dsa/companies/meta]]

## Problem

Given a 2D board of characters and a list of words, find all words in the board.

Each word must be constructed from letters of **sequentially adjacent** cells (4-directional), and the same letter cell may not be used more than once per word.

```
Input:
board = [["o","a","a","n"],
         ["e","t","a","e"],
         ["i","h","k","r"],
         ["i","f","l","v"]]
words = ["oath","pea","eat","rain"]

Output: ["eat","oath"]
```

## Why This Is Hard (and Important)

Word Search I finds a single word with DFS. Word Search II finds **many words** — the naive approach runs Word Search I independently for each word: O(words × rows × cols × 4^L). With 1000 words on a 12×12 board, that's ~10^9 operations.

The key insight: **use a Trie to search all words simultaneously**. As you DFS from each board cell, walk the Trie together. One DFS pass finds all words that start at that cell.

## Approach

### Step 1: Build a Trie from the word list

```
words = ["eat", "oath"]
Trie:
  root
  ├── e
  │   └── a
  │       └── t [word="eat"]
  └── o
      └── a
          └── t
              └── h [word="oath"]
```

Mark the end-of-word node with the actual word string (not just a boolean) — this way you don't need to reconstruct the word from the path.

### Step 2: DFS from every board cell

For each cell, start DFS and simultaneously descend the Trie. If the current character isn't a child of the Trie node, prune. If you reach a Trie node marked with a word, record it.

### Step 3: Optimize — prune exhausted Trie branches

After finding a word, **delete it from the Trie** (the node and its `word` marker). This prevents finding the same word twice and speeds up subsequent searches by pruning branches with no remaining words.

## Solution (Python)

```python
def find_words(board: list[list[str]], words: list[str]) -> list[str]:
    # Build Trie
    trie = {}
    for word in words:
        node = trie
        for ch in word:
            node = node.setdefault(ch, {})
        node['#'] = word  # end-of-word marker stores the word itself

    rows, cols = len(board), len(board[0])
    found = []

    def dfs(r: int, c: int, node: dict) -> None:
        ch = board[r][c]
        if ch not in node:
            return

        next_node = node[ch]

        # Check if we completed a word
        if '#' in next_node:
            found.append(next_node['#'])
            del next_node['#']  # remove to avoid duplicate finds

        # Mark as visited
        board[r][c] = '#'

        for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and board[nr][nc] != '#':
                dfs(nr, nc, next_node)

        # Restore cell
        board[r][c] = ch

        # Prune: if this Trie node has no children, remove it from parent
        # (Handled implicitly by the '#' deletion above; full pruning is optional)

    for r in range(rows):
        for c in range(cols):
            dfs(r, c, trie)

    return found
```

## Optimized: Full Trie Pruning

After collecting a word, prune empty Trie branches bottom-up. This makes subsequent DFS calls faster:

```python
def dfs_pruned(r, c, parent_node, ch_key):
    ch = board[r][c]
    if ch not in parent_node[ch_key]:
        return

    curr_node = parent_node[ch_key]

    if '#' in curr_node:
        found.append(curr_node.pop('#'))

    board[r][c] = '#'
    for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
        nr, nc = r + dr, c + dc
        if 0 <= nr < rows and 0 <= nc < cols and board[nr][nc] != '#':
            for child_key in list(curr_node.keys()):
                dfs_pruned(nr, nc, curr_node, child_key)
    board[r][c] = ch

    # Bottom-up pruning: delete leaf nodes that have no words left
    if not curr_node:
        del parent_node[ch_key]
```

## Complexity

| | Value |
|---|---|
| Trie build | O(W × L) where W = words, L = avg word length |
| DFS (worst case) | O(rows × cols × 4^L) — same as Word Search I |
| DFS (with pruning) | Much better in practice: branches with no words are pruned early |
| Space | O(W × L) for Trie + O(L) recursion depth |

## Key Insight

The Trie transforms "search for word W on board" from a serial problem (one word at a time) into a parallel problem (all words simultaneously). The DFS descends the Trie and the board at the same time — whenever there's no Trie child matching the board character, we prune the entire subtree.

The in-place visited marking (`board[r][c] = '#'`) with restoration after DFS is the classic "no-visited-set" trick that avoids O(rows × cols) extra memory per DFS call.

## Interview Tips

- This problem is almost always asked in two stages: first Word Search I (single word), then "now find all words from a list." Be ready to evolve your solution.
- Mention Trie *before* coding. Saying "I'd use a Trie to avoid redundant work across words" immediately shows the key insight.
- The pruning optimization (deleting found words from the Trie) is a tier-1 follow-up. If you mention it unprompted, you distinguish yourself.
- Python's `setdefault` for Trie construction is clean in interviews.

## Variants / Follow-ups

- **Word Search I (single word):** Pure DFS without Trie. [[dsa/grid/problems/word-search]]
- "What if words have wildcards ('.')?" → At wildcard Trie level, try all children.
- "What if the board wraps around?" → Adjust boundary conditions for modular arithmetic.
- "What if there are 100,000 words?" → Trie is even more critical. Also consider DAWG (Directed Acyclic Word Graph) for space-efficient Trie.

## Sources
- [[dsa/companies/apple]]
- [[dsa/companies/google]]
