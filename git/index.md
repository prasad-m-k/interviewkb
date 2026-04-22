# Git Knowledge Base — Index
Last updated: 2026-04-22

## Overview
- [[git/overview]] — High-level synthesis of Git for interviews and daily use

## Topics
- [[git/topics/core-concepts]] — Working tree, staging, commits, objects, refs
- [[git/topics/branching-and-merging]] — Branch models, merge strategies, rebase
- [[git/topics/collaboration]] — Remotes, forks, pull requests, code review
- [[git/topics/workflows]] — Git Flow, trunk-based development, release branching
- [[git/topics/ci-cd]] — Git in continuous integration and delivery pipelines

## Concepts
- [[git/concepts/commit]] — Snapshot, SHA, tree/blob/commit object model
- [[git/concepts/branch]] — Lightweight pointer to a commit; HEAD
- [[git/concepts/staging-area]] — Index: the three-tree architecture
- [[git/concepts/merge]] — Fast-forward vs. 3-way merge; merge commits
- [[git/concepts/rebase]] — Linear history rewriting; interactive rebase
- [[git/concepts/remote]] — origin, upstream, fetch vs. pull, tracking branches
- [[git/concepts/stash]] — Temporary shelving of dirty working tree
- [[git/concepts/cherry-pick]] — Apply individual commits to another branch
- [[git/concepts/tags]] — Lightweight vs. annotated; semantic versioning
- [[git/concepts/hooks]] — Client- and server-side automation triggers
- [[git/concepts/reflog]] — Safety net: recovering lost commits
- [[git/concepts/bisect]] — Binary search through history to find regressions

## Patterns
- [[git/patterns/git-flow]] — Feature/release/hotfix branching model
- [[git/patterns/trunk-based-development]] — Short-lived branches, continuous integration
- [[git/patterns/conventional-commits]] — Structured commit messages; changelog generation
- [[git/patterns/squash-and-merge]] — Clean history via squashed PRs

## Scenarios (Interview)
- [[git/scenarios/interview-questions]] — 30+ scenario-based interview questions with answers
- [[git/scenarios/merge-conflict]] — Detecting, resolving, preventing conflicts
- [[git/scenarios/hotfix-production]] — Safe hotfix workflow without disrupting main
- [[git/scenarios/revert-bad-commit]] — revert vs. reset; when to use each
- [[git/scenarios/bisect-regression]] — Finding the breaking commit with git bisect
- [[git/scenarios/clean-up-history]] — Squash, fixup, reorder before merging
- [[git/scenarios/large-push-failure]] — "send-pack: unexpected disconnect"; fix with http.postBuffer

## CI/CD
- [[git/cicd/overview]] — How Git events drive pipelines
- [[git/cicd/github-actions]] — Triggers, workflow YAML, branch protection rules
- [[git/cicd/branching-strategies]] — Aligning branch model to deployment strategy

## Reference
- [[git/cheatsheet]] — Complete GitHub + Git CLI cheatsheet
- [[git/flashcards]] — Anki-ready Q&A flashcard deck

## Sources
- [[git/sources/pro-git]] — Pro Git book (Scott Chacon); canonical reference
