# Flashcards — Top 10 MLOps Questions (FAANG)

**Topic:** [[mlops/topics/mlops]]
**Created:** 2026-04-21

Format: read the question, formulate your answer, then expand the callout to check.

---

## Card 1
**Q: What is training-serving skew, why is it dangerous, and how do you prevent it?**

> [!note]- Answer
> **What it is:** The features used during training differ from the features computed at serving time. The model was optimized on one distribution but receives another in production.
>
> **Why it's dangerous:** It's *silent* — no exception is thrown. The model deploys, offline metrics looked great, but online performance is worse and there's no obvious error.
>
> **Common causes:**
> - Training and serving teams duplicate feature logic independently, then diverge
> - Training reads from a batch warehouse; serving reads from a real-time stream with different null handling or aggregation windows
> - Preprocessing (scaler, encoder) is fit on training data but not serialized and reused at serving time
> - Data leakage: training features include future information (violates point-in-time correctness)
>
> **Prevention:**
> 1. Use a **feature store** — single feature definition computed once, served to both training and online inference
> 2. **Serialize preprocessing** — save the fitted scaler/encoder inside the model artifact (sklearn `Pipeline`, TF SavedModel with preprocessing layers)
> 3. **Log serving features** — log actual feature values sent to the model; compare distributions against training data periodically
> 4. **Shadow mode testing** — run the new model alongside the old one before go-live; compare feature distributions
>
> *See also:* [[mlops/concepts/training-serving-skew]], [[mlops/concepts/feature-store]]

---

## Card 2
**Q: What is the difference between data drift, concept drift, and label drift?**

> [!note]- Answer
> All three are forms of *distribution shift* that cause a deployed model to degrade over time — without any code change.
>
> | Type | What changes | Example |
> |---|---|---|
> | **Data drift** (covariate shift) | Distribution of input features X | User age skews older as app grows |
> | **Concept drift** | Relationship between X and target Y | Spam patterns change as spammers adapt to your classifier |
> | **Label drift** | Distribution of the output Y | Fraud rate increases during a recession |
> | **Prediction drift** | Distribution of model outputs | Model starts predicting one class disproportionately |
>
> **How to detect:**
> - Continuous features: Kolmogorov-Smirnov test or Population Stability Index (PSI)
> - Categorical features: Chi-squared test or PSI
> - PSI rule of thumb: < 0.1 = fine, 0.1–0.25 = monitor, > 0.25 = retrain
>
> **How to respond:** alert → investigate (which features?) → trigger retraining → validate new model → promote
>
> *See also:* [[mlops/concepts/data-drift]]

---

## Card 3
**Q: Design a feature store. What are its components and why do you need one?**

> [!note]- Answer
> **Why you need it:** Without a feature store, teams recompute the same features independently in training code and serving code — causing training-serving skew and duplicated effort.
>
> **Core components:**
>
> ```
> Raw data → Feature pipeline → Offline store  (training, batch scoring)
>                             → Online store   (real-time inference, low latency)
> ```
>
> - **Offline store**: historical feature values; backed by a data warehouse (BigQuery, Hive, S3 + Parquet). Used to build training datasets.
> - **Online store**: latest feature values; backed by a low-latency KV store (Redis, DynamoDB, Bigtable). Queried at prediction time in milliseconds.
> - **Feature pipelines**: compute features from raw data and write to both stores. Run on a schedule or event trigger.
> - **Feature registry**: catalog of feature definitions, ownership, and lineage.
>
> **Key design challenge — point-in-time correctness:** When building training datasets, you must use only features that were *available at the timestamp of each training example* — not features computed after the fact. Violating this causes data leakage.
>
> **Follow-up: what if a feature is not in the online store?** You have two options: (1) add it to the materialization pipeline, or (2) compute it on-the-fly at serving time (acceptable only for cheap, fast transforms).
>
> *See also:* [[mlops/concepts/feature-store]]

---

## Card 4
**Q: Walk me through how you would deploy a new ML model to production with zero downtime.**

