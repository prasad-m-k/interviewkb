# cgroups & Container Resource Isolation

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/concepts/memory-management]], [[sre/concepts/disk-and-io]], [[sre/concepts/process-signals]], [[sre/scenarios/cgroup-noisy-neighbor]]

## Overview
Control groups (cgroups) are the kernel mechanism that lets multiple workloads share one machine without one tenant starving another — the foundation Borg, Kubernetes, and Docker all build container resource limits on top of. SREs need this because "the box has capacity but my pod is slow" is almost always a cgroup limit, not a real hardware shortage.

## Key Concepts

### 1. cgroups v1 vs v2
- **v1:** Each resource controller (cpu, memory, blkio, ...) has its own independent hierarchy — a process can sit at different points in the cpu tree vs. the memory tree. Flexible but leads to inconsistent limit application across controllers.
- **v2:** Single unified hierarchy — one tree, all controllers attach to the same node per process. Simpler to reason about; this is what modern Kubernetes and Borg-style schedulers default to.

### 2. CPU throttling (the "why is my pod slow when the node has idle CPU" question)
- A container's CPU cgroup gets a **quota** per **period** (e.g. 50ms quota per 100ms period = 0.5 CPU).
- If the container bursts past quota within a period, the kernel throttles it — the process is runnable but denied CPU until the next period, even though other cores sit idle.
- This shows up as elevated **latency with low measured CPU%**, because `top`/`docker stats` average CPU usage over the whole period and hide the throttled gaps.
- Diagnose via `cat /sys/fs/cgroup/.../cpu.stat` → `nr_throttled` and `throttled_time`. A nonzero and growing `nr_throttled` with normal-looking average CPU is the signature of this exact failure mode.

### 3. Memory limits and OOM within a cgroup
- A container hitting its **cgroup memory limit** gets OOM-killed by the kernel even if the *host* has free memory — this is a scoped OOM kill, distinct from the host-wide OOM Killer in [[sre/concepts/memory-management]].
- Check `memory.max` (v2) / `memory.limit_in_bytes` (v1) against `memory.current` and `memory.events` (look at the `oom_kill` counter) to confirm it was the cgroup limit, not the host, that triggered the kill.
- Page cache counts against the cgroup's memory usage too — a container can look like it's near its limit purely from page cache that the kernel would reclaim under pressure, which is a common false alarm.

### 4. blkio / I/O isolation (the "noisy neighbor" problem)
- The `blkio` (v1) / `io` (v2) controller can cap a container's I/O bandwidth or IOPS on a shared device.
- Without a cap, one tenant running a large sequential read/write can saturate the device's queue and starve every other tenant's I/O latency — classic multi-tenant noisy-neighbor failure, distinct from the single-process disk-saturation case in [[sre/concepts/disk-and-io]].
- `ionice` sets scheduling class/priority for a process's I/O within the scheduler; the `blkio`/`io` cgroup controller enforces a hard bandwidth/IOPS ceiling — they solve related but different problems (priority vs. hard cap) and interviewers expect you to know which one actually stops a neighbor from saturating the disk.

## SRE Tools
- `cat /sys/fs/cgroup/<path>/cpu.stat` — throttling counters (`nr_periods`, `nr_throttled`, `throttled_time`)
- `cat /sys/fs/cgroup/<path>/memory.current` / `memory.max` / `memory.events` — cgroup memory state and OOM count
- `systemd-cgtop` — live per-cgroup resource usage, analogous to `top` but per container/slice
- `docker stats` / `kubectl top pod` — higher-level views that wrap the same cgroup counters
- `crictl stats` — cgroup-level stats when debugging directly on a Kubernetes node

## Common interview angles
- "A pod's CPU graph shows 40% average utilization but users report latency spikes — what's going on?" → CPU throttling within a period; check `nr_throttled`, not average utilization.
- "Container was OOM-killed but `free -h` on the host shows plenty of memory — why?" → cgroup memory limit, not host memory pressure.
- "Two tenants share a disk; one's batch job is stalling the other's read latency — how do you isolate it?" → blkio/io cgroup bandwidth cap, not just `ionice`.

## Sources
- [[sre/companies/google]]
