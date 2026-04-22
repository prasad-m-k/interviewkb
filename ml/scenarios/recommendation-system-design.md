# Scenario: Recommendation System Design

**Category:** ML System Design
**Difficulty:** Hard
**Seen at:** Netflix, YouTube, Spotify, Amazon, Meta, TikTok, LinkedIn

## The Problem
Design a recommendation system for a video streaming platform (100M users, 10M videos). Goal: maximize watch time.

## Framework

### 1. Requirements
- **Objective**: maximize watch time (not just clicks — optimize for engagement quality)
- **Latency**: < 100ms for homepage load (serving budget)
- **Scale**: 100M users × 10M videos = 10^15 possible user-item pairs
- **Freshness**: new content and user preferences should be incorporated
- **Constraints**: no recommending already-watched; diversity; cold-start for new users/items

### 2. Data
| Signal | Type | Strength |
|---|---|---|
| Watch completion % | Implicit positive | Strong |
| Explicit rating (thumbs up/down) | Explicit | Very strong |
| Click-through (but < 30% watched) | Weak positive / negative | Weak |
| Scroll past | Implicit negative | Moderate |
| User demographics, device | Context | Auxiliary |
| Video metadata (genre, title, duration) | Item feature | Cold-start |

**Label design:** positive = watch > 70% completion; negative = click + watch < 10%, or explicit dislike. This filters out "accidental clicks."

### 3. Architecture: Two-Stage Pipeline

```
10M Videos
    │
    ▼
┌─────────────────────────────────┐
│  Stage 1: RETRIEVAL             │  target: 100-500ms latency
│  Candidate generation           │  output: ~200 candidates
│  • Two-tower ANN (~500)         │
│  • Collaborative filter (~500)  │
│  • Trending / fresh content     │
│  • Previously liked genres      │
└─────────────────────────────────┘
                 │
                 ▼ ~200 candidates
┌─────────────────────────────────┐
│  Stage 2: RANKING               │  target: <50ms for 200 items
│  • Heavy neural ranker          │
│  • 100+ features per item       │
│  • Predicts P(watch > 70%)      │
└─────────────────────────────────┘
                 │
                 ▼ top 20
┌─────────────────────────────────┐
│  Stage 3: RE-RANKING / FILTERS  │  business rules
│  • Diversity (no 5 consecutive  │
│    action movies)               │
│  • Filter already-watched       │
│  • Safety / policy filters      │
└─────────────────────────────────┘
```

### 4. Model Design

#### Two-Tower (Retrieval)
```
User tower:                    Item tower:
  user_id embedding              item_id embedding
  + age, gender, country         + genre, duration
  + recent watch history (avg)   + creator embedding
  + device type                  + release date
       │                              │
  [MLP layers]                 [MLP layers]
       │                              │
  user_vec (64-128d)           item_vec (64-128d)
                  └───score: dot product───┘
```

**Training:** contrastive loss — maximize dot product for watched videos, minimize for non-watched. Use in-batch negatives + hard negatives.

**Serving:** precompute all item vectors; build HNSW index over item embeddings; ANN search with user vector at query time.

#### Neural Ranker (Stage 2)
Features per user-item pair:
- User features: embedding, history stats, demographics
- Item features: embedding, metadata, freshness
- Cross features: user-item interaction (did user watch similar videos?)
- Context: time of day, device, session length

Architecture: wide & deep (wide = memorization from cross features; deep = generalization from dense features).

### 5. Handling Key Challenges

**Cold Start for new users:**
- Use context signals (device, country, time)
- Serve popular content in their language/region
- Quick onboarding questions to capture initial preferences

**Cold Start for new videos:**
- Use content-based features (title embedding, genre, creator history)
- Inject new content randomly to a small fraction of users to gather feedback
- Promote through topic/genre buckets

**Position Bias:**
- Items shown higher in the list get more clicks regardless of quality
- Fix: add position as a feature during training; at inference, set position to 0 (or a fixed baseline) so the model scores independent of position
- Or use Inverse Propensity Scoring (IPS) to re-weight training examples

**Feedback Loop:**
- Recommending only what users already like → filter bubble
- Fix: ε-greedy exploration (occasionally recommend outside predicted preferences); diversity constraints in re-ranking

### 6. Evaluation

| Stage | Metric |
|---|---|
| Retrieval | Recall@K (are relevant items in the top-K candidates?) |
| Ranking | NDCG@10, MAP |
| Overall offline | Weighted average precision |
| Online A/B | Watch time per session, 7-day retention, diversity score |

**Key**: online metrics (watch time) should be the north star; offline NDCG is a proxy.

### 7. Monitoring
- Score distribution drift (model staleness)
- Coverage: % of catalog shown to users (diversity health)
- Feedback loop detection: Is the recommendation entropy decreasing over time?
- Latency P99 for retrieval and ranking stages

## Sources
- [[ml/topics/ml-system-design]]
- [[ml/concepts/embeddings]]
- [[ML overview]]
