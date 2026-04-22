# Scenario: Overfitting Diagnosis and Remediation

**Category:** Model Debugging
**Difficulty:** Medium
**Seen at:** Every ML interview

## The Scenario
Your model achieves 97% training accuracy but only 72% validation accuracy. The training loss keeps decreasing past epoch 15 but the validation loss starts climbing. You need to diagnose and fix it.

## Step 1: Confirm It's Overfitting (Not Leakage or a Bug)

```python
# Check 1: Are train and val on the same distribution?
print("Train label distribution:", pd.Series(y_train).value_counts(normalize=True))
print("Val label distribution:  ", pd.Series(y_val).value_counts(normalize=True))

# Check 2: Are any validation examples identical to training examples?
# (data duplication bug)
train_set = set(map(tuple, X_train.tolist()))
leaks = [i for i, x in enumerate(X_val.tolist()) if tuple(x) in train_set]
print(f"Val examples found in train: {len(leaks)}")

# Check 3: Plot loss curves
import matplotlib.pyplot as plt
plt.plot(train_losses, label='train')
plt.plot(val_losses, label='val')
plt.axvline(best_epoch, linestyle='--', label='best epoch')
```

## Step 2: Quantify the Gap

| Train | Val | Gap | Interpretation |
|---|---|---|---|
| 97% | 72% | 25pp | Severe overfitting |
| 87% | 82% | 5pp | Moderate; typical for complex models |
| 72% | 70% | 2pp | Fine; almost no overfitting |

## Step 3: Apply Remediations in Priority Order

### 3a. Get More Data (most effective)
```python
# Learning curve analysis — does performance improve with more data?
for frac in [0.1, 0.2, 0.5, 0.8, 1.0]:
    X_sub, _, y_sub, _ = train_test_split(X_train, y_train, train_size=frac)
    score = cross_val_score(model, X_sub, y_sub, cv=5).mean()
    print(f"Fraction {frac}: val score = {score:.3f}")
# If score still rising at 1.0 → more data will help
```

### 3b. Data Augmentation
See [[ml/patterns/data-augmentation]]. Add augmentation that reflects known invariances.

### 3c. Add Regularization
```python
# L2 weight decay
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-2)

# Dropout
class Model(nn.Module):
    def __init__(self):
        self.fc1 = nn.Linear(512, 256)
        self.drop = nn.Dropout(0.5)
        self.fc2 = nn.Linear(256, num_classes)
    def forward(self, x):
        return self.fc2(self.drop(F.relu(self.fc1(x))))

# Early stopping (already implemented above)
```

### 3d. Reduce Model Complexity
```python
# For XGBoost: reduce depth and increase min_child_weight
model = XGBClassifier(max_depth=3, min_child_weight=5, subsample=0.8)

# For neural net: fewer layers or neurons
# For transformer: reduce number of layers or heads
```

### 3e. Cross-Validate to Detect Data Split Artifact
```python
from sklearn.model_selection import cross_val_score
scores = cross_val_score(model, X, y, cv=5, scoring='f1')
print(f"CV scores: {scores.mean():.3f} ± {scores.std():.3f}")
# If std is high → high variance in data or unstable model
```

## Step 4: Check for Data Leakage (if performance seems too good everywhere)
See [[ml/concepts/overfitting]] for the full leakage checklist.

## The Most Common Fix in Practice
1. **Early stopping** (free; implement first)
2. **More data or augmentation** (if possible)
3. **Reduce model complexity** (simpler network or shallower trees)
4. **L2 regularization / weight decay** (tune λ via validation)
5. **Dropout** (for neural networks)

## Sources
- [[ml/concepts/overfitting]]
- [[ml/concepts/bias-variance-tradeoff]]
- [[ml/concepts/regularization]]
- [[ML overview]]
