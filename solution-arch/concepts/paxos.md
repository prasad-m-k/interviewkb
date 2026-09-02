---
uid: 8f3c1a72-4e9d-4b6a-a1c5-2d7e9f0b3a84
---

# Paxos

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/leader-election]], [[solution-arch/concepts/google-spanner]]

## What it is
Paxos is the original distributed consensus protocol (Lamport, 1998) for getting a majority of unreliable nodes to agree on a single value despite node crashes and message loss. It's the algorithm [[solution-arch/concepts/distributed-consensus]] contrasts with Raft in its comparison table — this page covers the actual two-phase protocol; see that page for the high-level Paxos-vs-Raft trade-off and for where Google's own infrastructure (Spanner, Chubby) uses it.

## How it works (Basic Paxos)
Roles: Proposer, Acceptor, Learner (a single node can play more than one role).

**Phase 1 — Prepare / Promise:**
```
Proposer picks a proposal number n (must be higher than any it has used before)
Proposer → Prepare(n) → majority of Acceptors

Acceptor receives Prepare(n):
  IF n > highest_promised_n it has ever seen:
    Promise: "I will not accept any proposal numbered < n"
    IF Acceptor already accepted a value v under some earlier proposal n' < n:
      Reply Promise(n, n', v)   ← tells the proposer about that already-accepted value
    ELSE:
      Reply Promise(n, null, null)
  ELSE:
    Reject (or ignore)
```

**Phase 2 — Accept / Accepted:**
```
Proposer collects Promise replies from a majority.
IF any Acceptor reported an already-accepted value:
  Proposer MUST adopt the highest-numbered one of those values — it cannot pick its own.
ELSE:
  Proposer is free to propose its own value.

Proposer → Accept(n, v) → majority of Acceptors

Acceptor receives Accept(n, v):
  IF n >= highest_promised_n:
    Accept it: store (n, v), reply Accepted(n, v)
  ELSE:
    Reject

Proposer receives Accepted from a majority → value v is CHOSEN.
Proposer/Learners broadcast the chosen value.
```

The rule that a proposer **must adopt the highest-numbered already-accepted value it learns about** is the crux of Paxos's safety: it guarantees that once a value is chosen by a majority, no later proposal round can choose a different value — only re-confirm the same one, even with proposers racing concurrently.

## Why proposal numbers matter — a failure walkthrough
```
Proposer A: Prepare(n=1) → majority promises → Accept(n=1, "X") → only 1 acceptor
             gets it before A crashes (no majority yet — X is NOT chosen)

Proposer B: Prepare(n=2) → majority promises; one Acceptor reports
             "I already accepted (n=1, X)"
  → B MUST propose "X" (not its own value Y), even though no majority chose it yet
  → B: Accept(n=2, "X") → majority accepts → "X" is now safely chosen

Without the "adopt the highest known value" rule, B could have proposed Y instead,
and different learners could end up believing different values were chosen.
```

## Multi-Paxos
Running Basic Paxos for every single log entry costs two full message round trips per value, every time. Multi-Paxos optimizes by holding a stable leader across many proposals: once one proposer establishes itself as leader for a range of proposal numbers, Phase 1 can be skipped for subsequent entries — only Phase 2 runs per log entry, collapsing to roughly one round trip per committed value. "Who is the stable leader and for how long" is an extra layer of machinery not specified in the original Paxos paper — a large part of why Paxos has a reputation for being simple to state but notoriously hard to implement correctly (see [[solution-arch/concepts/distributed-consensus]]'s Paxos-vs-Raft comparison, where Raft makes this leader-election layer explicit and fully specified from the start).

## Complexity
Best case (no contention, a single round): 2 message round trips (Prepare/Promise, then Accept/Accepted) to commit one value across a majority. Under proposer contention — dueling proposers keep issuing higher proposal numbers that interrupt each other — Basic Paxos can livelock, with no proposer ever completing a round before a higher-numbered one preempts it. This is a concrete motivation for both Multi-Paxos's stable-leader optimization and for Raft's explicit, randomized-timeout leader election.

## When to use
- You need a majority-based agreement primitive and either control the implementation or are choosing a system already built on one — Paxos itself is rarely hand-implemented in application code; you reach for a system built on it (Google Spanner, Chubby; ZooKeeper's ZAB protocol is Paxos-like) or on Raft (etcd, CockroachDB, Consul) instead.
- Understanding Paxos in depth matters most for interviews at companies whose core infrastructure is Paxos-based — see [[solution-arch/concepts/google-spanner]] for how Spanner layers Paxos, TrueTime, and 2PC together.

## Common interview angles
- "Walk through Basic Paxos's two phases." (Prepare/Promise establishes a proposal number and surfaces any already-accepted value; Accept/Accepted commits a value once a majority of acceptors accept it)
- "Why must a proposer adopt a previously-accepted value instead of its own?" (Safety: it guarantees only one value can ever be chosen, even with concurrent competing proposers)
- "What's the difference between Basic Paxos and Multi-Paxos?" (Multi-Paxos elects a stable leader to skip Phase 1 for most entries, turning a per-value 2-round-trip protocol into roughly one round trip per log entry)
- "Why is Paxos considered hard to implement even though it's provably correct?" (The original paper describes Basic Paxos for a single value; a production replicated log needs Multi-Paxos, log compaction, and membership/reconfiguration changes — none fully specified in the paper)
- "How does Paxos differ from Raft?" (Same core safety guarantee; see [[solution-arch/concepts/distributed-consensus]] — Raft makes leader election and log structure explicit and fully specified, which is why it was designed as the "understandable" alternative)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
