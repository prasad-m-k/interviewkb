# Hyperparameter Tuning

**Topic:** [[ml/topics/model-evaluation]], [[ml/topics/supervised-learning]]
**Related:** [[ml/concepts/cross-validation]], [[ml/concepts/bias-variance-tradeoff]]

## What it solves
Finding the combination of hyperparameters (learning rate, regularization strength, architecture choices) that maximizes validation performance. Hyperparameters are not learned by gradient descent.

## Methods

### Grid Search
Exhaustively try all combinations in a specified grid.
```python
from sklearn.model_selection import GridSearchCV
param_grid = {
    'max_depth': [3, 5, 7],
    'learning_rate': [0.01, 0.1, 0.3],
    'n_estimators': [100, 300]
}
# Tries 3 × 3 × 2 = 18 combinations
```
- Complete coverage of defined space
- Exponential in number of hyperparameters — only feasible for ≤ 3-4 params with small grids

### Random Search
Sample random combinations from defined distributions.
```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import loguniform, randint
param_dist = {
    'max_depth': randint(3, 10),
    'learning_rate': loguniform(1e-4, 1e-1),
    'n_estimators': randint(100, 1000)
}
# Try n_iter=50 random combinations
```
- Works better than grid search for same budget (Bergstra & Bengio, 2012)
- Reason: some hyperparameters matter much more than others; random search explores them more efficiently
- Use log-uniform distribution for learning rate (not uniform)

### Bayesian Optimization
Build a surrogate model of the objective function (usually Gaussian Process); use it to propose the next most promising configuration.
```
1. Try some random initial points
2. Fit GP to observed (hyperparams → val score) pairs
3. Maximize acquisition function (expected improvement) to pick next point
4. Evaluate at new point; update GP
5. Repeat
```
Libraries: **Optuna** (most popular), Hyperopt, Ray Tune, W&B Sweeps.

```python
import optuna

def objective(trial):
    lr = trial.suggest_loguniform('lr', 1e-5, 1e-1)
    depth = trial.suggest_int('max_depth', 3, 10)
    model = XGBClassifier(learning_rate=lr, max_depth=depth)
    return cross_val_score(model, X, y, cv=5).mean()

study = optuna.create_study(direction='maximize')
study.optimize(objective, n_trials=100)
```

### Population-Based Training (PBT)
- Train a population of models in parallel
- Periodically replace worst-performing models with copies of best, with perturbed hyperparameters
- Discovers schedules and hyperparameter trajectories, not just fixed values
- Used at DeepMind for game-playing agents

## Critical Hyperparameters by Model

### Gradient Boosting (XGBoost/LightGBM)
1. `learning_rate` (η): most important; 0.01-0.3; lower → more trees needed
2. `n_estimators`: use early stopping to find optimal automatically
3. `max_depth`: 3-7; controls tree complexity
4. `subsample`: row subsampling (0.5-0.9); regularizes
5. `colsample_bytree`: column subsampling per tree
6. `min_child_weight` / `min_data_in_leaf`: minimum samples in leaf; higher → smoother

### Neural Networks
1. `learning_rate`: single most important; use LR finder
2. `batch_size`: affects both speed and generalization
3. `weight_decay`: L2 regularization
4. `dropout_rate`: typically 0.1-0.5
5. Model architecture (depth, width, heads)
6. `warmup_steps` + schedule for Transformers

### XGBoost Tuning Order (practical priority)
```
1. n_estimators with early stopping (fast to find)
2. learning_rate + n_estimators together
3. max_depth + min_child_weight (model complexity)
4. gamma (minimum gain for split)
5. subsample + colsample_bytree (stochasticity)
6. reg_alpha + reg_lambda (regularization)
```

## Early Stopping (avoids tuning n_estimators)
```python
model = XGBClassifier(n_estimators=10000, learning_rate=0.05)
model.fit(X_train, y_train,
          eval_set=[(X_val, y_val)],
          early_stopping_rounds=50,  # stop if no improvement for 50 rounds
          verbose=False)
# model.best_iteration: optimal n_estimators
```

## Common interview angles
- Why does random search outperform grid search for the same budget? (some hyperparameters matter much more; random search explores these dimensions more; grid search wastes evaluations on unimportant dimensions)
- What is Bayesian optimization and why is it more efficient? (builds a surrogate model of the objective; uses previous results to make intelligent proposals; fewer evaluations needed)
- How do you avoid overfitting to the validation set during hyperparameter tuning? (use nested cross-validation; reserve a final test set that is never touched during tuning)
- What is the learning rate's relationship to batch size? (Linear scaling rule: if you double batch size, double learning rate to maintain the same training dynamics; empirical, not exact)

## Sources
- [[ML overview]]
