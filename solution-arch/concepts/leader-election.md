---
uid: 3d6b8e21-9a4f-4c7d-b092-1e5f8c2a6b73
---

# Leader Election

**Topic:** [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/paxos]], [[solution-arch/concepts/raft]], [[solution-arch/concepts/service-discovery]]

## What it is
Leader election is the general problem of getting a group of nodes to agree on exactly one of them acting as coordinator for some duty — accepting writes, running a scheduled job, holding a lock — without two nodes ever both believing they're the leader at once. It's a building block used *inside* full consensus protocols (Raft has leader election built in; see [[solution-arch/concepts/distributed-consensus]] for that specific mechanism), but it also shows up as a standalone problem, solved by simpler algorithms, whenever an application just needs "pick one coordinator" rather than full replicated-log consensus.

## How it works

### Bully Algorithm
Every node has a unique ID; the highest-ID live node wins.
```
Node C notices the leader is down, starts an election:
  C sends Election to every node with a HIGHER id than itself
  IF no higher node responds within a timeout:
    C declares itself leader, announces to all nodes
  IF a higher node responds "I'm alive":
    That node takes over running the election; C waits
```
Simple but chatty (O(n²) messages in the worst case) and elects purely on ID, not health or capacity — a poor fit if the highest-ID node happens to be under-resourced.

### Ring Algorithm
Nodes arranged in a logical ring; one election message circulates a single time around, each node comparing and forwarding.
```
Node detects leader failure, sends Election(self_id) to the next node in the ring
Each node: if incoming id > mine, forward it as-is
           if incoming id < mine, forward MY id instead
Message travels the full ring → highest id seen wins →
  a Coordinator message then circulates announcing the winner
```
Lower message complexity than Bully (O(n)), but a single failed node or link on the ring can stall an election in progress unless the ring implementation has failure-skip logic.

### Lease-Based Election (what production systems actually use)
Almost every real system reaches for this instead of hand-rolled Bully/Ring:
```
All candidate nodes race to create the same key with a TTL lease
  (e.g. etcd: a conditional PUT on /leader-lock with a lease — only one writer wins)

Winner: holds the lease, renews it periodically (heartbeat) before it expires
Losers: watch the key; if it disappears (lease expired / holder crashed), they race again

Leader crashes → lease not renewed → TTL expires → key deleted → election re-triggers automatically
```
ZooKeeper's classic pattern achieves the same effect slightly differently: candidates create *ephemeral sequential* znodes under a shared path; whichever candidate holds the lowest sequence number is leader, and every other candidate watches only the next-lowest node (never the leader directly) — this avoids a thundering herd of watch notifications firing on every single candidate when the leader steps down.

## Complexity
Bully: O(n²) messages, worst case. Ring: O(n) messages, one full loop. Lease-based (etcd/ZooKeeper): effectively O(1) from the application's perspective — a single conditional write — but this delegates the actual hard consensus problem to a system (etcd, ZooKeeper) that already solved it once, rather than re-solving split-brain-safe agreement per application.

## When to use
- Don't hand-roll Bully or Ring in a production system — use a lease/lock primitive on top of an already consensus-backed store (etcd, ZooKeeper, Consul, or a managed cloud equivalent) unless you're specifically building infrastructure at that layer.
- Reach for this whenever exactly one instance of something must run at a time: a cron/batch job leader, a singleton controller or operator, a primary-replica failover coordinator. See [[solution-arch/scenarios/design-distributed-job-scheduler]] and [[solution-arch/scenarios/design-global-fleet-software-rollout]] for where this shows up inside a larger system design.
- **Kubernetes multi-master control plane** is a production example of exactly this pattern: `kube-scheduler` and `kube-controller-manager` each run multiple replicas (`--leader-elect=true`) that race to hold a `Lease` object in `kube-system`, backed by etcd's compare-and-swap — the same lease-based election described above, not Raft directly (etcd itself uses Raft for its own internal replication — see [[solution-arch/concepts/raft]]). Full breakdown: [[k8s/topics/architecture]].
- If the requirement goes beyond "pick one coordinator" to "the whole group must agree on an ordered log of values," reach for full consensus instead — [[solution-arch/concepts/distributed-consensus]], [[solution-arch/concepts/paxos]] — leader election alone doesn't give you that.

## Common interview angles
- "How would you implement leader election without ZooKeeper or etcd?" (Bully or Ring algorithm — know the trade-off: Bully elects on raw ID, not health, and is O(n²) in messages)
- "Why do most real systems use etcd/ZooKeeper rather than Bully/Ring directly?" (Those systems are already consensus-backed; a lease-based election built on top avoids re-implementing split-brain-safe agreement from scratch)
- "What happens if the leader is alive but network-partitioned from the lease store?" (It fails to renew the lease in time, the lease expires, a new leader is elected elsewhere — the old leader must self-fence, i.e. stop acting as leader once it can no longer confirm it holds the lease, or you get split-brain)
- "How does Raft's leader election differ from standalone Bully/Ring?" (Raft's is embedded inside a consensus protocol — winning also requires having the most up-to-date replicated log, not just the highest ID; see [[solution-arch/concepts/distributed-consensus]])
- "What is 'fencing' and why does a lease-based leader need it?" (A leader that believes it still holds the lease — e.g. due to a long GC pause — but actually lost it must be prevented from writing; fencing tokens/epoch numbers attached to every write let downstream systems reject writes from a stale leader)

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
