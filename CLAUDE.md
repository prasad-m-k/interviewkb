# LLMWiki — Technical Interview Prep

This is a persistent, LLM-maintained knowledge base for technical interview preparation. You (the LLM) own the `mlops/` and `dsa/` directories entirely. You read from `sources/` but never modify it.

---

## Directory Layout

```
mk_kb/
├── CLAUDE.md              ← this file (schema + instructions)
├── sources/               ← raw source documents (immutable — read only)
├── mlops/                 ← MLOps interview prep knowledge base
└── dsa/                   ← Data Structures & Algorithms knowledge base
    ├── index.md           ← content catalog (update on every ingest)
    ├── log.md             ← append-only chronological log
    ├── overview.md        ← high-level synthesis of the full knowledge base
    ├── topics/            ← top-level topic survey pages
    ├── concepts/          ← individual concept deep-dives
    ├── patterns/          ← algorithmic / design patterns
    ├── problems/          ← specific problems with approaches and solutions
    ├── companies/         ← company-specific prep pages
    └── sources/           ← summaries of ingested source documents
```

---

## Page Taxonomy

### topics/
High-level survey pages. One page per major interview domain. Each page links out to relevant concepts, patterns, and problems within that domain.

Current domains:
- `topics/data-structures.md`
- `topics/algorithms.md`
- `topics/system-design.md`
- `topics/behavioral.md`
- `topics/language-python.md` (add per language as needed)

### concepts/
One page per individual concept. Examples: `concepts/hash-map.md`, `concepts/binary-search.md`, `concepts/cap-theorem.md`, `concepts/consistent-hashing.md`.

Concept page structure:
```
# <Concept Name>

**Topic:** [[topics/...]]
**Related:** [[concepts/...]], [[patterns/...]]

## What it is
One paragraph plain-English definition.

## How it works
Key mechanics. Diagrams in ascii/mermaid if helpful.

## Complexity
Time and space complexity where applicable.

## When to use
Decision criteria — what signals in a problem suggest this concept.

## Common interview angles
Typical questions, gotchas, follow-ups interviewers ask.

## Examples
Short concrete examples.

## Sources
- [[sources/...]]
```

### patterns/
Reusable problem-solving patterns. Examples: `patterns/sliding-window.md`, `patterns/two-pointers.md`, `patterns/bfs-dfs.md`, `patterns/dynamic-programming.md`, `patterns/divide-and-conquer.md`.

Pattern page structure:
```
# <Pattern Name>

**Topic:** [[topics/algorithms]]
**Related concepts:** [[concepts/...]]

## What it solves
The class of problems this pattern applies to.

## Template / skeleton
Pseudocode or Python skeleton.

## Signal phrases
Problem wording that suggests this pattern ("find all substrings...", "minimum window...").

## Complexity
Typical time/space.

## Problems using this pattern
- [[problems/...]]

## Sources
- [[sources/...]]
```

### problems/
One page per notable problem. Name files by problem slug: `problems/two-sum.md`, `problems/lru-cache.md`, `problems/word-ladder.md`.

Problem page structure:
```
# <Problem Name>

**Difficulty:** Easy / Medium / Hard
**Topic:** [[topics/...]]
**Pattern:** [[patterns/...]]
**Companies:** [[companies/...]]

## Problem
Brief statement of the problem.

## Approach
Walk through the reasoning, not just the code.

## Solution (Python)
\```python
# solution here
\```

## Complexity
Time: O(...) | Space: O(...)

## Key insight
The one thing you must not forget.

## Variants / follow-ups
Common twists interviewers add.

## Sources
- [[sources/...]]
```

### companies/
One page per target company. Examples: `companies/google.md`, `companies/amazon.md`, `companies/meta.md`.

Company page structure:
```
# <Company> Interview Prep

## Process
Rounds, format, what each round tests.

## Emphasis
What this company weights heavily vs. rarely asks.

## Frequently tested topics
- [[topics/...]], [[concepts/...]]

## Frequently seen problems
- [[problems/...]]

## Behavioral / culture fit
Key leadership principles or values to address.

## Tips from sources
Bullet points with citations.

## Sources
- [[sources/...]]
```

### sources/
One page per ingested source document. Name files by a short slug: `sources/cracking-coding-interview.md`, `sources/system-design-primer.md`.

