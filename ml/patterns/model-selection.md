# Model Selection Framework

**Topic:** [[ml/topics/supervised-learning]]
**Related:** [[ml/concepts/bias-variance-tradeoff]], [[ml/concepts/cross-validation]], [[ml/patterns/hyperparameter-tuning]]

## What it solves
A systematic decision process for choosing the right model architecture given data characteristics and constraints.

## The Decision Tree

```
What is the output type?
├── Continuous → Regression
└── Categorical → Classification (or ranking if ordering matters)

What is the data modality?
├── Tabular (rows + columns)
│   ├── < 10K rows → Logistic/Linear Regression (regularized), SVM
│   ├── 10K–1M rows → Gradient Boosting (XGBoost, LightGBM)
│   └── > 1M rows → LightGBM or Neural Network (MLP/TabNet)
├── Image → CNN or Vision Transformer (ViT)
├── Text / Sequences → Transformer (BERT/GPT/T5)
├── Graph → GNN
└── Time series
    ├── Simple trends + seasonality → ARIMA, Prophet
    ├── Complex patterns → LSTM, TCN, or Transformer
    └── Multi-variate features → LightGBM with lag features often wins

Do you need interpretability?
├── Yes → Logistic Regression, Decision Tree, or SHAP values on XGBoost
└── No → Gradient Boosting or Neural Network

What are the latency / resource constraints?
├── < 5ms → Logistic Regression (dot product), small tree
├── < 50ms → XGBoost (small depth), small neural net
└── < 500ms → Larger models; consider ONNX / quantization
```

## Starting Point Heuristics

| Problem | Start with | Why |
|---|---|---|
| Tabular, any size | LightGBM | Fast, robust, minimal preprocessing |
| Imbalanced binary | LightGBM + PR-AUC eval | scale_pos_weight; tree handles imbalance |
| Text classification | BERT fine-tune | Pre-trained semantics; 500+ examples sufficient |
| Image classification | ResNet50 / EfficientNet fine-tune | Pre-trained ImageNet weights |
| Recommendation (small) | Matrix factorization / ALS | Simple, fast, interpretable |
| Recommendation (large) | Two-tower neural network | Scalable; handles cold start |
| Time-series forecast | LightGBM with lag features | Often beats LSTM in practice |

## Evaluation-Driven Selection
Never select based on model complexity alone:
1. Establish a **strong baseline** (logistic regression, mean prediction)
2. Define the **right metric** for the problem before training anything
3. Use **k-fold cross-validation** to estimate generalization
4. Compare models on the val metric; use confidence intervals
5. Choose the **simplest model that meets the performance requirement** — not the most complex

## Production Constraints Matrix
Before finalizing, answer:

| Constraint | Impact on choice |
|---|---|
| Inference latency < 10ms | Logistic regression, small tree |
| Model must update in real-time | Online learning models; or lightweight retraining pipeline |
| Explainability required (regulated industry) | Linear model or tree with SHAP |
| Mobile / edge deployment | Quantized small NN; tree ≤ 100 nodes |
| Training data < 1000 labeled examples | Transfer learning; few-shot; classical ML |

## Common interview angles
- When would you choose logistic regression over XGBoost? (interpretability requirement, real-time low-latency, very small data, linear relationship)
- When would you choose XGBoost over a neural network for tabular data? (almost always on small-medium tabular data; neural nets win only with large data and complex feature interactions or mixed modalities)
- How do you avoid "model shopping" (overfitting to the test set)? (fix your evaluation metric and protocol before training; use nested CV; touch the test set only once)

## Sources
- [[ML overview]]
