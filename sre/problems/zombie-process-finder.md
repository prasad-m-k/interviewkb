# Zombie Process Finder and Analyst

**Difficulty:** Easy–Medium
**Topic:** [[sre/topics/linux-cli]]
**Pattern:** /proc filesystem parsing / Process tree
**Companies:** [[sre/companies/apple]], [[sre/companies/google]]

## Problem

Write a script that **finds all zombie processes** on a Linux system, identifies their parent process (which holds the zombie, because it hasn't called `wait()`), and outputs a report showing which parents are leaking zombies.

Expected output:
```
ZOMBIE PID  ZOMBIE CMD      PARENT PID  PARENT CMD
---------   --------        ----------  -----------
4521        defunct         1023        node
4522        defunct         1023        node
7891        defunct         2340        python3

Parents with zombie children:
  PID 1023 (node) holds 2 zombies
  PID 2340 (python3) holds 1 zombie
```

## Why This Is Tricky

- The zombie process itself cannot be killed (`kill -9` has no effect — it has no code, no resources, just an entry in the process table).
- The only fix is to kill or fix the **parent** — which must call `wait()`/`waitpid()` to reap the zombie.
- A production system with thousands of transient zombies (e.g., a shell spawning subprocesses) is not necessarily broken; a process that accumulates zombies over time and never reaps them is the real bug.
- Reading from `/proc` is the portable way; `ps` is a wrapper around `/proc`.

## Approach

1. List all numeric directories in `/proc` (each is a PID).
2. For each PID, read `/proc/<pid>/status` and check `State: Z`.
3. For zombies, also read `PPid:` (parent PID) from the same status file.
4. Group zombies by parent PID.
5. Look up the parent's command from `/proc/<ppid>/comm` or `status`.
6. Report zombies and parents.

## Solution (Python)

```python
import os
import sys
from collections import defaultdict

def read_proc_status(pid: int) -> dict:
    """Parse /proc/<pid>/status into a dict. Returns {} on permission error."""
    try:
        path = f"/proc/{pid}/status"
        result = {}
        with open(path) as f:
            for line in f:
                if ':' in line:
                    key, _, val = line.partition(':')
                    result[key.strip()] = val.strip()
        return result
    except (FileNotFoundError, PermissionError):
        return {}

def get_comm(pid: int) -> str:
    """Get process command name from /proc/<pid>/comm."""
    try:
        with open(f"/proc/{pid}/comm") as f:
            return f.read().strip()
    except (FileNotFoundError, PermissionError):
        return "unknown"

def find_zombies() -> list[dict]:
    zombies = []
    for entry in os.listdir("/proc"):
        if not entry.isdigit():
            continue
        pid = int(entry)
        status = read_proc_status(pid)
        if not status:
            continue
        state = status.get("State", "")
        # State line looks like: "Z (zombie)" or just "Z"
        if state.startswith("Z"):
            ppid_str = status.get("PPid", "0")
            ppid = int(ppid_str) if ppid_str.isdigit() else 0
            zombies.append({
                "pid":      pid,
                "ppid":     ppid,
                "name":     status.get("Name", "defunct"),
                "ppid_cmd": get_comm(ppid),
            })
    return zombies

def report(zombies: list[dict]) -> None:
    if not zombies:
        print("No zombie processes found.")
        return

    print(f"{'ZOMBIE PID':<12} {'ZOMBIE CMD':<16} {'PARENT PID':<12} {'PARENT CMD'}")
    print("-" * 55)
    for z in sorted(zombies, key=lambda x: x["ppid"]):
        print(f"{z['pid']:<12} {z['name']:<16} {z['ppid']:<12} {z['ppid_cmd']}")

    # Summarize by parent
    by_parent: dict[int, list[dict]] = defaultdict(list)
    for z in zombies:
        by_parent[z["ppid"]].append(z)

    print(f"\nParents holding zombies ({len(by_parent)} culprit(s)):")
    for ppid, zlist in sorted(by_parent.items(), key=lambda x: -len(x[1])):
        cmd = zlist[0]["ppid_cmd"]
        print(f"  PID {ppid} ({cmd}) holds {len(zlist)} zombie(s)")
        print(f"    Recommendation: check if PID {ppid} is calling wait() after fork()")

if __name__ == "__main__":
    report(find_zombies())
```

## Shell Equivalent

```bash
# List zombie processes with their parent
ps aux | awk '$8 == "Z" {print $2, $3, $13}' | head -20

# More detailed: show parent PID too
ps -eo pid,ppid,stat,comm | awk '$3 ~ /Z/ {print "zombie:", $1, "parent:", $2, $4}'

# Find and kill the parent of zombie PID 1234 (use with caution!)
PPID=$(awk '/PPid/{print $2}' /proc/1234/status)
echo "Parent is PID $PPID"
# Only kill if the parent is a runaway process and not critical
# kill -SIGCHLD $PPID  # Ask parent to reap children first
```

## Complexity

- **Time:** O(P) where P = number of processes on the system.
- **Space:** O(Z) where Z = number of zombies.

## Key Insight

Zombie processes **cannot be killed**. They're already dead — they have no process address space, no open file descriptors, and no runnable code. They exist only as entries in the kernel process table, waiting for their parent to call `wait()`. The fix is always in the parent:

1. **Fix the code:** Make the parent call `wait()` or `waitpid()` after every `fork()`.
2. **Use `SIGCHLD` with `SIG_DFL`:** Setting `signal(SIGCHLD, SIG_IGN)` tells the kernel to auto-reap children — but loses exit status.
3. **Kill the parent:** The zombie will be re-parented to PID 1 (init/systemd), which calls `wait()` automatically.
4. **Restart the service:** If the parent is a bug-ridden long-running service, a restart may be the fastest fix.

## Variants / Follow-ups

- "What's the maximum number of zombies that causes a real problem?" → When the process table fills up (default ~4M PIDs), new forks fail. A few hundred zombies are usually harmless.
- "How do you find zombies on a Kubernetes node?" → `kubectl exec` into the node or use `crictl ps --state=exited`. Also check `ps` on the container host.
- "What if init is accumulating zombies?" → PID 1 (systemd) should never have zombies — this indicates a systemd bug or a unit that spawned with `Type=forking` but the parent didn't signal readiness.
- "How would you script a zombie alert in Prometheus?" → Export `node_processes_state{state="Z"}` via `node_exporter` and alert when it stays above 0 for >5 minutes.

## Sources

- [[sre/companies/apple]]
- [[sre/concepts/process-signals]]
