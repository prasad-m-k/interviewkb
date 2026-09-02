# System Design: Global Fleet Software / Config Rollout System

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/service-discovery]], [[solution-arch/concepts/distributed-consensus]]
**Patterns:** [[solution-arch/patterns/blue-green-canary]], [[solution-arch/patterns/circuit-breaker]]

> Interview-prep scenario for a general Google L6 SRE NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

This page rolls out **binaries / OS images / system-level software** across a global machine fleet. It's the same progressive-canary discipline as [[solution-arch/scenarios/design-continuous-model-deployment-rollout]] (ML model artifact rollout) and [[solution-arch/scenarios/design-global-config-feature-flag-service]] (config-value rollout) applied to a third artifact type — all three are instances of one underlying pattern: **stage, gate on health, promote automatically, roll back at least as fast as you rolled forward.** See [[solution-arch/patterns/blue-green-canary]] for the core canary mechanics this specializes.

---

## Step 1: Requirements

**Functional:**
- Roll out a new build to a defined batch of the fleet, progressing through stage gates automatically
- Automatically detect regression at each stage (error rate, crash rate, resource-usage drift) and halt or roll back without waiting for a human
- Quarantine machines that individually fail to update, rather than blocking the whole batch on stragglers
- Support both scheduled rollouts (routine patch) and emergency rollouts (security fix) with different speed/safety trade-offs
- Give operators visibility into rollout progress and a manual pause/abort control at any stage

**Non-Functional:**
- **Rollback time ≤ time-to-detect**, ideally much less — a rollout that took 6 hours to reach 100% must not take 6 hours to roll back; detection plus rollback together should resolve faster than the blast radius grows
- **Capacity headroom maintained throughout** — a rolling restart reduces capacity for the batch currently restarting; batch size must respect remaining fleet capacity, not a fixed percentage blind to current load
- Fleet-wide rollout completion time target (routine: days, acceptable; emergency: hours)
- Near-zero user-visible impact during normal (non-rollback) progression
- Toil reduction: fully automated gate-and-promote; a human approves stage *boundaries* (or is paged only on an automatic rollback), not every individual batch

---

## Step 2: High-Level Architecture

```
Release artifact
   │
   ▼
Rollout Controller
   │  reads: fleet topology, current capacity, health-check config
   │
   ├─▶ Stage 1: single canary machine        (soak: minutes)
   ├─▶ Stage 2: single cluster / rack         (soak: tens of minutes)
   ├─▶ Stage 3: single region                (soak: hours)
   └─▶ Stage 4: global                       (soak: continues post-completion)

Each stage:
   Rollout Controller ──▶ batch of machines ──▶ update + restart
                              │
                     health-check gate ──── PASS ──▶ next stage
                              │
                            FAIL
                              │
                              ▼
                    auto-rollback this stage + halt
                    (page on-call, do not auto-retry blindly)
```

---

## Step 3: Health-Check Gates — What Actually Gates Progression

```
Fast signals (seconds):   error rate, crash-on-start, failed health endpoint
Slow signals (minutes):   memory growth trend, crash-loop over a soak window,
                           latency P99 drift vs. pre-rollout baseline

A gate that only checks "did it start" catches crashes but misses
slow leaks or gradual latency regressions — the soak window exists
specifically to surface signals that only appear after sustained load,
not just at process start.

Gate decision = fast signals (immediate halt) AND slow signals
                (halt after soak window elapses cleanly)
```

---

## Step 4: Capacity-Aware Batching

```
Naive: "restart 10% of the fleet at a time" — ignores current load.
Correct: batch size = min(fixed_percent, remaining_capacity_headroom)

If the fleet is already running near peak capacity (e.g. during
a traffic spike), a 10% batch restart could push remaining
capacity below demand — causing a self-inflicted capacity outage
DURING a routine rollout. The controller must check current
utilization before sizing each batch, and shrink batch size
(or pause the rollout) if headroom is thin.
```

---

## Step 5: Partial Failures — Stragglers vs. Systemic Regression

```
A few machines in a batch fail to update (bad disk, config drift,
unreachable): quarantine those specific machines — mark them
out of rotation, alert for manual remediation, and let the REST
of the batch proceed. Blocking the whole rollout on a handful of
stragglers turns an isolated hardware problem into a fleet-wide
rollout stall.

Distinguish this from a SYSTEMIC regression (the new build itself
is bad) — that must halt/rollback the whole stage, not just
quarantine a few hosts. The signal: are failures clustered on a
few specific machines (straggler) or spread proportionally across
the batch (systemic)?
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Stage granularity | Machine → cluster → region → global | Bounds blast radius geometrically; each stage's failure is contained before the next begins |
| Rollback speed target | Faster than rollout speed | Detection + rollback together must resolve before blast radius outpaces them |
| Batch sizing | Capacity-aware, not fixed-percent | Prevents a routine rollout from self-inflicting a capacity outage during high load |
| Straggler handling | Quarantine individual machines, don't block batch | Isolated hardware failure shouldn't stall a fleet-wide rollout |
| Emergency vs. routine rollout | Same pipeline, different stage soak times | One system, tunable speed/safety trade-off, rather than a separate emergency-only path that's rarely exercised and untrusted |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #canary-rollout #fleet-management #system-design #nalsd*
