# Natural Language Processing

**Related concepts:** [[ml/concepts/attention-mechanism]], [[ml/concepts/transformers]], [[ml/concepts/embeddings]], [[ml/concepts/loss-functions]]

## The NLP Stack (modern)

```
Raw text
  → Tokenization (BPE / WordPiece)
  → Token embeddings + Positional encoding
  → Transformer layers (self-attention + FFN)
  → Task-specific head
  → Output (classification, generation, span extraction...)
```

## Tokenization
- **Word-level**: simple; large vocabulary; can't handle OOV words
- **Character-level**: handles OOV; long sequences; slower
- **Subword (BPE, WordPiece, SentencePiece)**: balance between word and char; BERT/GPT use this
  - BPE: merge most frequent byte pairs iteratively
  - WordPiece: maximize likelihood of training data (used in BERT)
  - `##suffix` in BERT vocab = continuation of a word

## Pre-trained Language Models

### BERT (Encoder-only)
- Bidirectional: attends to both left and right context
- Pre-trained on: Masked Language Model (MLM) + Next Sentence Prediction (NSP)
- Best for: classification, NER, QA (extractive), semantic similarity
- Fine-tuning: add a task head on [CLS] token or token representations

### GPT / GPT-2 / GPT-3/4 (Decoder-only)
- Autoregressive: attends to left context only (causal mask)
- Pre-trained on: next token prediction
- Best for: generation, completion, few-shot learning
- Inference: sample token by token (temperature, top-k, top-p/nucleus sampling)

### T5 / BART (Encoder-Decoder)
- Seq2seq: encoder reads input; decoder generates output
- Pre-trained on: denoising objectives (span corruption in T5, token masking in BART)
- Best for: summarization, translation, data-to-text

## Fine-tuning Strategies
| Strategy | When | Trade-off |
|---|---|---|
| **Full fine-tune** | Large task-specific dataset (>10K examples) | Best performance; catastrophic forgetting risk |
| **Frozen encoder** | Very small dataset; quick prototyping | Fast; worse performance |
| **LoRA / QLoRA** | Large model; limited GPU memory | Near full fine-tune quality; ~10× fewer trainable params |
| **Prompt tuning / Prefix tuning** | Frozen model; soft prompts | Minimal storage per task; lower quality than full FT |
| **In-context learning (ICL)** | No fine-tuning; few examples in prompt | Zero infrastructure; not reliable on complex tasks |

## Key NLP Tasks and Approaches

| Task | Approach | Model |
|---|---|---|
| **Text classification** | [CLS] token → linear head | BERT fine-tune |
| **NER** | Token-level classification | BERT fine-tune on BIO/IOB tags |
| **QA (extractive)** | Predict start/end span | BERT with QA head |
| **Summarization** | Seq2seq generation | T5, BART, GPT |
| **Semantic similarity** | Cosine of sentence embeddings | SBERT (Sentence-BERT) |
| **Information retrieval** | Bi-encoder (embed query + doc); re-ranker | Bi-encoder + cross-encoder pipeline |

## Retrieval-Augmented Generation (RAG)
```
Query → Encode query → ANN search in vector DB → Retrieve top-k chunks
     → Concatenate with query → LLM generates answer grounded in retrieved text
```
Reduces hallucination by grounding generation in retrieved facts. Key components: chunking strategy, embedding model, vector DB (Faiss, Pinecone, Weaviate), reranker.

## Common Interview Angles
- BERT vs. GPT — when to use which? (BERT: understanding tasks; GPT: generation tasks)
- What is the attention complexity and why does it matter? (O(n²) — long documents are expensive; solutions: sliding window, BigBird, Longformer)
- How do you handle out-of-vocabulary words with WordPiece tokenization? (always tokenized to known subwords; no true OOV)
- What is catastrophic forgetting and how do you mitigate it in fine-tuning? (fine-tuning overwrites pre-trained weights; use low LR, LoRA, or continual pre-training)
- How does sampling temperature affect generation? (temperature → 0: argmax/greedy; temperature → ∞: uniform; temperature 0.7-1.0: typical range)
- What is perplexity? (exp(cross-entropy loss); lower = better language model; not a good metric for downstream tasks)

## Sources
- [[ML overview]]
