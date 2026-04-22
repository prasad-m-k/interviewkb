# Feature Engineering

**Related concepts:** [[ml/concepts/embeddings]], [[ml/concepts/class-imbalance]]
**Related patterns:** [[ml/patterns/feature-engineering-patterns]]

## Why It Matters
For tree-based models and classical ML, feature engineering is often the highest-leverage activity. For deep learning, the network learns features automatically — but preprocessing and normalization still matter.

## Numerical Features

### Scaling
| Method | Formula | When to use |
|---|---|---|
| **StandardScaler** | (x - μ) / σ | When distribution is roughly normal; required for SVMs, PCA, logistic regression |
| **MinMaxScaler** | (x - min) / (max - min) → [0,1] | When bounds are known/meaningful; neural network inputs |
| **RobustScaler** | (x - median) / IQR | When outliers are present |
| **Log transform** | log(1 + x) | Right-skewed distributions (income, counts, prices) |

**Tree-based models (RF, XGBoost) do NOT need scaling** — they use rank-order splits.

### Binning
Convert continuous to categorical buckets. Use when:
- Relationship with target is non-monotonic (e.g., age: young and old have similar behavior)
- Domain knowledge defines meaningful ranges (e.g., credit score bands)
- Quantile binning: equal-frequency buckets (handles skew better than equal-width)

### Polynomial Features
Create interaction terms: `x1*x2`, `x1²`. Use sparingly — explodes feature space; better to use a model that captures interactions natively (trees, deep learning).

## Categorical Features

### Encoding
| Method | Use when | Gotcha |
|---|---|---|
| **One-Hot** | Low cardinality (< 20 values); linear models | Creates sparse, wide features; curse of dimensionality for high-cardinality |
| **Ordinal** | Natural order exists (low/medium/high) | Don't impose false order |
| **Target encoding** | High cardinality (zipcode, user_id) | **Leakage risk** — must use out-of-fold mean; regularize with Bayesian smoothing |
| **Frequency encoding** | High cardinality; tree models | Loses category meaning; good for rare categories |
| **Embedding** | Very high cardinality; neural networks | Learns dense representation; see [[ml/concepts/embeddings]] |

**Target encoding formula with smoothing:** `(count * cat_mean + global_n * global_mean) / (count + global_n)` — prevents rare categories from being overfit to a single example.

### Handling Unknown Categories at Inference
- OHE → add "unknown" category at training time
- Target encoding → use global mean
- Embeddings → use zero vector or separate `<UNK>` embedding

## Temporal Features
From a timestamp, extract:
- Hour of day, day of week, day of month, month, quarter, year
- Is weekend, is holiday (domain-specific)
- Time since last event (recency)
- Rolling aggregates: 7-day mean, 30-day std (but compute with care to avoid leakage)

**Leakage trap:** rolling statistics using future values leak target information. Always compute on a window ending strictly before the label's time point.

## Text Features
- **TF-IDF**: term frequency × inverse document frequency; sparse; good baseline for short text
- **Embeddings**: contextual (BERT) or static (word2vec); dense; captures semantics; see [[ml/concepts/embeddings]]
- **N-grams**: bi/trigrams capture phrases; combine with TF-IDF

## Feature Selection
| Method | How | When |
|---|---|---|
| **Correlation filter** | Remove features correlated > threshold with each other | Quick first pass; linear relationships |
| **Feature importance** | Tree-based importance (impurity / permutation) | After fitting a tree; permutation importance more reliable |
| **LASSO** | L1 pushes coefficients to zero | Linear models; interpretability required |
| **Recursive Feature Elimination (RFE)** | Iteratively remove weakest features | Small-medium datasets |
| **Mutual information** | Non-linear dependency between feature and target | Non-linear relationships |

## Common Interview Angles
- What is the difference between filter, wrapper, and embedded feature selection methods?
- How do you handle high-cardinality categoricals (zipcode, user_id)? (target encoding with out-of-fold, or entity embeddings)
- What is feature leakage? Give an example. (using post-event data before the label; e.g., "loan was repaid" as a feature for predicting default)
- Why should you scale before PCA? (PCA maximizes variance; high-variance features dominate unless scaled)
- How would you build features for a time-series fraud model? (recency, frequency, velocity features; rolling stats; device/IP behavior)

## Sources
- [[ML overview]]
