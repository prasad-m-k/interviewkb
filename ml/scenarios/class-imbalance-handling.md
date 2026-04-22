# Scenario: Handling Class Imbalance

**Category:** Data Engineering / Model Training
**Difficulty:** Medium
**Seen at:** Stripe, PayPal, Amazon, Google (fraud / safety teams)

## The Scenario
You're building a fraud detection model. 99.8% of transactions are legitimate; 0.2% are fraudulent. Your initial model achieves 99.7% accuracy. Your manager says "great!" but you know something's wrong.

## Step 1: Diagnose the Real Problem

```python
from sklearn.metrics import classification_report, confusion_matrix

y_pred = model.predict(X_test)
print(classification_report(y_test, y_pred))
# Output:
#               precision  recall  f1-score  support
# 0 (legit)       1.00      1.00      1.00    9980
# 1 (fraud)       0.00      0.00      0.00      20
# accuracy                            0.998   10000
```

The model learned to predict "not fraud" always. Accuracy is meaningless here.

**Correct metric:** PR-AUC, Recall@Precision=90%, or F1 on the positive class.

## Step 2: Choose Your Strategy

```
Start here: what is the positive rate?
├── 1-10%: class weights + threshold tuning is usually sufficient
├── 0.1-1%: class weights + SMOTE + focal loss
├── < 0.1%: anomaly detection framing; or extreme oversampling
```

## Step 3: Apply the Playbook

### 3a. Fix the Metric First
```python
from sklearn.metrics import average_precision_score, roc_auc_score

y_score = model.predict_proba(X_test)[:, 1]
print(f"PR-AUC: {average_precision_score(y_test, y_score):.3f}")
print(f"ROC-AUC: {roc_auc_score(y_test, y_score):.3f}")
# Always use PR-AUC for imbalanced; ROC-AUC can be misleadingly high
```

### 3b. Class Weights (easiest, lowest cost)
```python
from sklearn.utils.class_weight import compute_class_weight
import numpy as np

weights = compute_class_weight('balanced', classes=[0,1], y=y_train)
class_weight_dict = {0: weights[0], 1: weights[1]}
# weights ≈ {0: 0.5, 1: 250} for 0.2% positive rate

# XGBoost
model = XGBClassifier(scale_pos_weight=len(y[y==0])/len(y[y==1]))

# PyTorch
pos_weight = torch.tensor([neg_count / pos_count])
loss = F.binary_cross_entropy_with_logits(logits, targets, pos_weight=pos_weight)
```

### 3c. SMOTE Oversampling (after class weights, if still insufficient)
```python
from imblearn.over_sampling import SMOTE
from imblearn.pipeline import Pipeline

# CRITICAL: apply SMOTE only inside each training fold, never before splitting
pipeline = Pipeline([
    ('smote', SMOTE(sampling_strategy=0.1, random_state=42)),  # 10:1 ratio, not 1:1
    ('model', XGBClassifier())
])
# Don't oversample all the way to 1:1 — slight imbalance is fine
```

### 3d. Threshold Tuning (often overlooked)
```python
from sklearn.metrics import precision_recall_curve
import numpy as np

prec, rec, thresholds = precision_recall_curve(y_val, y_score)
# Business requirement: "flag at least 80% of fraud, maximize precision"
target_recall = 0.80
idx = np.argmin(np.abs(rec - target_recall))
best_threshold = thresholds[idx]
print(f"Threshold for 80% recall: {best_threshold:.3f}")
print(f"Precision at that threshold: {prec[idx]:.3f}")
```

### 3e. Focal Loss for Neural Networks
```python
class FocalLoss(nn.Module):
    def __init__(self, gamma=2, alpha=0.25):
        super().__init__()
        self.gamma = gamma
        self.alpha = alpha
    def forward(self, logits, targets):
        p = torch.sigmoid(logits)
        bce = F.binary_cross_entropy_with_logits(logits, targets, reduction='none')
        p_t = p * targets + (1 - p) * (1 - targets)
        alpha_t = self.alpha * targets + (1 - self.alpha) * (1 - targets)
        return (alpha_t * (1 - p_t) ** self.gamma * bce).mean()
```

## Step 4: Validate Correctly
```python
# Always use stratified k-fold for imbalanced data
from sklearn.model_selection import StratifiedKFold
skf = StratifiedKFold(n_splits=5, shuffle=True)
# Each fold maintains the same 0.2% positive rate as the full dataset
```

## Production Monitoring
After deployment, monitor:
- **Fraud rate over time** (if it changes, model performance shifts)
- **False positive rate by customer segment** (imbalance may differ across segments)
- **Score distribution drift** → may indicate model staleness

## Sources
- [[ml/concepts/class-imbalance]]
- [[ml/concepts/precision-recall-auc]]
- [[ml/concepts/loss-functions]]
- [[ML overview]]
