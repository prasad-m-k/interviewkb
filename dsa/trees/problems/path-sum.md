# Path Sum I & II

**Difficulty:** Easy (I) / Medium (II)
**Topic:** [[dsa/trees/Tree overview]]
**Pattern:** DFS top-down, subtract target as you descend
**Companies:** Amazon, Apple, Microsoft, Google

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem I — Does a root-to-leaf path sum equal target?

Return `true` if any root-to-leaf path sums to `targetSum`.

```
         5
        / \
       4   8
      /   / \
     11  13   4
    /  \       \
   7    2        1

targetSum = 22
Path: 5 → 4 → 11 → 2 = 22 ✓
```

## Solution I (Python)

```python
def hasPathSum(root, targetSum: int) -> bool:
    if not root: return False
    targetSum -= root.val
    if not root.left and not root.right:     # leaf
        return targetSum == 0
    return hasPathSum(root.left, targetSum) or hasPathSum(root.right, targetSum)
```

**Why subtract at each node?** Avoids carrying a running sum separately. Reach a leaf with `remaining == 0` → found.

---

## Problem II — Return ALL root-to-leaf paths that sum to target

```
         5
        / \
       4   8
      /   / \
     11  13   4
    /  \     / \
   7    2   5   1

targetSum = 22
Paths: [5,4,11,2] and [5,8,4,5]
```

## Solution II (Python)

```python
def pathSum(root, targetSum: int):
    result = []

    def dfs(node, remaining, path):
        if not node: return
        path.append(node.val)
        remaining -= node.val
        if not node.left and not node.right and remaining == 0:
            result.append(path[:])           # snapshot the current path
        dfs(node.left,  remaining, path)
        dfs(node.right, remaining, path)
        path.pop()                           # BACKTRACK — undo the append

    dfs(root, targetSum, [])
    return result
```

## Traced execution (Path Sum II, small example)

```
Tree:    1
        / \
       2   3
targetSum = 3

dfs(1, 3, []):
  path=[1], remaining=2
  dfs(2, 2, [1]):
    path=[1,2], remaining=0
    leaf! remaining==0 → result=[[1,2]]
    pop → path=[1]
  dfs(3, 2, [1]):
    path=[1,3], remaining=-1
    leaf! remaining≠0 → skip
    pop → path=[1]
result = [[1,2]]  ✓  (1+2=3, not 1+3=4)
```

## Complexity

Time: O(n²) — O(n) nodes × O(n) to copy path at each leaf  
Space: O(n) path length + O(n) result

## Key insight

**`path.pop()` after both recursive calls is the backtracking step.** Without it, `path` accumulates all nodes ever visited. The pop restores the path to what it was before entering this node, allowing other branches to build their own paths.

## Variants / follow-ups

- **Path Sum III (LC 437):** count paths summing to target — path doesn't need to start at root or end at leaf. Use prefix sum + hashmap: `O(n)`.
  ```python
  def pathSumIII(root, targetSum):
      count = [0]
      prefix = {0: 1}
      def dfs(node, curr):
          if not node: return
          curr += node.val
          count[0] += prefix.get(curr - targetSum, 0)
          prefix[curr] = prefix.get(curr, 0) + 1
          dfs(node.left, curr); dfs(node.right, curr)
          prefix[curr] -= 1
      dfs(root, 0)
      return count[0]
  ```
- **Maximum path sum (any path, not root-to-leaf):** → [[dsa/trees/problems/binary-tree-maximum-path-sum]]

## Sources

- [[dsa/trees/Tree overview]]
- [[dsa/trees/patterns/dfs-subtree-returns]]
