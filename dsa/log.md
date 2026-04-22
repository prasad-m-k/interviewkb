# DSA Log

Append-only. Each entry: `## [YYYY-MM-DD] <type> | <title>`
Types: `ingest`, `query`, `lint`, `update`

Grep tip: `grep "^## \[" dsa/log.md | tail -10`

---

## [2026-04-21] update | Apple most-asked DSA problems
- Created: companies/apple (process, emphasis, frequently seen problems, behavioral tips)
- Created concepts: binary-tree (traversal orders, BST, height, LCA, serialization)
- Created patterns: tree-traversal (DFS pre/in/post templates, BFS level-order, LCA)
- Created problems: merge-intervals, serialize-deserialize-binary-tree, binary-tree-level-order-traversal, trapping-rain-water, product-of-array-except-self, lowest-common-ancestor, maximum-subarray
- Updated: dsa/index.md — added Apple-specific problem section, companies entry, new concept/pattern

## [2026-04-21] ingest | Data structure internals — senior-level concepts
- Created: concepts/hash-collision-resolution (chaining vs open addressing, Robin Hood hashing, tombstones)
- Created: concepts/hash-map-internals (load factor, rehashing, Python dict internals, Java HashMap treeification, adversarial hash flooding)
- Created: concepts/dynamic-array-amortized (doubling proof, O(1) amortized append, worst-case latency gotcha)
- Created: concepts/tree-balancing (BST degeneration, AVL, Red-Black, B-Tree, why databases use B-trees)
- Created: concepts/heap-internals (array layout, sift up/down, O(n) heapify proof, d-ary heaps)
- Updated: topics/data-structures — added internals links and Trees section
- Updated: concepts/hash-map, concepts/heap — linked to internals pages

## [2026-04-21] update | DSA knowledge base initialized
- Created directory structure: dsa/, topics/, concepts/, patterns/, problems/, flashcards/
- Created: dsa/index.md, dsa/log.md, dsa/overview.md
- Created topics: data-structures, algorithms
- Created concepts: hash-map, heap, graph, dynamic-programming, deque
- Created patterns: sliding-window, bfs-dfs, topological-sort, two-heaps
- Created 10 problems: lru-cache, course-schedule-ii, sliding-window-maximum, find-median-from-data-stream, top-k-frequent-elements, number-of-islands, merge-k-sorted-lists, task-scheduler, word-break, design-hit-counter
- Created flashcards: dsa-mlops-top10 (Obsidian), dsa-mlops-top10-anki.txt (Anki TSV)
- Domain: DSA for senior MLOps engineers at FAANG
