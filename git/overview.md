# Git — Knowledge Base Overview

Git is a distributed version control system (DVCS) designed for speed, data integrity, and support for distributed, non-linear workflows. Every clone is a full repository with complete history — there is no canonical "server" at the protocol level.

## The Three-Tree Architecture

Git maintains three "trees" at all times:

```
Working Tree  →  Staging Area (Index)  →  Repository (HEAD)
   git add ↗                git commit ↗
              git restore ↙               git reset ↙
```

- **Working tree** — files on disk as you see them
- **Staging area (index)** — a proposed next commit; lets you craft precise commits
- **HEAD** — pointer to the tip of the current branch (or a detached commit)

## Object Model

Everything in Git is a content-addressed object stored under `.git/objects/`:

| Object | Contains |
|--------|----------|
| **blob** | file content |
| **tree** | directory listing (blob refs + tree refs) |
  **commit** | tree ref, parent ref(s), author, message |
| **tag** | annotated tag — points to any object |

SHA-1 (transitioning to SHA-256) identifies every object. Same content → same SHA → deduplication is automatic.

## Key Mental Models

### Branches are cheap pointers
A branch is just a 41-byte file containing a SHA. Creating or deleting a branch is O(1). Branches do not "contain" commits — they point to the tip of a chain of commits.

### Merging vs. Rebasing
- **Merge** — creates a new merge commit that has two parents; preserves exact history
- **Rebase** — replays commits on top of another base; rewrites SHAs; produces linear history
- Rule of thumb: rebase local work before sharing; never rebase shared history

### Distributed model
- Every clone has the full history. `origin` is a convention, not a server role.
- `fetch` downloads objects; `merge`/`rebase` integrates them. `pull` = `fetch` + `merge`.

## Interview Themes

| Theme | Key questions |
|-------|--------------|
| History manipulation | rebase -i, reset, revert, reflog |
| Conflict resolution | merge conflicts, ours/theirs, rerere |
| Collaboration | fork vs. clone, PR workflow, code review |
| CI/CD integration | branch protection, status checks, deploy keys |
| Debugging | bisect, blame, log --follow |
| Performance at scale | shallow clones, sparse checkout, partial clone |

## Canonical Workflows

1. **GitHub Flow** — one main branch + short-lived feature branches + PRs. Simple, suits continuous deployment.
2. **Git Flow** — develop/main + feature/release/hotfix branches. Suits scheduled releases.
3. **Trunk-Based Development** — everyone commits to main (or very short branches < 1 day). Requires strong CI and feature flags.

## Related
- [[git/topics/core-concepts]]
- [[git/topics/workflows]]
- [[git/topics/ci-cd]]
- [[git/scenarios/interview-questions]]
- [[git/cheatsheet]]
