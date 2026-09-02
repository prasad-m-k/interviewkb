---
uid: 6c1f4a83-2e7b-4d9c-a538-8b0e3f9c1a72
---

# Raft

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/paxos]], [[solution-arch/concepts/leader-election]]

## What it is
Raft (Ongaro & Ousterhout, 2014) is a consensus protocol designed explicitly to be more understandable than Paxos while providing the same core guarantee: a majority of nodes agree on a single, ordered log of values despite crashes and message loss. [[solution-arch/concepts/distributed-consensus]] already covers Raft's shape at survey level (leader election, log replication, quorum math) — this page goes deeper into the actual RPCs, term numbers, and the specific safety argument that makes Raft provably correct, the way [[solution-arch/concepts/paxos]] does for Paxos.

## How it works
Raft splits consensus into three independently-understandable sub-problems: leader election, log replication, and safety.

### Terms — Raft's logical clock
```
Time is divided into TERMS, each with at most one leader.
Term numbers increase monotonically and act as a logical clock:
  Every RPC carries the sender's term.
  IF a node sees a term greater than its own:
    It updates its term and reverts to follower — even if it was a leader.
  A node never votes twice within the same term.
```

### Leader Election — RequestVote RPC
```
Follower's election timeout fires (randomized, typically 150-300ms —
the randomization is what prevents split votes from recurring indefinitely):
  → becomes Candidate, increments its term, votes for itself
  → sends RequestVote(term, candidateId, lastLogIndex, lastLogTerm) to all peers

Peer receiving RequestVote grants its vote IFF:
  1. candidate's term >= peer's current term, AND
  2. peer hasn't already voted in this term, AND
  3. candidate's log is AT LEAST AS UP-TO-DATE as the peer's own log
     (compare lastLogTerm first, then lastLogIndex — this is the
      "election restriction" that makes safety possible, see below)

Candidate wins with votes from a majority (including itself) → becomes Leader
  → immediately sends empty AppendEntries (heartbeats) to establish authority
```

### Log Replication — AppendEntries RPC
```
Client ──▶ Leader: "x = 5"
Leader appends (term, command) to its own log (not yet committed)
Leader sends AppendEntries(term, prevLogIndex, prevLogTerm, entries[], leaderCommit)
  to every follower

Follower receiving AppendEntries:
  IF its log doesn't contain an entry at prevLogIndex matching prevLogTerm:
    Reject — the leader must back up and retry with an earlier prevLogIndex
    (this is how Raft repairs a follower whose log has diverged)
  ELSE:
    Append the new entries, overwriting any conflicting ones after this point
    Reply success

Leader receives success from a MAJORITY (including itself)
  → entry is COMMITTED → leader applies it to its state machine, responds to client
  → leaderCommit propagates to followers on the next AppendEntries; they apply it too
```

### The Log Matching Property
```
If two logs contain an entry with the same index and term,
  the logs are IDENTICAL in every entry up through that index.

This holds because:
  - A leader creates at most one entry per (term, index) pair
  - AppendEntries' consistency check (prevLogIndex/prevLogTerm) fails
    whenever the follower's log doesn't match at that point

Consequence: once a follower has ack'd an entry, its full history up to
that point doesn't need re-verifying — the consistency check is inductive.
```

### Safety — why an out-of-date node can't become leader
```
Election restriction (from the RequestVote rule above): a candidate can only
win if its log is at least as up-to-date as a MAJORITY of the cluster.

Since a committed entry is, by definition, on a majority of logs, any future
election-winning majority must overlap that commit majority by at least one
node — and that node will refuse to vote for a candidate whose log is behind.

This is the same "majority overlap" argument that makes Paxos safe (see
[[solution-arch/concepts/paxos]]), but Raft states it directly as an election
precondition instead of deriving it from proposal-number bookkeeping.
```

## Complexity
Normal operation: one round trip (AppendEntries) per committed log entry to a majority — the same cost class as Multi-Paxos's steady state, but without needing a separate stable-leader optimization layered on afterward, because Raft has a stable leader by construction between elections. Leader failure: bounded by the election timeout (typically 150-300ms) plus one RequestVote round trip — this is the concrete availability cost any leader-based consensus protocol pays on failover.

## When to use
- Raft is the default choice today for a hand-rolled or embedded consensus layer — it's what etcd, CockroachDB, TiKV, and Consul are built on, precisely because its specification is complete and directly implementable, without the extra invention Multi-Paxos requires (see [[solution-arch/concepts/paxos]]).
- When choosing infrastructure rather than implementing consensus yourself, you rarely pick "Raft vs Paxos" directly — you pick a system (etcd vs. ZooKeeper vs. a Paxos-based store) and inherit whichever protocol it uses.
- Kubernetes's own HA control plane is a concrete production example: etcd's internal leader is elected via Raft (this page); the scheduler/controller-manager leader on top of it is a separate, lease-based mechanism — see [[solution-arch/concepts/leader-election]] and [[k8s/topics/architecture]] for the full multi-master picture.

## Common interview angles
- "Why does Raft use randomized election timeouts?" (Identical timeouts across followers would cause repeated split votes — every follower would time out and start a new election simultaneously, forever)
- "What happens if two nodes both think they're leader?" (Terms prevent this: a stale leader that regains connectivity sees a higher term in an incoming RPC and immediately reverts to follower — it can never accept client writes in a term it has lost)
- "How does Raft handle a follower whose log has diverged from the leader's?" (AppendEntries' consistency check fails at the divergence point; the leader decrements nextIndex for that follower and retries, eventually finding the matching point and overwriting the follower's conflicting entries)
- "Why can't a node with a stale log win an election?" (RequestVote's up-to-date check refuses the vote; since a committed entry lives on a majority, any election-winning majority must include at least one voter who rejects a behind candidate)
- "How does Raft compare to Paxos?" (Same core guarantee; see [[solution-arch/concepts/distributed-consensus]] for the comparison table and [[solution-arch/concepts/paxos]] for Paxos's own protocol — Raft's leader election and log structure are explicit and complete in the original spec, which Multi-Paxos leaves as an exercise)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
