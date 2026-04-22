# Unsupervised Learning

**Related concepts:** [[ml/concepts/embeddings]], [[ml/concepts/class-imbalance]]
**Related topics:** [[ml/topics/feature-engineering]]

## What it is
Learning structure from unlabeled data. No target labels — the model finds patterns, clusters, or compressed representations.

## Major Paradigms

### Clustering
Find natural groupings in data.

| Algorithm | How it works | When to use | Gotcha |
|---|---|---|---|
| **K-Means** | Assign to nearest centroid; update centroids | When K is known; convex clusters | Sensitive to outliers; fails on non-convex |
| **DBSCAN** | Density reachability; core/border/noise points | Unknown K; non-convex clusters; outlier detection | `eps` and `min_samples` are tricky to tune |
| **Hierarchical** | Agglomerative (bottom-up) or divisive; dendrogram | Small datasets; want dendrogram | O(n²) memory; not scalable |
| **GMM** | Soft cluster assignment; Gaussian components; EM | Overlapping clusters; probabilistic membership | Sensitive to initialization; can collapse |

**Choosing K for K-Means:** Elbow method (inertia vs K) or silhouette score (higher = better separation).

### Dimensionality Reduction
Compress high-dimensional data while preserving structure.

| Algorithm | Preserves | Use case |
|---|---|---|
| **PCA** | Global variance (linear) | Feature compression; whitening; noise removal |
| **t-SNE** | Local neighborhood structure (non-linear) | 2D/3D visualization only; not for new data |
| **UMAP** | Both local and global (non-linear) | Better than t-SNE for large datasets; can transform new data |
| **Autoencoders** | Learned compressed representation | Feature learning; anomaly detection (reconstruction error) |

**PCA mechanics:** SVD on covariance matrix → principal components are eigenvectors; explained variance ratio tells you how many components to keep.

### Anomaly / Outlier Detection
Find data points that don't fit the norm.

- **Isolation Forest** — randomly partitions data; anomalies are isolated in fewer splits; efficient; works on high-dimensional data
- **Autoencoder reconstruction error** — train on normal data; anomalies have high reconstruction loss
- **LOF (Local Outlier Factor)** — density-based; local density ratio vs neighbors
- **One-Class SVM** — learns a boundary around normal data

### Self-Supervised / Representation Learning
Learning representations without explicit labels.

- **Word2Vec / GloVe** — predict neighboring words (unsupervised but uses text structure as signal)
- **BERT masked language modeling** — predict masked tokens → rich contextual embeddings
- **Contrastive learning (SimCLR, MoCo)** — pull augmented views of same image together; push different images apart

## Common Interview Angles
- When would you use unsupervised learning before supervised? (feature learning, data exploration, cold-start problems)
- K-Means vs. DBSCAN trade-offs
- Why is t-SNE not good for inference on new data? (non-parametric; no out-of-sample extension)
- How would you detect anomalies in a streaming setting? (sliding window + Isolation Forest or LSTM reconstruction error)
- What is the curse of dimensionality? (distances become meaningless; all points equidistant; volume of space grows exponentially; need exponentially more data)

## Sources
- [[ML overview]]
