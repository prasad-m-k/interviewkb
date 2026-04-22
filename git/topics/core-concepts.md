# Core Git Concepts

**Related:** [[git/overview]], [[git/concepts/commit]], [[git/concepts/branch]], [[git/concepts/staging-area]]

## The Three Trees

Git's design revolves around three trees (file collections):

```
HEAD          Index           Working Tree
(last commit) (staged)        (on disk)
    │              │                │
    └─── git diff --cached ────────▶│
    │              └── git diff ───▶│
    └──────── git diff HEAD ───────▶│
```

All Git operations move data between these three trees.

## Commits

A commit is a **snapshot**, not a diff. It stores a pointer to the root tree object representing the entire project state. Parent pointer(s) form the DAG.

```
commit abc123
  tree  def456
  parent 789abc
  author ...
  committer ...

  Add login feature
```

## Refs and HEAD

- **Branch ref** — `.git/refs/heads/<name>` — moves forward with each commit
- **Remote-tracking ref** — `.git/refs/remotes/origin/<name>` — updated on fetch
- **Tag ref** — `.git/refs/tags/<name>` — immutable (lightweight) or signed (annotated)
- **HEAD** — `.git/HEAD` — points to a branch ref (attached) or a SHA (detached)

## Staging Area (Index)

The index is a binary file (`.git/index`) that stores:
- File paths, modes, and SHA-1 blobs for the next commit
- Conflict markers during merges

This is what makes `git add -p` powerful — you can stage partial hunks from a file.

## Pack Files

Loose objects are periodically compacted into **pack files** which store deltas. This is why git repos stay small even with long history. `git gc` triggers repacking.

## Interview Angles
- What happens when you `git commit`? (write tree, commit object, advance HEAD)
- What is a detached HEAD? (HEAD points to a SHA, not a branch)
- How does git store binary files efficiently? (pack files + delta compression)
- Difference between `git diff`, `git diff --cached`, `git diff HEAD`

## Sources
- [[git/sources/pro-git]]
