# Commit

**Topic:** [[git/topics/core-concepts]]
**Related:** [[git/concepts/branch]], [[git/concepts/staging-area]], [[git/concepts/tags]]

## What it is
A commit is an immutable snapshot of the entire project tree at a point in time. It is **not** a diff — it stores the full state (efficiently, via shared tree/blob objects). Each commit has a SHA-1 identifier derived from its content.

## How it works

```
commit 3a7f1c2
  tree   b9d4e2a        ← root directory snapshot
  parent 8ef20c1        ← previous commit (none for initial commit)
  author  Alice <a@x.com> 1714000000 +0000
  committer Alice <a@x.com> 1714000000 +0000

  feat: add user login endpoint
```

Merge commits have **two** parent pointers.

## Commit Message Convention

```
<type>(<scope>): <short summary>   ← 50 chars max
                                    ← blank line
<body>                              ← 72 chars per line; explain WHY
                                    ← blank line
<footer>                            ← Closes #42, BREAKING CHANGE: ...
```

Types (Conventional Commits): `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`

## Complexity
Creating a commit: O(1) — just writes objects and moves HEAD.

## When to use
Commit early and often on local branches. Squash/fixup before sharing.

## Common interview angles
- "What is a commit SHA and what determines it?" (hash of content + parent + author + message + timestamp)
- "Can you change a commit message after pushing?" (yes, with `git commit --amend` + `git push --force-with-lease`, but only if no one else has the old SHA)
- "What is an empty commit and when is it useful?" (`git commit --allow-empty` — trigger CI without code changes)
- "How do you split a commit into two?" (rebase -i, `edit`, reset HEAD~, re-stage in parts)

## Examples

```bash
git commit -m "fix: prevent null pointer in auth middleware"

# Amend last commit (message or staged changes)
git commit --amend --no-edit

# Commit with GPG signing (required for some org policies)
git commit -S -m "feat: add feature"

# Empty commit to retrigger CI
git commit --allow-empty -m "ci: retrigger pipeline"
```

## Sources
- [[git/sources/pro-git]]
