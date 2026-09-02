# System Design: Distributed Job Scheduler for a Heterogeneous Fleet

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/idempotency]], [[solution-arch/concepts/service-discovery]], [[solution-arch/concepts/distributed-consensus]]
**Patterns:** [[solution-arch/patterns/bulkhead]], [[solution-arch/patterns/circuit-breaker]]

> Interview-prep scenario for a general Google L6 SRE NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail. The scheduling shape described here follows the well-documented public lineage of Google's Borg and its open-source descendant, Kubernetes.

**Relationship to a sibling scenario:** this page is the **general** batch/cron scheduling mechanism for a heterogeneous machine fleet. [[solution-arch/scenarios/design-ml-accelerator-capacity-autoscaling]] is the specialized case of ML training/inference workloads on a CPU+accelerator fleet — those ML jobs are a workload TYPE that could run on top of the general scheduler described here, with accelerator-aware bin-packing as an extension of the scheduling logic below rather than a separate system.

---

## Step 1: Requirements

**Functional:**
- Accept both **cron-like recurring** job submissions and **one-off** job submissions
- Schedule jobs onto a heterogeneous fleet (machines with varying CPU/memory/disk capacity), bin-packing by declared resource requests
- Enforce **priority classes** with preemption — a higher-priority job can reclaim resources from a running lower-priority job
- Retry failed jobs with bounded backoff; dead-letter (stop retrying, surface for operator/owner attention) once a retry budget is exhausted
- Isolate resource usage between jobs co-located on the same machine

**Non-Functional:**
- **Scheduling latency:** time from job submission to job start should stay low (seconds) under normal load and degrade predictably — not catastrophically — under overload
- **Fleet utilization:** bin-packing quality directly determines how much of the fleet's capacity is usable; poor packing wastes hardware spend at this scale
- **Bounded retry budget:** no job retries indefinitely; every job either succeeds, exhausts its retry budget and dead-letters, or is explicitly cancelled
- **Operator control surface:** during a partial fleet outage or overload, an operator (or an automated policy) can pause/deprioritize whole priority classes and cordon (stop scheduling onto) degraded machines, without needing to touch individual jobs
- **Recurring-job correctness:** a recurring job's next scheduled run must not silently overlap with a still-running or hung previous run

---

## Step 2: High-Level Architecture

```
Job Submission (cron-defined or one-off API call)
   │
   ▼
Scheduler (control plane)
   │  maintains: pending queue (priority-ordered), current
   │  fleet capacity view, running-job registry
   │
   │  scheduling loop:
   │    for each pending job (priority order):
   │      find a machine with sufficient free capacity
   │      (bin-packing: fit the job's resource request into
   │       the best-fitting machine, not just any machine
   │       with room, to avoid fragmenting the fleet)
   │      if none available AND job priority is high enough:
   │        preempt a lower-priority running job on some machine,
   │        reschedule the preempted job back onto the pending queue
   │      assign job → machine; record in running-job registry
   ▼
Fleet (heterogeneous machines, each running an agent that
        enforces the resource isolation the scheduler assigned)
   │
   ▼
Job execution → success / failure / timeout reported back
   │
   ▼
On failure: retry (bounded, backoff) or dead-letter
```

The scheduler's control plane itself needs to be highly available and consistent about "what's currently running where" — this is a natural fit for a consensus-backed store ([[solution-arch/concepts/distributed-consensus]]) so a scheduler failover doesn't double-schedule or lose track of running jobs.

---

## Step 3: Bin-Packing and Priority/Preemption

```
Each job declares a resource request (CPU, memory, and optionally
accelerator type/count). The scheduler's placement decision is a
bin-packing problem: fit jobs onto machines to maximize utilization
while respecting declared requests as hard reservations (not
aspirational hints — this is what makes co-located jobs' resource
isolation enforceable at all).

Priority classes (e.g. P0 latency-critical serving jobs, P1 standard
batch, P2 best-effort/preemptible batch):
  A P0 job that can't find free capacity may PREEMPT a running P2
  job: the P2 job is killed/checkpointed, its resources freed, the
  P0 job scheduled, and the preempted P2 job goes back on the pending
  queue to be rescheduled when capacity frees up.

This is a deliberate policy, not just "run the highest-priority job
in the queue first" — preemption means priority also applies to
ALREADY-RUNNING work, which is what actually protects latency-critical
jobs during a capacity crunch.
```

