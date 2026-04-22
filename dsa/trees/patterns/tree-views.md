# Pattern: Tree Views (Left / Right / Top / Bottom)

**Topic:** [[dsa/trees/Tree overview]]
**Related:** [[dsa/patterns/tree-traversal]]

## What it solves

"What nodes are visible when you look at the tree from the left / right / top / bottom?"

All four views use BFS. The difference is only **what you record per level or per horizontal distance**.

---

## Reference Tree (used throughout)

```
              1
            /   \
           2     3
          / \     \
         4   5     6
              \
               7
```

---

## Left View — First node at each level

Imagine standing to the left and looking right. You see the leftmost node at each depth.

```
Left view: 1, 2, 4, 7
```

**Algorithm:** BFS; record `queue[0]` (first node) of each level.

```python
from collections import deque

def leftView(root):
    if not root: return []
    result = []
    queue = deque([root])
    while queue:
        result.append(queue[0].val)       # first = leftmost at this level
        for _ in range(len(queue)):
            node = queue.popleft()
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
    return result
```

**DFS alternative** (track max depth seen so far):
```python
def leftView_dfs(root):
    result = []
    def dfs(node, depth):
        if not node: return
        if depth == len(result):           # first visit to this depth
            result.append(node.val)
        dfs(node.left,  depth + 1)         # left first → leftmost seen first
        dfs(node.right, depth + 1)
    dfs(root, 0)
    return result
```

---

## Right View — Last node at each level

Imagine standing to the right. You see the rightmost node at each depth.

```
Right view: 1, 3, 6, 7
```

**Algorithm:** BFS; record `level[-1]` (last node) of each level — or the last node dequeued.

```python
def rightView(root):
    if not root: return []
    result = []
    queue = deque([root])
    while queue:
        for i in range(len(queue)):
            node = queue.popleft()
            if i == 0:                    # will be overwritten; last survives
                last = node.val
            last = node.val               # keep updating → last at this level
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
        result.append(last)
    return result
```

**Cleaner:**
```python
def rightView(root):
    if not root: return []
    result = []
    queue = deque([root])
    while queue:
        level_size = len(queue)
        for i in range(level_size):
            node = queue.popleft()
            if i == level_size - 1:       # last node of this level
                result.append(node.val)
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
    return result
```

---

## Top View — First node at each horizontal distance

Imagine a bird's-eye view from above, looking straight down.

**Horizontal distance (HD):**  
- Root = 0  
- Going left: HD − 1  
- Going right: HD + 1

A node is in the top view if it's the **first node seen (closest to root) at its HD**.

```
Tree:
              1 (HD=0)
            /   \
     (HD=-1)2     3(HD=1)
          / \     \
   (HD=-2)4   5(HD=0)  6(HD=2)
              \
               7(HD=1)

Top view (first seen at each HD):
HD=-2 → 4
HD=-1 → 2
HD= 0 → 1  (5 also has HD=0 but 1 is seen first in BFS)
HD= 1 → 3  (7 also has HD=1 but 3 is seen first)
HD= 2 → 6

Top view: [4, 2, 1, 3, 6]
```

```python
from collections import deque, defaultdict

def topView(root):
    if not root: return []
    hd_map = {}                          # HD → first node value seen
    queue = deque([(root, 0)])           # (node, horizontal_distance)
    while queue:
        node, hd = queue.popleft()
        if hd not in hd_map:             # first time seeing this HD
            hd_map[hd] = node.val
        if node.left:  queue.append((node.left,  hd - 1))
        if node.right: queue.append((node.right, hd + 1))
    return [hd_map[hd] for hd in sorted(hd_map)]
```

---

## Bottom View — Last node at each horizontal distance

Looking up from below. For each HD, show the last node reached in BFS (deepest visible node).

```
Tree (same as above):
HD=-2 → 4  (only node)
HD=-1 → 2  (only node at this HD... wait, 2 is the only one)
HD= 0 → 5  (both 1 and 5 at HD=0; 5 is deeper → seen last in BFS)
HD= 1 → 7  (both 3 and 7 at HD=1; 7 is deeper → seen last in BFS)
HD= 2 → 6  (only node)

Bottom view: [4, 2, 5, 7, 6]
```

```python
def bottomView(root):
    if not root: return []
    hd_map = {}                          # HD → last node value seen (keep overwriting)
    queue = deque([(root, 0)])
    while queue:
        node, hd = queue.popleft()
        hd_map[hd] = node.val            # always overwrite → last (deepest) wins
        if node.left:  queue.append((node.left,  hd - 1))
        if node.right: queue.append((node.right, hd + 1))
    return [hd_map[hd] for hd in sorted(hd_map)]
```

**Only difference from top view:** `if hd not in hd_map` → `hd_map[hd] = node.val` (always overwrite).

---

## Comparison Table

| View | Record | BFS or DFS | Key |
|---|---|---|---|
| Left | First node per **level** | Either | `depth == len(result)` (DFS) or `queue[0]` (BFS) |
| Right | Last node per **level** | Either | `i == level_size - 1` (BFS) or right-first DFS |
| Top | First node per **HD** | BFS only | `if hd not in hd_map` |
| Bottom | Last node per **HD** | BFS only | always `hd_map[hd] = node.val` |

---

## Signal phrases

"left view" / "right view" / "top view" / "bottom view" / "right side view" (= right view) / "visible from above" / "print boundary"

---

## Boundary Traversal (variant combining all views)

Print all boundary nodes anti-clockwise: left boundary (top to bottom) + all leaves (left to right) + right boundary (bottom to top, excluding root).

```python
def boundaryTraversal(root):
    if not root: return [root.val]
    result = [root.val]

    def add_left_boundary(node):
        if not node or (not node.left and not node.right): return
        result.append(node.val)
        if node.left: add_left_boundary(node.left)
        else:         add_left_boundary(node.right)

    def add_leaves(node):
        if not node: return
        if not node.left and not node.right:
            result.append(node.val); return
        add_leaves(node.left)
        add_leaves(node.right)

    def add_right_boundary(node):
        if not node or (not node.left and not node.right): return
        if node.right: add_right_boundary(node.right)
        else:          add_right_boundary(node.left)
        result.append(node.val)   # append after recursion = bottom-up

    add_left_boundary(root.left)
    add_leaves(root.left)
    add_leaves(root.right)
    add_right_boundary(root.right)
    return result
```

---

## Problems using this pattern

- [[dsa/trees/problems/left-side-view]]
- [[dsa/trees/problems/right-side-view]]
- [[dsa/trees/problems/top-view]]
- [[dsa/trees/problems/bottom-view]]

## Sources

- [[dsa/trees/Tree overview]]
- [[dsa/patterns/tree-traversal]]
