# Git Scenario-Based Interview Questions

**Related:** [[git/scenarios/merge-conflict]], [[git/scenarios/hotfix-production]], [[git/scenarios/revert-bad-commit]], [[git/scenarios/bisect-regression]], [[git/scenarios/clean-up-history]]

---

## Fundamentals

### Q1: What happens internally when you run `git commit`?
**A:** Git:
1. Hashes each staged file into blob objects (if not already in object store)
2. Creates a tree object representing the directory structure
3. Creates a commit object pointing to the tree, parent commit(s), author, message
4. Moves the current branch pointer (e.g., `main`) to the new commit SHA

### Q2: Explain the difference between `git pull` and `git fetch`.
**A:** `git fetch` downloads new objects and updates remote-tracking refs (`origin/main`) but does not touch your local branch or working tree. `git pull` = `git fetch` + `git merge` (or rebase with `--rebase`). Prefer `fetch` + explicit `rebase` for transparency.

### Q3: What is a detached HEAD state and how do you recover from it?
**A:** HEAD points directly to a commit SHA instead of a branch name. Any commits made are not referenced by a branch — they'll be GC'd eventually. Recovery: `git switch -c new-branch` to capture the work.

### Q4: What is the difference between `git reset --soft`, `--mixed`, and `--hard`?
**A:**

| Flag | HEAD | Index | Working Tree | Use when |
|------|------|-------|--------------|---------|
| `--soft` | moves back | unchanged | unchanged | uncommit but keep staged |
| `--mixed` | moves back | reset | unchanged | uncommit + unstage; keep file changes |
| `--hard` | moves back | reset | reset | discard everything |

### Q5: How would you undo the last commit without losing the changes?
**A:** `git reset --mixed HEAD~1` — moves HEAD back one commit, unstages changes, but file changes remain in working tree. Then edit and recommit.

---

## Branching and Merging

### Q6: When does a fast-forward merge happen, and when doesn't it?
**A:** Fast-forward happens when the current branch has no commits that the other branch doesn't have (i.e., it's a direct ancestor). When both branches have diverged, git must do a 3-way merge. Use `--no-ff` to always create a merge commit.

### Q7: A colleague rebased a shared branch and now your local copy has diverged. What do you do?
**A:** 
```bash
git fetch origin
git rebase origin/shared-branch  # replay your local commits on top of new history
# or if you have no local commits:
git reset --hard origin/shared-branch
```
Communicate with the team before rebasing shared branches in the future.

### Q8: How do you merge only specific commits from another branch?
**A:** `git cherry-pick <sha>` for individual commits; `git cherry-pick A..B` for a range. Useful for backporting fixes to release branches.

### Q9: You have a merge conflict in a binary file. How do you resolve it?
**A:** Git can't show a diff for binary files. You must choose one version entirely:
```bash
git checkout --ours   path/to/image.png   # keep our version
git checkout --theirs path/to/image.png   # keep their version
git add path/to/image.png
git merge --continue
```
Or use a merge driver registered in `.gitattributes` for specific binary types.

### Q10: Explain `git rebase --onto` and when you'd use it.
**A:** `git rebase --onto <newbase> <upstream> <branch>` replays `<branch>` (everything after `<upstream>`) onto `<newbase>`. Used when a feature branch was started from another feature branch, and you want to rebase it onto main without dragging in the other feature's commits.

```bash
# feature-b was branched from feature-a; rebase onto main
git rebase --onto main feature-a feature-b
```

---

## History and Recovery

### Q11: You ran `git reset --hard HEAD~3` and lost commits. How do you recover them?
**A:**
```bash
git reflog                    # find the SHA before the reset (HEAD@{1} etc.)
git reset --hard HEAD@{1}     # jump back to it
# or: git branch recovered HEAD@{1}
```
Reflog retains entries for 90 days by default.

### Q12: How do you find which commit introduced a bug, given you have 2000 commits to search?
**A:** `git bisect` — binary search in O(log n) ≈ 11 steps:
```bash
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
# test each checkout, mark good or bad
git bisect run pytest tests/test_regression.py   # automate it
git bisect reset
```

### Q13: A secret (API key) was committed 10 commits ago and pushed. What do you do?
**A:**
1. **Immediately rotate the secret** — assume it's compromised.
2. Scrub history:
```bash
git filter-repo --path secrets.env --invert-paths
git push --force-with-lease --all
```
3. All team members must re-clone or reset their local repos.
4. Notify the team and check if GitHub Secret Scanning flagged it.

### Q14: How do you view all changes made to a file across its entire history, including renames?
**A:**
```bash
git log --follow -p -- path/to/file.py
```
`--follow` tracks renames. `-p` shows the patch for each commit.

### Q15: What is `git bisect skip` and when do you use it?
**A:** Marks a commit as untestable (e.g., broken build unrelated to the bug). Git skips it and picks another commit to test. The final result may be a range rather than a single commit.

---

## Collaboration and Workflow

### Q16: What is the difference between `git merge` and `git rebase` in the context of integrating a feature branch?
**A:** 
- **Merge** preserves full history with a merge commit; shows exactly when branches diverged and merged. Preferred when history accuracy matters.
- **Rebase** replays feature commits on top of main, producing linear history. Easier to read, but rewrites SHAs. Preferred for local cleanup before a PR.
- In practice: rebase locally to clean up, merge (or squash merge) when landing the PR.

