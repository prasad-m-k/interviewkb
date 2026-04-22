# Scalability and Reliability

**Related:** [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/database-sharding]], [[solution-arch/concepts/load-balancing]], [[solution-arch/concepts/distributed-consensus]]

---

## Replication

Keeping copies of data on multiple nodes.

```
Primary ──writes──▶  Replica 1
        ──writes──▶  Replica 2  ──reads──▶ Clients
        ──writes──▶  Replica 3
```

### Synchronous vs Asynchronous Replication

```
Synchronous:
Client ──▶ Primary ──▶ Replica ──ACK──▶ Primary ──ACK──▶ Client
           (waits for replica before confirming write)
           + Durable: no data loss on primary failure
           - Higher write latency

Asynchronous:
Client ──▶ Primary ──ACK──▶ Client
                └──background──▶ Replica
           + Low write latency
           - Risk of data loss if primary fails before replication
```

### Replication Topologies

| Topology | Description | Risk |
|----------|-------------|------|
| Single-leader | One primary, N replicas | Primary is SPOF |
| Multi-leader | Multiple primaries (across DCs) | Write conflicts |
| Leaderless | Any node accepts writes; quorum | Complexity |

---

## Partitioning (Sharding)

Splitting data across nodes so each node holds a subset.

```
Range Partitioning:          Hash Partitioning:
A–M  → Shard 1              hash(key) % N → Shard
N–Z  → Shard 2
```

**Hot spot problem:** if one key is disproportionately accessed, its shard becomes a bottleneck.
**Fix:** Compound key (e.g., `user_id + timestamp`) to distribute load.

See: [[solution-arch/concepts/database-sharding]]

---

## Consensus and Leader Election

In a distributed system, nodes must agree on a single value (e.g., who is the leader).

```
     Node A proposes: "I am leader"
     Node B proposes: "I am leader"
                │
         Consensus Protocol (Raft)
                │
     Quorum of (N/2 + 1) nodes agrees
                │
         One leader elected
```

- **Raft:** understandable implementation; leader-based; log replication
- **Paxos:** proven but complex; used in Google Spanner, Chubby
- **Quorum:** requires (N/2 + 1) nodes to agree; tolerates (N-1)/2 failures

See: [[solution-arch/concepts/distributed-consensus]]

---

## Failure Modes and Fault Tolerance

```
Failure type         What breaks              Mitigation
─────────────────────────────────────────────────────────
Node crash           Service unavailable      Redundancy + failover
Network partition    Nodes can't talk         CAP trade-off choice
Slow node            Cascading timeouts       Timeouts + circuit breaker
Corrupt data         Silent bad results       Checksums + validation
Byzantine fault      Malicious/wrong answer   BFT protocols (blockchain)
```

### Timeout and Retry Strategy

```
Client ──▶ Service (timeout = 500ms)
              │
         No response after 500ms
              │
         Retry with exponential backoff + jitter:
              │   attempt 1: wait 100ms
              │   attempt 2: wait 200ms + rand(0–50ms)
              │   attempt 3: wait 400ms + rand(0–50ms)
              │
         After N retries: fail fast / fallback
```

**Jitter** prevents the "thundering herd" where all retrying clients hit the service simultaneously.

---

## Stateless vs Stateful Services

| | Stateless | Stateful |
|--|-----------|---------|
| Definition | No local state between requests | Maintains state (session, connection) |
| Scale | Trivial — add instances behind LB | Requires sticky sessions or external state |
| Failure | Any instance can handle any request | State is lost if instance crashes |
| Examples | REST API, Lambda | WebSocket server, database, cache |

**Rule:** Externalise state. Store sessions in Redis, files in object storage, state in DB. Keep services stateless.

---

## Load Shedding and Backpressure

When a system is overloaded, it should **shed load gracefully** rather than collapse.

```
Normal:   Client ──100 RPS──▶ Service ──▶ DB (handles fine)

Overload: Client ──1000 RPS──▶ Service
                                   │
                    ┌──────────────┤
                    │  Queue full? │
                    └──────────────┤
                         │ yes     │ no
                         ▼         ▼
                    Return 429   Enqueue
                    (Too Many    & process
                    Requests)
```

**Backpressure:** producer slows down when consumer signals it's overwhelmed. Common in streaming systems (Kafka, Reactive Streams).

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
