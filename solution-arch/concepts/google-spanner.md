---
uid: 5a9c2f14-7b3e-4d8a-9c61-3f7a2e0b8d95
---

# Google Spanner

**Topic:** [[solution-arch/topics/data-architecture]], [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/paxos]], [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/acid-vs-base]], [[solution-arch/concepts/federated-query-engines]]

## What it is
Google Spanner is a globally distributed, horizontally scalable relational database that provides **external consistency** — a guarantee stronger than typical linearizability — across data centers on different continents, while still exposing SQL and full ACID transactions. It's the standard answer to "how do you get strong consistency AND horizontal scale AND multi-region," a combination CAP-theorem intuition suggests should be nearly impossible. Spanner doesn't defeat CAP — it engineers the probability of a partition down to near-zero with redundant infrastructure, and pays for strict time synchronization to get the consistency guarantee.

## How it works

### Data is sharded, and each shard is its own Paxos group
```
Table split into contiguous key-range shards ("splits")
Each shard is replicated (typically 5 replicas across zones/regions)
Each shard's replica set runs its own [[solution-arch/concepts/paxos]] instance:
  → one replica is the Paxos leader for that shard and handles writes
  → a majority (e.g. 3 of 5) must ack a write before it commits
```
This is why [[solution-arch/concepts/distributed-consensus]]'s Paxos-vs-Raft table lists Spanner under Paxos: every shard's replication is a *separate* Paxos instance, not one global one.

### TrueTime — the key differentiator
Every Spanner node has access to a TrueTime API backed by GPS receivers and atomic clocks in each datacenter, returning not a single timestamp but a bounded **interval**:
```
TT.now() → [earliest, latest]     (a bounded uncertainty window, typically single-digit ms)

Instead of trusting one clock, Spanner KNOWS the true time lies somewhere in this
interval — so it can reason about "did event A definitely happen before event B"
across machines with imperfectly synchronized clocks, as long as the bound holds.
```

### Commit-wait — turning clock uncertainty into external consistency
```
Transaction wants to commit with timestamp s
Spanner does NOT release the transaction's locks / make it visible until:
    TT.now().earliest > s
  (it waits out the remaining clock uncertainty before releasing)

Guarantee: if transaction T2 starts (in real, wall-clock time) after T1 commits,
T2 is assigned a timestamp > T1's, and is guaranteed to see T1's effects.
→ External consistency: commit order matches real-time order, globally —
  not just per-replica or per-shard.
```
The commit-wait duration is bounded by the TrueTime uncertainty interval — small (typically single-digit milliseconds) but nonzero: it's the concrete price Spanner pays for a guarantee stronger than what most distributed databases offer.

### Cross-shard transactions — Two-Phase Commit over Paxos groups
A transaction touching multiple shards (each its own Paxos group) can't rely on a single Paxos round, so Spanner layers [[solution-arch/concepts/acid-vs-base]]'s Two-Phase Commit across the Paxos leaders of every involved shard:
```
Coordinator (one participating shard's Paxos leader) runs 2PC:
  Phase 1 (Prepare): ask every participating shard's Paxos leader to prepare
                      (each leader Paxos-commits its own prepare record to its group)
  Phase 2 (Commit):  once all participants have prepared, the coordinator commits
                      and tells every participant to commit (again via its own Paxos group)

TrueTime assigns the transaction's commit timestamp; commit-wait applies before
the result becomes externally visible.
```

## Complexity
Read-only, single-shard transactions are fast — roughly the cost of one Paxos round to a nearby replica (Spanner also supports lock-free snapshot reads at a fixed timestamp using TrueTime, without contacting the Paxos leader at all, when slight staleness is acceptable). Read-write, multi-shard transactions are the expensive path — 2PC across multiple Paxos groups plus commit-wait — and this is the operation Spanner's design deliberately makes slower in exchange for external consistency.

## When to use
- You need horizontally-scaled SQL with real ACID transactions across shards or regions, not just eventual consistency — see [[solution-arch/concepts/acid-vs-base]] for the general trade-off Spanner is unusual in escaping.
- You need strict global ordering guarantees for correctness (e.g. a financial ledger spanning regions) and can afford the latency cost of commit-wait and cross-region Paxos consensus.
- Available publicly as Cloud Spanner; also the storage substrate under Google's F1 Query — see [[solution-arch/concepts/federated-query-engines]].
- Don't reach for a Spanner-shaped system for workloads that would be fine with regional ACID or eventual consistency — the operational and cost overhead (atomic-clock-grade time infrastructure or its cloud-managed equivalent, cross-region replication) isn't free, and most applications don't need external consistency specifically.

## Common interview angles
- "How does Spanner get strong consistency across regions without violating CAP?" (It doesn't violate CAP — it engineers partition probability down via redundant infrastructure and pays commit-wait latency for external consistency; during an actual partition it sacrifices availability on the affected shard, making it a CP system with very high *measured* availability, not literally a CA system)
- "What problem does TrueTime actually solve?" (Bounding clock uncertainty so the system can safely wait out that bound and guarantee real-time commit ordering, instead of trusting an unbounded or unsynchronized clock)
- "Why does Spanner need 2PC if it already uses Paxos?" (Paxos gives consensus *within* one shard's replica set; 2PC is layered on top to coordinate atomicity *across* multiple independent Paxos groups for a multi-shard transaction)
- "What's the latency cost of commit-wait?" (Bounded by the TrueTime uncertainty interval — small, typically single-digit ms, but nonzero, and paid on every read-write transaction's commit path)
- "How would you design a service that doesn't need Spanner-level consistency?" (Regional database + [[solution-arch/patterns/saga]] or [[solution-arch/patterns/outbox]] for cross-service consistency — usually far cheaper and sufficient)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #spanner #distributed-databases #system-design*
