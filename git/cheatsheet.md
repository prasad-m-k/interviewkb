# Git + GitHub CLI Cheatsheet

> All commands assume `gh` (GitHub CLI) is installed and authenticated. Run `gh auth login` once.

---

## Setup

```bash
# Identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Preferred editor
git config --global core.editor "code --wait"    # VS Code
git config --global core.editor "vim"

# Default branch name
git config --global init.defaultBranch main

# Always rebase on pull
git config --global pull.rebase true

# Safe force-push as default
git config --global alias.fpush "push --force-with-lease"

# Useful aliases
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.st "status -sb"
git config --global alias.sw "switch"

# View all config
git config --list --show-origin
```

---

## Repository

```bash
git init                            # init new repo
git init --bare                     # init bare repo (for remotes)
git clone <url>                     # clone
git clone <url> --depth=1           # shallow clone
git clone <url> --branch v1.2.0     # clone specific tag/branch
git clone --filter=blob:none --sparse <url>  # partial clone

gh repo create my-project --public  # create GitHub repo
gh repo clone owner/repo            # clone via GitHub CLI
gh repo fork owner/repo             # fork on GitHub + clone
gh repo view --web                  # open repo in browser
```

---

## Daily Workflow

```bash
git status                          # working tree status
git status -sb                      # compact status

git add file.py                     # stage a file
git add -p                          # interactively stage hunks
git add -A                          # stage all changes

git commit -m "feat: add login"     # commit
git commit --amend --no-edit        # amend last commit (keep message)
git commit --amend -m "new message" # amend with new message
git commit --allow-empty -m "ci: retrigger"  # empty commit

git diff                            # working tree vs. index
git diff --cached                   # index vs. HEAD
git diff HEAD                       # working tree vs. HEAD
git diff main..feature              # between two branches
git diff --stat                     # summary of changed files
```

---

## Branching

```bash
git branch                          # list local branches
git branch -a                       # list all (local + remote)
git branch -v                       # with last commit
git branch --merged main            # branches merged into main
git branch --no-merged main         # branches NOT merged into main

git switch main                     # switch branch
git switch -c feature/login         # create + switch
git switch -c feature/login origin/feature/login  # track remote

git branch -d feature/login         # delete (safe)
git branch -D feature/login         # force delete
git branch -m old new               # rename

git push origin --delete feature/login   # delete remote branch
git remote prune origin                  # remove stale remote-tracking refs
```

---

## Merging

```bash
git merge feature/login             # merge into current branch
git merge --no-ff feature/login     # always create merge commit
git merge --squash feature/login    # collapse + stage (then commit)
git merge --abort                   # abort conflict

git revert abc123                   # safe undo (new commit)
git revert -m 1 <merge-sha>         # revert a merge commit
git revert --no-commit abc..def     # revert range, stage only
```

---

## Rebase

```bash
git rebase main                     # rebase current branch onto main
git rebase origin/main              # rebase onto remote main
git rebase -i HEAD~3                # interactive: last 3 commits
git rebase -i origin/main           # interactive: all commits since main
git rebase --autosquash -i origin/main  # auto-apply fixup! commits
git rebase --abort                  # cancel
git rebase --continue               # after resolving conflict
git rebase --skip                   # skip a conflicting commit

git rebase --onto main feature-a feature-b  # rebase feature-b off feature-a onto main
```

---

## Remote Operations

```bash
git remote -v                       # list remotes
git remote add origin <url>         # add remote
git remote add upstream <url>       # add upstream (for forks)
git remote set-url origin <url>     # change URL
git remote remove origin            # remove remote

git fetch origin                    # download, don't merge
git fetch --all                     # fetch all remotes
git fetch --prune                   # prune stale remote refs
git fetch --tags                    # fetch tags
git fetch --unshallow               # convert shallow to full clone

git pull                            # fetch + merge
git pull --rebase                   # fetch + rebase (preferred)
git pull origin main --rebase       # explicit

git push origin feature/login       # push branch
git push -u origin feature/login    # push + set upstream tracking
git push --force-with-lease         # safe force push
git push --tags                     # push all tags
git push origin --delete branch     # delete remote branch
```

---

## Tags

```bash
git tag                             # list tags
git tag -l "v1.*"                   # filter
git tag v1.2.3                      # lightweight tag
git tag -a v1.2.3 -m "Release"      # annotated tag (preferred)
git tag -a v1.2.3 abc123 -m "Tag"   # tag past commit
git show v1.2.3                     # show tag details
git push origin v1.2.3              # push one tag
git push --tags                     # push all tags
git tag -d v1.2.3                   # delete local tag
git push origin --delete v1.2.3     # delete remote tag
```

---

## Stash

```bash
git stash                           # stash tracked changes
git stash push -m "WIP: login"      # named stash
git stash -u                        # include untracked files
git stash list                      # list all stashes
git stash show -p stash@{0}         # show diff
git stash pop                       # apply + remove from stack
git stash apply stash@{1}           # apply without removing
git stash drop stash@{0}            # remove entry
git stash clear                     # remove all
git stash branch rescue stash@{0}   # create branch from stash
```

