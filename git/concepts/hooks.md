# Git Hooks

**Topic:** [[git/topics/ci-cd]], [[git/topics/core-concepts]]
**Related:** [[git/cicd/github-actions]], [[git/concepts/commit]]

## What it is
Git hooks are shell scripts that run automatically at specific points in Git's workflow. They live in `.git/hooks/` (client-side, not version-controlled) or can be distributed via tools like `pre-commit`, `husky`, or `lefthook`.

## How it works

Git calls the hook script at the named lifecycle point. If a hook exits non-zero, the operation is aborted (for pre-* hooks).

```
.git/hooks/
├── pre-commit          ← runs before commit is created
├── commit-msg          ← validates commit message
├── prepare-commit-msg  ← pre-fills commit message
├── post-commit         ← runs after commit (notifications)
├── pre-push            ← runs before push (last local gate)
├── pre-rebase          ← runs before rebase
└── post-checkout       ← runs after checkout/switch
```

Server-side hooks (on GitHub/GitLab self-hosted):
- `pre-receive` — gate incoming push (enforce policy)
- `update` — per-branch gate
- `post-receive` — trigger CI, send notifications

## Client-Side Hook Examples

```bash
# .git/hooks/pre-commit — run linter before commit
#!/bin/bash
ruff check . || exit 1
mypy src/ || exit 1

# .git/hooks/commit-msg — enforce conventional commit format
#!/bin/bash
msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|test|chore|perf|ci)(\(.+\))?: .+"
if ! echo "$msg" | grep -qE "$pattern"; then
  echo "ERROR: commit message must follow Conventional Commits format"
  exit 1
fi
```

## Distributing Hooks (Team Use)

Hooks in `.git/hooks/` are not committed. Use a framework:

```bash
# husky (JS projects)
npx husky init
echo "npm test" > .husky/pre-commit

# pre-commit (Python/polyglot)
pip install pre-commit
# .pre-commit-config.yaml:
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
pre-commit install

# lefthook (Go, fast)
lefthook install
```

## Common interview angles
- "How do you skip a hook?" (`git commit --no-verify` — but this should be rare and deliberate)
- "Why are client-side hooks not reliable for enforcing policy?" (local, skippable with --no-verify; use server-side or CI instead)
- "Where should you enforce commit message format — hook or CI?" (both: hook for fast feedback, CI for enforcement)
- "How do you share hooks across a team?" (pre-commit framework or husky; hook files committed to repo)

## Sources
- [[git/sources/pro-git]]
