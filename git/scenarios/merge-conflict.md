# Scenario: Merge Conflict

**Related:** [[git/concepts/merge]], [[git/concepts/rebase]], [[git/scenarios/interview-questions]]

## What Causes Conflicts

A conflict occurs when two branches modify **the same part of the same file** in incompatible ways, and Git cannot determine which change to keep.

Conflicts do NOT occur for:
- Changes to different files
- Changes to different parts of the same file
- One branch deleting a file the other doesn't touch

## Anatomy of a Conflict

```
<<<<<<< HEAD
current branch version of the code
=======
incoming branch version of the code
>>>>>>> feature/login
```

- Everything between `<<<<<<<` and `=======` is from HEAD (your current branch)
- Everything between `=======` and `>>>>>>>` is from the incoming branch

## Resolution Workflow

```bash
# A merge conflict occurred
git merge feature/login
# CONFLICT (content): Merge conflict in src/auth.py
# Automatic merge failed; fix conflicts and then commit the result.

# Option 1: Edit manually
# Open src/auth.py, find conflict markers, edit to desired state, remove markers
git add src/auth.py
git commit    # or: git merge --continue

# Option 2: Use a merge tool
git mergetool           # opens configured tool (vimdiff, VSCode, IntelliJ)

# Option 3: Take one side entirely
git checkout --ours   src/auth.py    # keep our version
git checkout --theirs src/auth.py    # keep their version
git add src/auth.py
git merge --continue

# Abort if needed
git merge --abort
```

## Conflict During Rebase

```bash
git rebase origin/main
# CONFLICT (content): Merge conflict in src/auth.py

# Resolve the file
git add src/auth.py
git rebase --continue   # NOT git commit

# Or abort
git rebase --abort
```

Key difference from merge conflicts: in a rebase, you resolve each commit's conflict one at a time. The "ours" side is **the branch you're rebasing onto** (not your branch).

## Preventing Conflicts

- **Merge main frequently** — reduces the time your branch diverges
- **Small, focused PRs** — fewer files touched = less overlap
- **Communicate** — if you know you're both touching the same file, coordinate
- **Modular code** — conflicts in function signatures hurt more than in separate files
- **`git rerere`** — automatically replays past conflict resolutions

## Common Conflict Types and Resolutions

| Conflict type | Resolution approach |
|--------------|-------------------|
| Both modified same function | Manually merge logic; understand both changes |
| Both added different imports | Include both |
| One deleted, one modified | Decide if the file should exist; keep or delete |
| Lock file (package-lock.json) | Take theirs, then re-run `npm install` |
| Binary file | `--ours` or `--theirs`; or manually replace |

## Interview Answer Framework
> "I check the conflict markers, understand what each side was trying to do, then write the correct merged version rather than blindly picking one side. For repeated conflicts I'd enable `git rerere`."

## Sources
- [[git/sources/pro-git]]