---

## History and Log

```bash
git log                             # full log
git log --oneline                   # compact
git log --oneline --graph --all     # visual branch graph
git log -p                          # with diffs
git log --stat                      # with file stats
git log --author="Alice"            # filter by author
git log --since="2 weeks ago"       # filter by date
git log --grep="fix"                # filter by message
git log --all --follow -p -- file   # history of a file (with renames)
git log main..feature               # commits in feature not in main
git log --merges                    # only merge commits
git log --no-merges                 # exclude merge commits

git show abc123                     # show a commit
git show abc123:src/file.py         # show file at commit
git blame file.py                   # line-by-line authorship
git blame -L 10,20 file.py          # blame specific lines
```

---

## Undo / Reset

```bash
git restore file.py                 # discard working tree changes
git restore --staged file.py        # unstage

git reset HEAD~1                    # uncommit, keep changes staged (soft default? no: mixed)
git reset --soft HEAD~1             # uncommit, keep index
git reset --mixed HEAD~1            # uncommit, unstage (default)
git reset --hard HEAD~1             # uncommit + discard
git reset --hard origin/main        # match remote exactly

git clean -fd                       # remove untracked files + dirs
git clean -fdn                      # dry run first
git clean -fdx                      # include ignored files
```

---

## Reflog (Safety Net)

```bash
git reflog                          # HEAD movement history
git reflog show main                # branch movement history
git reflog --date=iso               # with timestamps

# Recover lost commits
git reset --hard HEAD@{3}           # jump to a past state
git branch recovered HEAD@{3}       # create branch from past state
```

---

## Bisect

```bash
git bisect start
git bisect bad                      # current is bad
git bisect good v1.0.0              # last known good
# test, then:
git bisect good                     # or: git bisect bad
git bisect skip                     # skip untestable commit
git bisect run pytest test_bug.py   # automate
git bisect reset                    # return to original HEAD
```

---

## Advanced

```bash
# Cherry-pick
git cherry-pick abc123              # apply one commit
git cherry-pick abc123..def456      # apply range
git cherry-pick --no-commit abc123  # stage only

# Worktree
git worktree add ../hotfix hotfix/crash
git worktree list
git worktree remove ../hotfix

# Submodules
git submodule add <url> path/       # add submodule
git submodule update --init --recursive  # init after clone
git submodule update --remote       # pull latest

# Clean history
git filter-repo --path secrets.env --invert-paths  # scrub file from history
```

---

## GitHub CLI (gh)

```bash
# Authentication
gh auth login
gh auth status

# Pull Requests
gh pr create --title "Add login" --body "Closes #42"
gh pr create --draft                # create draft PR
gh pr list                          # list open PRs
gh pr view 123                      # show PR details
gh pr view 123 --web                # open in browser
gh pr checkout 123                  # check out PR locally
gh pr review 123 --approve
gh pr review 123 --request-changes --body "Please fix tests"
gh pr merge 123 --squash --delete-branch
gh pr merge 123 --rebase
gh pr merge 123 --merge             # merge commit
gh pr close 123
gh pr diff 123                      # show diff

# Issues
gh issue create --title "Bug: crash on login" --label bug
gh issue list --assignee @me
gh issue view 42 --web
gh issue close 42 --comment "Fixed in #123"

# Releases
gh release create v1.2.3 --title "v1.2.3" --notes "Bugfixes"
gh release create v1.2.3 dist/*.tar.gz  # with assets
gh release list
gh release view v1.2.3

# Workflows (Actions)
gh workflow list
gh workflow run ci.yml              # manual trigger
gh run list                         # recent workflow runs
gh run view 12345                   # run details
gh run watch 12345                  # live logs

# Repo operations
gh repo view --web
gh repo clone owner/repo
gh repo fork owner/repo --clone
gh repo set-default owner/repo      # set default for gh commands

# SSH keys
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Work laptop"
gh ssh-key list
```

---

## .gitignore Quick Reference

```gitignore
# Patterns
*.log           # all .log files
!important.log  # except this one
/build          # only at root
build/          # only directories named build
**/temp         # in any subdirectory
doc/**/*.txt    # all .txt in doc/ tree

# Common entries
.env
.DS_Store
__pycache__/
*.pyc
node_modules/
.venv/
dist/
*.egg-info/
.idea/
.vscode/
```

Generate for your stack: `gh extension install github/gh-gitignore` or `gitignore.io`

---

## Git Config Quick Wins

```bash
# Set VS Code as diff/merge tool
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# Enable rerere (reuse recorded resolutions)
git config --global rerere.enabled true

# Colored output
git config --global color.ui auto

# Show branch in prompt (add to .zshrc / .bashrc)
# Using oh-my-zsh git plugin, or:
PS1='\w$(__git_ps1 " (%s)") \$ '
```
