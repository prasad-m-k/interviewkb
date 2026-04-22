# Binary Search Tree (BST)

**Topic:** [[dsa/trees/Tree overview]]
**Related:** [[dsa/concepts/binary-tree]], [[dsa/trees/problems/validate-bst]], [[dsa/trees/problems/kth-smallest-bst]]

## What it is

A BST is a binary tree where for every node `N`:
- All values in `N`'s **left** subtree are **strictly less than** `N.val`
- All values in `N`'s **right** subtree are **strictly greater than** `N.val`

This invariant holds recursively — not just for immediate children.

```
Valid BST:          Invalid (looks valid locally, fails globally):
      5                      5
     / \                    / \
    3   7                  3   7
   / \   \                / \
  2   4   8              2   6     ← 6 > 5 but is in left subtree of 5 ✗
```

**Key insight:** inorder traversal of a BST yields values in sorted ascending order.  
If inorder output isn't strictly increasing → not a valid BST.

---

## Core Operations

### Search
```python
def search(root, target):
    if not root or root.val == target:
        return root
    if target < root.val:
        return search(root.left, target)
    return search(root.right, target)
```

### Insert
```python
def insert(root, val):
    if not root:
        return TreeNode(val)
    if val < root.val:
        root.left = insert(root.left, val)
    else:
        root.right = insert(root.right, val)
    return root
```

### Delete
Three cases:
1. Node is a leaf → just remove it
2. Node has one child → replace node with its child
3. Node has two children → replace value with **inorder successor** (smallest in right subtree), then delete inorder successor

```python
def delete(root, key):
    if not root:
        return None
    if key < root.val:
        root.left = delete(root.left, key)
    elif key > root.val:
        root.right = delete(root.right, key)
    else:
        if not root.left:  return root.right   # case 1 & 2
        if not root.right: return root.left    # case 2
        # case 3: find inorder successor (min of right subtree)
        succ = root.right
        while succ.left:
            succ = succ.left
        root.val = succ.val
        root.right = delete(root.right, succ.val)
    return root
```

---

## Validate a BST

**Wrong approach:** only check `node.left.val < node.val < node.right.val`.  
This misses the global constraint (a node in the left subtree must be less than ALL ancestors, not just its immediate parent).

**Correct approach:** pass bounds down.

```python
def isValidBST(root, lo=float('-inf'), hi=float('inf')):
    if not root:
        return True
    if not (lo < root.val < hi):
        return False
    return (isValidBST(root.left,  lo, root.val) and
            isValidBST(root.right, root.val, hi))
```

Alternatively: inorder traversal + check strictly increasing.

```python
def isValidBST_inorder(root):
    prev = [float('-inf')]
    def inorder(node):
        if not node: return True
        if not inorder(node.left): return False
        if node.val <= prev[0]: return False
        prev[0] = node.val
        return inorder(node.right)
    return inorder(root)
```

---

## LCA in a BST

Unlike general trees (which need full DFS), a BST LCA can exploit the invariant:

```python
def lcaBST(root, p, q):
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left     # both in left subtree
        elif p.val > root.val and q.val > root.val:
            root = root.right    # both in right subtree
        else:
            return root          # split point = LCA
```

O(h) time, O(1) space. No recursion needed.

---

## Kth Smallest Element

Inorder traversal visits nodes in sorted order. Stop at the kth visit.

```python
def kthSmallest(root, k):
    stack, curr = [], root
    count = 0
    while curr or stack:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()
        count += 1
        if count == k:
            return curr.val
        curr = curr.right
```

---

## BST vs. Balanced BST

| | BST | AVL / Red-Black |
|---|---|---|
| Insert/Search/Delete | O(h) | O(log n) guaranteed |
| Worst case h | O(n) (skewed) | O(log n) |
| Python built-in | No (use `sortedcontainers.SortedList`) | No |
| Interview use | Problem structure / logic | Mention as follow-up |

In interviews: implement plain BST operations, then mention "in production you'd use a self-balancing variant."

---

## Common BST Interview Angles

| Question | Approach |
|---|---|
| Validate BST | Bounds propagation top-down |
| Kth smallest/largest | Iterative inorder; augment with subtree sizes for O(log n) |
| LCA in BST | Walk from root — split point is LCA |
| Convert sorted array → BST | Always pick mid as root (ensures balance) |
| Convert BST → sorted doubly linked list | Inorder; thread prev/next pointers |
| Range sum query [L, R] | DFS with pruning — skip entire subtrees outside range |

---

## Sources
- [[dsa/trees/Tree overview]]
- [[dsa/trees/problems/validate-bst]]
- [[dsa/trees/problems/kth-smallest-bst]]
