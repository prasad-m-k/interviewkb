# Feature Engineering Patterns

**Topic:** [[ml/topics/feature-engineering]]
**Related concepts:** [[ml/concepts/embeddings]], [[ml/concepts/class-imbalance]]

## What it solves
Systematic process for transforming raw data into features that improve model performance. The difference between a mediocre model and a production-grade one.

## Pattern 1: Encode Categoricals by Cardinality

```
Low cardinality (< 20 unique values)
  → Tree models: ordinal or leave as-is
  → Linear/neural: one-hot encoding

Medium cardinality (20–200 unique values)
  → Target encoding (with out-of-fold computation to prevent leakage)
  → Frequency encoding

High cardinality (user_id, zip code, product_id)
  → Embedding table (for neural networks)
  → Target encoding with Bayesian smoothing
  → Hash bucketing (when catalog is unbounded)
```

## Pattern 2: Velocity / Recency Features (for fraud, recommendations)
From event logs, compute aggregates at multiple time windows:
```python
# For each user, compute rolling statistics
features = {
    'tx_count_1h':    count of transactions in last 1 hour
    'tx_count_24h':   count in last 24 hours
    'tx_count_7d':    count in last 7 days
    'avg_amount_7d':  mean transaction amount last 7 days
    'days_since_last_tx': recency
    'unique_merchants_7d': cardinality
}
```
These capture **behavioral patterns** that static demographic features miss. Fraud spikes in short windows; churn correlates with low recency.

## Pattern 3: Log-Transform Right-Skewed Numerics
```python
import numpy as np
# Income, price, count, duration are always skewed
X['log_price'] = np.log1p(X['price'])   # log(1+x) handles 0 values
```
Why: linear models and distance-based models are sensitive to scale; log makes the distribution more Gaussian; reduces outlier impact.

## Pattern 4: Interaction Features for Linear Models
```python
# Cross-product of meaningful feature pairs
X['age_x_income'] = X['age'] * X['income']
X['is_weekend_x_hour'] = X['is_weekend'] * X['hour_of_day']
```
Tree models discover interactions automatically. Linear models and logistic regression cannot — add them manually if domain knowledge suggests an interaction.

## Pattern 5: Target Encoding with Out-of-Fold
```python
from category_encoders import TargetEncoder

# WRONG: fit on full dataset → leaks target
enc = TargetEncoder().fit(X, y)

# RIGHT: fit only within each training fold
for train_idx, val_idx in kfold.split(X, y):
    enc = TargetEncoder(smoothing=10)
    X_train_enc = enc.fit_transform(X[train_idx], y[train_idx])
    X_val_enc = enc.transform(X[val_idx])  # apply fitted encoder
```

Smoothing formula: `(cat_count × cat_mean + λ × global_mean) / (cat_count + λ)` — prevents rare categories from getting wild estimates.

## Pattern 6: Time-Series Features (Temporal Fingerprint)
```
From timestamp t, extract:
- Calendar: hour, dow, month, quarter, is_holiday, is_weekend
- Periodicity: sin(2π × hour/24), cos(2π × hour/24)   ← encode cyclicality
- Recency: time_since_last_event, time_to_next_event
- Rolling: 7d_mean, 30d_mean, 7d_std, 7d_max
- Lag features: value at t-1d, t-7d, t-30d
```
Cyclical encoding (sin/cos) is important for hour and day-of-week — ensures that 23:00 and 01:00 are close (not far apart as integers).

## Pattern 7: Missing Value Strategy
```
Missing at random:
  → Mean/median imputation (for symmetric/skewed distributions)
  → KNN imputation (preserves correlations)

Missing not at random (missingness is informative):
  → Create binary indicator feature is_<feature>_missing
  → This often improves model performance

Tree models:
  → Can handle NaN natively (XGBoost, LightGBM)

Neural networks:
  → Impute + add indicator column
```

## Signal phrases
"improve features for tabular data" / "my model is underfitting" / "high cardinality categorical" / "feature engineering for fraud/recommendation" / "dealing with missing values"

## Complexity
Data transformation is O(N × d). Target encoding is O(N × k) for k folds.

## Sources
- [[ml/topics/feature-engineering]]
- [[ML overview]]
