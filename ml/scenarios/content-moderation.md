# Content Moderation System (ML-Based)

**Difficulty:** Hard
**Topic:** [[ml/topics/ml-system-design]]
**Pattern:** Multi-stage pipeline, human-in-the-loop
**Companies:** [[ml/companies/google-ml]]

---

## Problem

Design an ML-powered content moderation system for a platform like YouTube or Google Search. The system must detect violating content (hate speech, CSAM, misinformation, spam) across billions of pieces of content, supporting both video and text, with latency targets ranging from real-time (live streaming) to batch (uploaded content).

---

## Clarifying Questions

- Which violation categories are in scope? (hate speech, spam, adult, CSAM, misinformation, copyright?)
- Is this for text, images, video, or all modalities?
- Is this pre-publish (block before going live) or post-publish (detect and remove)?
- What is the acceptable false positive rate (incorrectly removing legitimate content)?
- How should the system handle borderline cases?

---

## The Core Tension

Content moderation is not a standard ML optimization problem. The business tension is:
- **Too many false negatives** (missed violations): harm to users, regulatory risk, advertiser pullout
- **Too many false positives** (wrongly removed content): creator harm, chilling speech, PR damage

Google's approach: match the threshold and human review investment to the severity of the violation category.

```
Severity tiers (determine false positive tolerance):
  Tier 1 — CSAM, terrorism: near-zero false negatives, high false positive OK
  Tier 2 — Hate speech, dangerous misinformation: balanced; human review for borderline
  Tier 3 — Spam, policy gray areas: higher false positive tolerance; fast appeals
```

---

## System Architecture

