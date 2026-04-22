# Dynamic Programming Pattern

**Topic:** [[dsa/topics/algorithms]]
**Related concepts:** [[dsa/concepts/dynamic-programming]]

## What it solves
Optimization and counting problems where the solution to the full problem can be expressed as a combination of solutions to smaller subproblems (optimal substructure), and those subproblems repeat (overlapping subproblems).

## Template / skeleton

**1D DP — "can we reach state i?"**
```python
def dp_reachable(n, ...):
    dp = [False] * (n + 1)
    dp[0] = True                    # base case
    for i in range(1, n + 1):
        for j in range(i):          # check all possible previous states
            if dp[j] and transition(j, i):
                dp[i] = True
                break
    return dp[n]
```

**1D DP — "minimum cost to reach state i"**
```python
def dp_min_cost(n, costs):
    dp = [float('inf')] * (n + 1)
    dp[0] = 0
    for i in range(1, n + 1):
        for prev in valid_predecessors(i):
            dp[i] = min(dp[i], dp[prev] + cost(prev, i))
    return dp[n]
```

**2D DP — "edit distance / LCS"**
```python
def dp_2d(s, t):
    m, n = len(s), len(t)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    # base cases: dp[0][j] and dp[i][0]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s[i-1] == t[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    return dp[m][n]
```

## Interview approach checklist
1. **State definition**: write `dp[i] = ...` in plain English before touching code
2. **Recurrence**: express `dp[i]` in terms of `dp[j]` for `j < i`
3. **Base cases**: what is `dp[0]`? Are there edge cases for empty input?
4. **Order**: fill left-to-right for 1D, top-left to bottom-right for 2D
5. **Return value**: is it `dp[n]`, `max(dp)`, or something else?

## Signal phrases
- "minimum/maximum number of steps/coins/operations"
- "number of ways to..."
- "can you form/break/partition"
- "longest/shortest subarray/subsequence satisfying..."

## Problems using this pattern
- [[dsa/problems/word-break]]

## Sources
- [[DSA overview]]