---

## Step 4: Recurring Jobs and Idempotent Re-Run Semantics

```
Problem: a cron-like job is due to run again, but its previous
         instance is still running (or crashed without cleanly
         reporting failure). Scheduling a second overlapping
         instance risks duplicate side effects or resource
         contention with itself.

Fix:
  1. Before scheduling the next instance, check the running-job
     registry for an existing instance of this job definition.
  2. If one is still running past its expected duration: either
     skip this scheduled run (log + alert) or kill-and-restart,
     per a policy declared on the job definition (some jobs are
     safe to overlap, most aren't).
  3. Job logic itself should be written idempotently where
     possible — see [[solution-arch/concepts/idempotency]] — so
     that a retried or restarted run doesn't double-apply its
     effect (e.g. a batch job that upserts by key rather than
     blindly appending).

The scheduler enforces "don't start a conflicting overlapping run";
idempotency in the job's own logic is what makes a RETRY of a
partially-completed run safe — the two are complementary, not
substitutes for each other.
```

---

## Step 5: Resource Isolation Between Co-Located Jobs

```
Multiple jobs from different priority classes and different owning
teams run on the same physical machine. Without isolation, one job's
resource spike (a memory leak, a CPU-bound loop) can starve its
neighbors — the "noisy neighbor" problem.

Fix: per-job resource limits enforced by the machine-level agent
(cgroup-style CPU/memory limits, not just requests used for
scheduling math) — this is the same isolation goal as
[[solution-arch/patterns/bulkhead]] applied to co-located OS
processes instead of service-level connection/thread pools. A job
that exceeds its declared limit is throttled or OOM-killed rather
than allowed to degrade its neighbors.
```

---

## Step 6: Observability and Operator Control Under Overload

```
What's visible to an operator:
  - Pending-queue depth per priority class (a growing P2 queue while
    P0/P1 stay healthy is expected backpressure, not an incident;
    a growing P0 queue is)
  - Scheduling latency distribution (submit → start)
  - Preemption rate (a sustained high preemption rate signals the
    fleet is under-provisioned for its priority mix, not just a
    one-off spike)
  - Per-machine health (cordoned vs. schedulable)

What an operator (or an automated policy) can DO during a fleet
overload or partial outage, without touching individual jobs:
  - Pause or deprioritize an entire priority class (e.g. halt all
    P2 best-effort batch scheduling to free capacity for P0/P1)
  - Cordon degraded machines (stop new scheduling onto them; let
    running jobs drain) so the scheduler routes around a
    known-bad subset of the fleet automatically
  - Adjust the preemption threshold temporarily if preemption
    churn itself is becoming disruptive
```

---

## Step 7: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Placement algorithm | Best-fit bin-packing, not first-available | Reduces fleet fragmentation; materially improves utilization at scale |
| Priority enforcement | Preemption of running jobs, not queue-order-only | Protects latency-critical work even when it arrives after capacity is already consumed |
| Recurring-job overlap | Registry check + per-job overlap policy | Prevents silent double-execution without hardcoding one policy for every job type |
| Resource limits | Hard, agent-enforced per-job limits | Requests-only (no enforcement) allows noisy-neighbor degradation despite correct scheduling math |
| Retry policy | Bounded backoff + dead-letter | Unbounded retry either masks a real failure forever or wastes fleet capacity retrying a doomed job |
| Overload response | Priority-class-level operator controls, not per-job intervention | Lets a human (or policy) act at incident speed without triaging thousands of individual jobs |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #scheduling #batch-processing #resource-isolation #system-design #nalsd*
