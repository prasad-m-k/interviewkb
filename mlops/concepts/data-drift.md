# Data Drift

**Topic:** [[topics/mlops]]
**Related:** [[concepts/training-serving-skew]], [[concepts/model-registry]], [[concepts/experiment-tracking]]

## What it is
Data drift (also called *covariate shift*) is when the statistical distribution of model inputs in production shifts away from the distribution the model was trained on. The model's learned mappings become stale.

## Types of drift

| Type | What changes | Example |
|---|---|---|
| **Data drift** (covariate shift) | Distribution of input features X | User age distribution shifts as app gains older users |
| **Concept drift** | Relationship between X and target Y | "Spam" patterns change as spammers adapt |
| **Label drift** | Distribution of output Y | Fraud rate increases during a recession |
| **Prediction drift** | Distribution of model outputs | Model starts predicting one class more often |

## How to detect it
- **Statistical tests**: compare training distribution vs. live distribution per feature
  - Continuous features: Kolmogorov-Smirnov test, Population Stability Index (PSI)
  - Categorical features: Chi-squared test, PSI
- **PSI rule of thumb**: PSI < 0.1 = no drift, 0.1–0.25 = moderate, > 0.25 = significant
- **Monitoring dashboards**: track feature mean/std/quantiles over rolling windows
- **Model performance monitoring**: if ground truth labels are available (even delayed), track accuracy/AUC over time

## How to respond
1. **Alert**: trigger a notification when drift exceeds a threshold
2. **Investigate**: determine if drift is in important features or noise features
3. **Retrain**: trigger a retraining run on recent data (continuous training)
4. **Rollback**: if new model performs worse, roll back using [[concepts/model-registry]]

## Common interview angles
- "How do you monitor a model in production?" — mention drift detection + performance tracking + alerting + CT pipeline
- "What's the difference between data drift and concept drift?" — X distribution shifts vs. X→Y relationship shifts
- "When would you retrain a model?" — scheduled retraining, drift-triggered retraining, performance degradation threshold

## Tools
Evidently AI, WhyLogs (open source), Arize, Fiddler, NannyML, SageMaker Model Monitor, Vertex AI Model Monitoring

## Sources
*(none yet)*
