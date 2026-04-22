# Collaboration with Git and GitHub

**Related:** [[git/concepts/remote]], [[git/topics/workflows]], [[git/cicd/github-actions]]

## Fork vs. Clone

| | Fork | Clone |
|--|------|-------|
| **What** | Server-side copy under your account | Local copy of any repo |
| **When** | Contributing to repos you don't own | Working on repos you own/have access to |
| **Upstream** | `upstream` = original; `origin` = your fork | `origin` = the repo |

```bash
# After forking on GitHub:
git clone git@github.com:you/repo.git
git remote add upstream git@github.com:original/repo.git
git fetch upstream
git rebase upstream/main
```

## Pull Request Workflow

```
feature branch (local)
    │
    ├── git push origin feature/login
    │
    ▼
GitHub PR
    │
    ├── CI checks (lint, tests, build)
    ├── Code review + approvals
    ├── Merge (squash / merge commit / rebase)
    │
    ▼
main branch
```

## Remote Tracking

```bash
git remote -v                          # list remotes
git remote add origin <url>            # add remote
git fetch origin                       # download objects, don't integrate
git pull origin main                   # fetch + merge
git pull --rebase origin main          # fetch + rebase (preferred for clean history)
git push -u origin feature/login       # push + set upstream tracking
git push --force-with-lease            # safe force-push (fails if remote moved)
```

`--force-with-lease` is the safe alternative to `--force` — it only pushes if the remote hasn't been updated by someone else since your last fetch.

## Code Review Best Practices (from Git perspective)

- **Small PRs** — easier to review, less likely to conflict
- **One logical change per commit** — reviewers can follow reasoning
- **Rebase before merging** — eliminates merge noise
- **Descriptive commit messages** — title ≤ 72 chars, body explains *why*

## GitHub-specific CLI Operations

```bash
# GitHub CLI (gh)
gh pr create --title "Add login" --body "Closes #42"
gh pr view 123
gh pr checkout 123          # check out a PR locally
gh pr merge 123 --squash
gh pr review 123 --approve
gh issue create --title "Bug: login fails" --label bug
gh repo clone owner/repo
```

## Protected Branches

Branch protection rules enforce:
- Required status checks (CI must pass)
- Required approvals (N reviewers)
- No force-push
- No deletion
- Signed commits (optional)
- Linear history (no merge commits — requires squash or rebase)

## Interview Angles
- When would you use `--force-with-lease` over `--force`?
- Describe the fork-and-PR workflow for open source.
- How do branch protection rules enforce quality gates?
- What is a tracking branch and how does git use it?

## Sources
- [[git/sources/pro-git]]
