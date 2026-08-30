# Google Ads CTR Prediction System

**Difficulty:** Hard
**Topic:** [[ml/topics/ml-system-design]]
**Pattern:** Real-time inference, two-stage pipeline
**Companies:** [[ml/companies/google-ml]]

---

## Problem

Design the click-through rate (CTR) prediction system for Google Search Ads. Given a search query, user context, and candidate ads, predict the probability that a user will click each ad. The auction system uses this score to rank ads and determine pricing.

---

## Clarifying Questions

- Is CTR the only signal, or are we also predicting conversion?
- What is the latency budget? (ads must serve within the search response SLA: < 100ms total, CTR model < 10ms)
- How many ads are candidates per query? (typically 3-10 final ads from thousands of candidates)
- Is this for search ads only, or also display/YouTube?

---

## Why CTR Prediction is Hard at Google Scale

- **Scale:** ~99,000 searches/second; each triggers an ad auction
- **Latency:** CTR model must score candidates in < 10ms within the overall search latency budget
- **Sparse signals:** most (user, ad) pairs have never been seen before
- **Position bias:** ads shown higher have higher CTR regardless of relevance
- **Label noise:** click ≠ conversion; accidental clicks pollute labels
- **Feedback loop:** ads predicted to have high CTR get shown more → more clicks → self-reinforcing
- **Business stakes:** Google's primary revenue source; 1% CTR improvement = billions of dollars

---

## System Architecture

```
Search Query arrives
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│   Ad Retrieval (100ms budget starts)                    │
│   • Query → Ad index (BM25 + semantic match)            │
│   • Retrieve 1000–5000 candidate ads                    │
└─────────────────────────────────────────────────────────┘
      │ ~1000-5000 candidates
      ▼
┌─────────────────────────────────────────────────────────┐
│   Lightweight Filtering / Pre-scoring                   │
│   • Budget check (advertiser has active budget?)        │
│   • Geo/demographic targeting match                     │
│   • Simple logistic regression score (10-100 features)  │
│   • Reduces candidates to ~50                           │
└─────────────────────────────────────────────────────────┘
      │ ~50 candidates
      ▼
┌─────────────────────────────────────────────────────────┐
│   Full CTR Model Scoring                                │
│   • Deep neural network (Wide & Deep / DCN)             │
│   • Rich features: user, query, ad, context, cross      │
│   • Output: P(click | user, query, ad, context)         │
│   • Budget: < 5ms for 50 candidates                     │
└─────────────────────────────────────────────────────────┘
      │ CTR scores
      ▼
┌─────────────────────────────────────────────────────────┐
│   Auction                                               │
│   • Ad Rank = CTR × Quality Score × Bid                │
│   • Select top 3-10 ads                                 │
│   • Price = Vickrey auction (second-price)              │
└─────────────────────────────────────────────────────────┘
```

---

## Model: Wide & Deep (Google's Architecture)

Published by Google 2016 (Cheng et al.). Still the reference architecture for ads CTR.

**Wide component:** linear model over cross-product features ("user searched 'running shoes' AND ad is for 'Nike shoes'"). Captures memorization of known co-occurrences.

**Deep component:** sparse features → dense embeddings → MLP. Captures generalization to unseen combinations.

```
Input:
  - Categorical: user_id, ad_id, query_tokens, advertiser_id
  - Dense: historical CTR, user age, bid amount
  - Cross features: query × ad_category

Wide:
  y_wide = w × [raw_features, cross_product_features]

Deep:
  embeddings = Lookup(categorical_features)  # (N, D)
  h = concat([embeddings, dense_features])
  for layer in MLP_layers:
      h = ReLU(W @ h + b)
  y_deep = h

Final:
  y = sigmoid(y_wide + y_deep)  # P(click)
```

**Modern variant: DCN (Deep & Cross Network):** replaces the wide component with an explicit cross network that learns all-order feature interactions efficiently. DCN v2 is the current standard at Google.

---

## Features

| Feature Group | Examples | Challenge |
|---|---|---|
| Query features | Query tokens, query category, query intent | Sparse; billions of queries |
| User features | Search history, demographics, device, location | Privacy-sensitive; must anonymize |
| Ad features | Ad text, landing page, advertiser, category | Change frequently |
| Context | Time of day, day of week, country, device | Easy to compute |
| Cross features | Query × ad_category, user_interest × ad_topic | Explosion of feature space |
| Historical | User's CTR on this advertiser, ad's base CTR | High signal; cold start issue |

**Feature freshness:** CTR models need real-time user context. Feature store serves pre-computed embeddings with < 1ms latency. Raw features like "user just searched X 10 minutes ago" require real-time streaming pipelines.

---

## Handling Position Bias

Ads shown in position 1 get 5-10× more clicks than position 5, regardless of quality. If you train on raw clicks, the model learns position as a proxy for quality.

**Solutions:**
1. **Position as a training feature, zeroed at inference:** include position_id as a feature during training. At inference time, set position = 0 (removing the bias while keeping the model's learned correlation).
2. **Inverse Propensity Scoring (IPS):** weight each training example by 1/P(shown in position k). Denoises labels but increases variance.
3. **Examination model:** factorize CTR into `P(click) = P(examine | position) × P(click | examine, query, ad)`. Train the examination probability separately.

---

## Evaluation

**Offline:**
- **Log-loss (cross-entropy):** primary metric for calibration
- **AUC-ROC:** ranking quality (does the model rank clicks above non-clicks?)
- **Calibration curve:** P(click) must be well-calibrated; the auction multiplies CTR × bid, so an overconfident CTR prediction directly inflates prices

**Online A/B:**
- Primary: Revenue Per Search (RPS), CTR on served ads
- Guardrail: user click satisfaction, advertiser ROI

---

## Cold Start

| Scenario | Solution |
|---|---|
| New advertiser | Default to category average CTR; boost with quality score |
| New ad creative | Estimate from similar ads (same advertiser, category, token overlap) |
| New user | Use population average; real-time features (current query) dominate anyway |

---

## Training Infrastructure

- **Data volume:** hundreds of billions of (query, ad, user, click) log rows per day
- **Training frequency:** online learning preferred; mini-batch updates every few minutes using streaming logs
- **Online learning risk:** label delay (conversion may happen hours after click); incomplete labels cause underestimating CTR initially
- **Distributed training:** parameter server architecture for embedding tables (billions of unique IDs); gradient accumulation via AllReduce for dense layers

---

## Responsible AI Considerations (Google-specific)

Google expects you to raise these even if not asked:
- **Demographic parity in ads:** should women see fewer tech ads because of historical click patterns? Fairness constraints must be explicit.
- **Privacy:** user click history is PII. Training data must be aggregated/anonymized. Federated learning options for on-device.
- **Ad quality:** CTR × bid can surface misleading ads if quality score doesn't penalize them. Human review pipeline for flagged advertisers.

---

## Key Insight

CTR prediction is a calibration problem, not just a ranking problem. The auction multiplies your predicted probability by the advertiser's bid — if your model is systematically over-confident, advertisers overpay; under-confident, Google under-earns. Calibration evaluation is mandatory.

---

## Sources
- [[ml/topics/ml-system-design]]
- [[ml/companies/google-ml]]
- [[mlops/concepts/feature-store]]
- [[mlops/concepts/training-serving-skew]]
- [[ml/concepts/class-imbalance]]
