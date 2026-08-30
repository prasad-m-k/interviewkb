# LLM Serving System Design

**Difficulty:** Hard
**Topic:** [[ml/topics/ml-system-design]]
**Pattern:** [[ml/patterns/rag-pattern]], [[ml/patterns/rlhf]]
**Companies:** [[ml/companies/google-ml]]

---

## Problem

Design a production LLM serving system (like Gemini API or a Vertex AI generative endpoint) that handles millions of requests per day with low latency, high throughput, and cost efficiency.

---

## Clarifying Questions

- What model sizes are in scope? (7B / 70B / 100B+?)
- Is the primary use case chat (interactive, low TTFT) or batch processing (throughput over latency)?
- What is the SLA? (TTFT < 500ms? Streaming tokens at > 30 tokens/s?)
- Do we need multi-modal inputs (text + images)?
- Any fine-tuned or personalized model variants?

---

## Unique Challenges of LLM Serving

LLM inference is fundamentally different from traditional model serving:

| Challenge | Cause | Impact |
|---|---|---|
| Variable output length | Autoregressive generation; unknown until EOS | Can't pre-allocate memory |
| KV cache memory | Attention over all previous tokens is cached | 70B model with 8K context needs ~35GB just for KV cache |
| Memory-bandwidth bound | At batch size 1, decode is bandwidth-not-compute limited | GPU compute sits idle; throughput limited by HBM bandwidth |
| Long tail latency | Long prompts or long generations dominate p99 | Need scheduling to avoid head-of-line blocking |

---

## System Architecture

```
Client
  │ HTTP/gRPC
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  API Gateway / Load Balancer                                    │
│  • Auth, rate limiting, request validation                      │
│  • Route to appropriate model variant                           │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  Request Queue / Scheduler                                      │
│  • Priority queue (paid tier vs. free tier)                    │
│  • Continuous batching controller                               │
│  • Timeout / cancellation propagation                           │
└─────────────────────────────────────────────────────────────────┘
  │ batched requests
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  Inference Cluster (GPU/TPU Pool)                               │
│  • Tensor parallelism across GPUs for large models             │
│  • KV cache manager (paged memory — vLLM PagedAttention)        │
│  • Continuous batching: add/remove requests each iteration      │
│  • Speculative decoding for small model speedup                │
└─────────────────────────────────────────────────────────────────┘
  │ token stream
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  Response Streaming                                             │
│  • Server-Sent Events (SSE) or WebSocket for streaming tokens   │
│  • Safety filter on each token/chunk                           │
│  • Billing metering (input tokens + output tokens)             │
└─────────────────────────────────────────────────────────────────┘
```

---

## KV Cache Management

The KV cache is the biggest memory bottleneck. Every request keeps the entire attention state for all previously generated tokens.

**Naive approach:** allocate max_seq_len × kv_size memory per request upfront. Massive waste for short outputs; OOM for long ones.

**PagedAttention (vLLM):** inspired by OS virtual memory paging. KV cache is divided into fixed-size "pages" (blocks). Each request's KV cache is stored in non-contiguous pages; a page table maps logical to physical blocks.
- Eliminates internal fragmentation: memory allocated only for tokens that exist
- Enables KV cache sharing across requests with the same prefix (system prompt)
- ~90% memory utilization vs. ~30% with contiguous allocation

**Prefix caching:** if 1000 requests share the same system prompt ("You are a helpful assistant..."), compute the KV cache once and reuse it. Significant savings for chatbot applications.

---

## Batching Strategies

**Static batching:** all requests in a batch complete together. The batch waits for the slowest request. Throughput waste: a request that finished generating at step 50 sits idle until the last request finishes at step 500.

**Continuous batching (iteration-level scheduling):** at each token generation step, completed requests leave the batch immediately; new waiting requests join. No request waits for others. Standard in all modern serving frameworks (vLLM, TGI, TensorRT-LLM).

**Chunked prefill:** for long prompts, the prefill phase (processing the input) blocks the GPU for many steps. Chunked prefill interleaves prefill and decode, reducing TTFT spikes for other requests.

---

## Model Parallelism for Serving

For a 70B model (140GB in FP16), it cannot fit on a single A100 (80GB). Options:

