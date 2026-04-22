# Supervised Learning

**Related concepts:** [[ml/concepts/bias-variance-tradeoff]], [[ml/concepts/regularization]], [[ml/concepts/cross-validation]], [[ml/concepts/ensemble-methods]], [[ml/concepts/loss-functions]]

## What it is
Learning a mapping from inputs X to outputs Y from labeled examples. The model minimizes a loss function that measures how far its predictions are from the true labels.

## Classification vs. Regression
| | Classification | Regression |
|---|---|---|
| **Output** | Discrete class label | Continuous value |
| **Loss** | Cross-entropy, hinge | MSE, MAE, Huber |
| **Evaluation** | Accuracy, F1, AUC-ROC | RMSE, MAE, R² |
| **Examples** | Spam detection, churn | House price, demand forecast |

## Key Algorithms

### Linear Models
- **Logistic Regression** — linear decision boundary; fast; interpretable; baseline for binary classification
- **Linear Regression / Ridge / Lasso** — Lasso does feature selection (L1 sparsity); Ridge shrinks coefficients (L2)
- **SVM** — maximum-margin hyperplane; kernel trick for non-linear; great for high-dimensional, small-data problems

### Tree-Based Models
- **Decision Tree** — axis-aligned splits; interpretable; high variance alone → bag or boost
- **Random Forest** — bagging + feature subsampling; low variance; handles missing values; good baseline
- **Gradient Boosting (XGBoost / LightGBM / CatBoost)** — sequential trees correcting residuals; state-of-art for tabular data
  - XGBoost: regularized; handles sparse data; second-order gradients
  - LightGBM: leaf-wise growth; faster on large datasets
  - CatBoost: native categorical handling; less tuning needed

### Neural Networks
- MLPs for tabular, CNNs for images, RNNs/Transformers for sequences
- See [[ml/topics/deep-learning]]

## Decision Framework: Picking the Right Algorithm

```
Tabular data?
├── < 100K rows, interpretability needed → Logistic Regression / Linear SVM
├── < 100K rows, max performance → XGBoost / LightGBM
├── > 100K rows, complex features → Neural Network (TabNet, or MLP with embeddings)
└── Categorical-heavy → CatBoost

Image / audio / video → CNN or Vision Transformer
Text / sequence → Transformer (BERT, GPT)
```

## Common Interview Angles
- Why does logistic regression output probabilities? (sigmoid maps to [0,1]; log-likelihood loss)
- When does SVM beat logistic regression? (small datasets, high dimensions, kernel trick for nonlinearity)
- Random Forest vs. XGBoost? (RF = parallel, low variance; XGB = sequential, lower bias, more tuning)
- What is multiclass classification? (one-vs-rest, one-vs-one, softmax output)
- How do you handle a regression problem where outliers matter a lot? (Huber loss = MSE for small errors, MAE for large)

## Sources
- [[ML overview]]
