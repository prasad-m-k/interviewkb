# CAP Theorem

**Topic:** [[solution-arch/topics/data-architecture]], [[solution-arch/topics/scalability-and-reliability]]
**Related:** [[solution-arch/concepts/acid-vs-base]], [[solution-arch/concepts/distributed-consensus]]

## What it is
In a distributed system, you can guarantee at most **two** of three properties simultaneously:

- **C**onsistency — every read receives the most recent write (or an error)
- **A**vailability — every request receives a non-error response (may be stale)
- **P**artition Tolerance — the system continues operating even when network messages are dropped or delayed between nodes

**The catch:** in any real distributed system, network partitions *will* happen. So the real trade-off is always **C vs A** during a partition.

## How it works

```
         Consistency
              ▲
              │
    CA        │        CP
  (single-   │     (partition
   node)     │      tolerant,
             │      consistent)
             │
─────────────┼─────────────────────▶
             │                    Availability
    CP       │        AP
             │     (partition
             │      tolerant,
             │      available)
             │
```

### CP Systems (Consistency + Partition Tolerance)
- During a partition: reject writes (or reads) to maintain consistency
- Examples: **HBase, Zookeeper, etcd, MongoDB (w/ write concern majority)**
- Use when: financial transactions, inventory counts — stale data causes real harm

### AP Systems (Availability + Partition Tolerance)
- During a partition: continue serving requests, possibly returning stale data
- Examples: **Cassandra, CouchDB, DynamoDB (default), DNS**
- Use when: social feeds, product catalogues, caches — slight staleness acceptable

### CA Systems (Consistency + Availability, no partition tolerance)
- Only possible on a single node or within a cluster where partitions "cannot happen"
- Examples: traditional RDBMS (PostgreSQL, MySQL) on a single node
- In practice: not viable for distributed systems

## Complexity

CAP is a useful mental model but PACELC adds nuance:
- **PACELC:** Even when no partition (Else): trade off **Latency vs Consistency**

```
PACELC Decision Tree:
During Partition?
├── Yes → C or A?
│         ├── C → CP system (HBase, etcd)
│         └── A → AP system (Cassandra, DynamoDB)
└── No  → L or C?
          ├── L → optimise for low latency (fewer coordination roundtrips)
          └── C → optimise for consistency (more coordination)
```

## When to use
Use CAP as a framing tool when choosing a database or explaining a design trade-off in interviews.

## Common interview angles
- "Explain CAP and give examples of CP vs AP systems."
- "You're building a bank transfer system — CP or AP? Why?"
- "Your social media feed shows slightly old data sometimes — is that a CAP violation?"
- "If CAP says pick 2, why does Google Spanner claim to be CA?" (It achieves very high consistency + availability using atomic clocks and globally synchronised replicas — partition risk is extremely low, not zero.)
- "What is PACELC and how does it improve on CAP?"

## Examples

```
Scenario: Shopping cart
  During partition:
    If CP → cart becomes unavailable → bad UX (user can't add items)
    If AP → cart may show stale data → usually acceptable

Scenario: Bank balance
  During partition:
    If CP → read returns error → user cannot overdraw → correct
    If AP → read returns stale balance → user may overdraw → unacceptable

Decision: Use CP for money, AP for UX-sensitive features.
```

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