> [!note]- Answer
> **Step 1 — Register the model**
> After training completes, register the artifact to the model registry with metrics and lineage. Mark it `Staging`.
>
> **Step 2 — Offline validation**
> Run automated checks: does the model beat the current production baseline on a held-out eval set? Does it pass latency/throughput requirements? Fail fast here.
>
> **Step 3 — Shadow mode deployment**
> Deploy the new model alongside production but route its predictions to a log, not users. Compare prediction distributions and feature distributions against the live model. Catch training-serving skew before any user impact.
>
> **Step 4 — Canary release**
> Route a small slice of traffic (e.g., 1–5%) to the new model. Monitor online metrics (CTR, conversion, error rate) vs. the control. Gradually increase traffic if metrics hold.
>
> **Step 5 — Full rollout**
> Shift 100% of traffic. Keep the old model warm for fast rollback.
>
> **Step 6 — Promote in registry**
> Mark the new model `Production`; mark the old model `Archived`.
>
> **Rollback:** re-promote the previous version in the registry; it's already warm.

---

## Card 5
**Q: What is continuous training (CT), and how does it differ from CI/CD?**

> [!note]- Answer
> **DevOps CI/CD:**
> - **CI**: automatically build and test code on every commit
> - **CD**: automatically deploy passing builds to production
> - Assumption: once deployed, software doesn't degrade on its own
>
> **MLOps adds CT:**
> - **CT (Continuous Training)**: automatically retrain and redeploy the model when triggered
> - Necessary because models *do* degrade on their own — the world changes, data distributions shift, user behavior evolves — without any code change
>
> **CT trigger strategies:**
> - **Schedule**: retrain weekly/nightly on fresh data
> - **Data volume**: trigger when N new labeled examples arrive
> - **Drift threshold**: trigger when PSI or KS-stat exceeds a threshold (see [[mlops/concepts/data-drift]])
> - **Performance degradation**: trigger when online metrics drop below a threshold
>
> **CT pipeline steps:** data ingestion → validation → feature engineering → training → evaluation vs. baseline → conditional promotion
>
> *Key interview point:* CT is not "retrain whenever someone feels like it." It's an automated, gated pipeline with the same rigor as CI/CD.
>
> *See also:* [[mlops/concepts/ml-pipeline]], [[mlops/patterns/ml-training-pipeline]]

---

## Card 6
**Q: Describe the MLOps maturity levels. What does moving from Level 0 to Level 1 look like?**

> [!note]- Answer
> *Framework from Google's MLOps whitepaper.*
>
> | Level | Name | What it means |
> |---|---|---|
> | **0** | Manual | Data scientists run notebooks manually. Models are deployed by hand. No reproducibility, no automation. |
> | **1** | Pipeline automation | Training is a versioned, automated pipeline. CT is enabled. Model registry exists. |
> | **2** | CI/CD pipeline automation | The pipeline *itself* is built, tested, and deployed automatically on code changes. Full software engineering discipline applied to the ML system. |
>
> **Level 0 → Level 1 transition (common interview question):**
> 1. Convert notebook code into modular, version-controlled Python steps
> 2. Wire steps into an orchestrated pipeline (Kubeflow, Vertex AI Pipelines, Metaflow, Airflow)
> 3. Add data validation at the pipeline entry point
> 4. Add experiment tracking (MLflow, W&B) to every training run
> 5. Add a model registry with staging/production lifecycle
> 6. Build a CT trigger (schedule or drift-based)
> 7. Add automated evaluation gating before promotion
>
> **Most companies are at Level 0 or early Level 1.** Interviewers at FAANG often ask "where would you start?" — the answer is: data validation and experiment tracking first, because they give immediate value and are low-risk to add.

---

## Card 7
**Q: How do you monitor a production ML model? What metrics do you track?**

> [!note]- Answer
> Monitoring a production model spans four layers:
>
> **1. Infrastructure metrics** (same as any service)
> - Latency (p50, p95, p99), throughput (QPS), error rate, CPU/GPU utilization
>
> **2. Input data health**
> - Feature distribution drift (PSI, KS-test per feature)
> - Missing value rates, schema violations
> - Volume anomalies (sudden drop in traffic = data pipeline issue)
>
> **3. Prediction health**
> - Prediction distribution drift (are outputs skewing to one class?)
> - Confidence score distribution
>
> **4. Model performance** (requires ground truth labels — often delayed)
> - Accuracy, AUC, precision/recall, RMSE depending on task
> - Track over rolling windows; alert on statistically significant degradation
>
> **The challenge:** ground truth is often delayed (days, weeks, or never). This is why input/prediction monitoring matters — it's a leading indicator you can act on *before* you have labels.
>
> **Alerting strategy:** set thresholds at two levels — *warn* (investigate) and *critical* (trigger CT pipeline or page on-call).
>
> *See also:* [[mlops/concepts/data-drift]]

