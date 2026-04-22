# Word Break

**Difficulty:** Medium
**Topic:** [[dsa/topics/algorithms]]
**Pattern:** [[dsa/patterns/dynamic-programming]]
**Companies:** Google, Amazon, Meta, Apple

## Progress

- [ ] Read problem statement
- [ ] Solved independently
- [ ] Reviewed solution / key insight
- [ ] Can solve from memory

## Problem
Given a string `s` and a list of strings `wordDict`, return `True` if `s` can be segmented into a space-separated sequence of one or more dictionary words.

Example: `s = "leetcode"`, `wordDict = ["leet","code"]` → `True`
Example: `s = "applepenapple"`, `wordDict = ["apple","pen"]` → `True`

## Approach
This has the classic DP structure: can we reach position `i`?

Define `dp[i] = True` if `s[:i]` can be segmented into dictionary words.

**Recurrence:** `dp[i] = True` if there exists `j < i` such that `dp[j] = True` and `s[j:i]` is in the word dictionary.

**Base case:** `dp[0] = True` (empty string is trivially segmentable).

**Answer:** `dp[len(s)]`.

Convert `wordDict` to a `set` for O(1) lookup. The nested loop is O(n²) total.

## Solution (Python)
```python
from typing import List

def wordBreak(s: str, wordDict: List[str]) -> bool:
    word_set = set(wordDict)
    n = len(s)
    dp = [False] * (n + 1)
    dp[0] = True   # empty prefix is always valid

    for i in range(1, n + 1):
        for j in range(i):
            if dp[j] and s[j:i] in word_set:
                dp[i] = True
                break   # no need to check other j values

    return dp[n]
```

## Complexity
Time: O(n² × m) where n = len(s), m = avg word length (for substring hashing) | Space: O(n)

In practice, treat as O(n²) since string hashing is amortized O(m) but m is bounded.

## Key insight
The `break` after setting `dp[i] = True` prunes the inner loop. Without it, correctness is the same but we do extra work. More importantly: the substring `s[j:i]` check is O(m) not O(1), so for very long words, using `word_set` with a max-length cutoff (`if i - j > max_word_len: continue`) is a real optimization.

## Variants / follow-ups
- **Word Break II**: return all possible sentences (not just True/False) — use memoized DFS, not DP, because you need to reconstruct paths.
- **Concatenated Words**: given a word list, find all words that are concatenations of other words in the list — run Word Break for each word using the rest as the dictionary.
- **Interviewers ask**: "What if `wordDict` is huge?" → Build a trie from `wordDict`; walk the trie instead of doing set lookups. Reduces the inner loop to the length of the longest matching word.

## MLOps connection
Tokenization: given a vocabulary (like a BPE/WordPiece dictionary) and a string, can the string be fully tokenized? ML tokenizers solve exactly this — the Viterbi algorithm over the vocabulary is DP on strings. This problem is a simplified version of how a tokenizer checks whether an input can be encoded.

## Sources
- [[DSA overview]]
