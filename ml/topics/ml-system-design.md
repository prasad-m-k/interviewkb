# ML System Design

**Related scenarios:** [[ml/scenarios/recommendation-system-design]], [[ml/scenarios/search-ranking-design]], [[ml/scenarios/fraud-detection-design]]

## The ML System Design Framework

Use this structure for every ML system design interview question:

```
1. Requirements clarification (5 min)
2. Data (10 min)
3. Model / Algorithm (10 min)
4. Training pipeline (5 min)
5. Serving & infrastructure (5 min)
6. Evaluation — offline + online (5 min)
7. Monitoring & failure modes (5 min)
```

### 1. Requirements
- **Business objective**: what KPI does this system optimize? (CTR, revenue, safety)
- **Functional requirements**: what does the system output? (ranked list, probability score, generated text)
- **Constraints**: latency (p99), QPS, freshness, privacy/regulatory

### 2. Data
- What data is available? Volume? Labels?
- How are labels collected? (explicit ratings, implicit clicks, manual annotation)
- What is the label distribution? (imbalanced? → see [[ml/concepts/class-imbalance]])
- Label noise? (click-through ≠ conversion)
- **Training-serving skew** risk: are features computed the same way offline and online?

### 3. Model / Algorithm
Apply the decision framework:
- Tabular, low-latency → GBM (XGBoost/LightGBM)
- Sequence, NLP → Transformer
- Recommendation, large catalog → Two-tower neural network → ANN retrieval
- Real-time scoring with simple features → Logistic regression (fast, explainable)

### 4. Training Pipeline
- **Feature store**: point-in-time correct features; avoid leakage; online (low-latency) + offline (batch)
- **Training frequency**: batch re-train (daily/weekly) vs. online learning vs. streaming
- **Data splits**: time-based splits for anything temporal; stratified for class balance

### 5. Serving & Infrastructure
- **Batch scoring**: compute scores for all users nightly; serve from lookup table (low latency, stale)
- **Real-time scoring**: live inference on request (fresh, higher latency)
- **Two-stage pattern** (retrieval + ranking):
  - Stage 1: retrieve ~100-1000 candidates from millions (ANN, BM25, collaborative filter)
  - Stage 2: rank candidates with a heavier model (GBM or neural ranker)
  - Why: heavy ranker can't score every item; retrieval is the scalability lever

### 6. Evaluation
- **Offline**: AUC-ROC, NDCG@k, MAP, calibration
- **Online A/B**: primary business metric + guardrail metrics
- **Shadow mode**: run new model in parallel, log predictions, no traffic impact

### 7. Monitoring
- **Data drift**: input feature distributions shift → model performance degrades silently
- **Concept drift**: relationship between features and labels changes (e.g., user behavior during COVID)
- **Model staleness**: model trained on old data; scheduled re-training is the standard fix
- **Feedback loop**: model predictions influence future training data (recommendation → users only see recommended items → label bias)

## Common ML System Design Interview Questions
1. Design a recommendation system for a streaming service → [[ml/scenarios/recommendation-system-design]]
2. Design a spam/fraud detection system → [[ml/scenarios/fraud-detection-design]]
3. Design a search ranking system → [[ml/scenarios/search-ranking-design]]
4. Design a news feed ranking system (similar to recommendation)
5. Design an ad click-through rate prediction system (label: click; challenge: sparse labels, position bias)
6. Design a question-answering system over a document corpus (RAG pattern)

## Key Concepts to Know for ML System Design
- **Feature store** (online + offline) — [[mlops/concepts/feature-store]]
- **Training-serving skew** — [[mlops/concepts/training-serving-skew]]
- **Data drift** — [[mlops/concepts/data-drift]]
- **Two-tower retrieval** — dense embedding + ANN; basis for most modern recommendation
- **Position bias** — items shown higher get more clicks regardless of quality; correct with IPS or position feature
- **Exploration vs. exploitation** — ε-greedy, UCB, Thompson sampling for bandit problems
- **Cold start** — new user/item has no history; fall back to popularity, content-based, or metadata features

## Sources
- [[ML overview]]
- [[mlops/overview]]
