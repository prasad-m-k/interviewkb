# Scenario: Low Accuracy Debugging

**Category:** Model Debugging / Error Analysis
**Difficulty:** Medium-Hard
**Seen at:** Google, Meta, Amazon (applied scientist interviews)

## The Scenario
Your model achieves only 68% accuracy on the test set but you expected 85%+. The model is not overfitting (training accuracy is also 70%). Walk through your debug methodology.

## High-Level Framework

```
Low performance breaks down into two cases:
1. High bias (underfitting): train ≈ val ≈ low
   → Model can't represent the task
2. High variance (overfitting): train high, val low
   → This scenario is the other one
   → See [[ml/scenarios/overfitting-diagnosis]]
```

Since train ≈ val ≈ 70%, this is a **high bias / underfitting** problem.

## Step 1: Error Analysis — What Does the Model Get Wrong?

```python
from sklearn.metrics import confusion_matrix, classification_report
import seaborn as sns

y_pred = model.predict(X_test)

# Confusion matrix
cm = confusion_matrix(y_test, y_pred)
sns.heatmap(cm, annot=True, fmt='d')

# Per-class breakdown
print(classification_report(y_test, y_pred, target_names=class_names))

# Sample wrong predictions — READ THEM
mistakes = X_test[y_test != y_pred]
for i, (x, y_true, y_hat) in enumerate(zip(mistakes[:20], y_test[y_test!=y_pred], y_pred[y_test!=y_pred])):
    print(f"True: {y_true}, Predicted: {y_hat}")
    print(f"Example: {x}")
```

**Reading wrong examples is the highest-leverage debug activity.** Common patterns:
- Model confuses semantically similar classes (cat vs. dog)
- Certain subgroups perform differently (short sentences vs. long)
- Ambiguous or mislabeled examples
- Missing features (model doesn't have access to decisive information)

## Step 2: Check for the Root Causes

### 2a. Wrong or Insufficient Features
```python
# Mutual information between each feature and target
from sklearn.feature_selection import mutual_info_classif
mi = mutual_info_classif(X_train, y_train)
# If all MI values are very low → features don't contain predictive signal
```
Ask: does a human with access to these features correctly classify this example? If no → feature problem.

### 2b. Label Quality
```python
# Sample 100 examples from error cases; manually inspect labels
# Look for labeling inconsistencies, ambiguities
# If ~20%+ of "errors" are actually mislabeled → label noise is the problem
```
Label noise effectively sets a ceiling on achievable accuracy. Fix: re-label with higher quality, or use noise-robust loss functions.

### 2c. Model Too Simple
```python
# Progressive model complexity experiment
for depth in [2, 4, 8, None]:
    model = DecisionTreeClassifier(max_depth=depth)
    cv_score = cross_val_score(model, X, y, cv=5).mean()
    print(f"max_depth={depth}: val accuracy={cv_score:.3f}")
# If accuracy keeps increasing as depth grows → model too simple
```

### 2d. Training Distribution ≠ Test Distribution (Covariate Shift)
```python
# Train a binary classifier: "is this example from train or test?"
# If it can discriminate → distributions differ
X_combined = np.vstack([X_train, X_test])
y_dist = np.array([0]*len(X_train) + [1]*len(X_test))
dist_score = cross_val_score(LogisticRegression(), X_combined, y_dist, cv=5).mean()
print(f"Train/test discriminability: {dist_score:.3f}")
# Score > 0.6 → significant distribution shift
```

### 2e. Wrong Evaluation (Measurement Error)
```python
# Sanity check: what is the baseline (majority class predictor)?
from sklearn.dummy import DummyClassifier
baseline = DummyClassifier(strategy='most_frequent')
baseline_score = cross_val_score(baseline, X, y, cv=5).mean()
print(f"Majority class baseline: {baseline_score:.3f}")
# Is your model actually doing better than this baseline?
```

## Step 3: Fixes by Root Cause

| Root Cause | Fix |
|---|---|
| Features don't contain signal | Feature engineering; find new data sources |
| Model too simple | Larger model; more capacity; better architecture |
| Label noise | Re-label; noise-robust loss (symmetric cross-entropy) |
| Distribution shift | Collect data from target distribution; domain adaptation |
| Class imbalance | See [[ml/scenarios/class-imbalance-handling]] |
| Too little data | Collect more; use transfer learning; data augmentation |
| Wrong metric / task framing | Re-examine the problem definition |

## Step 4: Ablation Studies
Isolate which component is causing the drop:
```python
# Systematically remove features or components and measure impact
results = {}
for feature_set in [all_features, without_temporal, without_text, tabular_only]:
    score = cross_val_score(model, X[feature_set], y, cv=5).mean()
    results[str(feature_set)] = score
# Feature set with biggest accuracy drop = most important
```

## Sources
- [[ml/concepts/bias-variance-tradeoff]]
- [[ml/concepts/class-imbalance]]
- [[ml/topics/model-evaluation]]
- [[ML overview]]
