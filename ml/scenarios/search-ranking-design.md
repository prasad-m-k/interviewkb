# Scenario: Search Ranking System Design

**Category:** ML System Design
**Difficulty:** Hard
**Seen at:** Google, LinkedIn, Airbnb, DoorDash, Uber

## The Problem
Design a search ranking system. When a user types a query, return a ranked list of results. Optimize for relevance + business value.

## Framework

### 1. Requirements
- **Objective**: maximize relevance to query intent (and optionally: CTR, conversion)
- **Latency**: < 100ms end-to-end
- **Query types**: keyword, semantic (natural language), faceted (filters)
- **Scale**: billions of documents; millions of queries/day

### 2. Data and Labels

**Label collection challenge:** Search relevance labels are expensive.

| Method | Pros | Cons |
|---|---|---|
| **Explicit ratings** (crowdworkers rate query-doc pairs) | Clean; controlled | Expensive; doesn't reflect real users |
| **Click-through data** | Free; large scale | Position bias; CTR ≠ relevance |
| **Dwell time** | Stronger signal than click | Noisy; varies by query type |
| **Conversion** (purchase, apply, book) | Strongest signal | Sparse; delayed feedback |

**Handling position bias in clicks:**
- Items ranked #1 get 10× more clicks than #10 regardless of relevance
- Correct with **Inverse Propensity Scoring (IPS)**: weight each click by 1/P(click at position k)
- Or: use unbiased learning to rank with a bias model trained jointly

### 3. Architecture: Multi-Stage Retrieval + Ranking

```
Query
  │
  ▼
┌──────────────────────────────────────────────┐
│  RETRIEVAL                                   │
│  • BM25 (keyword matching, lexical)          │  ~10M → 1000 candidates
│  • Dense retrieval (bi-encoder + ANN)        │
│  • Both combined (hybrid)                    │
└──────────────────────────────────────────────┘
  │ ~1000 candidates
  ▼
┌──────────────────────────────────────────────┐
│  RE-RANKING (Learning to Rank)               │  1000 → 10 results
│  • Cross-encoder (query × doc jointly)       │
│  • Rich features: semantic, engagement, QD   │
└──────────────────────────────────────────────┘
  │ top 10
  ▼
Final ranked list
```

### 4. Retrieval Stage

**BM25 (TF-IDF variant):**
```
BM25(q, d) = Σᵢ IDF(qᵢ) × (f(qᵢ,d) × (k₁+1)) / (f(qᵢ,d) + k₁(1-b+b×|d|/avgdl))
```
- Good for exact keyword matching; fast (inverted index)
- Fails on synonyms, paraphrases, semantic intent

**Dense Retrieval (Bi-Encoder):**
```
score(q, d) = encode_query(q) · encode_doc(d)
```
- Embed query and document separately; ANN search over document embeddings
- Good for semantic understanding
- Retrieval model: BERT fine-tuned on (query, relevant_doc, negative_doc) triplets with contrastive loss

**Hybrid BM25 + Dense (best practice):**
- Run both in parallel; merge candidate lists (dedup + union)
- Both captures different relevant documents

### 5. Re-ranking: Learning to Rank

Three paradigms:

| Paradigm | What it learns | Loss | Example models |
|---|---|---|---|
| **Pointwise** | Predict absolute relevance score | MSE or BCE | Logistic regression on features |
| **Pairwise** | Predict which of two docs is more relevant | Pairwise hinge / RankNet | RankNet, LambdaRank |
| **Listwise** | Optimize entire ranked list directly | LambdaMART, ListNet | LambdaMART, NDCG optimization |

**LambdaMART** (XGBoost variant for ranking): state-of-art for feature-based ranking; directly optimizes NDCG.

**Cross-encoder re-ranker (neural):**
```
Input: [CLS] query [SEP] document [SEP]
→ BERT encoder → [CLS] embedding → linear → relevance score
```
- Processes query and document jointly → captures interactions
- 10-100× more expensive than bi-encoder → only feasible on small candidate set

### 6. Features for the Ranker

| Feature Category | Examples |
|---|---|
| **Query-document match** | BM25 score, exact match count, query term coverage |
| **Semantic similarity** | Cosine similarity of embeddings |
| **Document quality** | PageRank/authority, freshness, spam score, content length |
| **User-specific** | User's click history on this query, personalization signal |
| **Context** | Time of day, device, location |
| **Engagement** | Historical CTR for doc on this query, avg dwell time |

### 7. Evaluation

**Offline:**
- **NDCG@10**: discounted cumulative gain; position-aware; industry standard
- **MRR**: mean reciprocal rank; good for tasks where one correct answer exists
- **MAP**: mean average precision; for multiple relevant results

**Online:**
- **CTR** (click-through rate): measure carefully for position bias
- **Time to click** (faster = user found what they wanted)
- **Zero-result rate**: % of queries that return no useful results
- **Session abandonment rate**: user leaves without clicking

### 8. Query Understanding
Before ranking, understand the query:
- **Intent classification**: informational / navigational / transactional
- **Entity recognition**: "restaurants in SF" → location-aware search
- **Spell correction**: handle typos with noisy channel model or seq2seq
- **Query expansion**: add synonyms or related terms to improve recall

## Sources
- [[ml/topics/ml-system-design]]
- [[ml/concepts/embeddings]]
- [[ml/topics/nlp]]
- [[ML overview]]
