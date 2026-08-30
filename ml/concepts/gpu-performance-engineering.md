# GPU Performance Engineering

**Topic:** [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/distributed-training]], [[ml/scenarios/llm-service-design]], [[ml/concepts/ml-compilers-and-runtimes]]
**Tags:** #ResponsibleAI

## What it is

The discipline of diagnosing and closing the gap between a GPU's theoretical peak performance and what a real AI workload actually achieves. This sits between framework code (a PyTorch `matmul` call) and hardware (streaming multiprocessors, HBM memory controllers) — the layer an "AI Frameworks" team owns when the JD says "benchmark, profile, debug, and optimize large language model training and inference workloads across GPUs and Microsoft hardware."

## How it works

### The roofline model — the first question to ask

```
Performance is bounded by whichever is smaller:
  compute_bound  = peak FLOPs of the chip
  memory_bound   = memory_bandwidth × arithmetic_intensity

Arithmetic intensity (AI) = FLOPs performed / bytes moved from memory

  AI below the chip's "ridge point"  → MEMORY-BANDWIDTH BOUND
    (the chip's math units sit idle waiting for data)
  AI above the ridge point           → COMPUTE BOUND
    (the chip is doing as much math as physically possible)
```

This is the same finding as the decode-phase analysis in [[ml/scenarios/llm-service-design]] (autoregressive decode at batch size 1 is memory-bandwidth bound, not compute bound) — generalized into the diagnostic tool you reach for FIRST on any slow kernel, before profiling anything. Compute-bound and memory-bound problems have completely different fixes; optimizing the wrong one wastes days.

### CUDA vs ROCm vs Triton — the hardware-portability landscape

```
CUDA (NVIDIA)
  Vendor-specific, lowest-level, most mature tooling (Nsight).
  Kernels written here are NOT portable to AMD hardware.

ROCm / HIP (AMD)
  AMD's CUDA-equivalent stack. HIP is source-level similar to
  CUDA (often a near-mechanical port), but still a SEPARATE
  codebase to write, test, and tune per vendor.

Triton (vendor-neutral)
  A Python-embedded DSL that compiles down to GPU code for
  multiple backends. The key portability lever for a team that
  must support "NVIDIA and AMD GPUs and Microsoft silicon" (this
  JD's exact language) without hand-writing and maintaining a
  separate kernel per vendor. Triton is also the backend
  TorchInductor emits into inside torch.compile — see
  [[ml/concepts/ml-compilers-and-runtimes]].
```

The practical implication for a multi-hardware platform team: prefer writing performance-critical kernels in Triton (or relying on a compiler that emits Triton) over hand-rolled CUDA/HIP wherever the perf ceiling allows it, because a hand-tuned CUDA kernel is a maintenance liability the moment the platform needs to also run on AMD or Microsoft's own silicon (Maia).

### Kernel fusion — the highest-leverage fix for memory-bound workloads

```
Unfused (3 kernel launches, 3 round-trips to HBM):
  x1 = relu(x0)        # read x0, write x1
  x2 = x1 * scale       # read x1, write x2
  x3 = x2 + bias         # read x2, write x3

Fused (1 kernel launch, 1 round-trip to HBM):
  x3 = fused_relu_scale_bias(x0)   # read x0, write x3 once
```

Every intermediate tensor written to and read back from HBM is bandwidth spent on nothing but data movement between two math operations. Fusion is the single most common fix for a memory-bandwidth-bound elementwise/normalization-heavy workload — it's why "does the compiler fuse this" is a recurring practical question, not just a theory question.

### Precision and Tensor Cores

Modern GPUs' peak advertised FLOPs assume Tensor Core execution at FP16/BF16/FP8 with the right data layout. A kernel running plain FP32 math on Tensor-Core-capable hardware is leaving most of the chip's throughput unused — this is why [[ml/concepts/distributed-training]]'s mixed-precision section is not just a memory optimization, it's also a raw-throughput one.

### Profiling tools — finding which bound you're actually hitting