Source page structure:
```
# <Source Title>

**Type:** book / article / video / course / blog post
**Ingested:** YYYY-MM-DD
**Topics covered:** [[topics/...]]

## Summary
2–4 paragraph summary of key content.

## Key takeaways
Bullet list of the most important insights.

## What it updated
- Updated [[concepts/...]] with ...
- Created [[patterns/...]]
- Added to [[companies/...]]
```

---

## Workflows

### Ingest
When the user provides a new source (file path, URL, or pasted content):

1. Read the source thoroughly.
2. Discuss key takeaways with the user — ask what to emphasize before writing.
3. Create `mlops/sources/<slug>.md` with a full summary and key takeaways.
4. For each significant concept, pattern, or problem in the source:
   - If the mlops page exists: open it and integrate the new information — update, expand, note contradictions.
   - If it doesn't exist: create the page using the appropriate template above.
5. Update relevant `topics/` pages to link to any new concept/pattern/problem pages.
6. Update `mlops/index.md` — add new pages, update summaries for modified pages.
7. Append an entry to `mlops/log.md`.

A single source typically touches 5–15 wiki pages. That is expected and good.

### Query
When the user asks a question:

1. Read `mlops/index.md` to identify relevant pages.
2. Read those pages in full.
3. Synthesize an answer with citations to wiki pages (`[[...]]`).
4. If the answer is non-trivial (a comparison, an analysis, a decision framework), offer to file it as a new wiki page — good answers compound the knowledge base.

### Lint
When the user asks for a wiki health check:

1. Read all pages in `mlops/index.md`.
2. Check for:
   - Contradictions between pages
   - Stale claims superseded by newer sources
   - Orphan pages with no inbound links
   - Concepts mentioned but lacking their own page
   - Missing cross-references between related concepts/patterns/problems
   - Data gaps (important topics with thin coverage)
3. Report findings as a prioritized list.
4. Suggest new sources to look for or new questions to investigate.
5. Append a lint entry to `mlops/log.md`.

---

## index.md Format

```markdown
# Wiki Index
Last updated: YYYY-MM-DD

## Overview
- [[overview]] — High-level synthesis of the full knowledge base

## Topics
- [[topics/data-structures]] — Arrays, linked lists, trees, graphs, heaps, hash maps
- [[topics/algorithms]] — Sorting, searching, traversal, dynamic programming
...

## Concepts
- [[concepts/binary-search]] — O(log n) search on sorted arrays; common interview staple
...

## Patterns
- [[patterns/sliding-window]] — Variable/fixed window over arrays or strings
...

## Problems
- [[problems/two-sum]] — Medium; hash map; Amazon, Google
...

## Companies
- [[companies/google]] — Heavy on algorithms, system design, ambiguity
...

## Sources
- [[sources/cracking-coding-interview]] — Book; comprehensive DSA coverage
...
```

---

## log.md Format

Each entry starts with `## [YYYY-MM-DD] <type> | <title>` for easy grepping.

Types: `ingest`, `query`, `lint`, `update`

```markdown
## [2026-04-21] ingest | Cracking the Coding Interview
- Created: sources/cracking-coding-interview, concepts/hash-map, patterns/sliding-window (×3 more)
- Updated: topics/algorithms, topics/data-structures
- Notes: Emphasized hash map patterns and the two-pointer section

## [2026-04-21] query | What sliding window problems does Google ask?
- Filed answer as: problems/minimum-window-substring
```

---

## Conventions

- All internal links use Obsidian wikilink syntax: `[[path/page-name]]` (no `.md` extension).
- File names: lowercase, hyphen-separated. No spaces.
- Keep concept pages focused on one concept. Resist combining two concepts on one page — link instead.
- When new information contradicts an existing page, note the contradiction explicitly rather than silently overwriting: use a `> **Note:** ...` blockquote.
- Never delete content from `log.md` — it is append-only.
- The user reads the wiki in Obsidian. Use Obsidian-compatible markdown. Mermaid diagrams are supported.

## graphify

This project has a graphify knowledge graph at graphify-out/.

Rules:
- Before answering architecture or codebase questions, read graphify-out/GRAPH_REPORT.md for god nodes and community structure
- If graphify-out/wiki/index.md exists, navigate it instead of reading raw files
- After modifying code files in this session, run `graphify update .` to keep the graph current (AST-only, no API cost)
