# Binary Tree

**Topic:** [[dsa/topics/data-structures]]
**Related:** [[dsa/concepts/graph]], [[dsa/patterns/tree-traversal]]

## What it is
A binary tree is a rooted tree where each node has at most two children (left and right). A **Binary Search Tree (BST)** adds the invariant: `left.val < node.val < right.val` for every node. A **complete binary tree** fills all levels left-to-right (this is how heaps are stored as arrays).

## How it works

**Node definition:**
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

**Key traversals:**
| Order | Sequence | Use case |
|---|---|---|
| Inorder (L → root → R) | Sorted output for BST | BST validation, kth smallest |
| Preorder (root → L → R) | Serialize tree structure | Serialization, copy |
| Postorder (L → R → root) | Process children before parent | Delete tree, path sums |
| Level-order (BFS) | Level by level | Shortest path, level sums |

## Complexity
| Operation | BST (balanced) | BST (worst) | Any binary tree |
|---|---|---|---|
| Search | O(log n) | O(n) | O(n) |
| Insert | O(log n) | O(n) | — |
| Delete | O(log n) | O(n) | — |
| Traversal | O(n) | O(n) | O(n) |
| Space (recursion) | O(log n) | O(n) | O(h) |

Height `h` = O(log n) for balanced trees, O(n) for skewed.

## When to use
- **Ordered data with O(log n) ops**: BST (or balanced variant like Red-Black)
- **Level-by-level processing**: BFS / level-order
- **Path problems**: DFS (preorder or postorder depending on direction)
- **LCA, diameter, balanced check**: DFS returning info from subtrees

## Common interview angles
- **Height / depth**: recursive `1 + max(height(left), height(right))`
- **Diameter**: at each node, `left_height + right_height`; track global max
- **Is balanced**: DFS returns height; returns -1 if subtree is unbalanced
- **LCA**: DFS; node is LCA if it's the root of the subtree containing both p and q
- **Serialize/deserialize**: preorder DFS with null markers; reconstruct from same traversal
- **Level-order**: BFS with a `for _ in range(len(queue))` inner loop to process one level at a time

## Examples
```python
# Inorder (iterative — avoids recursion limit)
def inorder(root):
    stack, result = [], []
    curr = root
    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        result.append(curr.val)
        curr = curr.right
    return result

# Height
def height(root):
    if not root: return 0
    return 1 + max(height(root.left), height(root.right))

# Check if BST (pass bounds down)
def isValidBST(root, lo=float('-inf'), hi=float('inf')):
    if not root: return True
    if not (lo < root.val < hi): return False
    return isValidBST(root.left, lo, root.val) and isValidBST(root.right, root.val, hi)
```

## Sources
- [[DSA overview]]
