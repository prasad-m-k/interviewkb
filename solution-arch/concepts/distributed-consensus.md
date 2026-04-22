# Distributed Consensus

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/database-sharding]]

## What it is
Consensus is the problem of getting a group of nodes to agree on a single value — essential for leader election, distributed locks, configuration management, and replicated state machines.

## Why it's hard

```
Node A: "I propose value X"
Node B: "I propose value Y"
Node C: (crashes mid-vote)
Network: drops some messages

Without a protocol, nodes may never agree (or agree on different values).
```

## Raft (Understandable Consensus)

Raft divides consensus into three sub-problems: leader election, log replication, safety.

```
        ┌─────────────┐
        │   Leader    │◀── all writes go here
        └──────┬──────┘
               │ replicate log entries
       ┌───────┼──────────┐
       ▼       ▼          ▼
  ┌────────┐ ┌────────┐ ┌────────┐
  │Follower│ │Follower│ │Follower│
  └────────┘ └────────┘ └────────┘
  
  Leader election:
    Leader sends heartbeats every 150ms
    If follower misses heartbeat → election timeout (150–300ms random)
    Candidate increments term, requests votes
    Majority vote (N/2 + 1) → new leader
```

**Log replication:**
```
Client ──▶ Leader: "x = 5"
Leader appends to log, sends AppendEntries to followers
Followers append to their logs, ack
Leader receives majority acks → commits, applies to state machine
Leader responds to client
```

**Safety:** A leader must have the most up-to-date log to win election (no stale leader can win).

## Quorum

A quorum is the minimum number of nodes that must agree for an operation to succeed.

```
N = 5 nodes
Write quorum: W = 3
Read quorum:  R = 3

W + R > N  →  3 + 3 = 6 > 5  →  reads always overlap with writes
→ Strong consistency guaranteed

Tolerate failures: floor((N-1)/2) = 2 node failures
```

Tuning quorum:
```
W=1, R=N: fast writes, slow reads (write-optimised)
W=N, R=1: slow writes, fast reads (read-optimised)
W=N/2+1, R=N/2+1: balanced, strongly consistent
W=1, R=1: fast both, eventual consistency only
```

## Leader Election

```
Before election:
  Node A: leader (healthy)
  Node B: follower
  Node C: follower

Node A crashes:

  B and C detect missing heartbeats (timeout)
  B sends RequestVote to C (term=2)
  C grants vote
  B becomes leader (majority: B + C)
  B starts sending heartbeats

If A recovers:
  A receives heartbeat from B with term=2 > A's term=1
  A reverts to follower
```

## Paxos vs Raft

| | Paxos | Raft |
|--|-------|------|
| Understandability | Hard (academic) | Designed to be understandable |
| Phase | Prepare + Accept | Leader election + Log replication |
| Used in | Google Spanner, Chubby | etcd, CockroachDB, TiKV |
| Multi-Paxos | Log variant (complex) | Natural multi-entry log |

## Split-Brain

When a network partition divides nodes into two groups that each believe they are the majority:

```
Before partition:
  [A] [B] [C] [D] [E]    (5 nodes, quorum = 3)

Network partition:
  [A] [B] [C]  |  [D] [E]
  (quorum=3)   | (only 2, no quorum — should not elect leader)

Risk: if quorum is miscounted, both sides elect a leader → split-brain
→ Two leaders accept writes → data diverges
```

**Prevention:** Strict majority quorum (> N/2) ensures only one partition can form a quorum.

## Common interview angles
- "What is the split-brain problem?" (Two nodes both believe they are leader; prevented by quorum)
- "How does etcd/Zookeeper use Raft/ZAB for leader election in Kubernetes?" (etcd stores cluster state; Raft ensures all control-plane nodes agree)
- "What is the quorum formula for reads and writes?" (W + R > N)
- "Why is Paxos considered hard?" (Multi-Paxos has many implementation details not in the original paper)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
