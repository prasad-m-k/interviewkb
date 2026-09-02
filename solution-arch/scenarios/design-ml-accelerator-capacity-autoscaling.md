# System Design: Capacity Management & Autoscaling for a Mixed CPU + Accelerator (GPU/TPU) Fleet

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/rate-limiting]], [[solution-arch/concepts/distributed-caching]]
**Patterns:** [[solution-arch/patterns/bulkhead]], [[solution-arch/patterns/circuit-breaker]]

> Interview-prep scenario grounded in Google's L6 SU (Semantic Understanding Platform) SRE job description and public NALSD interview patterns — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

---

## Step 1: Requirements

**Functional:**
- Schedule inference and training jobs onto the best-fit compute type (CPU, GPU, TPU) given the job's resource profile
- Autoscale each accelerator pool based on both forecasted demand and real-time signal
- Support burst capacity for planned launches (a known future demand spike) and unplanned viral traffic (an unknown one)
- Bin-pack heterogeneous job shapes onto heterogeneous accelerator types to avoid stranded capacity
- Degrade gracefully under saturation — serve a smaller/cheaper model rather than queue or fail requests outright

**Non-Functional:**
- Accelerator utilization target high enough to be cost-efficient — idle accelerator time is the single most expensive line item at this scale (see [[solution-arch/topics/cost-architecture-finops]])
- Scale-ahead, not scale-reactive, wherever accelerator provisioning lead time is long — reactive autoscaling that works fine for stateless CPU services actively fails for accelerators
- Bounded degradation: quality loss under saturation must be graceful and monotonic, never a hard failure of the request
- Headroom for N-x unplanned traffic spikes without breaching serving SLOs, sized from historical incident data, not guessed
- Fair allocation across the accelerator pool when multiple internal consumers compete for the same scarce hardware during a spike

---

## Step 2: Why Accelerator Autoscaling Isn't Just "CPU Autoscaling With Bigger Numbers"

```
CPU autoscaling assumption: provisioning a new instance takes seconds,
so a reactive control loop (scale when utilization crosses X%) works —
by the time new capacity is needed, it's already coming online.

Accelerator reality: GPU/TPU provisioning — driver init, model load
onto the device, warm-up — can take minutes, not seconds. A purely
reactive loop scales AFTER the traffic has already arrived, which
means the saturation window (queueing, degraded quality, or drops)
already happened by the time new capacity is live.

Conclusion: accelerator capacity needs a PREDICTIVE component
(forecast-driven scale-ahead) layered on top of a reactive component
(real-time signal correction for forecast error), not reactive alone.
```

---

## Step 3: Predictive + Reactive Scaling Architecture

```
Demand Forecast (offline, periodic)
  historical traffic pattern + known launch calendar
  → target accelerator pool size, T-minus-lead-time before need
       │
       ▼
Scale-Ahead Controller
  provisions accelerator capacity ahead of forecasted demand
  (sized to the provisioning lead time, e.g. scale 15 min ahead
  if cold-start takes 10 min)
       │
       ▼
Reactive Correction Layer
  real-time queue depth / utilization signal
  corrects for forecast error in either direction:
    under-forecast → burst-scale additional capacity (best-effort,
                      may not land in time — this is where graceful
                      degradation kicks in as the safety net)
    over-forecast  → scale down to reclaim idle (expensive) capacity
```

The predictive layer handles the common case (known daily/weekly patterns, planned launches); the reactive layer is explicitly there to correct forecast error, not to be the primary mechanism — for an accelerator fleet, waiting for the reactive signal alone means the saturation window already happened.

---

## Step 4: Bin-Packing Heterogeneous Jobs

```
Problem: jobs vary widely in resource shape (a small classification
model fits comfortably on a fraction of a GPU; a large multimodal
model needs a full TPU pod slice). Naive one-job-per-accelerator
scheduling strands capacity — a job needing 20% of a GPU still
occupies the whole device if packing isn't multiplexed.

Approach:
  - Classify jobs by resource profile (memory footprint, compute
    shape, latency sensitivity) at registration time
  - Co-locate multiple small jobs on one accelerator where the
    accelerator/runtime supports multiplexing (e.g. MIG-style GPU
    partitioning), reserve whole devices for large jobs that need them
  - Keep latency-sensitive serving jobs on a separate bulkhead
    (see [[solution-arch/patterns/bulkhead]]) from best-effort batch/
    training jobs, so a batch job's resource spike never starves a
    latency-sensitive serving request
```

---

## Step 5: Graceful Degradation Under Saturation

```
Saturation ladder (each step degrades gracefully rather than failing):

  1. Normal: full model on preferred accelerator type
  2. Mild saturation: route to a queue with bounded wait, still full model
  3. Heavy saturation: fall back to a smaller/cheaper model variant
     (lower quality, but still a valid response) rather than queue further
  4. Severe saturation: shed lowest-priority traffic classes first
     (ties into admission-control/load-shedding design — see the
     dedicated rate-limiting/load-shedding scenario for that mechanism)

The key design choice: quality degrades in discrete steps BEFORE
outright failure, so the platform's most valuable traffic (e.g. a
low-latency product surface) keeps getting *a* response, just a
cheaper one, instead of a hard error.
```

---

## Step 6: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Scaling model | Predictive scale-ahead + reactive correction | Accelerator provisioning lead time makes pure reactive scaling too slow to avoid a saturation window |
| Bin-packing | Multiplex small jobs, reserve whole devices for large ones | Naive 1:1 job-to-accelerator scheduling strands capacity and inflates cost |
| Isolation | Bulkhead: serving vs. batch/training on separate pools | A resource-hungry batch job must never starve latency-sensitive serving |
| Saturation response | Graceful multi-step degradation (smaller model → load shedding) | A hard failure is strictly worse than a lower-quality-but-valid response |
| Headroom sizing | Derived from historical spike/incident data | Arbitrary headroom either wastes cost (over-sized) or fails during real spikes (under-sized) |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #capacity-planning #autoscaling #ml-infrastructure #system-design #nalsd*
