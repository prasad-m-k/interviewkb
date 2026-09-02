# System Design: Cross-Region Petabyte-Scale Data Migration & DR

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/distributed-caching]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/cap-theorem]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]]

> Interview-prep scenario for a general Google L6 SRE NALSD prompt — architectural specifics are plausible public-knowledge system design reasoning, not verified insider Google infrastructure detail.

The classic framing is "move 100 PB of data from the East Coast to the West Coast," but the same design answers the broader "multi-region data migration / DR plan for a multi-PB storage cluster" prompt — moving data for a planned migration and moving data to establish a DR replica are the same mechanics with a different trigger.

---

## Step 1: Requirements

**Functional:**
- Transfer the full dataset (100 PB, worked example below) to the destination region
- Keep the source system consistent and serving live traffic throughout the migration — this is not an offline batch job against a frozen dataset
- Verify destination integrity — prove the copy is correct, not just "the transfer job reported success"
- Support a cutover with a bounded, well-understood consistency window (source and destination briefly diverge; that divergence must be closed before cutover completes)

**Non-Functional:**
- **Total migration time target:** bounded and predictable, not open-ended
- **Bandwidth/cost budget:** network egress cost has a real dollar figure at this scale and must be weighed against alternatives
- **Zero data loss, verified:** not assumed from "the job didn't error" — see Step 5
- **Minimal service disruption during cutover:** ideally near-zero customer-visible impact, achieved by a delta-sync phase rather than a single stop-the-world copy

---

## Step 2: The Physics First — Why Bandwidth Alone Doesn't Work

```
100 PB = 100 * 10^15 bytes = 8 * 10^17 bits

Over a (generous, dedicated) 100 Gbps link at 100% theoretical
utilization:
  8*10^17 bits / (100*10^9 bits/sec) = 8*10^6 seconds ~= 92 days

Real-world utilization is well under 100% (contention, retries,
protocol overhead) — the realistic number is materially worse
than 92 days for network-only transfer of the FULL dataset.

This is the moment to say out loud in the interview: "a truck full
of hard drives has enormous bandwidth." Physically shipping storage
appliances (Google-style Transfer Appliance, AWS Snowball-class
devices) moves petabytes at effectively hundreds of Gbps to Tbps of
"bandwidth" once you count the aggregate capacity of the drives
being shipped, at a fraction of the network-egress cost and time.
```

---

## Step 3: Parallel Transfer Strategy

```
Bulk transfer (the 100 PB baseline):
  - Multiple physical appliances filled and shipped in parallel,
    OR multiple independent network paths/links running concurrently
    if network transfer is chosen for part of the data
  - Partition the dataset by shard/partition key so each
    appliance/link owns a disjoint slice — no coordination needed
    between them during the bulk phase

This bulk phase is intentionally NOT trying to catch writes that
happen on the source while it's running — that's Step 4's job.
```

---

## Step 4: Consistency During Migration — Delta Sync, Not a Single Cutover

```
Problem: the source keeps taking writes for the days/weeks the bulk
         transfer takes. A single cutover at the end would lose
         every write that landed after the bulk copy started.

Fix — two-phase migration:
  Phase 1 (bulk): copy the dataset as of some snapshot time T0.
                   Takes days/weeks via parallel physical/network transfer.
  Phase 2 (delta sync): replay every write that happened on the
                   source between T0 and now, continuously, until
                   the gap between source and destination is small
                   enough to close in a short cutover window
                   (minutes, not days).
  Cutover: briefly pause writes (or dual-write) on the source,
                   apply the final delta, verify, flip traffic to
                   destination.

The delta-sync phase is what turns "zero data loss" from an
assumption into something actually achievable — the bulk phase
alone cannot guarantee it.
```

---

## Step 5: Verification at Scale — You Cannot Re-Read Everything

```
Problem: re-reading and byte-comparing 100 PB at the destination
         to "prove" correctness is itself a multi-week operation —
         defeats the purpose.

Fix: hierarchical verification, not exhaustive comparison.
  - Per-chunk checksums computed during transfer (e.g. per 64MB/1GB
    block), verified immediately as each chunk lands
  - Merkle-tree-style hash aggregation: hash of hashes up to a
    per-shard root hash, then up to a full-dataset root hash —
    comparing two root hashes proves equivalence of everything
    beneath them without re-reading the underlying bytes at
    verification time
  - Statistical sampling: full-content spot-check of a
    randomly-selected subset as a defense-in-depth belt-and-suspenders
    layer, not the primary correctness proof
```

---

## Step 6: Cost — Network vs. Physical Shipment Crossover

```
Rough shape of the trade-off (illustrative, not vendor-specific):
  Network egress cost scales with TOTAL BYTES moved, roughly
    linearly, regardless of how long it takes.
  Physical appliance shipment cost scales with NUMBER OF APPLIANCES
    (and their round-trip logistics time), largely independent of
    how much of each appliance's capacity is used.

At small data volumes, network transfer wins (no logistics overhead).
At petabyte scale, physical shipment tends to win on both cost and
time, which is exactly why cloud providers built dedicated
appliance-based transfer services for this use case rather than
expecting customers to always push large migrations over the network.
```

---

## Step 7: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Bulk transfer method | Physical appliances (parallelized) for the majority of the volume | Network-only transfer of 100 PB is measured in months even on a dedicated 100 Gbps link |
| Consistency approach | Bulk snapshot + continuous delta sync + short cutover window | A single stop-the-world copy either loses in-flight writes or requires unacceptable downtime |
| Verification method | Chunk checksums + Merkle-tree root-hash comparison + sampling | Exhaustive re-read-and-compare at 100 PB scale is itself infeasible |
| Cutover | Brief pause/dual-write + final delta + verify + traffic flip | Minimizes the customer-visible disruption window to minutes rather than the full migration duration |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #data-migration #disaster-recovery #capacity-planning #system-design #nalsd*
