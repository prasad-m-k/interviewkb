# Troubleshooting: The Noisy Neighbor (cgroup Resource Contention)

**Topic:** [[sre/topics/linux-cli]]
**Pattern:** [[sre/patterns/troubleshooting-framework]] (USE Model, applied per-cgroup instead of per-host)
**Concept:** [[sre/concepts/cgroups-and-containers]]
**Companies:** [[sre/companies/google]]
**Level:** L6 — this is a step up from single-host debugging; the machine is healthy, the container isn't, and you have to prove *why* with cgroup-level evidence, not host-level tools.

## Scenario
Your team runs a multi-tenant fleet (Borg-style scheduling — one physical machine, many containers from unrelated services). One service's p99 latency has tripled. The on-call for the *host* insists the machine is fine: `top` shows 40% CPU utilization and `free -h` shows several GB free. The service owner insists their code hasn't changed. Diagnose it.

## Step 1: Reframe the question
The first mistake candidates make is trusting host-level metrics for a container-level problem. At L6, the interviewer wants you to say out loud: "Host-level utilization and container-level allocation are two different resource pools — I need to look at the cgroup, not the host." This is the RED (service) vs. USE (resource) split from [[sre/patterns/troubleshooting-framework]], except the "resource" being measured is the container's slice of the host, not the host itself.

## Step 2: Check CPU throttling, not CPU utilization
```bash
# Find the container's cgroup path (via docker/crictl or /proc/<pid>/cgroup)
cat /sys/fs/cgroup/.../cpu.stat
```
Look at `nr_throttled` and `throttled_time`. A container can be throttled dozens of times a second while the host shows moderate average CPU%, because throttling happens *within* a period (e.g. quota exhausted at 50ms into a 100ms window) and the remaining idle time in that period still counts toward a low host-wide average. Rising `nr_throttled` with otherwise unremarkable host CPU is the smoking gun. Detail: [[sre/concepts/cgroups-and-containers]].

## Step 3: Check for a memory-side false alarm
```bash
cat /sys/fs/cgroup/.../memory.current
cat /sys/fs/cgroup/.../memory.max
cat /sys/fs/cgroup/.../memory.events   # look at oom_kill
```
Rule out two look-alikes:
- The container is near its `memory.max` because of reclaimable **page cache**, not a leak — not an actual problem, but can trigger throttling-adjacent slowdowns as the kernel works harder to reclaim.
- The container was actually OOM-killed and restarted (check `oom_kill` count and process start time) — this reads as "latency" in a dashboard that's really "the process cold-started twice."

## Step 4: Check I/O contention from a genuinely different tenant
```bash
cat /sys/fs/cgroup/.../io.stat        # or blkio.throttle.io_service_bytes on v1
iostat -xz 1                          # host-wide device saturation
```
If the *device* (not the container's own cgroup) shows high `%util`/`await`, another tenant on the same machine is saturating shared I/O bandwidth. This is the actual "noisy neighbor" — a batch job or log-shipping sidecar with no `io`/`blkio` cap monopolizing the disk queue for everyone co-located on it.

## Step 5: Resolution, in order of blast radius
1. **Immediate:** reschedule the affected container to a different host (bypasses the contention without touching either tenant's config) — the fastest reversible mitigation.
2. **Short term:** apply or tighten an `io`/`blkio` cap on the noisy tenant so it can't monopolize shared bandwidth again.
3. **Root cause / prevention:** the scheduler placed two I/O-heavy or bursty-CPU tenants on the same host without accounting for their resource *profiles*, not just their static requests — flag as a bin-packing/scheduling gap, not a one-off incident.

## Why this is an L6-level question
An L4/L5 answer stops at "the container hit its limit." An L6 answer explains *why the host-level tools lied* (averaging hides throttling; free memory hides a scoped OOM; overall CPU% hides a specific noisy tenant), and proposes a fix that addresses the scheduling/isolation gap, not just the single incident.

## Sources
- [[sre/concepts/cgroups-and-containers]]
- [[sre/companies/google]]