- **Tensor parallelism (TP=2):** split each weight matrix across 2 GPUs. Both participate in every forward pass. Requires NVLink for low-latency communication. Standard choice for single-node serving.
- **Pipeline parallelism (PP):** split layers across nodes. High pipeline bubble for interactive use cases; better for batch processing.
- **TP × PP:** e.g., TP=4 within node, PP=2 across nodes for very large models.

---

## Latency Decomposition

```
TTFT (Time to First Token)
  = Time to receive request
  + Queue wait time
  + Prefill time (O(prompt_len²) compute)

TPS (Tokens Per Second, decode phase)
  = Memory bandwidth / (KV cache size per token + weight size per token)
  ← memory-bandwidth bound, not compute bound at small batch sizes

Total generation time ≈ TTFT + output_len / TPS
```

**Optimization targets:**
- Reduce TTFT: priority scheduling, chunked prefill, smaller models or quantization
- Increase TPS: larger batch size (increases GPU utilization), quantization (reduces memory bandwidth)
- Reduce output_len: streaming (shows output early without reducing length), speculative decoding (same length, faster)

---

## Speculative Decoding

Use a small draft model (e.g., 7B) to generate K candidate tokens speculatively. The large verifier model (70B) checks all K tokens in parallel (using its KV cache). Accept tokens where the verifier agrees; regenerate from the first disagreement.

**Speedup:** if the draft model accepts most tokens, K tokens are generated at the cost of 1 large model forward pass instead of K. Typical: 2-3× throughput improvement for long generation. Zero impact on output quality (mathematically equivalent to sampling from the large model alone).

---

## Model Quality vs. Cost Tradeoff

Not all requests need the largest model:

```
Router
  │
  ├── Simple queries → Small model (Gemma-2B, ~0.1× cost)
  │
  ├── Standard queries → Medium model (Gemini Nano, ~0.5× cost)
  │
  └── Complex reasoning → Large model (Gemini Pro/Ultra, 1× cost)
```

**Cascaded serving:** start with small model; if confidence is low (entropy of output tokens is high), escalate to larger model. Used in production to reduce cost without sacrificing quality.

---

## Safety Layer

Safety filters run on every request and response:
- **Input:** prompt injection detection, PII scrubbing, topic blocking
- **Output:** harmful content classifier (fine-tuned distilled model), PII redaction
- **Latency impact:** classifier must run in < 20ms; usually distilled models or specialized fast classifiers

---

## Napkin Math (Google Interview)

Assumptions: 1M requests/day, avg 200 input tokens, avg 500 output tokens, Gemini-70B scale model.

- **Throughput needed:** 1M / 86400 ≈ 12 requests/second
- **Compute per request:** prefill (200 tokens) + decode (500 tokens)
- **With A100 80GB, TP=2, FP16:** ~10-20 requests/second throughput (batch-dependent)
- **GPUs needed:** ~1-4 A100 pairs for this load (with continuous batching and quantization)
- **KV cache per request (4K context, 70B, 80 layers, 128 head_dim, FP16):** 80 × 2 × 4096 × 128 × 2 bytes ≈ 160MB per request

A100 80GB with TP=2 (160GB total) minus model weights (~140GB) leaves ~20GB for KV cache → supports ~125 concurrent requests at 160MB each. With shorter contexts or INT8, this doubles or quadruples.

---

## Evaluation

- **Latency:** TTFT p50/p99, TPS p50 (tokens/second)
- **Throughput:** total tokens generated per GPU per second
- **Quality:** model accuracy on benchmarks; human preference in A/B tests
- **Cost:** cost per 1K tokens (compute + memory + networking)
- **Availability:** uptime, error rate (failed generations, timeout rate)

---

## Key Insight

LLM serving is dominated by memory bandwidth, not compute. The fastest path to throughput improvement is: (1) larger effective batch size via continuous batching, (2) reduced memory footprint via quantization and PagedAttention, (3) speculative decoding for decode-heavy workloads.

---

## Sources
- [[ml/concepts/llm-fundamentals]]
- [[ml/concepts/distributed-training]]
- [[ml/patterns/rag-pattern]]
- [[ml/companies/google-ml]]
