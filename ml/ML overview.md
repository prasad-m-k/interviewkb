# ML Knowledge Base — Overview

## What This Wiki Covers
Interview preparation for Machine Learning roles: ML Engineer, Applied Scientist, Research Engineer, and Data Scientist. Covers foundational concepts, deep learning, NLP, system design, and practical debugging scenarios.

---

## Mental Model for ML Interviews

### The Three Layers Interviewers Probe
1. **Conceptual depth** — can you explain *why* something works, not just *what* it is?
2. **Practical judgment** — given this problem, which model/metric/approach would you use and why?
3. **System thinking** — how does this scale? What breaks in production?

### The Three Types of ML Interview Questions
| Type | Example | How to prepare |
|---|---|---|
| **Concept/Theory** | "Explain bias-variance tradeoff" | Know the mechanism, not the definition |
| **Scenario/Debug** | "Your model's val loss diverges at epoch 10. What do you check?" | Build systematic debug checklists |
| **System Design** | "Design a recommendation system for 100M users" | Learn the 2-stage pipeline pattern |

---

## Key Themes Across the Wiki

### 1. Bias-Variance is Everywhere
Every ML decision — model complexity, regularization, data size, ensemble strategy — is a bias-variance trade-off. Frame your answers through this lens.

### 2. Metrics Must Match the Business Problem
- Classification with class imbalance → don't use accuracy; use F1 or PR-AUC
- Ranking problems → NDCG, MAP
- Regression → RMSE if outliers matter, MAE if they don't
- Always ask: what does a false positive cost vs. a false negative?

### 3. Overfitting is the Most Common Failure Mode
And it has multiple root causes: too little data, too complex a model, data leakage, no regularization, train/test distribution mismatch. Know all five.

### 4. Neural Networks Fail in Systematic Ways
Loss explosions, vanishing gradients, slow convergence, and poor generalization each have specific causes and specific fixes. Interviewers test whether you can diagnose, not just define.

### 5. Production ML ≠ Notebook ML
Feature stores, training-serving skew, data drift, feedback loops — the system around the model is often harder than the model itself. See [[ml/topics/ml-system-design]].

---

## Top 10 Concepts to Know Cold
1. [[ml/concepts/bias-variance-tradeoff]] — the core framework
2. [[ml/concepts/gradient-descent]] — how models learn
3. [[ml/concepts/regularization]] — overfitting prevention
4. [[ml/concepts/cross-validation]] — honest evaluation
5. [[ml/concepts/precision-recall-auc]] — the right metric for the job
6. [[ml/concepts/attention-mechanism]] — foundation of modern NLP
7. [[ml/concepts/transformers]] — architecture you must know in 2024+
8. [[ml/concepts/ensemble-methods]] — trees: Random Forest vs. XGBoost
9. [[ml/concepts/class-imbalance]] — the practical problem most datasets have
10. [[ml/concepts/batch-normalization]] — training stability

---

## Top System Design Questions
1. Recommendation system (Netflix, YouTube, Amazon) → [[ml/scenarios/recommendation-system-design]]
2. Search ranking (Google, LinkedIn) → [[ml/scenarios/search-ranking-design]]
3. Fraud/anomaly detection (Stripe, PayPal) → [[ml/scenarios/fraud-detection-design]]
4. Feed ranking (Instagram, Twitter) — similar to recommendation
5. Ads prediction (CTR, pCVR) — variant of ranking with business constraints

---

## Interview Strategy
- **Lead with the framework, then fill in details.** For system design: requirements → data → model → evaluation → serving → monitoring.
- **State trade-offs explicitly.** "I would use X over Y here because [trade-off], unless [condition]."
- **Connect concepts.** "Dropout is regularization, and regularization shifts the model toward higher bias / lower variance."
- **Reference numbers.** "BERT-base has 110M parameters, training needs ~16 A100-hours, inference latency ~20ms on CPU."
