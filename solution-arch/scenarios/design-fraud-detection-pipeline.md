# System Design: Fraud Detection Pipeline for a Payments Platform

**Difficulty:** Hard
**Concepts:** [[solution-arch/concepts/message-queues]], [[solution-arch/concepts/idempotency]], [[solution-arch/concepts/distributed-caching]]
**Patterns:** [[solution-arch/patterns/event-driven-architecture]], [[solution-arch/patterns/outbox]], [[solution-arch/patterns/inbox]], [[solution-arch/patterns/circuit-breaker]], [[solution-arch/patterns/bulkhead]]

---

## Step 1: Requirements

**Functional:**
- Score every transaction in real time and return one of: **approve / decline / step-up auth (e.g. OTP) / hold for manual review**
- Combine two decision layers: a fast **rules engine** (velocity limits, blocklists, geo/IP mismatch) and an **ML model score**
- Case management queue for flagged transactions, with analyst decisions feeding back as labeled training data
- Support multiple transaction types (card-present, card-not-present, ACH, wallet transfer) — each with different fraud signal profiles
- Return **reason codes** with every decline (regulatory requirement — adverse-action notices)
- Support shadow-mode model evaluation: a new model scores live traffic without affecting decisions, before rollout

**Non-Functional:**
- **Latency:** synchronous scoring must fit inside the payment network's authorization window — P99 < 200–300ms end to end, including feature lookup, rules, and model inference
- **Throughput:** peak transaction volume (e.g. Black Friday) can run 10x baseline; the pipeline must absorb bursts without shedding scoring
- **Availability:** 99.99%+, with an explicit, deliberate **fail-open vs fail-closed** policy when the scoring service itself is degraded (this is a business decision, not an engineering default — see Step 5)
- **Consistency:** velocity counters (e.g. "3rd transaction on this card in 60 seconds") need **strong, low-latency** consistency — two concurrent transactions on the same card must both see an up-to-date counter, or the velocity check is useless
- **Accuracy trade-off:** false positive (legitimate transaction declined) costs customer trust and revenue; false negative (fraud approved) costs a chargeback — the acceptable ratio between them is a tunable business threshold, not a fixed target
- **Auditability:** every decision must be logged with the model version, feature values, and rule hits that produced it, retained for the compliance-mandated retention period
- **Freshness:** fraud patterns shift quickly (new attack rings, compromised BINs) — features need near-real-time updates; models need frequent (daily/weekly) retraining, not quarterly

---

## Step 2: High-Level Architecture

```
                       Transaction Request
                                │
                       ┌─────────────────┐
                       │ Payment Gateway │
                       └─────────────────┘
                                │
                                  sync call (must respond within auth window)
                                ▼
          ┌───────────────────────────────────────────┐
          │ Fraud Scoring Service                     │
          │                                           │
          │ 1. Feature enrichment                     │
          │    ┌─────────────────────────────────┐    │
          │    │ Online feature store (Redis)    │    │
          │    │  velocity counters, device rep, │    │
          │    │  IP reputation                  │    │
          │    └─────────────────────────────────┘    │
          │ 2. Rules engine (fast filter)             │
          │    blocklist hit → auto-decline           │
          │ 3. ML model score (gradient-boosted /     │
          │    ensemble; circuit breaker guards it)   │
          │ 4. Decision: approve/decline/step-up/hold │
          └───────────────────────────────────────────┘
                                │
                                  decision (sync response) + async event
                   ┌────────────┴─────────────┐
                   │                          │
                   ▼                          ▼
         ┌───────────────────┐      ┌──────────────────┐
         │ Payment Gateway   │      │ Outbox → Kafka   │
         │ (approve/decline) │      │ (decision event) │
         └───────────────────┘      └──────────────────┘
                                              │
           ┌───────────────────────┬──────────┴─────────┐
           │                       │                    │
           ▼                       ▼                    ▼
┌─────────────────────┐    ┌──────────────┐    ┌────────────────┐
│ Case Management     │    │ Ledger/Audit │    │ Feature Store  │
│ (flagged → analyst) │    │ (compliance) │    │ Update (async) │
└─────────────────────┘    └──────────────┘    └────────────────┘
           │
             analyst decision (confirmed fraud / false positive)
           ▼
┌───────────────────────┐
│ Labeled Training Data │
│ → Offline Retraining  │
└───────────────────────┘
```

The synchronous path (gateway → scoring → decision) is on the critical latency path and must stay minimal. Everything downstream of the decision — case management, audit logging, feature-store updates, training-data capture — is **async via the outbox pattern**, so a slow downstream consumer never adds latency to a payment authorization.

---

## Step 3: The Synchronous Scoring Path

