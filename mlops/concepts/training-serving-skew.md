# Training-Serving Skew

**Topic:** [[topics/mlops]]
**Related:** [[concepts/feature-store]], [[concepts/data-drift]], [[concepts/ml-pipeline]]

## What it is
Training-serving skew is when the features used during model training differ from the features computed at serving time. The model was optimized on one distribution but receives a different distribution in production — causing silent performance degradation.

## How it happens
The most common causes:

1. **Duplicate feature logic** — training code and serving code compute the same feature independently and diverge over time (bugs, refactors, different libraries).
2. **Data pipeline differences** — training reads from a batch warehouse; serving reads from a real-time stream. The aggregation windows, null handling, or join logic differs subtly.
3. **Data leakage in training** — training features include information from the future (violates point-in-time correctness), so the model sees "better" inputs in training than it ever will in production.
4. **Preprocessing inconsistency** — scaler/encoder fit on training data but not serialized and reused at serving time (common sklearn mistake).

## Why it's dangerous
It's silent. The model deploys, offline metrics looked great, but online metrics are worse — and there's no obvious error to debug. It's one of the top causes of the "works in notebooks, fails in production" problem.

## How to prevent it
1. **Use a [[concepts/feature-store]]** — single feature definition computed once, served to both training and online inference.
2. **Serialize preprocessing** — save the fitted scaler/encoder as part of the model artifact (sklearn Pipeline, TF SavedModel with preprocessing layers).
3. **Log serving features** — log the actual feature values sent to the model in production. Periodically compare their distribution against training data.
4. **Shadow mode testing** — run the new model in shadow alongside the old one; compare feature distributions before going live.

## Common interview angles
- "What is training-serving skew and how do you detect it?" — define it, then describe logging + distribution monitoring
- "Walk me through how you'd design a system to prevent it" — feature store + serialized preprocessing + feature logging

## Sources
*(none yet)*
