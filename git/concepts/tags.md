# Tags

**Topic:** [[git/topics/core-concepts]]
**Related:** [[git/concepts/commit]], [[git/topics/ci-cd]]

## What it is
A tag is a named reference to a specific commit, typically used to mark release points. Unlike branches, tags do not move — they are permanent markers.

## How it works

### Lightweight Tag
Just a file in `.git/refs/tags/<name>` pointing to a commit SHA. No metadata.

### Annotated Tag
A full Git object with tagger name, email, date, message, and optional GPG signature. Stored as a tag object that points to a commit. This is what `git describe` uses.

```
v1.2.3 (annotated tag)
  object: abc123 (commit)
  type: commit
  tagger: Alice <a@x.com>
  date: ...
  message: Release 1.2.3
```

## Complexity
Creating: O(1). Listing: O(tags).

## When to use
- Mark every production release with a tag (`v1.2.3`)
- Trigger release pipelines (CI watches for `v*` tags)
- Semantic versioning: MAJOR.MINOR.PATCH

## Common interview angles
- "Lightweight vs. annotated tags — which should you use for releases?" (annotated — carries metadata, signable, used by git describe)
- "How do you push tags to remote?" (`git push --tags` or `git push origin v1.2.3`)
- "How do you delete a remote tag?" (`git push origin --delete v1.2.3`)
- "What is `git describe`?" (describes current commit relative to nearest annotated tag: `v1.2.3-14-gabc1234`)

## Examples

```bash
# Lightweight tag
git tag v1.2.3

# Annotated tag (preferred for releases)
git tag -a v1.2.3 -m "Release 1.2.3: add login feature"

# Tag a past commit
git tag -a v1.1.0 abc123 -m "Retroactive tag"

# List tags
git tag
git tag -l "v1.*"            # filter by pattern

# Show tag details
git show v1.2.3

# Push tags
git push origin v1.2.3       # push one tag
git push --tags               # push all local tags

# Delete tag
git tag -d v1.2.3                        # local
git push origin --delete v1.2.3          # remote

# Sign a tag with GPG
git tag -s v1.2.3 -m "Signed release"
git tag -v v1.2.3            # verify signature
```

## Sources
- [[git/sources/pro-git]]
