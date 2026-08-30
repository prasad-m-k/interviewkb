# ML Compilers & Runtimes

**Topic:** [[ml/topics/deep-learning]]
**Related:** [[ml/concepts/gpu-performance-engineering]], [[ml/concepts/distributed-training]], [[ml/scenarios/llm-service-design]]
**Tags:** #ResponsibleAI

## What it is

The software layer that translates a model definition (a PyTorch graph, an ONNX file) into optimized executable code for a specific hardware target. This is the layer that closes the gap between "a research checkpoint that runs" and "a production-efficient service" — directly the capability an AI Frameworks team owns when the JD talks about accelerating model onboarding and reducing deployment time.

## How it works

### Eager execution vs graph compilation

```
Eager execution (PyTorch default)
  Each op executes immediately as the Python interpreter reaches
  it. Easy to debug (real Python stack traces, can print/inspect
  any tensor). But the framework never sees the WHOLE computation
  graph at once — so it can't fuse ops, reorder for better memory
  locality, or eliminate dead computation across op boundaries.

Graph compilation
  Capture the full computation graph once (ahead of time or via
  tracing), then optimize globally — operator fusion (see
  [[ml/concepts/gpu-performance-engineering]]), dead code
  elimination, memory planning, layout optimization — before
  running it, potentially thousands of times, at that optimized
  form. The one-time compilation cost is amortized across all
  subsequent executions.
```

### The PyTorch-native compiler stack: torch.compile

```
torch.compile pipeline:
  1. TorchDynamo   — captures the Python bytecode into an FX graph,
                      falling back to eager for anything it can't
                      safely trace (a "graph break")
  2. AOTAutograd   — captures the backward pass graph ahead of time
                      too, not just forward
  3. TorchInductor — the codegen backend: emits Triton kernels for
                      GPU targets (fused where possible — see
                      [[ml/concepts/gpu-performance-engineering]]),
                      C++ for CPU

Guards & recompilation:
  Compiled code is only valid for the input shapes/dtypes/
  conditions it was compiled against. A "guard" checks this on
  every call; if violated (e.g., a new sequence length), Dynamo
  RECOMPILES — an expensive, often invisible tax in production if
  inputs vary more than expected.
```

### ONNX Runtime — the hardware-portable interchange layer

```
ONNX (Open Neural Network Exchange): a hardware/framework-neutral
model format. A model trained in PyTorch (or TensorFlow, or
anything else) exports to ONNX once.

ONNX Runtime: a runtime with pluggable "Execution Providers" (EPs)
per hardware backend — CUDA EP, ROCm EP, TensorRT EP, CPU EP, and
others. The model doesn't change; only the EP selected at deploy
time changes.

Why this matters for a platform spanning NVIDIA, AMD, and Microsoft
silicon: it decouples "which framework was this model authored in"
from "which hardware is it running on" — the same decoupling
motivation behind preferring Triton over hand-written CUDA in
[[ml/concepts/gpu-performance-engineering]], one layer higher in
the stack (whole-model portability, not single-kernel portability).
```

### XLA — the comparison point from the TPU world

[[ml/concepts/distributed-training]] already covers XLA as the compiler underpinning Google's TPU stack: it requires static, fixed-shape computation graphs known at compile time, which is why JAX (designed around this constraint) is TPU's preferred framework. torch.compile's guard/recompile model is PyTorch's answer to the same fundamental tension (dynamic-shape flexibility vs compile-time optimization opportunity) but chooses graceful, per-call recompilation over XLA's harder static-shape requirement — a different point on the same trade-off curve, worth naming explicitly if asked to compare.

## Complexity

The core trade-off is compile-time cost vs steady-state execution speed. A model compiled once and run a million times amortizes the compile cost to nothing; a model whose shape changes every call (recompiling constantly) can end up SLOWER than eager execution, because you pay compilation cost repeatedly without ever reaching the steady state that justified it.

## When to use

```
- Any workload where steady-state throughput/latency matters more
  than development iteration speed — i.e., production training and
  serving, not exploratory research notebooks
- Model onboarding: taking a new architecture from a research
  checkpoint to a performance-SLA-compliant production service is,
  concretely, the work of getting it correctly and efficiently
  through this compilation layer
- Multi-hardware platforms: ONNX Runtime's EP model when the same
  model must run correctly across several hardware backends without
  hardware-specific model rewrites
```

## Common interview angles

```
Q: "Why doesn't just shipping PyTorch eager mode to production work
    at scale?"
A: Eager mode never sees the full graph, so it can't fuse ops or
   plan memory globally — every intermediate tensor round-trips
   through HBM (see [[ml/concepts/gpu-performance-engineering]]'s
   fusion section), and Python interpreter overhead sits on the
   critical path of every single op dispatch. Compilation removes
   both costs at the price of an upfront compile step and reduced
   debuggability.

Q: "A model recompiles on almost every request in production, and
    latency is terrible. What's happening and how do you fix it?"
A: A guard is failing on nearly every call — most likely a shape
   that varies per request (e.g., sequence length) that the
   compiler treated as static. Fix: mark that dimension dynamic
   explicitly so Dynamo compiles once for the general case instead
   of once per exact shape, trading a small amount of per-call
   optimality for eliminating the recompilation tax entirely.

Q: "Compare ONNX Runtime and torch.compile — when would a platform
    team choose one over the other?"
A: torch.compile optimizes a PyTorch model for execution, still
   inside the PyTorch ecosystem, with the best fusion/codegen
   quality for PyTorch-authored models. ONNX Runtime targets
   framework- and hardware-agnostic deployment: a model authored in
   any framework, served through a runtime with pluggable
   per-hardware execution providers. A platform team serving models
   authored across multiple frameworks, or needing one deployment
   path across many hardware SKUs, leans ONNX Runtime; a team
   entirely inside PyTorch optimizing for peak performance on known
   hardware leans torch.compile.
```

## Examples

```
Model onboarding checklist through this layer:
  1. Confirm the model traces cleanly under TorchDynamo (no
     unexpected graph breaks on the hot path)
  2. Identify which input dimensions are truly dynamic (batch size?
     sequence length?) and mark them explicitly — avoid silent
     per-shape recompilation in production
  3. Benchmark compiled vs eager on the target hardware SKU (see
     [[ml/concepts/ml-benchmarking-and-regression-detection]]) —
     confirm the compile-time cost is actually amortized at
     expected production call volume before shipping
  4. If the model must also serve on a second hardware vendor,
     evaluate ONNX Runtime export as the portability path instead
     of a second hand-tuned compile target
```

## Sources
- [[ml/concepts/gpu-performance-engineering]]
- [[ml/concepts/distributed-training]]
- [[ml/companies/microsoft-ai-frameworks]]
