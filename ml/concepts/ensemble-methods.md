# Ensemble Methods

**Topic:** [[ml/topics/supervised-learning]]
**Related:** [[ml/concepts/bias-variance-tradeoff]], [[ml/concepts/decision-trees]], [[ml/concepts/regularization]]

## What it is
Combining multiple models to produce predictions better than any individual model. Reduces variance (bagging), bias (boosting), or both (stacking).

## Bagging (Bootstrap Aggregating)

**Mechanism:**
1. Sample N bootstrap datasets (with replacement) from training data
2. Train one model on each dataset independently
3. Aggregate predictions: average (regression) or majority vote (classification)

**Why it works:** Each model is trained on a different sample → diversity → averaging cancels out errors. Bias is unchanged; variance is reduced by factor ~1/N (for uncorrelated models).

### Random Forest
Bagging + additional randomness:
- Each split considers only a random subset of features (`max_features = sqrt(d)` for classification)
- Makes trees less correlated → bigger variance reduction

**Key hyperparameters:**
- `n_estimators`: more trees → lower variance; diminishing returns; 100-500 is typical
- `max_depth`: controls individual tree variance; `None` (full depth) is common
- `max_features`: subset of features per split; lower = less correlated trees
- `min_samples_leaf`: smoothing; higher = less overfit

**Strengths**: handles missing values, mixed types, scales automatically, robust to outliers, fast to train in parallel, built-in feature importance.

## Boosting

**Mechanism:** Build trees sequentially; each tree corrects the residual errors of the previous ensemble.

### Gradient Boosting
```
F₀(x) = initial prediction (e.g., mean)
For m = 1 to M:
    rᵢ = -∂L(yᵢ, F_{m-1}(xᵢ)) / ∂F    # pseudo-residuals (negative gradient)
    Train tree hₘ on rᵢ
    Fₘ = F_{m-1} + η · hₘ              # η: learning rate (shrinkage)
```

### XGBoost
- Regularized objective: `L + Ω(tree) = Σ l(yᵢ, ŷᵢ) + γT + 0.5λ‖w‖²`
- Second-order Taylor expansion: uses Hessian for better gradient approximation
- Column and row subsampling (like Random Forest)
- Handles missing values natively: learns default split direction
- `scale_pos_weight`: handles class imbalance

### LightGBM
- **Leaf-wise growth** (vs. level-wise in XGBoost): grows the leaf with highest loss reduction first → fewer leaves needed for same accuracy
- Much faster on large datasets (100K+ rows)
- GOSS (Gradient-based One-Side Sampling): keep high-gradient examples, sample low-gradient ones
- EFB (Exclusive Feature Bundling): bundle mutually exclusive sparse features

### CatBoost
- Native categorical handling: ordered target statistics prevent leakage
- Ordered boosting: build each tree on a permutation-consistent subset; prevents overfitting
- Minimal feature engineering needed; often best out-of-the-box

## Random Forest vs. XGBoost — When to Use Which

| | Random Forest | XGBoost/LightGBM |
|---|---|---|
| **Bias** | Higher | Lower |
| **Variance** | Lower | Can be higher |
| **Training** | Parallel | Sequential |
| **Tuning** | Minimal | More hyperparameters |
| **Missing values** | Impute first | Native support |
| **Overfitting risk** | Low | Higher (but regularized) |
| **Best when** | Quick baseline, small data | Competitions, max accuracy |

**Rule of thumb:** Start with Random Forest as a baseline; switch to XGBoost/LightGBM for squeezing out the last percent.

## Stacking
Train a **meta-learner** on the out-of-fold predictions of base models:
```
Level 0: Train RF, XGB, LR on training data via k-fold
Level 1: Stack their OOF predictions → train a meta-model (usually LR or XGB)
```
- Often 0.5-1% improvement over best single model
- More complex to implement correctly (avoid leakage between levels)

## Voting Ensembles
Simple: majority vote for classification, mean for regression. Effective when models are diverse (different architectures, different features).

## Common interview angles
- Why does bagging reduce variance but not bias? (averaging uncorrelated models reduces variance; each model has same expected bias as individual model)
- Why are boosting models vulnerable to overfitting? (if learning rate is high and trees are deep, can memorize noise; regularize with η, max_depth, subsample)
- Random Forest feature importance: two types? (impurity-based: mean decrease in Gini across splits — fast but biased toward high-cardinality features; permutation importance: permute feature values, measure accuracy drop — unbiased, slower)
- When does XGBoost outperform deep learning on tabular data? (XGBoost often wins on small/medium tabular data, especially with categorical features; DL wins when data is large and has spatial/sequential structure)
- What is shrinkage in boosting and why does it help? (multiply each tree's contribution by a small learning rate η; adds more trees, each making smaller corrections; regularization; better generalization)

## Sources
- [[ML overview]]
