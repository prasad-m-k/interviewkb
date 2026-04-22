# Overfitting

**Topic:** [[ml/topics/supervised-learning]]
**Related:** [[ml/concepts/bias-variance-tradeoff]], [[ml/concepts/regularization]], [[ml/concepts/cross-validation]]

## What it is
A model overfits when it learns the training data too well — including noise and sampling artifacts — and fails to generalize to new data. Manifests as a large gap between training and validation performance.

```
Overfitting signal:
  Training accuracy:   98%
  Validation accuracy: 71%
  → Gap = 27 points
```

## Root Causes
1. **Model too complex** for the amount of data (deep network on 500 examples)
2. **Too little training data** relative to model capacity
3. **Insufficient regularization**
4. **Data leakage** — target information in features → artifically perfect training metrics
5. **No early stopping** — trained too many epochs

## Diagnosis Checklist

```
Step 1: Plot training vs. validation loss curves
        → Does val loss diverge upward while train continues down?  → Overfitting
        → Both plateau high?                                        → Underfitting (high bias)
        → Both plateau low?                                         → Good fit

Step 2: Learning curves (performance vs. training set size)
        → Small training set size → large train/val gap?
        → Gap closes as data increases → need more data

Step 3: Confusion matrix analysis
        → Does the model perfectly predict training class distribution?
        → Systematically wrong on certain subgroups at test time?

Step 4: Check for leakage
        → Are any features derived from the label or from post-event data?
        → Was preprocessing (scaling, encoding) fit on the full dataset?
```

## Prevention Strategies

### Regularization
- L1/L2 weight penalties: see [[ml/concepts/regularization]]
- Dropout: randomly zero activations during training
- Early stopping: halt training when val loss stops improving
- Label smoothing: soften hard labels to prevent overconfidence

### Data
- **More training data**: always helps with variance
- **Data augmentation**: creates effective diversity without new labels; see [[ml/patterns/data-augmentation]]
- **Clean labels**: label noise amplifies overfitting

### Model Complexity
- Reduce depth or width of neural network
- Reduce `max_depth` for tree models
- Fewer features (feature selection, PCA)
- Simpler architecture for dataset size

### Training
- Reduce learning rate (prevents memorization)
- Use batch normalization (implicit regularization)
- Cross-validation to detect overfitting early

## Special Case: Memorization without Generalization
Neural networks with enough capacity can memorize random labels (Zhang et al. 2017). This proves that classical learning theory bounds don't fully explain DL generalization — regularization and implicit biases (SGD's preference for flat minima) are critical.

## Detecting Data Leakage (a hidden form of overfitting)
Data leakage causes unrealistically high training (and test) performance that disappears in production.

Common sources:
| Source | Example | Fix |
|---|---|---|
| **Target encoding leak** | Encode category using full-dataset mean | Use out-of-fold mean only |
| **Future feature** | "Was flagged for fraud" as input for fraud model | Ensure features are only available at prediction time |
| **Row duplication** | Same user in train and test | Group-split by user |
| **Preprocessing before split** | Scale using full dataset mean | Fit scaler only on training fold |
| **Label distribution leak** | Oversampling before splitting | SMOTE/oversampling after splitting |

## Common interview angles
- Your model is 99% accurate in development but 65% in production. What happened? (data leakage, train/test distribution mismatch, or overfitting to test set by repeated evaluation)
- What is the difference between overfitting and high variance? (they are the same thing from different angles; overfitting describes the symptom on training vs. val; high variance describes the statistical property of the model)
- How does early stopping prevent overfitting? (stops training before the model has memorized noise; equivalent to constraining the effective model complexity via training time)
- Can you overfit a linear model? (yes — if you have more features than examples, or if you add polynomial features; L2 regularization is the standard fix)
- What is double descent? (test error decreases, increases, then decreases again as model size grows past interpolation threshold; occurs in deep learning; suggests very large models generalize well despite memorizing training data)

## Sources
- [[ML overview]]
