# Find Top-N Disk Space Consumers

**Difficulty:** Easy–Medium
**Topic:** [[sre/topics/linux-cli]]
**Pattern:** Heap / Priority Queue
**Companies:** [[sre/companies/apple]], [[sre/companies/google]], [[sre/companies/meta]]

## Problem

Write a script that, given a root directory, **finds the top N directories consuming the most disk space** (not individual files), and reports them in descending order with human-readable sizes.

Constraints:
- Must handle permission errors gracefully (skip unreadable dirs, continue).
- Must handle symlinks correctly (do not follow them to avoid loops).
- For large filesystems, avoid loading all paths into memory — use a fixed-size heap.

## Why This Is Tricky

- `os.walk` follows symlinks by default on some versions — must set `followlinks=False`.
- Permissions errors on `/proc`, `/sys`, or root-owned dirs are common and will crash a naive script.
- The size of a directory itself (its inode) is not the same as the total size of its contents — you need recursive accumulation.
- A naive `sorted(all_dirs)` uses O(D) memory. A min-heap of size N uses O(N) memory regardless of how many dirs exist.

## Approach

1. Walk the directory tree with `os.walk(followlinks=False)`.
2. For each directory, sum the sizes of its **direct** files (not subdirs — `os.walk` already recurses).
3. Accumulate a running total per directory using a bottom-up rollup (or just use `shutil.disk_usage` per dir).
4. Maintain a min-heap of size N to track the largest N directories seen so far.
5. Print the result sorted descending.

## Solution (Python)

```python
import os
import heapq
import sys

def human_readable(size_bytes: int) -> str:
    for unit in ('B', 'KB', 'MB', 'GB', 'TB'):
        if size_bytes < 1024:
            return f"{size_bytes:.1f} {unit}"
        size_bytes /= 1024
    return f"{size_bytes:.1f} PB"

def top_disk_consumers(root: str, top_n: int = 10) -> list[tuple[int, str]]:
    """
    Returns list of (size_bytes, path) for the top_n largest directories,
    sorted descending. Does not follow symlinks.
    """
    dir_sizes: dict[str, int] = {}

    for dirpath, dirnames, filenames in os.walk(root, topdown=False, followlinks=False):
        # Remove hidden system paths that cause permission errors
        dirnames[:] = [d for d in dirnames if not d.startswith('/proc') and not d.startswith('/sys')]

        total = 0
        for fname in filenames:
            fpath = os.path.join(dirpath, fname)
            try:
                # lstat avoids following symlinks
                total += os.lstat(fpath).st_size
            except (OSError, PermissionError):
                pass

        # Add sizes of immediate subdirectories (already computed, bottom-up)
        for dname in os.listdir(dirpath):
            subpath = os.path.join(dirpath, dname)
            if os.path.isdir(subpath) and not os.path.islink(subpath):
                total += dir_sizes.get(subpath, 0)

        dir_sizes[dirpath] = total

    # Use a min-heap of size top_n to avoid sorting all dirs
    heap: list[tuple[int, str]] = []
    for path, size in dir_sizes.items():
        if len(heap) < top_n:
            heapq.heappush(heap, (size, path))
        elif size > heap[0][0]:
            heapq.heapreplace(heap, (size, path))

    return sorted(heap, reverse=True)

def main():
    root   = sys.argv[1] if len(sys.argv) > 1 else "."
    top_n  = int(sys.argv[2]) if len(sys.argv) > 2 else 10

    print(f"Top {top_n} directories under {root}:\n")
    for size, path in top_disk_consumers(root, top_n):
        print(f"  {human_readable(size):>12}  {path}")

if __name__ == "__main__":
    main()
```

## Shell Equivalent

```bash
# Quick one-liner — works for interactive use, NOT for huge trees
du -sh /var/log/* 2>/dev/null | sort -rh | head -n 10

# Safer version that handles permission errors
du --max-depth=2 /var/ 2>/dev/null | sort -rn | head -20 | \
  awk '{printf "%s\t%s\n", $1, $2}' | numfmt --from-unit=1024 --to=iec --field=1
```

## Complexity

- **Time:** O(D log N) where D = total directories, N = top_n. Each dir does one heap operation.
- **Space:** O(D) for `dir_sizes` (the rollup map). O(N) for the heap. For huge filesystems, improve by streaming the walk and keeping only the current branch in `dir_sizes` (tree-based pruning).

## Key Insight

The min-heap trick: to find the top N of a large stream, maintain a min-heap of size N. When a new item is larger than the heap's minimum, evict the minimum and push the new item. Final heap is the top N without sorting everything.

## Variants / Follow-ups

- "Find top N individual files (not directories)" → same approach, walk files instead of dirs, no rollup needed.
- "Files not accessed in 30 days" → filter with `os.lstat(fpath).st_atime < cutoff`.
- "Find and delete empty directories" → `if not dirnames and not filenames: os.rmdir(dirpath)`.
- "What if the filesystem is NFS-mounted and slow?" → Add `os.stat_result` caching, run with `ThreadPoolExecutor` for parallel stat calls per directory (I/O-bound, safe to parallelize).
- "How do you handle hard links so they're not double-counted?" → Track seen `(dev, inode)` pairs in a set and skip duplicates.

## Sources

- [[sre/companies/apple]]
- [[sre/companies/google]]
