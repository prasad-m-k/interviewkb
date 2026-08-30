# Microsoft AI Frameworks Interview — Top 15 Flashcards

**Company:** [[ml/companies/microsoft-ai-frameworks]]
**Format:** Front (Q) / Back (A)
**Tags:** #ResponsibleAI

---

## GPU Performance Engineering

**Q1. What's the first question to ask when a GPU workload is slower than expected?**
A: Is it compute-bound or memory-bandwidth bound? Use the roofline model: compare the workload's arithmetic intensity (FLOPs / bytes moved) against the chip's ridge point. Below the ridge point, the workload is memory-bound and more compute optimization won't help; above it, the workload is compute-bound and memory optimization won't help. See [[ml/concepts/gpu-performance-engineering]].

---

**Q2. Why is kernel fusion the highest-leverage fix for a memory-bound workload?**
A: Every intermediate tensor written to and read back from HBM between two unfused ops is bandwidth spent purely on data movement, not computation. Fusing multiple ops into one kernel launch eliminates those round-trips — one read, one write, instead of N reads and N writes for N chained ops.

---

**Q3. Why would a platform team choose Triton over hand-written CUDA, even if CUDA gets a few percent more peak performance?**
A: Maintenance cost across hardware vendors. A CUDA kernel only runs on NVIDIA hardware; supporting AMD (ROCm) or in-house silicon (Microsoft's Maia) means a second hand-written implementation to keep functionally and performance-equivalent forever. Triton (or a compiler that emits Triton, like TorchInductor) trades a few points of peak performance for one maintained implementation across vendors.

---

**Q4. A workload has low GPU utilization but no single kernel looks slow in the profiler. What's happening?**
A: Likely launch-overhead-bound — many small kernel launches where fixed per-launch overhead dominates actual compute time. Neither the compute-bound nor memory-bound fix applies; the fix is fusion or graph capture to reduce the NUMBER of launches.

---

## ML Compilers & Runtimes

**Q5. Why doesn't shipping PyTorch eager mode to production work at scale?**
A: Eager execution never sees the full computation graph at once — it can't fuse ops or plan memory globally, and every op pays Python interpreter dispatch overhead on the critical path. Graph compilation (torch.compile) captures the whole graph once and optimizes it globally, amortizing that cost across many subsequent runs. See [[ml/concepts/ml-compilers-and-runtimes]].

---

**Q6. A model recompiles on nearly every request in production. What's the likely cause and fix?**
A: A "guard" — the check that validates compiled code is still valid for the current input — is failing almost every call, most likely because a dimension (like sequence length) that varies per request was treated as static at compile time. Fix: mark that dimension explicitly dynamic so Dynamo compiles once for the general case instead of recompiling per exact shape.

---

**Q7. ONNX Runtime vs torch.compile — when would a platform team pick one over the other?**
A: torch.compile optimizes a PyTorch model within the PyTorch ecosystem for best-in-class fusion/codegen on known hardware. ONNX Runtime targets framework- and hardware-agnostic deployment via pluggable Execution Providers per backend (CUDA, ROCm, TensorRT, CPU) — the right choice when serving models authored in multiple frameworks or needing one deployment path across many hardware SKUs.

---

**Q8. How does XLA's static-shape requirement compare to torch.compile's approach to dynamic shapes?**
A: XLA (Google's TPU compiler) requires fixed, known-at-compile-time shapes — a hard constraint, which is why JAX is designed around it. torch.compile takes a softer approach: it compiles per-shape and recompiles when shapes change (guard failures), trading some recompilation overhead for not requiring fully static shapes upfront. Different points on the same compile-time-vs-flexibility trade-off. See [[ml/concepts/distributed-training]] for the TPU/XLA context.

---

## Benchmarking & Regression Detection

**Q9. Why isn't a single benchmark run a valid benchmark?**
A: GPU timing has real variance (thermal throttling, co-tenant noise, clock boost). A meaningful benchmark pins every variable that affects the number (model version, batch size, precision, hardware SKU, driver version), excludes warm-up runs, and reports a statistic across N runs (median/p90) — not one sample. See [[ml/concepts/ml-benchmarking-and-regression-detection]].

---

**Q10. Why is a fixed percentage threshold (e.g., "fail if >5% slower") a bad regression-detection gate?**
A: It either misses real regressions in low-variance benchmarks or false-positives constantly on high-variance ones. A statistically-aware threshold — comparing the new measurement's distribution against the baseline's own historical variance — is what separates a real regression gate from a noisy script.

---

**Q11. What's the difference in scope between a Senior and a Principal engineer solving the same "model is slow" problem, per this JD?**
A: A Senior engineer profiles it, finds the fix, and ships it. A Principal engineer asks why the regression wasn't caught before shipping and builds the automated benchmarking/observability mechanism that catches the next fifty regressions like it — turning a one-off fix into a durable platform capability. See [[ml/companies/microsoft-ai-frameworks]].

---

## System Design / Cross-Stack

**Q12. Design the process for onboarding a new model architecture from research checkpoint to production, with a defined performance SLA.**
A: (1) Confirm it traces cleanly under the compiler with no unexpected graph breaks; (2) identify truly dynamic input dimensions and mark them explicitly to avoid recompilation storms; (3) benchmark compiled vs eager on the target hardware SKU using a pinned, statistically-sound methodology; (4) capture that benchmark as the baseline future changes are measured against; (5) if it must serve on a second hardware vendor, evaluate ONNX Runtime instead of a second hand-tuned compile target.

---

**Q13. You're training on 512 GPUs and throughput stops scaling past 128. What's the likely cause?**
A: A communication bottleneck — AllReduce cost is fixed per step regardless of GPU count, but at 512 GPUs the gradient volume and cross-node interconnect (InfiniBand, ~25GB/s) becomes the binding constraint instead of compute. Compare NVLink's ~600GB/s intra-node bandwidth to InfiniBand's inter-node bandwidth — the gap is usually where the ceiling comes from. See [[ml/concepts/distributed-training]].

---

## Behavioral / Leadership (Principal-Specific)

**Q14. Tell me about a time you aligned researchers, an infrastructure team, and a hardware partner around a technical direction they didn't initially agree on.**
A: Frame it around DATA as the persuasion mechanism — a benchmark result or profiling finding that made the trade-off concrete — rather than authority or seniority. Mirrors the JD's own language: "align teams around durable architectures and success measures."

---

**Q15. Why does this JD explicitly mention NVIDIA GPUs, AMD GPUs, AND Microsoft silicon in the same sentence, and what does that imply for how you should answer performance questions?**
A: It signals hardware portability is a first-class constraint, not an edge case — solutions hand-tuned to a single vendor are a maintenance liability for this team. Volunteer vendor-portable approaches (Triton over hand-written CUDA, ONNX Runtime's execution-provider model) unprompted in performance answers; it directly mirrors the JD's framing and differentiates from a narrowly single-vendor answer.

---

## Sources
- [[ml/companies/microsoft-ai-frameworks]]
- [[ml/concepts/gpu-performance-engineering]]
- [[ml/concepts/ml-compilers-and-runtimes]]
- [[ml/concepts/ml-benchmarking-and-regression-detection]]
