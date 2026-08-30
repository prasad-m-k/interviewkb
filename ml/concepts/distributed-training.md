# Distributed Training

**Topic:** [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/llm-fundamentals]], [[ml/concepts/gradient-descent]], [[ml/concepts/batch-normalization]]

---

## What it is

Distributed training splits the work of training a neural network across multiple devices (GPUs/TPUs). Required when: the model is too large for one device's memory, or training on a single device is prohibitively slow.

The three orthogonal parallelism strategies are **data parallelism**, **model parallelism**, and **pipeline parallelism**. Modern systems use all three simultaneously (3D parallelism).

---

## Data Parallelism

**Idea:** each device holds a full copy of the model. The dataset is sharded across devices. Each device computes gradients on its local mini-batch. Gradients are aggregated (averaged) across all devices before the weight update.

**Gradient aggregation strategies:**
- **AllReduce (Ring-AllReduce):** each GPU sends gradients to its neighbor in a ring. After two passes (reduce-scatter + all-gather), every GPU has the sum. Used by PyTorch DDP, Horovod. Communication cost: `O(2 × N × gradients / N)` = independent of device count, which is why it scales.
- **Parameter Server:** dedicated server nodes aggregate gradients from workers. Simpler but bandwidth-limited at the PS.

**When to use:** model fits in one GPU's memory. Scale from 1 to 1000s of GPUs with minimal code change.

**Limitation:** if the model itself exceeds one GPU's VRAM (e.g., 70B parameter model needs ~140GB in FP16, but an A100 has 80GB), data parallelism is insufficient.

---

## Model Parallelism (Tensor Parallelism)

**Idea:** split individual weight matrices across multiple devices. For a large matrix multiply `Y = XW`, partition `W` column-wise: GPU0 holds columns 0–511, GPU1 holds columns 512–1023. Both compute partial results in parallel; outputs are gathered.

**Used for:** attention heads (split across devices), feed-forward layers (split the hidden dimension).

**Communication:** requires an AllReduce or AllGather at the end of each layer, after every forward/backward pass. High communication frequency → needs NVLink (high-bandwidth GPU interconnect), not slow PCIe or network.

**Megatron-LM** (NVIDIA) is the reference implementation. Used in training GPT-3 scale models.

---

## Pipeline Parallelism

**Idea:** partition the model's layers across devices. Device 0 holds layers 0–7, device 1 holds layers 8–15, etc. Training proceeds in micro-batches: Device 0 processes micro-batch 1 and passes activations to Device 1, while Device 0 starts processing micro-batch 2.

**Pipeline bubble:** at startup and shutdown, some devices are idle waiting for activations. Effective when the number of micro-batches >> number of pipeline stages. GPipe and PipeDream are the key papers.

**When to use:** very deep models; or when tensor parallelism communication overhead is too high (e.g., slower interconnects).

---

## 3D Parallelism

Modern large model training (GPT-4, Gemini) combines all three:
- **Pipeline parallelism** across nodes (slow inter-node networking)
- **Tensor parallelism** within a node (fast NVLink)
- **Data parallelism** across replica groups

This is sometimes called **3D parallelism** (DeepSpeed + Megatron-LM).

---

## ZeRO Optimizer (DeepSpeed)

The memory bottleneck in data parallelism is that every device stores full optimizer states (e.g., Adam stores 3× the model: weights + first moment + second moment).

ZeRO (Zero Redundancy Optimizer) shards optimizer state, gradients, and optionally model parameters across data-parallel ranks:
- **ZeRO Stage 1:** shard optimizer states only (4× memory reduction)
- **ZeRO Stage 2:** shard optimizer states + gradients (8× reduction)
- **ZeRO Stage 3 / FSDP:** shard optimizer states + gradients + parameters (full sharding). PyTorch FSDP implements this.

**Trade-off:** more sharding = less memory per device = higher communication volume.

---

## Gradient Accumulation

Simulates a large batch without fitting it in memory:
```python
for i, (X, y) in enumerate(loader):
    loss = model(X, y) / accumulation_steps
    loss.backward()  # accumulate gradients
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```
Effective batch size = micro-batch size × accumulation steps. Common when GPU memory limits micro-batch to 1–4 samples.

---

## Mixed Precision Training

Training in FP32 is memory-intensive. Mixed precision (AMP) keeps:
- **FP16 / BF16:** forward pass, backward pass (2 bytes/param vs 4)
- **FP32 master copy:** optimizer state (to avoid precision loss in weight updates)

**BF16 vs FP16:**
- BF16 has same exponent range as FP32 (avoids overflow/underflow) but lower mantissa precision
- BF16 is preferred for training; FP16 requires gradient scaling (loss scaling) to avoid underflow
- Google TPUs are optimized for BF16

---

## Communication Bottleneck

At scale, communication dominates compute time. Key ratios:
- NVLink bandwidth: ~600 GB/s (within a node, 8× A100s)
- InfiniBand (inter-node): ~200 Gb/s = ~25 GB/s
- At 70B model: gradient size ~140GB in FP16 → AllReduce at ~25 GB/s takes ~5.6s per step without pipelining

Techniques to mitigate:
- Gradient compression (PowerSGD, Top-K sparsification)
- Overlap communication with computation (PyTorch DDP does this automatically)
- Reduce precision of gradients during AllReduce (FP16 gradients)

---

## Google TPU Architecture

Google uses TPUs (Tensor Processing Units) instead of GPUs. Key differences:
- **Systolic array** architecture: optimized for matrix multiply; cannot do dynamic control flow efficiently
- **XLA compiler:** requires static computation graphs (shapes must be known at compile time); JAX is designed for this
- **HBM bandwidth:** TPU v4 has ~1.2 TB/s; very memory-bandwidth-bound tasks benefit more
- **Pod connectivity:** TPU pods are interconnected via 3D torus topology with high bandwidth; eliminates the NVLink vs. InfiniBand distinction

For Google interviews: knowing that TPUs favor **fixed-shape, compiled graphs** and that **JAX/XLA** is the preferred framework signals genuine depth.

---

## Complexity Summary

| Strategy | Memory per device | Communication frequency | Code complexity |
|---|---|---|---|
| Data Parallelism | Full model copy | Once per step (AllReduce) | Low |
| Tensor Parallelism | 1/N of each layer | Every forward/backward pass | High |
| Pipeline Parallelism | 1/N of layers | Micro-batch boundaries | Medium |
| ZeRO / FSDP | 1/N of everything | More frequent than DP | Medium |

---

## Common Interview Angles

1. "What's the difference between data parallelism and model parallelism? When would you use each?"
2. "Explain the AllReduce operation and why Ring-AllReduce scales better than a parameter server."
3. "A 70B model needs 140GB in FP16. Your GPUs have 80GB each. How would you train it?"
4. "What is ZeRO and how does it differ from standard data parallelism?"
5. "Why does Google prefer BF16 over FP16 for training on TPUs?"
6. "You're training a model on 512 GPUs and notice that throughput doesn't scale beyond 128. What's likely happening?"

---

## Sources
- [[ml/concepts/gradient-descent]]
- [[ml/concepts/llm-fundamentals]]
