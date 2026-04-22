# Git Flashcards (Anki Format)

> Import into Anki: use the "Basic" card type. Front = Q, Back = A.
> Or use the Obsidian Anki Sync plugin — each `## Q:` / `**A:**` block becomes a card.

---

## Fundamentals

## Q: What are the three trees in Git and what does each contain?
**A:**
1. **Working Tree** — files as they exist on disk
2. **Staging Area (Index)** — snapshot of the proposed next commit
3. **Repository (HEAD)** — the last committed snapshot

`git add` moves from working tree → index. `git commit` moves from index → HEAD.

---

## Q: What four object types does Git store, and what does each contain?
**A:**
- **blob** — raw file content
- **tree** — directory listing (refs to blobs + subtrees)
- **commit** — tree ref, parent SHA(s), author, message
- **tag** (annotated) — points to any object with metadata + optional signature

---

## Q: What is HEAD in Git?
**A:** HEAD is a pointer to the currently checked-out branch (or commit). Normally it points to a branch name (e.g., `refs/heads/main`), which in turn points to the tip commit. When HEAD points directly to a SHA, it's called **detached HEAD**.

---

## Q: What is a detached HEAD and how do you recover work from it?
**A:** HEAD points to a commit SHA instead of a branch. Any commits made won't be referenced by a branch and will be GC'd.

Recovery: `git switch -c new-branch` to capture the floating commits into a named branch.

---

## Q: What is the difference between `git diff`, `git diff --cached`, and `git diff HEAD`?
**A:**
- `git diff` — working tree vs. **index** (unstaged changes)
- `git diff --cached` — index vs. **HEAD** (staged changes, what will be committed)
- `git diff HEAD` — working tree vs. **HEAD** (all uncommitted changes)

---

## Q: What does `git add -p` do?
**A:** Interactively stages **hunks** (sections) of files. Lets you craft precise commits by choosing which changes to include, even within a single file. Options: `y` (stage), `n` (skip), `s` (split), `e` (edit hunk manually).

---

## Branching and Merging

## Q: What is a Git branch at the filesystem level?
**A:** A file in `.git/refs/heads/<name>` containing the 40-character SHA of the tip commit. Creating/deleting a branch is O(1) — it just writes or removes this small file.

---

## Q: When does a fast-forward merge happen vs. a 3-way merge?
**A:**
- **Fast-forward**: the current branch is a direct ancestor of the target — git just moves the pointer. No new commit.
- **3-way merge**: both branches have diverged — git finds the common ancestor and combines both sets of changes, creating a merge commit with two parents.

---

## Q: What is the golden rule of rebasing?
**A:** Never rebase commits that have been pushed to a **shared/public branch**. Rebasing rewrites SHAs; anyone who has fetched the old SHAs will have a diverged history.

---

## Q: What are the interactive rebase actions? Name 5.
**A:**
- `pick` — keep commit as-is
- `reword` — keep commit, change message
- `squash` — melt into previous commit, merge messages
- `fixup` — melt into previous commit, discard this message
- `drop` — delete the commit entirely
- `edit` — pause to amend the commit

---

## Q: What is `git merge --squash`?
**A:** Stages all changes from the feature branch as a single unstaged change in the current branch, without creating a merge commit or preserving branch history. You then commit it as one clean commit. The feature branch has no parent link in the resulting commit.

---

## Q: How do you resolve a conflict for a binary file?
**A:** Binary files can't be text-merged. You must choose one side entirely:
```bash
git checkout --ours path/file.png    # keep our version
git checkout --theirs path/file.png  # keep their version
git add path/file.png
git merge --continue
```

---

## Q: What does `git rebase --onto` do?
**A:** Replays a range of commits onto a new base, transplanting them from their original parent.

```bash
git rebase --onto <newbase> <upstream> <branch>
```

Useful when feature-b was branched from feature-a and you want to rebase it directly onto main.

---

## Undo and Recovery

## Q: What is the difference between `git reset --soft`, `--mixed`, and `--hard`?
**A:**
| Flag | HEAD | Index | Working Tree |
|------|------|-------|--------------|
| `--soft` | moves | unchanged | unchanged |
| `--mixed` (default) | moves | reset | unchanged |
| `--hard` | moves | reset | reset |

---

## Q: When should you use `git revert` vs. `git reset`?
**A:**
- `git revert` — creates a new commit undoing the change. **Safe for shared branches.** Preserves history.
- `git reset` — moves HEAD backward, rewrites history. **Only for local, unpushed commits.**

Rule: if the commit is already pushed and others may have it, always use `revert`.

---

## Q: How do you recover commits lost after `git reset --hard`?
**A:**
```bash
git reflog           # find the SHA or HEAD@{n} before the reset
git reset --hard HEAD@{3}   # jump back to it
```
Reflog retains entries for 90 days by default.

---

## Q: What is the reflog and what is it NOT?
**A:** The reflog is a **local, append-only journal** of every time HEAD or a branch pointer moved. It is NOT pushed to the remote — it's per-clone. It allows recovery of commits lost via reset, rebase, branch deletion.

---

## Q: How do you revert a merge commit?
**A:**
```bash
git revert -m 1 <merge-commit-sha>
```
`-m 1` selects "mainline parent 1" — the branch you merged into. This tells git which parent to treat as the base for the revert.

---

## Remote and Collaboration

## Q: What is the difference between `git fetch` and `git pull`?
**A:**
- `git fetch` — downloads new objects and updates remote-tracking refs (`origin/main`). Does **not** touch your local branch or working tree. Safe.
- `git pull` — `fetch` + `merge` (or `rebase` with `--rebase`). Integrates the remote changes into your current branch.