```
Content Ingestion
     │
     ▼
┌──────────────────────────────────────────────────────────────────┐
│  Pre-Processing                                                   │
│  • Video: extract frames (1 fps), transcribe audio → text        │
│  • Text: normalize, language detection, translation if needed    │
│  • Hash-based dedup: perceptual hash (pHash) against known-bad  │
│    database (PhotoDNA for CSAM; hash-matching is O(1), exact)    │
└──────────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────────┐
│  Fast Classifiers (P99 < 50ms)                                   │
│  • Lightweight text classifier (DistilBERT / FastText)           │
│  • Image classifier (EfficientNet or ViT)                        │
│  • Output: confidence score + violation category                 │
│  • Decision: [auto-approve] [auto-remove] [escalate to Stage 2] │
└──────────────────────────────────────────────────────────────────┘
     │ borderline cases
     ▼
┌──────────────────────────────────────────────────────────────────┐
│  Heavy Classifiers (P99 < 2s)                                    │
│  • Multi-modal model: text + image/video frames (joint embedding)│
│  • Larger model (e.g., fine-tuned Gemini) for nuanced reasoning │
│  • Context-aware: does the video title match the content?        │
│  • Outputs updated confidence + category + severity              │
└──────────────────────────────────────────────────────────────────┘
     │ still borderline
     ▼
┌──────────────────────────────────────────────────────────────────┐
│  Human Review Queue                                              │
│  • Prioritized by: severity tier, viral potential, channel size  │
│  • Human annotators make final decision                          │
│  • Decision feeds back to training data                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Hash-Based Detection (Exact Matching)

For known-bad content (CSAM, terrorist manifestos), exact matching is always faster and more reliable than ML:
- **Perceptual hashing (pHash):** generates a compact fingerprint of an image/video. Similar images produce similar hashes (Hamming distance < threshold). O(1) lookup against a known-bad database.
- **PhotoDNA:** Microsoft's CSAM detection; computes a hash robust to resizing, color changes, minor edits.
- **Text hashing:** n-gram fingerprints for spam/duplicate content.

**Rule:** exact/near-exact matching before ML classifiers. Catches re-uploads of known-bad content with zero false negatives and zero latency overhead vs. ML.

---

## Classifier Design

### Text Moderation
- **Multilingual:** Google operates in 100+ languages. Options: multilingual BERT (mBERT), XLM-RoBERTa, or translate-then-classify. Translating-then-classifying in English is simple but loses nuance and introduces latency.
- **Label hierarchy:** a single "hate speech" label is too coarse. Multi-label: hate speech → (race, gender, religion, sexual orientation). Important for policy specificity and appeal handling.
- **Implicit vs. explicit:** "Go back to your country" is hate speech without explicit slurs. Context matters; lightweight models fail here. Escalate to larger LLM.

### Video Moderation
- Video = sequence of frames + audio + speech text + metadata (title, description, comments)
- **Multi-modal fusion:** independent classifiers per modality → aggregate scores → final decision
- **Temporal context:** a single disturbing frame in an educational video about history is not a violation. Models must consider surrounding frames and speech context.
- Key model: VideoMAE, Video-LLaMA for rich video understanding.

---

## Handling Class Imbalance

- Violations are rare events: spam is < 1%, severe violations are < 0.01% of content.
- Standard accuracy is meaningless (99.99% accuracy by predicting "all clean").
- Metrics: precision, recall, F1 per violation category. PR-AUC over thresholds.
- Training: weighted cross-entropy, oversampling violations, hard negative mining.
- See [[ml/concepts/class-imbalance]].

---

## Real-Time vs. Batch

| Use case | Modality | SLA | Approach |
|---|---|---|---|
| Live stream | Video + audio | Real-time, < 5s delay | Sliding window over 5s chunks; fast classifier only |
| Uploaded video | Video + text | < 5 min post-upload | Full pipeline; heavy classifier + human queue |
| Comment / post | Text | < 1s | Lightweight text classifier; shadow mode before launch |
| Search query | Text | < 10ms | Inline fast classifier within search serving |

---

## Evaluation

**Metrics:**
- **Recall by category and tier:** critical for high-severity violations (near-100% recall required for CSAM)
- **False Positive Rate:** wrong removals; drives creator trust and appeals volume
- **Time to Remove (TTR):** how quickly is violating content taken down after upload?
- **Appeals success rate:** proxy for false positive quality

**Measurement pitfall:** labels come from human reviewers who disagree. Inter-annotator agreement (Cohen's κ) must be measured; only use samples where reviewers agree as ground truth for automated metrics.

---

## Feedback Loop and Training Data Management

- **Human decisions → training data:** reviewers' decisions are the most valuable training signal. Pipeline must track which human decision label annotated which example.
- **Active learning:** prioritize human review for examples near the classifier's decision boundary. Maximizes label value per annotation hour.
- **Drift:** violation patterns evolve (new memes, coded language, adversarial evaders). Model retraining must be frequent. See [[mlops/concepts/data-drift]].
- **Adversarial content:** bad actors actively probe what gets detected and modify content to evade. Models must be retrained on evasion patterns.

---

## Responsible AI Considerations

Google interviews expect this — raise it proactively:

- **Disparate impact:** hate speech classifiers trained on internet text often have higher false positive rates on Black English or LGBTQ+ speech (confusing self-identification with slurs). Evaluate separately by demographic if proxy signals exist.
- **Appeal process:** every wrongful removal must have a fast, effective appeals path.
- **Human reviewer wellbeing:** exposure to CSAM and violent content causes PTSD. Google has programs for mandatory rotation, counseling.
- **Transparency:** creators should know why content was removed; policies must be legible.

---

## Key Insight

Content moderation is a risk management system, not a pure ML accuracy problem. The threshold is not set at max F1 — it is set by the cost of each error type for each violation category. High-severity violations justify high false positive rates. Design the human review layer as a first-class part of the system, not a fallback.

---

## Variants / Follow-Ups

- "How would you handle misinformation, where 'false' is contested?" → authoritative source weighting, third-party fact-checker integration
- "How would you prevent the system from being gamed by adversarial uploads?" → adversarial training, hash chaining, behavioral signals (account age, upload velocity)
- "How do you scale the human review queue during breaking news events?" → surge staffing prediction, priority routing by virality

---

## Sources
- [[ml/topics/ml-system-design]]
- [[ml/concepts/class-imbalance]]
- [[mlops/concepts/data-drift]]
- [[ml/companies/google-ml]]