```
NVIDIA Nsight Systems   → timeline view: what's running when,
                           gaps = idle GPU time (often launch
                           overhead or host-device sync stalls)
NVIDIA Nsight Compute    → per-kernel deep dive: achieved vs peak
                           FLOPs, achieved vs peak bandwidth,
                           occupancy, register/shared-memory
                           pressure
PyTorch Profiler          → framework-level view: op-by-op time,
                           CPU-vs-GPU time, memory timeline —
                           first stop before dropping to Nsight
AMD rocprof                → ROCm-equivalent of the above
```

A common failure mode profiling reveals that a napkin-math estimate misses entirely: **launch-overhead-bound** workloads — many small kernels where GPU launch latency (microseconds each) dominates actual compute time. Neither "compute-bound" nor "memory-bound" fixes apply here; the fix is fusion or graph capture (see [[ml/concepts/ml-compilers-and-runtimes]]) to reduce the number of launches, not faster math.

## Complexity

Not algorithmic. Think in terms of **utilization percentage**: achieved FLOPs / peak FLOPs, or achieved bandwidth / peak bandwidth, whichever the roofline model says is the binding constraint. A100 in FP16 peaks near 312 TFLOPS; a kernel achieving 30 TFLOPS on a compute-bound workload is at ~10% utilization — that gap is the optimization target, not an abstract "make it faster."

## When to use

```
- Any role turning a working-but-slow model into a production-
  efficient one (exactly this JD's "reduce deployment time and
  hardware footprint")
- Any platform team supporting multiple GPU vendors, where
  vendor-portable tooling (Triton, compiler-generated kernels)
  is preferred over hand-written CUDA/HIP per vendor
- Diagnosing an unexplained throughput ceiling in distributed
  training (see [[ml/concepts/distributed-training]]'s Q6:
  "throughput doesn't scale beyond 128 GPUs" — the roofline model
  plus a profiler is exactly how you'd start that investigation)
```

## Common interview angles

```
Q: "Your training job uses only 20% of the GPU's theoretical peak
    FLOPs. Walk through your diagnostic process."
A: Start with the roofline model, not the profiler: compute the
   workload's arithmetic intensity and compare to the chip's ridge
   point. If memory-bound, look for fusion opportunities and
   unnecessary HBM round-trips first — more compute optimization
   won't help a memory-bound kernel. If compute-bound and still
   under peak, check precision (is it actually hitting Tensor
   Cores?) and occupancy (Nsight Compute) before assuming the
   algorithm itself needs to change.

Q: "Why would a platform team choose Triton over hand-written CUDA
    for a new fused kernel, even though CUDA usually gets a few
    percent more performance in expert hands?"
A: Maintenance cost across hardware targets. A CUDA kernel is
   NVIDIA-only; supporting AMD or in-house silicon means a second
   (or third) hand-written implementation to keep functionally and
   performance-equivalent forever. Triton (or a compiler emitting
   Triton) trades a few points of peak performance for one
   maintained implementation across vendors — usually the right
   trade for a platform, even if not for a single hand-tuned
   showcase kernel.

Q: "A workload has many small ops and low GPU utilization even
    though no single kernel looks slow in the profiler. What's
    happening?"
A: Likely launch-overhead-bound, not compute- or memory-bound —
   each kernel launch has fixed overhead, and with enough small ops
   that overhead dominates. Fusion or graph capture
   ([[ml/concepts/ml-compilers-and-runtimes]]) reduces the number
   of launches; this is invisible to the roofline model because the
   roofline model assumes the kernel is actually running, not
   waiting to be launched.
```

## Examples

```
Diagnostic order for "model is slower than expected on GPU":
  1. Roofline: compute arithmetic intensity, compare to ridge point
  2. Nsight Systems / PyTorch Profiler timeline: look for gaps
     (idle GPU) — host-device sync? data loading stall? launch
     overhead from many small ops?
  3. Nsight Compute on the hottest kernel: achieved vs peak
     FLOPs/bandwidth, occupancy, precision actually used
  4. Fix: fusion (memory-bound), precision/Tensor Core usage
     (compute-bound underutilized), or graph capture (launch-bound)
```

## Sources
- [[ml/concepts/distributed-training]]
- [[ml/scenarios/llm-service-design]]
- [[ml/concepts/ml-compilers-and-runtimes]]
- [[ml/companies/microsoft-ai-frameworks]]
