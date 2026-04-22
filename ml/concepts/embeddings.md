# Embeddings

**Topic:** [[ml/topics/nlp]], [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/attention-mechanism]], [[ml/concepts/transformers]]

## What it is
A learned dense vector representation of discrete objects (tokens, entities, items, users). Maps high-cardinality discrete inputs into a continuous vector space where semantic similarity corresponds to geometric proximity.

## Word Embeddings

### Word2Vec (Mikolov et al., 2013)
Two training objectives, both predict context from target or vice versa:
- **CBOW** (Continuous Bag of Words): predict center word from surrounding context
- **Skip-gram**: predict surrounding context from center word (better for rare words)

Training trick: **negative sampling** — instead of softmax over full vocabulary, train binary classifiers for observed pairs vs. random noise pairs. Makes training O(k) per example, not O(V).

**Properties of learned embeddings:**
```
king - man + woman ≈ queen
paris - france + italy ≈ rome
```
Linear structure: analogies are encoded as vector offsets.

### GloVe (Global Vectors)
Factorizes the log co-occurrence matrix. Combines global (count-based) + local (window-based) information. Often similar performance to Word2Vec.

### Limitations of Static Embeddings
- One vector per word regardless of context
- "bank" (river) = "bank" (financial) — same embedding
- Solution: contextual embeddings

## Contextual Embeddings (BERT, etc.)
Every token gets a different embedding depending on its context:
```python
from transformers import BertTokenizer, BertModel
tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
model = BertModel.from_pretrained('bert-base-uncased')

text = "The bank robber ran"
inputs = tokenizer(text, return_tensors='pt')
outputs = model(**inputs)
# outputs.last_hidden_state: [batch, seq_len, 768]
# Each token has a 768-dim contextual embedding
```

For sentence-level embedding: use [CLS] token output or mean-pool all token embeddings.

## Sentence Embeddings (SBERT)
Plain BERT [CLS] embeddings are not good sentence embeddings (not contrastive-trained for semantic similarity). Use **Sentence-BERT** (SBERT):
- Fine-tuned with siamese network + NLI data
- Cosine similarity between embeddings is meaningful
- `sentence-transformers` library; models: `all-MiniLM-L6-v2`, `all-mpnet-base-v2`

## Item / User Embeddings
Used in recommendation systems:
- **Collaborative filtering**: factorize user-item matrix; each user/item gets an embedding; predict rating as dot product
- **Two-tower model**: separate encoder for user features and item features; train so relevant user-item pairs have high cosine similarity

## Embedding Tables (for tabular / categorical)
```python
# PyTorch embedding table
import torch.nn as nn
embedding = nn.Embedding(num_embeddings=1000, embedding_dim=32)
# Lookup: embedding(torch.tensor([5, 23, 7]))  → [3, 32] tensor
```
- Row = learned embedding for each category index
- For user_id with 1M users → 1M × 64 embedding table = 256MB
- Initialized randomly; learned jointly with the rest of the model

## Similarity Search (Vector Databases)
For retrieval at scale (semantic search, ANN for recommendations):

1. **Exact search**: dot product or cosine over all vectors — O(n·d); only feasible for small n
2. **Approximate Nearest Neighbor (ANN)**:
   - **Faiss** (Facebook): HNSW, IVF-PQ; GPU-accelerated; best for in-process
   - **HNSW**: graph-based; O(log n) search; great recall
   - **IVF** (Inverted File): cluster vectors; search only nearest clusters; fast but lower recall
   - **Product Quantization (PQ)**: compress vectors to save memory; small accuracy loss
3. **Vector DBs**: Pinecone, Weaviate, Qdrant, Milvus — add persistence, filtering, metadata

**Recall@k vs. Latency trade-off**: higher recall requires searching more candidates → higher latency.

## Common interview angles
- Why can't you use one-hot encoding for word tokens in a neural network? (1M-dimensional sparse vector; no similarity structure; can't generalize)
- What are the three differences between Word2Vec and BERT embeddings? (Word2Vec: static, non-contextual, trained unsupervised on local windows; BERT: contextual, bidirectional, fine-tuned on downstream tasks)
- How do you build a semantic search system? (embed queries and documents with SBERT; store document embeddings in vector DB; ANN search at query time)
- What is the "embedding collapse" problem in contrastive learning? (all embeddings converge to same point; fix with hard negatives, projection head, or momentum encoder)
- How do you handle out-of-vocabulary items in a recommendation embedding table? (OOV embedding, hash bucketing, or fallback to content-based features)

## Sources
- [[ML overview]]
