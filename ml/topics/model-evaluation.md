# Model Evaluation

**Related concepts:** [[ml/concepts/cross-validation]], [[ml/concepts/precision-recall-auc]], [[ml/concepts/class-imbalance]], [[ml/concepts/bias-variance-tradeoff]]

## The Evaluation Hierarchy

```
1. Offline metrics  →  2. Online A/B test  →  3. Business KPIs
        ↑                       ↑                      ↑
  Fast, cheap            Ground truth           What matters
  Proxy for              for model              Ultimate goal
  real behavior          quality
```

Never ship based on offline metrics alone. Always validate with online testing.

## Classification Metrics

### The Confusion Matrix
```
                  Predicted Positive    Predicted Negative
Actual Positive        TP                    FN
Actual Negative        FP                    TN
```

| Metric | Formula | When to use |
|---|---|---|
| **Accuracy** | (TP+TN) / All | Balanced classes; rarely the right metric |
| **Precision** | TP / (TP+FP) | Cost of false positives is high (spam, fraud alerts) |
| **Recall** | TP / (TP+FN) | Cost of false negatives is high (cancer screening, fraud) |
| **F1** | 2·P·R / (P+R) | Imbalanced classes; need balance of P and R |
| **F-beta** | (1+β²)·P·R / (β²·P+R) | β>1 weights recall more; β<1 weights precision more |

### AUC Metrics
- **ROC-AUC**: area under TPR vs FPR curve; threshold-independent; equals P(score(positive) > score(negative)); not useful for severe imbalance
- **PR-AUC (Average Precision)**: area under Precision-Recall curve; better for imbalanced data; tells you how well the model ranks positives
- **Rule**: use ROC-AUC when negatives matter; use PR-AUC when positives are rare

### Log Loss (cross-entropy)
- Penalizes confident wrong predictions heavily
- Use when you need calibrated probabilities, not just rankings

## Regression Metrics
| Metric | Formula | Use when |
|---|---|---|
| **MSE** | mean((y - ŷ)²) | Outliers should be penalized heavily |
| **RMSE** | √MSE | Same scale as target; interpretable |
| **MAE** | mean(\|y - ŷ\|) | Outliers should not dominate |
| **R²** | 1 - SS_res/SS_tot | Explained variance; 1 = perfect, 0 = baseline mean |
| **MAPE** | mean(\|y - ŷ\|/y) | Percentage error; bad when y≈0 |
| **Huber** | MSE if \|e\|<δ, MAE otherwise | Robust: outlier-tolerant but smooth near 0 |

## Ranking Metrics
| Metric | What it measures |
|---|---|
| **NDCG@k** | Normalized Discounted Cumulative Gain; relevance of top-k results with position discount |
| **MAP** | Mean Average Precision; quality of ranked list of binary-relevant items |
| **MRR** | Mean Reciprocal Rank; rank of first correct answer (useful for QA) |
| **Hit Rate@k** | Fraction of queries where the correct item is in top-k |

## Validation Strategies

### Standard k-Fold
Split data into k folds; train on k-1, evaluate on held-out fold; repeat k times; average metrics. Use stratified k-fold for classification to maintain class ratio in each fold.

### Time-Series Split (Walk-Forward)
```
Train: [1..t]      Val: [t+1..t+h]
Train: [1..t+h]    Val: [t+h+1..t+2h]
...
```
Never use random split on time-series — it leaks future into training.

### Nested Cross-Validation
Outer loop: honest performance estimate. Inner loop: hyperparameter tuning. Prevents optimistic bias from tuning on the same data you evaluate on.

## A/B Testing
- **Split**: route X% of traffic to new model, (100-X)% to baseline
- **Metrics**: primary (business KPI) + secondary (guardrails — don't regress)
- **Statistical significance**: p < 0.05; always compute sample size needed before starting
- **Novelty effect**: users click more on new things even if they're worse; run for ≥ 2 weeks
- **Interference**: if users interact with each other (social graph), A/B bias occurs → use cluster-level assignment

## Common Interview Angles
- Why is accuracy bad for imbalanced classes? (dominated by majority; model that predicts all-negative gets 99% accuracy on 1% positive dataset)
- When would you prefer PR-AUC over ROC-AUC? (rare positive events; fraud, medical diagnosis)
- What is data leakage and how does it inflate metrics? (features computed using future data or target encode using full dataset; fix with proper pipeline)
- How do you evaluate a ranking model offline? (NDCG, MAP — but these require relevance judgments; implicit feedback via clicks has position bias)
- What is calibration? (P(y=1 | score=0.8) should be ~0.8; use Platt scaling or isotonic regression to calibrate)

## Sources
- [[ML overview]]
