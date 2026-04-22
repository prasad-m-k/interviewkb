# Conventional Commits

**Topic:** [[git/topics/core-concepts]]
**Related concepts:** [[git/concepts/commit]], [[git/concepts/tags]], [[git/concepts/hooks]]

## What it solves
Provides a machine-readable commit message format that enables:
- Automated semantic versioning (bump MAJOR/MINOR/PATCH based on commit types)
- Automated CHANGELOG generation
- Structured, human-readable history
- Tooling integrations (commitlint, semantic-release, release-please)

## Template / Skeleton

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Types

| Type | SemVer bump | When to use |
|------|-------------|-------------|
| `feat` | MINOR | New feature for the user |
| `fix` | PATCH | Bug fix for the user |
| `BREAKING CHANGE` (footer) | MAJOR | Breaking API change |
| `docs` | none | Documentation only |
| `style` | none | Formatting, whitespace |
| `refactor` | none | Restructuring without behavior change |
| `perf` | none | Performance improvement |
| `test` | none | Adding or fixing tests |
| `chore` | none | Build system, tooling |
| `ci` | none | CI configuration |

### Examples

```
feat(auth): add OAuth2 login with Google

fix(cart): prevent duplicate items when clicking rapidly

feat!: remove deprecated /api/v1 endpoints

BREAKING CHANGE: /api/v1 is removed. Migrate to /api/v2.

docs(readme): add local setup instructions

chore(deps): bump requests from 2.28.0 to 2.31.0
```

## Signal phrases
- "Generate changelogs automatically"
- "Automate semantic versioning"
- "Enforce consistent commit message style"
- `feat:`, `fix:`, `chore:` prefixes in the codebase

## Complexity
Tooling overhead is minimal. Biggest cost is team habit change.

## Tooling

```bash
# Enforce with commitlint
npm install --save-dev @commitlint/cli @commitlint/config-conventional
echo "module.exports = {extends: ['@commitlint/config-conventional']}" > commitlint.config.js

# Automate with semantic-release
npx semantic-release

# Validate in pre-commit hook
# .git/hooks/commit-msg — see [[git/concepts/hooks]]
```

## Problems using this pattern
- [[git/scenarios/clean-up-history]]

## Sources
- [[git/sources/pro-git]]