---

## Q: What is `--force-with-lease` and why prefer it over `--force`?
**A:** `--force-with-lease` only force-pushes if the remote branch **hasn't changed since your last fetch**. It prevents overwriting a teammate's commits that were pushed between your fetch and your push. `--force` overwrites regardless.

---

## Q: What is a remote-tracking branch?
**A:** A local, read-only reference (e.g., `origin/main`) that mirrors the last known state of the remote branch. Updated by `git fetch`. Your local `main` and `origin/main` are separate refs — they diverge until you fetch/pull/push.

---

## Q: How do you delete a remote branch?
**A:**
```bash
git push origin --delete branch-name
```
Or via GitHub CLI: `gh pr merge` with `--delete-branch`, or in the PR UI after merging.

---

## Q: Describe the fork-and-PR workflow.
**A:**
1. Fork the repo on GitHub (creates your copy)
2. Clone your fork: `git clone git@github.com:you/repo.git`
3. Add upstream: `git remote add upstream <original-url>`
4. Stay current: `git fetch upstream && git rebase upstream/main`
5. Create branch, make changes, push to your fork
6. Open PR from your fork's branch to the original repo's main
7. After merge: `git pull upstream main && git push origin main`

---

## CI/CD and Operations

## Q: Why are shallow clones used in CI, and what breaks with them?
**A:** Shallow clones (`--depth=1`) download only recent commits, making CI checkout much faster. Breaks:
- `git describe` (needs tag ancestry)
- `git bisect` (needs full history)
- `git log --follow` (needs full history for renames)

Fix: `git fetch --unshallow` or set `fetch-depth: 0` in GitHub Actions.

---

## Q: How do you tag a Docker image with the Git SHA for traceability?
**A:**
```bash
SHA=$(git rev-parse --short HEAD)
docker build -t myapp:${SHA} .
docker push myapp:${SHA}
```
Every deployment is now traceable: image tag → Git SHA → exact source code.

---

## Q: What is a pre-receive hook and how is it different from a pre-commit hook?
**A:**
- **pre-commit** — client-side, local. Runs before commit is created. Can be skipped with `--no-verify`.
- **pre-receive** — server-side, on the Git server. Runs when a push arrives. **Cannot be bypassed** by the client. Used to enforce org-wide policies (branch naming, no secrets, signed commits).

---

## Q: What is GitOps?
**A:** GitOps uses Git as the single source of truth for infrastructure and deployment configuration. A reconciliation agent (ArgoCD, Flux) continuously ensures the live environment matches the state declared in Git. Benefits: full audit trail, rollback = `git revert`, PR review for infra changes.

---

## Q: How do branch protection rules create a quality gate?
**A:** Branch protection rules (GitHub Settings → Branches) can require:
1. Required status checks (CI jobs must pass via Commit Status API)
2. Required approvals (N reviewers)
3. No force-push / no deletion
4. Linear history (squash or rebase only)
5. Signed commits

The Merge button stays locked until all conditions pass.

---

## Advanced

## Q: What is `git bisect` and what is its time complexity?
**A:** `git bisect` performs a **binary search** through commit history to find which commit introduced a bug. Given good and bad commit, it tests the midpoint each step. Time complexity: **O(log n)** — ~10 steps for 1000 commits. Automate with `git bisect run <test-script>`.

---

## Q: What is `git rerere`?
**A:** "Reuse Recorded Resolution" — git caches how you resolved a conflict and automatically replays that resolution if the same conflict appears again. Enable with: `git config rerere.enabled true`. Useful when frequently rebasing long-running branches onto main.

---

## Q: What is `git worktree`?
**A:** Allows checking out **multiple branches simultaneously into separate directories**, all sharing one `.git` directory.
```bash
git worktree add ../hotfix hotfix/crash
# work in ../hotfix without disturbing current branch
git worktree remove ../hotfix
```
Useful for reviewing PRs or running tests on another branch without stashing.

---

## Q: How does `git cherry-pick` differ from `git merge`?
**A:**
- `cherry-pick` — applies the **diff of specific commit(s)** onto the current branch. Creates new commits with new SHAs. No parent link to the source branch.
- `merge` — integrates the entire diverged history of a branch, creating a merge commit with two parents.

Use cherry-pick to backport individual fixes; use merge to integrate entire feature branches.

---

## Q: What does `git filter-repo` do and when would you use it?
**A:** Rewrites Git history to remove or transform content — typically used to scrub accidentally committed secrets or large binary files from the entire history.
```bash
git filter-repo --path secrets.env --invert-paths
```
After this: rotate the secret immediately (assume compromised), force-push all branches, have all teammates re-clone.

---

## Q: What is sparse checkout and when is it useful?
**A:** Sparse checkout lets you work with only a **subset of files** from a large repository without downloading everything. Useful in monorepos to reduce clone size and scope.
```bash
git clone --filter=blob:none --sparse <url>
git sparse-checkout set services/auth services/payments
```

---

## Q: What is the difference between an annotated tag and a lightweight tag?
**A:**
- **Lightweight** — just a ref pointing to a commit SHA. No metadata.
- **Annotated** — a full Git object with tagger, date, message, optional GPG signature. Used by `git describe`. **Prefer for releases.**

---

## Q: How do you find which commit last changed a specific line in a file?
**A:**
```bash
git blame file.py
git blame -L 42,55 file.py    # specific line range
```
Shows the commit SHA, author, and date for each line.

---

## Q: What does `git log --follow` do?
**A:** Follows a file's history **across renames**. Without `--follow`, git log stops at the point where the file was renamed.
```bash
git log --follow -p -- src/auth.py   # full history + diffs
```
