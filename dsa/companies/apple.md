# Apple Interview Prep

## Process
- **Phone screen** (30 min): behavioral + 1 coding problem, usually Easy–Medium
- **Technical phone screen** (45–60 min): 1–2 coding problems, Medium difficulty
- **On-site** (4–5 rounds, each 45–60 min):
  - 2–3 coding rounds
  - 1 system design round (senior roles)
  - 1 hiring manager / behavioral round

Rounds are independent — each interviewer makes their own hire/no-hire call. There is no debrief loop; a single strong no-hire can block the offer.

## Emphasis
Apple weights **clean, production-quality code** heavily. Interviewers look for:
- Well-named variables, no magic numbers
- Edge case handling *proactively mentioned*, not prompted
- Time and space complexity stated before coding begins
- Iterative solutions preferred over recursive for large inputs (Python recursion limit)

Apple is **less LeetCode-grindy** than Meta or Google. They favor problems with real-world analogues and probe "what would you change if the input were 1 billion elements?"

## Frequently tested topics
- **Trees** (heavy ~50% of rounds): [[dsa/trees/index]] — BST, views, path sums, invert, serialize, LCA
- **Arrays / Two Pointers**: Trapping Rain Water, Product of Array Except Self
- **Intervals**: Merge Intervals, Insert Interval
- **Strings**: Serialize tree, anagrams, palindromes
- **Dynamic Programming**: Max subarray, word break
- **Graphs / BFS**: [[dsa/problems/number-of-islands]], [[dsa/problems/binary-tree-level-order-traversal]]
- **Design**: [[dsa/problems/lru-cache]], [[dsa/problems/design-hit-counter]]

## Frequently seen problems

### Trees (appear in ~50% of Apple coding rounds)

| Problem | Difficulty | What Apple tests |
|---|---|---|
| [[dsa/problems/binary-tree-level-order-traversal]] | Medium | BFS level snapshot; often a warm-up |
| [[dsa/trees/problems/right-side-view]] | Medium | Last per level; follow-up: left view |
| [[dsa/trees/problems/diameter-of-binary-tree]] | Easy | Bottom-up DFS; follow-up leads to max path sum |
| [[dsa/trees/problems/binary-tree-maximum-path-sum]] | Hard | Return gain up + accumulate globally; negative values |
| [[dsa/problems/lowest-common-ancestor]] | Medium | DFS postorder; know BST vs. general tree |
| [[dsa/problems/serialize-deserialize-binary-tree]] | Hard | Preorder + null markers; full round-trip |
| [[dsa/trees/problems/validate-bst]] | Medium | Global invariant, not local; bounds propagation |
| [[dsa/trees/problems/kth-smallest-bst]] | Medium | Iterative inorder; "can you avoid recursion?" |
| [[dsa/trees/problems/construct-from-preorder-inorder]] | Medium | Divide & conquer; hashmap for O(n) |
| [[dsa/trees/problems/invert-binary-tree]] | Easy | Warm-up; know both recursive and iterative |
| [[dsa/trees/problems/path-sum]] | Easy/Med | Path Sum I (exists?) and II (all paths); backtracking |
| [[dsa/trees/problems/flatten-to-linked-list]] | Medium | In-place O(1) space transformation |
| [[dsa/trees/problems/symmetric-tree]] | Easy | Paired recursion; tests clean thinking |
| [[dsa/trees/problems/count-good-nodes]] | Medium | Top-down carry; less common but reported |

**Apple tree interview tips:**
- Every recursive tree solution → expect "can you do this iteratively?" Have both ready.
- For level-order problems, explain `len(queue)` as a level snapshot before writing code.
- BST problems: always state the global invariant, not just parent-child, before coding.
- Max path sum: draw the two-quantity distinction (return-to-parent vs. local-best) on the board first.

### Arrays / Two Pointers
- [[dsa/problems/merge-intervals]] — reported in almost every Apple interview thread
- [[dsa/problems/trapping-rain-water]] — tests two-pointer depth
- [[dsa/problems/product-of-array-except-self]] — tests prefix/suffix pattern
- [[dsa/problems/maximum-subarray]] — Kadane's; tests DP intuition on arrays

### Other
- [[dsa/problems/number-of-islands]] — BFS/DFS flood fill
- [[dsa/problems/lru-cache]] — system design-adjacent coding

## Behavioral / culture fit
Apple culture is less explicit than Amazon's (no published leadership principles). Interviewers probe for:
- **Ownership and craftsmanship** — "I cared about getting this right, not just shipping"
- **Cross-functional collaboration** — working with hardware, design, and firmware teams
- **Handling ambiguity** — "the spec was unclear; here's how I drove clarity"
- **Privacy and security awareness** — Apple's core brand; mention it when relevant

Common behavioral questions:
- "Tell me about a time you disagreed with a technical decision."
- "Describe a project where you had to balance speed and correctness."
- "How do you handle code review feedback you disagree with?"

## Tips from sources
- Apple interviewers often follow up Medium problems with "what if the input doesn't fit in memory?" — always mention external sort / streaming variants
- Trees appear in roughly half of Apple coding rounds; review all four traversal orderings cold
- Apple likes interval problems — internalize Merge Intervals and Insert Interval patterns
- Recursive solutions get a follow-up: "can you do this iteratively?" — have both ready for tree problems
- Whiteboard (or shared editor) cleanliness matters; Apple engineers care about code aesthetics

## Sources
*(none yet)*
