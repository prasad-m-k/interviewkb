# Process Management and Signals

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/concepts/file-manipulation]], [[sre/concepts/disk-and-io]]

## Process states

| State | Code | Meaning |
|---|---|---|
| Running | R | On CPU or runnable, waiting for CPU |
| Sleeping (interruptible) | S | Waiting for event (I/O, timer, signal) |
| Sleeping (uninterruptible) | D | Waiting for I/O — **cannot be killed** |
| Stopped | T | Paused by SIGSTOP or debugger |
| Zombie | Z | Exited but parent hasn't called `wait()` — entry in process table only |

**D state is a red flag**: a process stuck in D state is usually waiting on slow/hung disk or NFS. You cannot kill it with SIGKILL. Fix: resolve the underlying I/O issue.

**Zombie processes**: don't consume CPU/memory (already dead), but do consume a PID slot. If the parent is alive and leaking zombies, it has a bug in its reaping logic. Fix: kill the parent (or fix the code).

---

## ps — process snapshot

```bash
# All processes, full detail
ps aux

# BSD-style columns: USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
# a = all users, u = user-oriented format, x = no controlling terminal

# Show process tree
ps auxf

# Filter by name
ps aux | grep nginx

# Show threads
ps -eLf | grep python

# Sort by CPU usage (top 10)
ps aux --sort=-%cpu | head -11

# Sort by memory
ps aux --sort=-%mem | head -11

# Show a specific process's environment
cat /proc/<PID>/environ | tr '\0' '\n'
```

---

## top / htop — live monitor

Key `top` fields:
- **load average**: `1m 5m 15m` — runnable + uninterruptible processes on average. > CPU count means overloaded.
- **%Cpu**: `us` (user), `sy` (kernel), `wa` (I/O wait), `id` (idle)
- **VSZ**: virtual memory size (reserved)
- **RSS**: resident set size (actual physical RAM in use)

```bash
# top keyboard shortcuts (while running)
k          # kill a process by PID
r          # renice (change priority)
1          # toggle per-CPU breakdown
M          # sort by memory
P          # sort by CPU
f          # field selection / column ordering
```

**I/O wait** (`%wa`) > 20% is a sign of a disk or NFS bottleneck — look at `iostat` next.

---

## Signals

Signals are asynchronous notifications sent to processes.

| Signal | Number | Default action | Common use |
|---|---|---|---|
| SIGHUP | 1 | Terminate | Reload config (daemons often handle this) |
| SIGINT | 2 | Terminate | Ctrl+C — interrupt |
| SIGQUIT | 3 | Core dump | Ctrl+\ — quit with core |
| SIGKILL | 9 | Terminate (uncatchable) | Force kill — last resort |
| SIGTERM | 15 | Terminate (catchable) | Graceful shutdown request |
| SIGSTOP | 19 | Stop (uncatchable) | Pause process |
| SIGCONT | 18 | Continue | Resume a stopped process |
| SIGUSR1/2 | 10/12 | User-defined | App-specific: log rotate, stats dump |

**SIGTERM vs SIGKILL**: always try SIGTERM first — it lets the process clean up (flush buffers, close connections, deregister from service discovery). SIGKILL is uncatchable and immediate — use only when SIGTERM fails after a timeout.

```bash
# Send SIGTERM (default)
kill <PID>
kill -15 <PID>
kill -TERM <PID>

# Force kill
kill -9 <PID>
kill -KILL <PID>

# Send SIGHUP (reload config for nginx, rsyslog, etc.)
kill -HUP $(cat /var/run/nginx.pid)
# or
pkill -HUP nginx

# Kill by name
pkill nginx
killall nginx

# Kill all processes of a user
pkill -u baduser

# Send signal to a process group (negative PID)
kill -TERM -<PGID>
```

---

## lsof — list open files

Everything in Linux is a file. `lsof` shows what files, sockets, and pipes every process has open.

```bash
# All open files for a process
lsof -p 1234

# Who has a file open?
lsof /var/log/app.log

# What ports is nginx listening on?
lsof -i -P -n | grep nginx

# All listening TCP ports
lsof -i TCP -s TCP:LISTEN

# Files opened by a user
lsof -u deploy

# Network connections to a specific port
lsof -i :8080

# Find deleted files still held open (disk-full scenario)
lsof | grep '(deleted)'

# How many file descriptors does PID 1234 have open?
lsof -p 1234 | wc -l
```

---

## strace — trace system calls

```bash
# Trace a running process
strace -p <PID>

# Trace with timestamps
strace -t -p <PID>

# Trace only specific syscalls
strace -e trace=open,read,write -p <PID>

# Trace including child processes
strace -f -p <PID>

# Count syscall frequency
strace -c -p <PID>

# Run a command under strace
strace ls /tmp
```

**Interview use case**: "A process is slow and you've ruled out CPU/memory — what next?" → `strace -p <PID>` to see if it's stuck in a `read()` waiting on slow I/O, or `futex()` waiting on a lock, or `nanosleep()` in a retry loop.

---

## /proc — process information filesystem

```bash
# Command line used to start process
cat /proc/<PID>/cmdline | tr '\0' ' '

# Environment variables
cat /proc/<PID>/environ | tr '\0' '\n'

# Open file descriptors
ls -la /proc/<PID>/fd

# Memory maps
cat /proc/<PID>/maps

# Resource limits
cat /proc/<PID>/limits

# I/O stats for a process
cat /proc/<PID>/io

# System-wide: CPU, memory, network
cat /proc/cpuinfo
cat /proc/meminfo
cat /proc/net/dev
```

---

## Background jobs and nohup

```bash
# Run in background
command &

# Bring to foreground
fg %1

# Send to background (after Ctrl+Z to stop)
bg %1

# Disown from terminal (won't be killed on shell exit)
nohup command > output.log 2>&1 &

# List jobs
jobs

# Preferred for long-running tasks: tmux or screen
tmux new -s session_name
```

---

## Common interview scenarios

**"A process is using 100% CPU"**
```bash
top -b -n1 | head -20              # identify PID
ps aux | sort -k3 -rn | head -5    # confirm
strace -p <PID> -c                 # what syscalls?
# Is it spinning in a tight loop? → application bug
# Is it running user code continuously? → profiling needed
```

**"Server load average is high but CPUs look idle"**
```bash
# Load average counts D-state (uninterruptible sleep) too
# Check for D-state processes:
ps aux | awk '$8 == "D" {print}'
iostat -xz 1 5    # disk I/O — look for high %await or %util
```

**"Out of file descriptors (EMFILE / ENFILE error)"**
```bash
ulimit -n                          # soft limit for current shell
cat /proc/sys/fs/file-max          # system-wide limit
lsof -p <PID> | wc -l             # how many this process has open
# Fix temporarily:
ulimit -n 65536
# Fix permanently: /etc/security/limits.conf or systemd LimitNOFILE=
```

## Sources
*(none yet)*
