# ML Interview Prep

## Fundamentals

### Bias-Variance Tradeoff
- Bias² + Variance + Irreducible noise
- High bias → underfitting → bigger model, more features
- High variance → overfitting → more data, regularization
- [[ml/concepts/bias-variance-tradeoff]]

### Gradient Descent
- Mini-batch SGD (default)
- Adam: momentum + adaptive LR (AdamW for Transformers)
- Learning rate is #1 hyperparameter
- Gradient clipping for exploding gradients
- [[ml/concepts/gradient-descent]]

### Backpropagation
- Chain rule through computation graph
- Forward pass caches activations
- Vanishing gradient → ReLU, residual connections
- Exploding gradient → gradient clipping
- [[ml/concepts/backpropagation]]

## Model Evaluation

### Metrics
- Accuracy → bad for imbalance
- Precision / Recall / F1
- ROC-AUC → both classes matter
- PR-AUC → rare positives (fraud, medical)
- NDCG@k → ranking quality
- [[ml/concepts/precision-recall-auc]]

### Validation
- k-Fold (k=5 or 10)
- Stratified for classification
- Time-series split for temporal data
- Nested CV for hyperparameter tuning
- [[ml/concepts/cross-validation]]

### Calibration
- Predicted probabilities should match true frequencies
- Fix: Platt scaling, isotonic regression, temperature scaling

## Supervised Learning

### Classical Models
- Logistic Regression → baseline, interpretable, fast
- SVM → small data, high dimensions
- Decision Trees → interpretable, high variance alone

### Ensemble Methods
- Random Forest → bagging, low variance, robust
- XGBoost / LightGBM → boosting, low bias, max accuracy
- Stacking → meta-learner on OOF predictions
- [[ml/concepts/ensemble-methods]]

### Regularization
- L1 (Lasso) → sparsity, feature selection
- L2 (Ridge) → shrinkage, all features kept
- Dropout → ensemble of thinned networks
- Early stopping → cheapest form
- [[ml/concepts/regularization]]

## Deep Learning

### Architectures
- MLP → tabular, simple tasks
- CNN → images; skip connections (ResNet)
- RNN / LSTM → sequences; LSTM gates solve vanishing gradient
- Transformer → self-attention; O(n²); modern standard
- [[ml/topics/deep-learning]]

### Activation Functions
- ReLU → CNN/MLP hidden layers; dead neuron risk
- GELU → Transformers; smooth
- Sigmoid → binary output, LSTM gates only
- Softmax → multiclass output
- [[ml/concepts/activation-functions]]

### Normalization
- BatchNorm → CNNs; needs large batch; train ≠ inference
- LayerNorm → Transformers; per-token; any batch size
- RMSNorm → modern LLMs (LLaMA)
- [[ml/concepts/batch-normalization]]

## NLP and Transformers

### Attention
- Q, K, V → scaled dot-product → weighted sum
- Scale by √dₖ prevents softmax saturation
- Multi-head → different subspaces learned per head
- Causal mask for autoregressive generation
- [[ml/concepts/attention-mechanism]]

### Model Families
- BERT (encoder) → understanding, classification, NER
- GPT (decoder) → generation, completion
- T5 / BART (enc-dec) → seq2seq, summarization
- [[ml/concepts/transformers]]

### Fine-tuning
- Full fine-tune → best, needs data
- LoRA → low-rank adapters; frozen base; efficient
- QLoRA → quantized + LoRA; 70B on one GPU
- [[ml/patterns/transfer-learning]]

### Embeddings
- Word2Vec → static, word-level
- BERT → contextual per token
- SBERT → sentence-level, cosine-ready
- Vector DB → ANN search (HNSW, Faiss)
- [[ml/concepts/embeddings]]

## Feature Engineering

### Numerical
- StandardScaler → linear/SVM/neural
- Log transform → right-skewed distributions
- Binning → non-monotonic relationships
- Trees: no scaling needed

### Categorical
- One-hot → low cardinality
- Target encoding → high cardinality (out-of-fold only)
- Embedding table → neural networks
- [[ml/topics/feature-engineering]]

### Time-Series
- Lag features, rolling stats
- Sin/cos encoding for cyclical features (hour, day-of-week)
- Velocity / recency features for fraud

## Key Challenges

### Class Imbalance
- Accuracy is misleading → use PR-AUC
- Class weights (scale_pos_weight)
- SMOTE → synthetic minority oversampling
- Focal loss → neural networks
- Threshold tuning → PR curve
- [[ml/concepts/class-imbalance]]

### Overfitting
- Train acc >> Val acc = high variance
- Fixes: data, augmentation, regularization, smaller model
- [[ml/concepts/overfitting]]
- [[ml/scenarios/overfitting-diagnosis]]

### Training Failures
- Loss flat → LR too low, vanishing gradient, data bug
- Loss explodes → LR too high, exploding gradient
- NaN → log(0), division by zero, exploding gradient
- One-batch overfit test → most important debug step
- [[ml/scenarios/model-not-converging]]

## ML System Design

### Two-Stage Pattern
- Retrieval → millions to hundreds (fast, approximate)
- Ranking → hundreds to tens (accurate, expensive)
- Re-ranking → business rules, diversity, safety

### Key Systems
- Recommendation → two-tower + ANN + neural ranker
- Search → BM25 + dense retrieval + cross-encoder ranker
- Fraud detection → rules + XGBoost + GNN
- [[ml/topics/ml-system-design]]

### Production Challenges
- Training-serving skew → feature store
- Data drift → monitor input distributions
- Concept drift → scheduled retraining
- Feedback loops → exploration, diversity constraints
- Position bias → IPS correction, position feature
