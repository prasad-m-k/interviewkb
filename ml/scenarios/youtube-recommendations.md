# YouTube Video Recommendation System

**Difficulty:** Hard
**Topic:** [[ml/topics/ml-system-design]]
**Pattern:** Two-stage retrieval + ranking
**Companies:** [[ml/companies/google-ml]]

---

## Problem

Design a video recommendation system for YouTube. The system must serve personalized video recommendations to 2 billion logged-in users, with a catalog of 800 million videos (500 hours of video uploaded per minute).

---

## Clarifying Questions (ask first in interview)

- What is the primary objective: maximize watch time? CTR? Satisfaction? User return rate?
- Are we designing the homepage feed, the "Up Next" sidebar, or both?
- What is the latency budget? (p99 < 200ms is typical)
- Do we personalize for anonymous users?
- Any safety/content moderation constraints?

---

## Requirements

**Functional:**
- Given a user and context (current video, time of day, device), return a ranked list of ~20 videos
- Support cold start for new users and new videos
- Refresh recommendations in real-time as user watches

**Non-Functional:**
- Latency: < 200ms p99 for the entire request
- Scale: 2B users, 800M videos, ~1M requests/second
- Freshness: trending videos must surface within minutes of upload

---

## Architecture: Two-Stage Pipeline

This is the canonical Google architecture (described in the 2016 YouTube DNN paper by Covington et al.).

```
Request
   │
   ▼
┌──────────────────────────────────────────────────────────────┐
│  Stage 1: Candidate Retrieval (fast, coarse)                 │
│                                                              │
│  From 800M videos → retrieve 100–500 candidates             │
│                                                              │
│  Methods (run in parallel, merge):                           │
│  • Collaborative filtering (user embedding → ANN search)     │
│  • Context-based (recent watch history → similar videos)     │
│  • Trending / fresh content (real-time signals)              │
│  • Session-based (currently watching video → related)        │
└──────────────────────────────────────────────────────────────┘
   │ 100-500 candidates
   ▼
┌──────────────────────────────────────────────────────────────┐
│  Stage 2: Ranking (slow, precise)                            │
│                                                              │
│  Score each candidate with a heavy neural ranker             │
│  Features: user history, video metadata, context, cross-     │
│  feature interactions                                        │
│                                                              │
│  Multi-objective: combine watch time + satisfaction +        │
│  diversity + freshness via linear scalarization              │
└──────────────────────────────────────────────────────────────┘
   │ Ranked list of 20
   ▼
┌──────────────────────────────────────────────────────────────┐
│  Post-Ranking (rules + diversity)                            │
│                                                              │
│  • Deduplicate (no two videos from same channel in top 5)    │
│  • Safety filter (demonetized / policy violations)           │
│  • Freshness boost (recent videos get score boost)           │
│  • Topic diversity (avoid showing 20 cooking videos)         │
└──────────────────────────────────────────────────────────────┘
```

---

## Stage 1: Candidate Retrieval

### Collaborative Filtering via Two-Tower Network
- Train a bi-encoder: User Tower + Video Tower → embeddings in shared space
- User embedding input: watch history, search history, demographics
- Video embedding input: title, description, category, watch time stats
- Train with in-batch negatives (sampled softmax)
- Serve: pre-compute all video embeddings → FAISS / ScaNN index; query with user embedding at request time

**Why ANN, not exact search:** scanning 800M vectors per request at p99 < 200ms is not feasible with exact search. ScaNN (Google's ANN library) delivers top-100 recall ~95% with 5-10ms latency at this scale.

### Freshness
New videos have no interaction history. Fixes:
- Add video age as a feature in retrieval
- Dedicate a freshness retrieval source (recently uploaded + similar to watch history)
- Exploration policy: reserve 5-10% of candidates for new content

---

## Stage 2: Ranking

### Features
| Feature Group | Examples |
|---|---|
| User features | Watch time in last 24h, search history, subscriptions, device |
| Video features | Length, category, channel size, average watch %, age |
| Context | Time of day, day of week, device, country |
| Cross features | "User has watched 5 cooking videos this week" × "Video is cooking" |
| Historical | Has user watched this creator before? Completion rate on similar videos |

### Model
- Wide & Deep (Google 2016): memorization (wide/linear) + generalization (deep/MLP)
- Modern: Transformer-based ranking (learn interaction between candidate features + query)

### Training Labels
- **Watch time** (not binary click): using watch time as label avoids clickbait
- **Satisfaction** (thumbs up, post-watch survey): separate ranking objective
- **Not Interested** (explicit negative): strong negative signal

**Key insight: position bias.** Videos shown in position 1 get more clicks regardless of quality. Correct with Inverse Propensity Scoring (IPS) or by adding the impression position as a feature during training (then set position = 0 at inference to remove the bias).

---

## Multi-Objective Optimization

Maximizing only watch time produces engagement-bait. YouTube ranks on a combination:
```
final_score = w1 × watch_time + w2 × satisfaction + w3 × freshness - w4 × clickbait_penalty
```
Weights are tuned via A/B experiments against long-term user return rate, not just immediate CTR.

---

## Data & Training Pipeline

- **Labels:** user activity logs (impression, click, watch duration, satisfaction signal)
- **Training frequency:** daily re-train is standard; for rapid trend capture, online learning or micro-batch updates every hour
- **Training-serving skew risk:** features must be computed identically offline (batch) and online (serving). Feature store ([[mlops/concepts/feature-store]]) is the standard solution.

---

## Cold Start

| Scenario | Solution |
|---|---|
| New user (no history) | Trending videos; onboarding questions to elicit topics |
| New video (no interactions) | Use content features (title, description, thumbnail); trending channel boosts |
| New user + new video | Content-based retrieval; default to popularity |

---

## Evaluation

- **Offline:** AUC-ROC, NDCG@10, calibration of watch time predictions
- **Online A/B:** primary metric = total watch time per user per day; guardrail = satisfaction, diversity, creator fairness
- **Interleaving experiments:** for ranking changes, interleave two rankers in the same session; faster signal than A/B

---

## Failure Modes to Discuss

- **Filter bubble:** user watches one topic → all recommendations become that topic → satisfaction drops
  - Mitigation: diversity injection in post-ranking; explore-exploit policies
- **Feedback loop:** recommended videos become the training labels → popular videos dominate
  - Mitigation: counterfactual logging, off-policy evaluation
- **Clickbait:** thumbnails optimized for click, not satisfaction
  - Mitigation: watch time label instead of click; satisfaction survey
- **Misinformation amplification:** recommendation amplifies sensational content
  - Mitigation: authoritative source boost; content moderation at retrieval

---

## Key Insight

The two-stage pattern (retrieval + ranking) is universal because: no single model can both scan 800M items quickly AND score each with rich features accurately. Retrieval is the scalability lever; ranking is the quality lever.

---

## Sources
- [[ml/topics/ml-system-design]]
- [[ml/companies/google-ml]]
- [[mlops/concepts/feature-store]]
- [[mlops/concepts/training-serving-skew]]
