# Precision, Recall, and AUC

**Topic:** [[ml/topics/model-evaluation]]
**Related:** [[ml/concepts/class-imbalance]], [[ml/concepts/loss-functions]], [[ml/concepts/cross-validation]]

## What it is
A family of metrics for evaluating binary classifiers that go beyond accuracy. Especially important for imbalanced datasets and problems where the costs of false positives and false negatives differ.

## The Confusion Matrix
```
                  Predicted Positive    Predicted Negative
Actual Positive        TP                    FN
Actual Negative        FP                    TN
```

## Core Metrics

### Precision
```
Precision = TP / (TP + FP)
```
"Of all the examples I predicted positive, what fraction are actually positive?"
High precision → few false alarms. Use when **false positives are costly** (spam filters, medical screening alerts).

### Recall (Sensitivity, True Positive Rate)
```
Recall = TP / (TP + FN)
```
"Of all the actual positives, what fraction did I catch?"
High recall → few missed positives. Use when **false negatives are costly** (cancer detection, fraud prevention).

### F1 Score
```
F1 = 2 · Precision · Recall / (Precision + Recall)
```
Harmonic mean of precision and recall. Good single metric when you need balance between P and R. The harmonic mean penalizes extreme imbalance between the two.

### F-beta
```
Fβ = (1 + β²) · P · R / (β²·P + R)
```
- β > 1: weights recall more (β=2: recall twice as important as precision)
- β < 1: weights precision more
- Use when one error type is more costly than the other

### Specificity (True Negative Rate)
```
Specificity = TN / (TN + FP)
```
"Of all the actual negatives, what fraction did I correctly identify?" Used in medical tests (complementary to recall).

## ROC Curve and AUC

**ROC curve:** plot TPR (recall) vs. FPR (= FP / (FP+TN)) across all classification thresholds.

```
TPR
 1 |        _____
   |      /
   |    /
   |  /
   | /
 0 |___________
   0           1
              FPR
```

**AUC-ROC** (Area Under the Curve):
- 1.0: perfect classifier
- 0.5: random classifier (diagonal line)
- Equals P(score(positive example) > score(negative example))
- **Threshold-independent**: evaluates the ranking, not a specific decision

**Limitation of ROC-AUC for imbalanced data:** FPR denominator contains TN, which can be very large with many negatives → FPR is always tiny even with many false positives → ROC looks optimistic.

## PR Curve and PR-AUC

**PR curve:** plot Precision vs. Recall across all thresholds.

```
Precision
  1 |___
    |    \
    |     \___
    |          \___
  0 |____________
    0             1
                Recall
```

**PR-AUC** (Average Precision):
- Better for **imbalanced datasets** — focuses on positive class performance
- High PR-AUC means the model can achieve both high precision and high recall simultaneously
- If the positive class is rare, a good model should still rank positives high → high PR-AUC

**Rule:** Use ROC-AUC when both classes matter equally. Use PR-AUC when the positive class is rare or when false positives and false negatives have very different costs.

## Calibration
A model is calibrated if predicted probabilities reflect true frequencies:
- If your model predicts p=0.8 for 100 examples, ~80 of them should actually be positive

**Check:** reliability diagram / calibration plot (predicted probability vs. actual fraction positive)

**Fix:**
- **Platt scaling**: fit a logistic regression on model scores
- **Isotonic regression**: fit a non-decreasing step function on model scores
- **Temperature scaling**: divide logits by a learned temperature T

## Choosing the Right Metric for the Problem

| Problem | Recommended Metric | Why |
|---|---|---|
| Fraud detection | PR-AUC | Rare positives; missing fraud is costly |
| Spam filter | Precision-first | False positives (blocking legitimate mail) are unacceptable |
| Cancer screening | Recall-first | False negatives (missing cancer) are catastrophic |
| Ad CTR prediction | Log loss + AUC | Need calibrated probabilities for auction |
| Recommendation | NDCG@k, Hit Rate@k | Ranking quality matters |
| Balanced classification | Accuracy or F1 | Both classes equally important |

## Common interview angles
- Why is F1 the harmonic mean and not the arithmetic mean? (arithmetic mean can be high even if one metric is very poor; harmonic mean is more sensitive to low values — penalizes extreme imbalance)
- ROC-AUC vs. PR-AUC — when does each mislead you? (ROC-AUC misleads on imbalanced data — high TN rate inflates AUC; PR-AUC is honest about positive class performance)
- What does AUC of 0.5 mean? (random classifier; model has no discriminative power)
- A model achieves 99% accuracy on a 1% positive rate dataset. Is this a good model? (almost certainly not — a model predicting all negatives achieves the same; check recall and precision on the positive class)
- What is calibration and why does it matter for business decisions? (a model that predicts 0.9 but is correct only 60% of the time will lead to wrong decisions in systems that use probabilities for thresholding or ranking)

## Sources
- [[ML overview]]
