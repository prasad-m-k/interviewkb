# Pro Git (Scott Chacon & Ben Straub)

**Type:** book (free online at git-scm.com/book)
**Ingested:** 2026-04-22
**Topics covered:** [[git/topics/core-concepts]], [[git/topics/branching-and-merging]], [[git/topics/collaboration]], [[git/topics/workflows]], [[git/topics/ci-cd]]

## Summary

Pro Git is the canonical reference for Git, written by a GitHub co-founder. Available free online. Covers everything from first principles (object model, three-tree architecture) through advanced topics (rerere, filter-repo, submodules, worktrees, partial clone).

Part 1–3 (fundamentals through remotes) are essential for any developer. Part 7 (Git tools) covers the advanced features most engineers don't know: rerere, subtree, filter-repo, credential helpers, bundle. Part 10 (Git internals) explains the plumbing commands and object model in depth.

## Key Takeaways

- Git's object model (blob/tree/commit/tag) is the foundation for understanding all behaviors
- The three-tree architecture (working tree / index / HEAD) explains every add/commit/reset/restore operation
- Branches are cheap pointers — this is the insight that makes feature branching practical
- Rebase rewrites history; the golden rule is never rebase shared commits
- `git reflog` is the safety net that makes destructive operations recoverable
- Server-side hooks (pre-receive) are the only reliable enforcement point for policy

## What it updated
- Created all pages under [[git/]] from scratch using Pro Git as the primary reference
- Formed the basis for [[git/overview]], [[git/topics/core-concepts]], [[git/concepts/commit]], [[git/concepts/branch]], [[git/concepts/rebase]], [[git/concepts/merge]], [[git/concepts/reflog]], [[git/concepts/bisect]], [[git/scenarios/interview-questions]]
