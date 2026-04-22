# Scenario: Large Push Fails with "send-pack: unexpected disconnect"

**Difficulty:** Beginner trap
**Topic:** [[git/topics/collaboration]]
**Related:** [[git/concepts/remote]], [[git/cheatsheet]]

## Problem

Pushing a repo to GitHub fails mid-transfer with:

```
send-pack: unexpected disconnect while reading sideband packet
fatal: the remote end hung up unexpectedly
Everything up-to-date
```

## Root Cause

Git's default HTTP POST buffer is **1 MB**. When the pack being sent exceeds that size — common on initial pushes of repos containing large files (minified JS, images, generated assets) — the connection drops before the transfer completes.

The misleading "Everything up-to-date" message appears because git prints it before detecting the disconnect.

## Exact Steps That Fix It

```bash
# Step 1: Increase the HTTP POST buffer to 500 MB
git config http.postBuffer 524288000

# Step 2: Retry the push (--progress shows transfer so you can confirm it's working)
git push -u origin main --progress
```

Expected output on success:
```
Writing objects: 100% (340/340), 1.29 MiB | 12.55 MiB/s, done.
To https://github.com/prasad-m-k/interviewkb.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

## Why This Works

`http.postBuffer` controls how many bytes Git buffers before sending over HTTPS. Setting it to `524288000` (500 × 1024 × 1024) lets Git send packs larger than the default 1 MB limit in a single HTTP POST.

## Make It Permanent (optional)

```bash
# Apply globally so all repos on this machine use the larger buffer
git config --global http.postBuffer 524288000
```

## Diagnosis Checklist

If the push still fails after increasing the buffer:

| Check | Command |
|-------|---------|
| Is the remote reachable? | `git remote show origin` |
| Are you authenticated? | `gh auth status` |
| Is the pack itself corrupted? | `git fsck` |
| Is any single file over GitHub's 100 MB limit? | `git ls-files \| xargs ls -lh \| sort -k5 -rh \| head` |
| Is SSH faster than HTTPS? | `git remote set-url origin git@github.com:user/repo.git` |

## Alternative: Switch to SSH

SSH does not have the HTTP buffer limitation and is generally more reliable for large pushes:

```bash
git remote set-url origin git@github.com:<user>/<repo>.git
git push -u origin main
```

Requires your SSH public key to be added at GitHub → Settings → SSH keys.

## Sources
- [[git/sources/pro-git]]