```
POST /score
{
  transaction_id, card_id, amount, merchant_id, ip, device_id, timestamp
}

Scoring service logic:
  1. Feature lookup (Redis, ~5-10ms):
     - velocity: count of transactions on this card in last 60s/1h/24h
     - device/IP reputation score (precomputed, async-updated)
     - merchant risk tier (precomputed)
  2. Rules engine (in-process, ~1ms):
     - card_id in blocklist? → DECLINE, reason: "blocklist"
     - velocity > threshold? → DECLINE or STEP_UP, reason: "velocity"
  3. If not auto-decided by rules → ML model inference (~50-100ms):
     - features: velocity + reputation + amount z-score + merchant history + time-of-day
     - output: fraud probability score
  4. Threshold decision:
     score < T_low  → APPROVE
     T_low..T_high  → STEP_UP (OTP) or HOLD for review
     score > T_high → DECLINE
  5. Return decision + reason code
  6. Emit decision event to outbox (same transaction as any local state write)
```

---

## Step 4: Velocity Counters — the Concurrency Problem

```
Problem: two transactions on the SAME card arrive within milliseconds
         (e.g. a stolen card being tested rapidly). If each reads a
         stale velocity count, both may pass the check independently.

Fix: atomic increment-and-check in Redis, not read-then-write:

  INCR velocity:{card_id}:60s   ← atomic
  EXPIRE velocity:{card_id}:60s 60  (sliding window via sorted set for precision)
  IF result > threshold: flag

This must be a single atomic operation — a naive
"GET count, check, SET count+1" has a race window that defeats
the entire point of a velocity check.
```

See [[solution-arch/concepts/rate-limiting]] for the general sliding-window/token-bucket mechanics this borrows from.

---

## Step 5: Fail-Open vs Fail-Closed — a Deliberate Business Decision

```
Scoring service degraded/unavailable. Two options, both defensible:

FAIL-CLOSED (decline/hold everything on scoring outage):
  + No fraud gets through undetected
  − Every legitimate transaction blocked during the outage
  − Availability of the fraud service becomes the availability
    ceiling of the entire payment platform

FAIL-OPEN (approve everything on scoring outage, flag for
async post-hoc review):
  + Payment platform stays available even if fraud scoring is down
  − Fraud exposure during the outage window
  − Requires a fast async re-score once the service recovers
```

Most payment platforms choose **fail-open with async catch-up scoring**, wrapped in a [[solution-arch/patterns/circuit-breaker]]: if the scoring service is unhealthy, the circuit opens, transactions are approved and simultaneously queued for async re-scoring, and an alert fires. [[solution-arch/patterns/bulkhead]] isolates the rules engine and ML model into separate resource pools, so a slow model doesn't also starve the (cheap, high-value) rules-engine checks.

---

## Step 6: Feedback Loop and Retraining

```
Analyst reviews a HOLD case → confirms fraud / clears as legitimate
Chargeback arrives weeks later → confirms fraud that was APPROVED

Both outcomes flow back as labeled examples:
  case_management / chargeback events → outbox → Kafka → data lake

Offline retraining pipeline (daily/weekly):
  Pull labeled data → retrain model → evaluate offline (precision/recall
  at current threshold) → shadow-deploy → compare shadow vs. production
  decisions on live traffic → promote if shadow model wins on the
  business-chosen precision/recall trade-off
```

The chargeback-driven labels are inherently **delayed** (weeks) — this is why the pipeline needs both a fast rules layer (catches known patterns immediately) and a model layer (catches patterns learned from delayed feedback), rather than relying on the model alone.

---

## Step 7: Trade-offs

| Decision | Choice | Rationale |
|---|---|---|
| Synchronous vs async scoring | Synchronous, in the auth path | Fraud decision must gate the transaction, not follow it |
| Degraded-service policy | Fail-open + async re-score + circuit breaker | Payment availability outweighs bounded fraud exposure during rare outages |
| Rules engine + ML, not ML-only | Both, rules first | Rules give deterministic, auditable, instant blocks for known-bad patterns; ML catches novel patterns rules can't enumerate |
| Velocity counters | Redis atomic INCR, not app-level read-modify-write | Eliminates the race window that would defeat the check under concurrent transactions |
| Downstream fan-out (case mgmt, audit, retraining) | Outbox → Kafka, fully async | Keeps every non-decision-critical consumer off the payment authorization latency path |
| Threshold tuning | Two-threshold band (approve/step-up/decline) | Binary approve/decline forces either too many false declines or too much fraud through; a middle band routes ambiguous cases to step-up auth instead |

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]

---
*Tags: #google #fraud-detection #payments #system-design*
