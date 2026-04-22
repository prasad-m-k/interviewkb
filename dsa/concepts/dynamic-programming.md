# Dynamic Programming

**Topic:** [[dsa/topics/algorithms]]
**Related:** [[dsa/patterns/dynamic-programming]], [[dsa/problems/word-break]]

## What it is
Dynamic programming (DP) solves problems by breaking them into overlapping subproblems and storing results to avoid recomputation. Applies when the problem has **optimal substructure** (optimal solution is built from optimal subsolutions) and **overlapping subproblems** (same subproblems solved multiple times).

## How it works

**Bottom-up (tabulation):**
1. Define `dp[i]` semantically in plain English
2. Write the recurrence relation: `dp[i]` in terms of previous states
3. Identify base cases
4. Fill the table in the correct order
5. Return the target state

**Top-down (memoization):**
1. Write a recursive solution
2. Add a cache (e.g., `@lru_cache` or `memo = {}`)
3. Return cached result if already computed

Bottom-up is preferred in interviews: no recursion depth issues, often simpler to trace.

## Complexity
Typically O(n) or O(n²) time and space. Space can often be optimized to O(1) or O(k) when only the last few states are needed.

## When to use
Signal phrases in problem statements:
- "minimum / maximum number of ways"
- "how many ways to reach / make / partition"
- "can you form / break / spell"
- "longest / shortest subsequence or subarray"

## Common interview angles
- **1D DP**: Fibonacci, Climbing Stairs, House Robber, Word Break
- **2D DP**: Longest Common Subsequence, Edit Distance, Unique Paths
- **Knapsack**: 0/1 Knapsack, Coin Change, Partition Equal Subset Sum
- **String DP**: Word Break, Palindrome Partitioning

## Examples
```python
# Word Break (1D DP)
def wordBreak(s, wordDict):
    word_set = set(wordDict)
    dp = [False] * (len(s) + 1)
    dp[0] = True  # empty string is always breakable
    for i in range(1, len(s) + 1):
        for j in range(i):
            if dp[j] and s[j:i] in word_set:
                dp[i] = True
                break
    return dp[len(s)]

# Coin Change (1D DP — unbounded knapsack)
def coinChange(coins, amount):
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i:
                dp[i] = min(dp[i], dp[i - coin] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1
```

## Sources
- [[DSA overview]]
