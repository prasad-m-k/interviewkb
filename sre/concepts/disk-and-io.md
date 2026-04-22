# Disk and I/O

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/concepts/process-signals]], [[sre/concepts/file-manipulation]]

## df — disk space by filesystem

```bash
# Human-readable
df -h

# Include filesystem type
df -hT

# Inode usage (a filesystem can be full in inodes while having free blocks)
df -i

# A specific path
df -h /var/log
```

**Inode exhaustion**: `df -h` shows free space but the system can't create new files → `df -i` shows 100% inode usage. Common on filesystems with millions of tiny files (mail servers, cache dirs, temp dirs).

```bash
# Find the directory with the most files (inode consumers)
for d in /*; do echo "$d: $(find $d -xdev | wc -l)"; done 2>/dev/null | sort -t: -k2 -rn | head
```

---

## du — disk usage by directory

```bash
# Disk usage of current directory, summarized
du -sh .

# Top-level directories sorted by size
du -sh /* 2>/dev/null | sort -h | tail -20

# Recursive drill-down
du -sh /var/log/* | sort -h

# Find largest files under a path
find /var -type f -exec du -h {} + | sort -h | tail -20

# Faster alternative
du -ah /var/log | sort -h | tail -20
```

---

## iostat — disk I/O statistics

```bash
# 1-second snapshots, 5 times
iostat -xz 1 5

# Key columns in -x output:
# r/s, w/s       — reads/writes per second
# rkB/s, wkB/s   — KB read/written per second
# await          — average I/O wait time (ms) — most important
# %util          — % of time device was busy (>80% = saturated)
# svctm          — average service time (deprecated, ignore)

# Watch a specific device
iostat -x sda 1
```

**Diagnosing I/O bottleneck**:
- `%util` near 100% → disk saturated
- `await` >> `svctm` → long queue depth, requests waiting
- High `r/s` with high `await` → read-heavy workload starving; check for missing indexes (DB)
- High `w/s` with spikes → write bursts; check for log rotation, batch jobs

---

## The "disk is full" investigation playbook

```bash
# Step 1: which filesystem?
df -h

# Step 2: which directory is consuming space?
du -sh /* 2>/dev/null | sort -h

# Step 3: drill down
du -sh /var/log/* | sort -h
du -sh /var/log/app/* | sort -h

# Step 4: find the single largest files
find /var -type f -size +500M -ls 2>/dev/null

# Step 5: deleted files still held open by processes
lsof | grep '(deleted)'
# → restart the process or send SIGHUP to release the space

# Step 6: check inodes
df -i
```

**The deleted-but-held-open trap**: you `rm` a 10GB log file. `df` still shows 0 bytes free. The logging process still has the file open — the kernel keeps the data blocks allocated until the last file descriptor closes. `lsof | grep deleted` shows the PID. Restart it or `kill -HUP`.

---

## Common interview scenarios

**"Disk is full, logs are critical, can't delete anything yet"**
```bash
# Truncate (not delete) the file — keeps FD valid, frees space
> /var/log/bigfile.log          # truncate to 0 bytes
# or
truncate -s 0 /var/log/bigfile.log
```

**"Application writing to disk is slow"**
```bash
iostat -xz 1 10         # is the disk saturated?
iotop                   # which process is writing most?
strace -p <PID> -e write,fsync   # is it fsyncing on every write?
```

**"NFS mount is hanging"**
```bash
# NFS hangs cause D-state processes — can't be killed
ps aux | awk '$8=="D"'

# Check mount options: soft vs hard
mount | grep nfs
# soft mount: returns error after timeout
# hard mount: retries forever — dangerous if NFS server dies

# Check NFS server reachability
showmount -e nfs-server
rpcinfo -p nfs-server
```

## Sources
*(none yet)*
