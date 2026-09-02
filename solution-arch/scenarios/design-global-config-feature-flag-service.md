# System Design: Global Configuration / Feature-Flag Service

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/distributed-caching]]
**Patterns:** [[solution-arch/patterns/blue-green-canary]], [[solution-arch/patterns/circuit-breaker]]

> Interview-prep scenario for a general Google L6 SRE NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

Canarying a config/flag push is the same discipline applied to a different artifact type as [[solution-arch/scenarios/design-global-fleet-software-rollout]] (binary/OS rollout) and [[solution-arch/scenarios/design-continuous-model-deployment-rollout]] (ML model rollout). All three answer "how do you ship a change to a huge fleet without it becoming an outage" — the artifact differs (config value vs. binary vs. model weights), the rollout/rollback discipline is the same shape. Worth saying explicitly in an interview if asked more than one of these prompts back to back.

---

## Step 1: Requirements

**Functional:**
- Push a config/flag change with a defined, staged rollout plan (e.g. 1% → 10% → 50% → 100% of the fleet)
- Serve current config values to reading services with low latency, globally
- Automatically roll back a change when a regression is detected — not only on human intervention
- Maintain a full audit trail of every config change: who/what pushed it, when, to what rollout percentage, and what the effect was

**Non-Functional:**
- **Read latency:** config reads are frequently on the request hot path of the services consuming them — must be low-single-digit milliseconds, typically served from a local cache, not a round trip to a central store per read
- **Consistency guarantee:** stronger than plain eventual consistency, or explicitly bounded/documented staleness (e.g. "config propagates fleet-wide within N seconds") — an inconsistent config read is qualitatively worse than a stale product-data read, because it can misconfigure a service into an active outage rather than just showing slightly old data
- **Blast-radius bound:** a single bad push must be structurally limited in how much of the fleet it can reach before detection and rollback
- **Rollback time bound:** from "regression detected" to "fleet back on last-known-good config" must be fast — minutes, not the time it took to roll the bad change out in the first place

---

## Step 2: High-Level Architecture

```
Config author -> Push API -> Config Control Plane (source of truth,
                              versioned, validated)
                                   |
                                   | propagate (staged, see Step 3)
                                   v
                     Regional Config Distribution Layer
                       (replicated store per region)
                                   |
                                   | pull/push to local cache
                                   v
                  Consuming services' local in-process cache
                       (read path never leaves the process)

Health signal loop (parallel, continuous):
  Consuming services' error/latency metrics -> Monitoring ->
  Automated regression detector -> triggers rollback in
  Config Control Plane if a push correlates with a regression
```

Config reads never hit the control plane directly on the request path — every consuming service reads from a local cache that's kept warm by the regional distribution layer, which is what keeps read latency low regardless of how far a service is from the control plane.

---

## Step 3: Why Config Needs Stronger Consistency Than Typical Data

```
A stale PRODUCT read (e.g. a slightly out-of-date recommendation)
degrades quality slightly. A stale or INCONSISTENT config read can:
  - leave some fraction of the fleet running an old, incompatible
    config against a new binary version (or vice versa)
  - cause a split-brain where different replicas of the same
    service disagree on a safety-critical setting (e.g. a rate
    limit, a feature gate, a kill switch)

This is exactly the [[solution-arch/concepts/cap-theorem]] trade-off,
but the "cost of being wrong" term is much higher for config than
for most application data — which is why config systems commonly
lean on [[solution-arch/concepts/distributed-consensus]] (Paxos/Raft-
backed) for the control plane's own source-of-truth writes, even
though the READ path (from local caches) stays fast and only
eventually consistent with a small, bounded, documented staleness
window (e.g. "new config is visible fleet-wide within 30 seconds").
```

---

## Step 4: Blast-Radius Control — Staged Rollout

```
A bad config pushed to 100% of the fleet in one step is the single
biggest self-inflicted-outage risk in this design. The fix is
identical in shape to a canary binary rollout
(see [[solution-arch/patterns/blue-green-canary]]):

  1% of fleet  -> observe error/latency metrics for a bake period
  10% of fleet -> observe again
  50% of fleet -> observe again
  100% of fleet

Region-scoped staging is often layered on top of percentage
staging: roll to one region fully before any other region sees
the change at all, so a region-specific incompatibility (e.g. a
regional dependency version mismatch) is caught before it's
global.
```

---

## Step 5: Automated Detection and Rollback

```
Don't rely on a human noticing. Every staged rollout step is
gated on automated regression detection:

  Push to N% -> bake period -> compare N% cohort's error rate /
  latency / key business metric against the control (unchanged)
  cohort -> statistically significant regression detected?
    YES -> automatic rollback to last-known-good config,
           halt the rollout, alert the owning team
    NO  -> proceed to next rollout percentage

This closes the loop the same way a canary ML-model rollout does
(see [[solution-arch/scenarios/design-continuous-model-deployment-rollout]])
— the mechanism (staged exposure + automated compare + auto-rollback)
is reused across artifact types rather than reinvented per system.
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Control-plane writes | Consensus-backed (Paxos/Raft-style) | Config's source of truth needs strong correctness — a split-brain config write is worse than most application-data conflicts |
| Read path | Local in-process cache, bounded staleness | Keeps config reads off the network entirely for the common case, at the cost of a small, documented propagation delay |
| Rollout strategy | Staged percentage + region scoping, automated gating | Bounds blast radius of a bad push without slowing down the common case (a good push) unnecessarily |
| Rollback trigger | Automated regression detection, not human-only | Rollback speed is the main lever on outage duration; waiting for a human to notice is the slow path |
| Audit trail | Every push versioned and logged with rollout state | Required for postmortems and for proving compliance/change-control processes are followed |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #configuration #feature-flags #canary-rollout #system-design #nalsd*