---

## Card 8
**Q: How do you ensure reproducibility of a machine learning experiment?**

> [!note]- Answer
> Reproducibility means: given the same inputs, you can re-run a training job and get the same (or statistically equivalent) model. Required for debugging, auditing, and regulatory compliance.
>
> **Reproducibility checklist:**
>
> | What | How |
> |---|---|
> | **Code** | Log git commit SHA with every run |
> | **Data** | Version your dataset (DVC, Delta Lake, LakeFS); log dataset hash or version ID |
> | **Environment** | Pin library versions (`requirements.txt`, `poetry.lock`, or Docker image digest) |
> | **Random seeds** | Set and log seeds for `random`, `numpy`, `torch`/`tensorflow`, data shuffle |
> | **Hyperparameters** | Log all params via experiment tracker; never hardcode |
> | **Hardware** | Note GPU type — some ops are non-deterministic across hardware |
>
> **The practical answer for interviews:** use an experiment tracker (MLflow, W&B) that logs all of the above automatically on every run, and use the model registry to link deployed models back to their exact training run.
>
> **Gotcha:** full bit-for-bit reproducibility is often impossible with GPU training (non-deterministic CUDA ops). "Statistically equivalent" (same metrics within noise) is the practical goal.
>
> *See also:* [[mlops/concepts/experiment-tracking]]

---

## Card 9
**Q: How would you version ML models and safely roll back a bad deployment?**

> [!note]- Answer
> **Versioning via a model registry:**
> Every trained model is registered as an immutable artifact with:
> - Version number (auto-incremented)
> - Metrics from eval
> - Lineage: dataset version, git SHA, experiment run ID
> - Lifecycle stage: `Staging` → `Production` → `Archived`
>
> **Promotion flow:**
> ```
> Training pipeline → registers v42 as Staging
> Validation gate   → passes eval, promotes v42 to Production (v41 → Archived)
> Serving layer     → always loads the model tagged Production
> ```
>
> **Rollback procedure:**
> 1. Detect degradation via monitoring alerts
> 2. In the model registry, re-promote the previous good version (e.g., v41) to `Production`
> 3. Serving layer picks up the new tag on next refresh (or immediately if using polling/push)
> 4. v42 is demoted to `Archived` with a note
> 5. Post-mortem: what caused v42 to degrade? Data issue? Training bug?
>
> **Key design principle:** the serving layer should be *tag-driven*, not version-number-driven. It loads "whatever is tagged Production" — so rollback is just a registry state change, not a redeployment.
>
> *See also:* [[mlops/concepts/model-registry]]

---

## Card 10
**Q: Design an end-to-end ML pipeline for a recommendation system at FAANG scale.**

> [!note]- Answer
> *This is a system design question. Structure your answer in layers.*
>
> **Data layer**
> - Raw events (clicks, views, purchases) land in a data lake (S3/GCS)
> - Batch pipeline (Spark/Beam) computes offline features → feature store offline store
> - Streaming pipeline (Kafka + Flink) computes real-time features → feature store online store (Redis)
>
> **Training pipeline**
> - Trigger: daily schedule or drift detection
> - Steps: pull training data from feature store offline store (point-in-time correct) → validate → train candidate generation model + ranking model → evaluate vs. baseline → register to model registry if better
> - Experiment tracking: every run logged with params, metrics, dataset version
>
> **Serving layer**
> - Candidate generation: retrieve top-K candidates per user (ANN search, e.g., FAISS)
> - Ranking: score candidates with ranking model (online store feature lookup → model inference)
> - Latency budget: typically <100ms total; online feature lookup must be <10ms
>
> **Deployment strategy**
> - New model enters shadow mode → canary (1%) → gradual rollout
> - Serving layer is tag-driven off model registry
>
> **Monitoring**
> - Feature drift per feature group
> - CTR, conversion rate as online business metrics
> - Latency p99, error rate as infra metrics
> - Trigger CT pipeline if drift threshold exceeded
>
> *Concepts to reference:* [[mlops/concepts/feature-store]], [[mlops/patterns/ml-training-pipeline]], [[mlops/concepts/data-drift]], [[mlops/concepts/model-registry]]
