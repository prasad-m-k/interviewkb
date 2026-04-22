# Scenario: Fraud Detection System Design

**Category:** ML System Design
**Difficulty:** Hard
**Seen at:** Stripe, PayPal, Square, Amazon, Google, Airbnb

## The Problem
Design a real-time fraud detection system for a payment platform. Flag fraudulent transactions before they are approved (< 50ms latency) or block them after (< 200ms).

## Framework

### 1. Requirements
- **Objective**: maximize fraud caught (recall) while keeping false positive rate low (precision)
- **Latency**: < 100ms for real-time blocking; < 500ms for review queue
- **Scale**: 10K+ transactions per second
- **Data**: severe class imbalance (0.1–1% fraud)
- **Constraint**: cannot use ground truth labels in real time (labels come hours/days later)

### 2. Data and Labels

**The label delay problem:** True fraud is often not confirmed for hours/days (chargebacks, investigation). Real-time scoring uses features only — labels arrive later.

| Label source | Delay | Quality |
|---|---|---|
| Chargeback initiated | 1-30 days | High; user-confirmed fraud |
| Fraud reported by user | Hours | High |
| Rule-based catches | Real-time | Medium; may have false positives |
| Model re-label (from audit) | Weeks | Highest |

**Survivorship bias:** Transactions blocked by current rules/model never get labels → training data misses "hard negatives" blocked by the old system. Fix: block a fraction of suspicious transactions with a randomized 5% fallthrough to gather labels.

### 3. Architecture: Multi-Layer Defense

```
Transaction arrives
  │
  ▼
┌─────────────────────────────────────┐
│  Layer 1: HARD RULES (< 1ms)        │  Block obvious fraud: known bad IPs,
│  Blocklist, velocity checks         │  >5 failed cards in 1 min
└─────────────────────────────────────┘
  │ passes
  ▼
┌─────────────────────────────────────┐
│  Layer 2: FAST ML MODEL (< 20ms)    │  Lightweight model; real-time features
│  Logistic Regression / XGBoost      │  from feature store
│  Low-latency scoring                │
└─────────────────────────────────────┘
  │ score > high threshold → block
  │ score in review range → queue
  ▼
┌─────────────────────────────────────┐
│  Layer 3: DEEP MODEL (< 100ms)      │  Graph + sequence + entity features
│  GNN or sequence model              │  More features; runs async or in-band
│  + manual review queue              │  based on risk score
└─────────────────────────────────────┘
```

### 4. Feature Engineering for Fraud

**Velocity features** (most important):
```python
{
  'num_cards_used_1h': count of distinct cards from this device in 1 hour,
  'num_txn_1h': transaction count from this user in 1 hour,
  'total_amount_24h': sum of transaction amounts in 24 hours,
  'num_failed_1h': count of declined transactions in 1 hour,
  'new_merchant_flag': first time user transacting with this merchant,
  'device_age_days': how old is this device fingerprint,
  'distance_from_last_txn_km': physical distance from last transaction,
}
```

**Identity and context:**
- IP address reputation (proxy/VPN flag)
- Device fingerprint (browser, OS, resolution)
- Billing address match to shipping address
- Time of day / day of week (fraud spikes at specific times)
- Merchant risk category

**Graph features:**
- Connected components: does this card share a device with known fraud cards?
- Shared address or email across multiple flagged accounts
- GNN can score the entire neighborhood simultaneously

### 5. Model Architecture

**Real-time model (XGBoost, ~5ms latency):**
- ~100 pre-computed features from feature store (velocity, identity, context)
- Threshold tuned for business constraint (e.g., catch 80% of fraud at < 0.5% false positive rate)

**Graph Neural Network (for connected fraud rings):**
```
Nodes: transactions, cards, devices, merchants, users
Edges: "same device", "same IP", "same email domain", "same billing address"
```
- Fraudsters reuse infrastructure → creates graph clusters
- GNN aggregates neighborhood signals: one fraudulent node raises risk of connected nodes
- Slower (200ms); used for async risk enrichment or review queue

### 6. Handling Class Imbalance (critical for fraud)
See [[ml/scenarios/class-imbalance-handling]].
- `scale_pos_weight` in XGBoost
- Focal loss for neural models
- Business-driven threshold: set threshold to satisfy P(fraud blocked | flagged) requirement

### 7. Feedback Loop Challenge
Model predicts → blocks fraud → no label for blocked transactions. Training data is biased.

**Fixes:**
1. **Randomized exploration**: pass 1-5% of high-risk transactions through with monitoring (accept and log)
2. **Reject inference**: estimate labels for rejected transactions using domain heuristics
3. **Champion-challenger**: run new model on small traffic slice to gather fresh labeled data

### 8. Evaluation

**Offline:**
- PR-AUC (not ROC-AUC — imbalanced)
- Recall@Precision=X%: "at what recall can we operate at 90% precision?"
- Dollar amount of fraud caught vs. dollar amount of false positives (business metric)

**Online:**
- Fraud rate ($ fraud / $ total)
- False positive rate (legitimate transactions blocked / total transactions)
- Latency P99

### 9. Monitoring
- **Score distribution drift** → may indicate new fraud patterns
- **Feature drift** (velocity distributions change) → may need retraining
- **Precision/recall monitoring** on labeled subset
- **Alert on sudden spike in block rate** → possible model malfunction or fraud campaign

## Sources
- [[ml/topics/ml-system-design]]
- [[ml/concepts/class-imbalance]]
- [[ml/concepts/precision-recall-auc]]
- [[ML overview]]
