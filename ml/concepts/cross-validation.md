# Cross-Validation

**Topic:** [[ml/topics/model-evaluation]]
**Related:** [[ml/concepts/bias-variance-tradeoff]], [[ml/concepts/overfitting]], [[ml/patterns/hyperparameter-tuning]]

## What it is
A resampling technique to estimate how well a model generalizes to unseen data, by systematically using different portions of the data as training and validation sets.

## Standard k-Fold

```
Data: [F1][F2][F3][F4][F5]

Fold 1: Train=[F2,F3,F4,F5]  Val=[F1]
Fold 2: Train=[F1,F3,F4,F5]  Val=[F2]
Fold 3: Train=[F1,F2,F4,F5]  Val=[F3]
Fold 4: Train=[F1,F2,F3,F5]  Val=[F4]
Fold 5: Train=[F1,F2,F3,F4]  Val=[F5]

Final score: mean ± std across 5 folds
```

- k=5 or k=10 are standard; k=10 gives slightly better estimates at 2× cost
- Report both mean and std: `0.84 ± 0.03` — high std means unstable model or too little data
- Leave-One-Out (LOO): k=N; unbiased but very high variance in estimate; rarely used

## Stratified k-Fold
Ensures each fold has the same class distribution as the full dataset. **Always use for classification**, especially with imbalanced classes.

```python
from sklearn.model_selection import StratifiedKFold
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
for train_idx, val_idx in skf.split(X, y):
    ...
```

## Time-Series Split (Walk-Forward Validation)
For temporal data: training must always precede validation in time.

```
Fold 1: Train=[Jan-Mar]   Val=[Apr]
Fold 2: Train=[Jan-Apr]   Val=[May]
Fold 3: Train=[Jan-May]   Val=[Jun]
```

**Never shuffle time-series data before splitting.** This leaks future information into training.

## Group k-Fold
When data has groups that should not be split across train/val (e.g., multiple images from the same patient, multiple sessions from the same user). Prevents information leakage across groups.

```python
from sklearn.model_selection import GroupKFold
gkf = GroupKFold(n_splits=5)
for train_idx, val_idx in gkf.split(X, y, groups=user_ids):
    ...
```

## Nested Cross-Validation
Used when you need to both tune hyperparameters and get an honest performance estimate.

```
Outer loop: honest performance estimate (5 folds)
  Inner loop: hyperparameter search (3 folds)

For each outer fold:
  1. Inner CV on training portion → best hyperparameters
  2. Retrain with best hyperparameters on full training portion
  3. Evaluate on outer held-out fold
```

Without nesting: hyperparameters are tuned on the same data used to estimate performance → optimistically biased estimate.

## Train/Val/Test Split Philosophy
```
Train (70%)   → model learns from this
Val (15%)     → hyperparameter tuning / early stopping
Test (15%)    → final, one-time evaluation — touch ONCE
```

**Never tune on the test set.** If you evaluate on the test set during development, it becomes a validation set — you need a fresh test set.

## Common Mistakes
- **Data leakage in the pipeline**: fit scaler/encoder on training + validation data before splitting → information from val leaks into train. Fix: fit preprocessing **only on training fold**.
- **Shuffling time-series**: breaks temporal ordering → inflated metrics
- **Class imbalance ignoring**: k-fold without stratification → some folds may have few or no positive examples

## Complexity
k-fold = k × cost of one training run. For expensive models, use k=3 or k=5 rather than k=10.

## Common interview angles
- What is the purpose of a test set if you already have a validation set? (val set is used during development; test set is a simulation of deployment; val metrics are optimistic due to selection)
- Why does k-fold give a better estimate than a single train/val split? (averages over multiple random splits; reduces variance in the estimate)
- How do you do CV for time-series? (walk-forward; train only on past; never random split)
- What is data leakage and how does k-fold help? (proper k-fold: fit preprocessing inside each fold; prevents leakage from val into training; see [[ml/topics/model-evaluation]])

## Sources
- [[ML overview]]
