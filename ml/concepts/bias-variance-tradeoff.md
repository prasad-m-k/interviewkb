# Bias-Variance Tradeoff

**Topic:** [[ml/topics/supervised-learning]]
**Related:** [[ml/concepts/regularization]], [[ml/concepts/overfitting]], [[ml/concepts/cross-validation]], [[ml/concepts/ensemble-methods]]

## What it is
Every model's prediction error on unseen data can be decomposed into three terms:

```
Total Error = Bias² + Variance + Irreducible Noise
```

- **Bias**: error from wrong assumptions. A high-bias model underfits — it can't capture the true complexity of the data.
- **Variance**: error from sensitivity to training data. A high-variance model overfits — it learns noise as signal.
- **Irreducible noise**: inherent randomness in the labels. Can't be eliminated by any model.

## How it works

```
         High Bias                    High Variance
         (Underfitting)               (Overfitting)
              |                              |
    Train ≈ Val ≈ High error     Train << Val
         |                              |
  Model too simple               Model too complex
  Can't represent signal         Memorizes training data
```

**The trade-off**: reducing bias (more complex model) tends to increase variance, and vice versa. The goal is to find the sweet spot.

## Diagnosing with Learning Curves

```
Error
  |
  |  ~~~~~~~~~~~~~~~~~~~~ (val)       ← high bias: val plateaus high
  |  ~~~~~~~~~~~~~~~~~~~~ (train)       (train ≈ val ≈ both high)
  |__________________________
              Training set size

Error
  |
  |            ~~~~~~~~~~~~ (val)     ← high variance: large gap
  |  _________ (train)                 (train low, val high)
  |__________________________
              Training set size
```

## Levers

| To reduce bias | To reduce variance |
|---|---|
| More complex model | Simpler model |
| More features | Feature selection / regularization |
| Less regularization | More regularization (L1/L2/dropout) |
| More training time | Early stopping |
| Better architecture | More training data |
| | Bagging / ensembling |

## Connection to Ensemble Methods
- **Bagging** (Random Forest): reduces variance without increasing bias — average of many high-variance, low-bias trees
- **Boosting** (XGBoost): reduces bias by sequentially correcting errors — starts with weak (high-bias) learners and reduces bias each step; can also reduce variance

## When to use
Frame every ML debugging scenario through this lens:
- Low train accuracy → high bias → need more capacity or better features
- High train accuracy + low val accuracy → high variance → need regularization, more data, or simpler model

## Common interview angles
- "Your model has 99% train accuracy and 70% val accuracy. What do you do?" → high variance → regularize, get more data, reduce model complexity
- "Your model has 60% train and 60% val accuracy on a balanced binary classification task. What do you do?" → high bias → bigger model, more features, less regularization
- Why does more data help with variance but not bias? (more data makes the average of models more stable, but if the model is fundamentally too simple, more data can't help it capture complexity)
- What is the Bias-Variance trade-off in k-fold CV? (k=N is leave-one-out: low bias, high variance in the estimate; k=5-10 is a good balance)

## Sources
- [[ML overview]]
