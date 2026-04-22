# ACID vs BASE

**Topic:** [[solution-arch/topics/data-architecture]]
**Related:** [[solution-arch/concepts/cap-theorem]], [[solution-arch/concepts/distributed-consensus]]

## ACID (Traditional RDBMS)

| Property | Meaning | Example |
|----------|---------|---------|
| **A**tomicity | All-or-nothing: either all operations succeed or none do | Transfer: debit + credit either both happen or neither |
| **C**onsistency | Database moves from one valid state to another (constraints respected) | Balance can't go negative if constraint exists |
| **I**solation | Concurrent transactions don't interfere with each other | Two users updating same row don't corrupt each other |
| **D**urability | Committed transactions survive crashes | Written to WAL before ack; survives power loss |

```
BEGIN TRANSACTION
  UPDATE accounts SET balance = balance - 100 WHERE id = 1  -- debit
  UPDATE accounts SET balance = balance + 100 WHERE id = 2  -- credit
COMMIT

If crash after debit but before credit:
  → Transaction rolled back (atomicity)
  → Balances unchanged (consistency)
```

### Isolation Levels (trade off correctness vs concurrency)

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|-----------|---------------------|-------------|-------------|
| Read Uncommitted | ✅ Possible | ✅ Possible | ✅ Possible | Highest |
| Read Committed | ❌ Prevented | ✅ Possible | ✅ Possible | High |
| Repeatable Read | ❌ Prevented | ❌ Prevented | ✅ Possible | Medium |
| Serialisable | ❌ Prevented | ❌ Prevented | ❌ Prevented | Lowest |

**Default:** most DBs use Read Committed (PostgreSQL) or Repeatable Read (MySQL InnoDB).

---

## BASE (Distributed / NoSQL systems)

| Property | Meaning |
|----------|---------|
| **B**asically **A**vailable | System guarantees availability (may return stale/partial data) |
| **S**oft state | State may change over time even without new input (convergence) |
| **E**ventually consistent | All replicas will eventually converge to the same value |

```
Write to DynamoDB replica A:
  A: balance = $100

Replica B not yet synced:
  B: balance = $80  (stale)

Read from B immediately after write:
  Returns $80  (eventual consistency — will converge to $100 shortly)
```

---

## ACID vs BASE Decision Framework

```
Use ACID when:                          Use BASE when:
────────────────────────────────────    ────────────────────────────────────
Financial transactions                  Social media feeds
Inventory management                    Shopping cart
Medical records                         Product catalogue
Legal compliance data                   User preferences
Order processing                        Recommendation data
Multi-entity consistency required       High write throughput needed
Strong guarantees required              Availability > consistency
```

---

## Distributed ACID

Traditional RDBMS ACID works on a single node. Distributed ACID is much harder:

- **Two-Phase Commit (2PC):** coordinator asks all nodes to prepare, then commits. Blocking if coordinator crashes.
- **Three-Phase Commit (3PC):** adds a pre-commit phase; non-blocking but more round trips.
- **Saga Pattern:** break transaction into compensatable steps (see [[solution-arch/patterns/saga]])
- **Google Spanner:** achieves distributed ACID using TrueTime and Paxos — possible but expensive.

---

## Common interview angles
- "Why can't distributed systems guarantee ACID?" (Network partitions mean you can't atomically update two separate nodes without complex coordination)
- "What are the isolation levels and what anomalies does each prevent?"
- "When would you choose a NoSQL (BASE) database over PostgreSQL?"
- "How do you handle distributed transactions without 2PC?" (Saga pattern, outbox pattern)
- "What is a write-ahead log and how does it ensure durability?"

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
