# Feature Store

**Topic:** [[topics/mlops]]
**Related:** [[concepts/training-serving-skew]], [[concepts/ml-pipeline]], [[concepts/data-drift]]

## What it is
A feature store is a centralized data layer that manages the creation, storage, and serving of ML features. It provides the same feature values to both training and serving, ensuring consistency across the ML lifecycle.

## How it works
A feature store has two serving paths:
- **Offline store** (batch): historical feature values for training. Backed by a data warehouse or object store (S3, BigQuery, Hive).
- **Online store** (low-latency): latest feature values for real-time inference. Backed by a key-value store (Redis, DynamoDB, Bigtable).

A *feature pipeline* computes features from raw data and writes to both stores. During training you pull from the offline store; during serving you pull from the online store.

```
Raw data → Feature pipeline → Offline store (training)
                            → Online store  (serving)
```

## Why it matters
Without a feature store, teams recompute the same features independently in training code and serving code. This causes [[concepts/training-serving-skew]] and wastes engineering effort.

## When to use
- Multiple models reuse the same features (e.g., user activity features shared by recommendation and fraud models)
- Real-time serving requires low-latency feature lookup
- You need point-in-time correct features for training (avoiding data leakage)

## Common interview angles
- "How would you design a feature store?" — cover offline/online split, point-in-time correctness, feature versioning
- "What is point-in-time correctness?" — when building training data, you must use only features available *at* the timestamp of each training example, not features computed later. Violating this causes data leakage.
- "How do you prevent training-serving skew?" — single feature definition, computed once, served from both stores

## Key concepts
- **Point-in-time correctness**: training data only uses features that were available at prediction time
- **Feature versioning**: ability to roll back to a previous feature definition
- **Feature sharing**: data scientists discover and reuse features without recomputing
- **Materialization**: the process of computing features and writing them to the store

## Tools
Feast (open source), Tecton, Vertex AI Feature Store, Databricks Feature Store, SageMaker Feature Store, Hopsworks

## Sources
*(none yet)*
