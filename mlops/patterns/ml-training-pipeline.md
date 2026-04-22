# ML Training Pipeline Pattern

**Topic:** [[topics/mlops]]
**Related concepts:** [[concepts/ml-pipeline]], [[concepts/feature-store]], [[concepts/experiment-tracking]], [[concepts/model-registry]]

## What it solves
How to go from raw data to a deployed model artifact in a reliable, repeatable, automated way — replacing ad-hoc notebook runs.

## Pattern skeleton

```
┌─────────────────────────────────────────────────────┐
│                   Training Pipeline                  │
│                                                      │
│  1. Data ingestion    → raw dataset artifact         │
│  2. Data validation   → schema check, stats, alerts  │
│  3. Feature engineering → feature table artifact     │
│  4. Train/val split   → split dataset artifacts      │
│  5. Model training    → model checkpoint artifact    │
│  6. Model evaluation  → metrics artifact             │
│  7. Model push        → register to model registry   │
│         if metrics > baseline threshold              │
└─────────────────────────────────────────────────────┘
```

## Step details

**1. Data ingestion**
Pull data from source (data warehouse, lake, streaming). Log dataset version/hash for reproducibility.

**2. Data validation**
Check schema, value ranges, null rates, row count. Fail fast here rather than discover problems after a 6-hour training run. Tools: Great Expectations, TFX Data Validation.

**3. Feature engineering**
Compute features. If a [[concepts/feature-store]] exists, pull from it here instead of recomputing.

**4. Train/val split**
Split with a fixed seed or time-based split (for temporal data). Log split parameters.

**5. Model training**
Train with logged hyperparameters. Log metrics, parameters, and artifacts to [[concepts/experiment-tracking|experiment tracker]].

**6. Model evaluation**
Compare new model against current production baseline. If new model doesn't beat baseline, do not push.

**7. Model push (conditional)**
Register the model artifact to the [[concepts/model-registry]] with metrics and lineage. Mark as `Staging` — a separate CD process promotes to `Production`.

## Trigger strategies
- **Schedule**: retrain weekly/nightly on fresh data
- **Data trigger**: trigger when N new labeled examples arrive
- **Drift trigger**: trigger when [[concepts/data-drift]] exceeds threshold
- **Manual**: data scientist triggers on demand

## Signal phrases (interview)
"design a retraining pipeline", "how do you automate model training", "continuous training", "CT pipeline"

## Sources
*(none yet)*
