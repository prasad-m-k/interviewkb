# Class Imbalance

**Topic:** [[ml/topics/supervised-learning]], [[ml/topics/model-evaluation]]
**Related:** [[ml/concepts/precision-recall-auc]], [[ml/concepts/loss-functions]]

## What it is
When one class is substantially more frequent than another. Common in real-world problems: fraud (0.1%), medical diagnosis (1-5%), defect detection (<1%). A naive model predicts the majority class and achieves high accuracy but zero utility.

## Why Accuracy Fails
On a 99%/1% binary dataset:
- A model that always predicts "negative" achieves 99% accuracy
- Recall on the positive class: 0%
- This model is useless

Always use **precision, recall, F1, or PR-AUC** for imbalanced classification. See [[ml/concepts/precision-recall-auc]].

## The Full Playbook

### 1. Do Nothing First (establish a baseline)
Run a model with no imbalance handling. Understand the baseline PR-AUC and see whether imbalance is even causing a problem.

### 2. Threshold Tuning (cheapest, often best)
The default threshold of 0.5 is rarely optimal. Adjust based on the business trade-off:
```python
from sklearn.metrics import precision_recall_curve
prec, rec, thresholds = precision_recall_curve(y_true, y_score)
# Pick threshold that maximizes F1 or satisfies business constraint
f1 = 2 * prec * rec / (prec + rec)
best_threshold = thresholds[f1.argmax()]
```

### 3. Class Weights (in-training rebalancing)
```python
# scikit-learn: auto-compute class weights
from sklearn.utils.class_weight import compute_class_weight
weights = compute_class_weight('balanced', classes=[0,1], y=y_train)
# {0: 0.505, 1: 50.5} for 1% positive rate

# XGBoost
xgb_model = XGBClassifier(scale_pos_weight=99)  # neg_count / pos_count

# PyTorch
pos_weight = torch.tensor([neg/pos])
loss = F.binary_cross_entropy_with_logits(logits, targets, pos_weight=pos_weight)
```

### 4. Resampling

#### Oversampling the minority class
- **Random oversampling**: duplicate minority examples → risk of overfitting to exact duplicates
- **SMOTE (Synthetic Minority Oversampling Technique)**: create synthetic examples by interpolating between minority class neighbors
  ```
  new_example = x + λ · (x_neighbor - x)    λ ~ Uniform(0, 1)
  ```
  - Better than random oversampling; creates diverse examples
  - ADASYN: SMOTE variant that focuses on hard-to-classify minority examples

#### Undersampling the majority class
- **Random undersampling**: discard majority examples → loses information
- **Tomek Links**: remove majority examples that are close to the boundary
- **NearMiss**: undersample majority examples nearest to minority examples

#### Combined
- **SMOTEENN**: SMOTE + ENN (edited nearest neighbor to clean noise)

### 5. Focal Loss (for neural networks)
Down-weight easy examples to focus training on hard ones:
```
L = -αₜ · (1 - pₜ)^γ · log(pₜ)
```
See [[ml/concepts/loss-functions]].

### 6. Anomaly Detection Framing
If positives are extremely rare (<0.1%), treat as anomaly detection:
- Train only on negative (majority) class
- At inference, flag examples with high reconstruction error (autoencoder) or low density (Isolation Forest)
- Avoids the need for positive labels during training

### 7. Stratified Sampling
Always use stratified splits and stratified k-fold — ensures each fold has the same positive rate as the full dataset. See [[ml/concepts/cross-validation]].

## Decision Guide

| Ratio | Strategy |
|---|---|
| 1:2 to 1:5 | Usually fine; try class weights |
| 1:10 to 1:50 | Class weights + threshold tuning; try SMOTE |
| 1:100 to 1:1000 | SMOTE + focal loss + PR-AUC evaluation |
| > 1:1000 | Consider anomaly detection framing |

## Common interview angles
- Why is accuracy a bad metric for imbalanced classification? (majority-class baseline achieves high accuracy; use F1/PR-AUC instead)
- SMOTE — how does it work and what are its limitations? (generates synthetic minority examples via interpolation; limitation: doesn't work well for high-dimensional data; may generate noisy examples in feature spaces with complex structure)
- Class weights vs. oversampling — when to use which? (class weights are simpler, no data augmentation; oversampling can improve on models that don't support class weights; for deep learning, focal loss > either)
- What is the PR-AUC and why is it better than ROC-AUC for imbalanced data? (PR-AUC focuses on positive class performance; ROC-AUC can look good even when positive recall is poor because TN rate dominates the FPR denominator)
- How would you handle a 0.01% positive rate in production? (anomaly detection framing; or train with heavy oversampling + class weights; evaluate with very low recall thresholds; monitor false positive rate carefully)

## Sources
- [[ML overview]]