### Q17: Describe the fork-and-PR workflow for an open-source contribution.
**A:**
```bash
# 1. Fork on GitHub
git clone git@github.com:you/repo.git
git remote add upstream git@github.com:original/repo.git
git fetch upstream
git rebase upstream/main               # get latest
git switch -c fix/typo-in-readme
# make changes
git push origin fix/typo-in-readme
gh pr create                           # open PR to upstream
# after merge: clean up
git switch main
git pull upstream main
git push origin main
git branch -d fix/typo-in-readme
```

### Q18: What is `--force-with-lease` and why use it over `--force`?
**A:** `--force-with-lease` only force-pushes if the remote branch hasn't changed since your last fetch. It prevents overwriting commits pushed by a teammate between your fetch and push. `--force` overwrites regardless — dangerous on shared branches.

### Q19: How do branch protection rules integrate with CI/CD?
**A:** Branch protection rules (GitHub settings) can require:
- **Required status checks** — CI jobs must pass (reported via Commit Status API)
- **Required reviews** — N approvals needed
- **No force-push** — history is immutable
- **Linear history** — only squash or rebase merges allowed

This creates a quality gate: CI runs automatically on each PR push, and the merge button is gated on green status.

### Q20: How do you squash all commits on a feature branch into one before merging?
**A:**
```bash
# Method 1: interactive rebase
git rebase -i origin/main     # mark all but first as 'squash' or 'fixup'

# Method 2: merge --squash
git switch main
git merge --squash feature/login
git commit -m "feat: add login"

# Method 3: let GitHub do it (Squash and merge button)
gh pr merge 123 --squash
```

---

## CI/CD and Operations

### Q21: How do you set up a CI pipeline that deploys on tag push?
**A:** In GitHub Actions:
```yaml
on:
  push:
    tags:
      - 'v*'
jobs:
  deploy:
    steps:
      - uses: actions/checkout@v4
      - run: make build
      - run: make deploy
```
Tag with: `git tag -a v1.2.3 -m "Release" && git push --tags`

### Q22: How do you trace a running Docker image back to the exact source code?
**A:** Tag images with the Git SHA:
```bash
SHA=$(git rev-parse --short HEAD)
docker build -t myapp:${SHA} .
docker push myapp:${SHA}
```
The running container's image tag tells you exactly which commit to inspect.

### Q23: Your CI pipeline uses a shallow clone and `git describe` is failing. How do you fix it?
**A:** `git describe` needs the full tag history. In GitHub Actions:
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0    # 0 = full history
```
Or fetch tags explicitly: `git fetch --tags --unshallow`

### Q24: What is a pre-receive hook and how is it different from a pre-commit hook?
**A:**
- **pre-commit** — client-side, runs locally before a commit is created. Can be skipped with `--no-verify`.
- **pre-receive** — server-side, runs on the Git server when a push arrives. Cannot be bypassed by the client. Used to enforce org-wide policies (commit signing, no secrets, branch naming conventions).

### Q25: How do you implement a Git-based deployment strategy without environment branches?
**A:** Prefer tag-based or SHA-based deployments:
- Dev: auto-deploy every push to main
- Staging: deploy via workflow dispatch with a SHA input
- Prod: deploy on `v*` tag push with a manual approval gate

Environment branches (`staging`, `prod`) are an anti-pattern — they diverge from main and create a second merge burden.

---

## Advanced / Senior-Level

### Q26: How does `git rerere` work and when is it valuable?
**A:** rerere (Reuse Recorded Resolution) caches how you resolved a conflict the first time and automatically replays that resolution if the same conflict appears again. Valuable when:
- Long-running feature branches frequently rebase on main
- A recurring merge conflict appears across many PRs
```bash
git config rerere.enabled true
```

### Q27: What is `git worktree` and when would you use it?
**A:** `git worktree` lets you check out multiple branches into separate directories simultaneously, all sharing the same `.git` directory. Useful for:
- Reviewing a PR while keeping your current work intact
- Running tests on another branch without stashing
```bash
git worktree add ../hotfix-review hotfix/login-crash
cd ../hotfix-review && make test
git worktree remove ../hotfix-review
```

### Q28: Explain sparse checkout and when to use it in a monorepo.
**A:** Sparse checkout lets you clone only a subset of files/directories from a large monorepo:
```bash
git clone --filter=blob:none --sparse https://github.com/org/monorepo.git
cd monorepo
git sparse-checkout set services/auth services/payments
```
Dramatically reduces clone time and disk usage when you only work on one service.

### Q29: How would you audit every change ever made to a production configuration file?
**A:**
```bash
git log --all --follow -p -- config/production.yaml
# Shows every commit that touched this file, including renames, across all branches
git log --all --follow --format="%h %ai %an %s" -- config/production.yaml
```
For stronger auditability: enforce signed commits and protect the branch.

### Q30: A monorepo CI pipeline is slow because it runs all tests on every commit. How do you use Git to optimize it?
**A:** Use Git to detect which paths changed and run only affected tests:
```bash
CHANGED=$(git diff --name-only origin/main...HEAD)
if echo "$CHANGED" | grep -q "^services/auth/"; then
  run_auth_tests
fi
if echo "$CHANGED" | grep -q "^services/payments/"; then
  run_payments_tests
fi
```
Tools like `nx affected`, `turborepo`, and `bazel` build on this principle.

---

## Sources
- [[git/sources/pro-git]]
- [[git/scenarios/merge-conflict]]
- [[git/scenarios/hotfix-production]]
- [[git/scenarios/revert-bad-commit]]
- [[git/scenarios/bisect-regression]]
