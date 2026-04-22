# Remote

**Topic:** [[git/topics/collaboration]]
**Related:** [[git/concepts/branch]], [[git/topics/ci-cd]]

## What it is
A remote is a named reference to another repository (usually on GitHub, GitLab, or similar). The most common remote name is `origin` (the repo you cloned from). For forks, `upstream` refers to the original source.

## How it works

Remotes are stored in `.git/config`:
```ini
[remote "origin"]
    url = git@github.com:user/repo.git
    fetch = +refs/heads/*:refs/remotes/origin/*
```

**Remote-tracking branches** (e.g., `origin/main`) are local read-only references that mirror the last known state of the remote. They update on `fetch`.

```
fetch: download objects + update remote-tracking refs (no local branch changes)
pull:  fetch + merge (or rebase with --rebase)
push:  upload local commits to remote
```

## Complexity
Network-bound. Data transferred = only new objects (Git is efficient).

## When to use
- `fetch` before starting work to see what's changed upstream
- `pull --rebase` to stay current without merge commits
- `push --force-with-lease` when you need to update a branch you've rebased

## Common interview angles
- "Difference between `fetch` and `pull`?" (fetch is safe; pull integrates)
- "What is `--force-with-lease` and why prefer it over `--force`?" (checks remote hasn't advanced since your last fetch; prevents overwriting others' work)
- "How do you delete a remote branch?" (`git push origin --delete branch-name`)
- "How do you add a second remote?" (`git remote add upstream <url>`)
- "What is a remote-tracking branch?" (local ref that mirrors remote state; updated on fetch)

## Examples

```bash
git remote -v                            # list remotes with URLs
git remote add upstream <url>            # add upstream for fork workflow
git remote set-url origin <new-url>      # change remote URL (e.g., SSH → HTTPS)
git remote remove origin                 # remove a remote

git fetch origin                         # download all, update tracking refs
git fetch origin main                    # fetch specific branch
git fetch --prune                        # remove stale remote-tracking refs

git pull origin main                     # fetch + merge
git pull --rebase origin main            # fetch + rebase

git push origin feature/login            # push branch
git push -u origin feature/login         # push + track (shortens future push/pull)
git push origin --delete old-branch      # delete remote branch
git push --force-with-lease              # safe force push
git push --tags                          # push all tags
```

## Sources
- [[git/sources/pro-git]]
