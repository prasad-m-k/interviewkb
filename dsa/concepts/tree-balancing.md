# Tree Balancing — BST, AVL, Red-Black, B-Tree

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/concepts/heap-internals]]

## Why balance matters
A binary search tree promises O(log n) search by halving the search space at each node. But that guarantee only holds when the tree is balanced (height ≈ log n). An unbalanced BST can degrade to a linked list — O(n) for every operation.

```
Balanced BST (height log n):     Degenerate BST (height n):
        5                          1
       / \                          \
      3   7                          2
     / \ / \                          \
    2  4 6  8                          3
                                        \
                                         4  ← O(n) to reach leaf
```

**When does a plain BST degenerate?** When you insert already-sorted data. Inserting 1, 2, 3, 4, 5 in order builds a right-leaning chain.

## AVL Trees
AVL trees maintain strict balance: the heights of the left and right subtree of every node differ by at most 1 (balance factor ∈ {-1, 0, 1}).

**Rotations**: when an insert or delete violates the balance condition, the tree fixes itself via rotations — O(1) pointer updates.

```
Left rotation at node x:          Right rotation at node y:
    x                 y                y               x
     \               / \              /               / \
      y     →       x   z            x      →        z   y
     / \             \              / \
    (z) (ignored)    (z)           z  (ignored)
```

Four cases: Left-Left, Right-Right, Left-Right, Right-Left. The last two require two rotations (double rotation).

**Guarantees**: height ≤ 1.44 log n — strictly O(log n) search, insert, delete.
**Cost**: stricter balance means more rotations on insert/delete (up to O(log n) rotations per insert).
**Used when**: read-heavy workloads where O(log n) lookup must be tight (e.g., in-memory sorted maps where reads vastly outnumber writes).

## Red-Black Trees
Red-Black trees are BSTs where every node is colored red or black, and the coloring enforces a weaker balance invariant:

1. Root is black
2. No two consecutive red nodes (red node's parent must be black)
3. Every path from root to null has the same number of black nodes (*black-height*)

**Guarantees**: height ≤ 2 log n — still O(log n), but the constant is worse than AVL.
**Cost**: rebalancing requires at most **2 rotations** per insert and **3 rotations** per delete — better write performance than AVL.
**Used by**: Java `TreeMap`/`TreeSet`, C++ `std::map`/`std::set`, Linux CFS scheduler, Python `sortedcontainers` internals, Java `HashMap` bucket treeification (since Java 8).

## AVL vs Red-Black — the tradeoff

| | AVL | Red-Black |
|---|---|---|
| Balance | Stricter (height ≤ 1.44 log n) | Looser (height ≤ 2 log n) |
| Lookup | Slightly faster | Slightly slower |
| Insert/delete rotations | More | Fewer (≤ 2 insert, ≤ 3 delete) |
| Best for | Read-heavy | Write-heavy |

**Interview answer**: "If lookups dominate, prefer AVL. If writes are frequent, prefer Red-Black. In practice, most standard library sorted maps use Red-Black because write-heavy patterns are more common in general-purpose code."

## B-Trees
B-trees generalize the BST to nodes with **multiple keys and multiple children** (branching factor `t`: each node has between `t-1` and `2t-1` keys).

```
B-tree node (t=3, up to 5 keys):
[ 10 | 20 | 30 | 40 ]
 ↓    ↓    ↓    ↓   ↓
(<10)(10-20)(20-30)(30-40)(>40)
```

**Why databases use B-trees (not red-black trees) for indexes:**
1. **Disk I/O is the bottleneck.** Reading one tree node = one disk page read (~4KB–16KB). A fat node with 100+ keys means you traverse far fewer levels than a binary tree for the same n. A B-tree on 1 billion records might have height 3–4; a red-black tree has height ~60.
2. **Cache line efficiency.** Even in memory, a wide node fits in one or two cache lines; a binary tree forces many cache misses chasing pointers.
3. **Sequential reads are fast.** B-trees store keys in sorted order within each node, making range scans efficient (scan the leaf level left-to-right).

**B+ tree** (used by most databases — InnoDB, PostgreSQL): like a B-tree but all actual data lives in leaf nodes; internal nodes store only keys for routing. Leaves are linked in a doubly-linked list for efficient range scans.

## Common interview angles
- "Why can a BST degrade to O(n)?" — sorted insertions produce a chain; height becomes n
- "What's the difference between AVL and Red-Black trees?" — tradeoff between read and write performance (see table above)
- "Why do databases use B-trees rather than red-black trees?" — disk I/O minimization; one node = one disk page read
- "What is a B+ tree?" — B-tree variant where all data is in leaves, leaves are linked for range scans
- "What's the height of a balanced BST on n nodes?" — ⌊log₂ n⌋

## Sources
*(none yet)*
