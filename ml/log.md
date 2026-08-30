# ML Knowledge Base — Log

## [2026-08-16] ingest | Microsoft AI Frameworks — Senior/Principal SWE interview prep

- User provided the live JD text directly (pasted) for "Senior and/or Principal Software Engineer - AI Frameworks" (Microsoft Careers portal query=200045418, pid=1970393556944478) after WebFetch/WebSearch attempts to render the live posting were blocked by the portal's JS-only content (same limitation hit on the earlier CoreAI Responsible AI req in solution-arch/). Did not fabricate location or compensation, since the pasted JD text didn't include them.
- Created company page: companies/microsoft-ai-frameworks — role snapshot, Senior vs Principal scope-comparison table (built and verified with a column-alignment script after the first hand-typed pass drifted — see alignment note below), full required/preferred qualifications by level, what the role weights heavily (systems fundamentals over ML theory, cross-stack debugging, hardware-vendor portability, durable-platform-capability framing, model onboarding velocity, Azure capacity efficiency), and practice questions spanning GPU diagnosis, compilers/onboarding, benchmarking, and Principal-level cross-team leadership.
- Created concept: gpu-performance-engineering — roofline model (compute- vs memory-bound) as the required first diagnostic step, CUDA vs ROCm vs Triton portability trade-offs, kernel fusion, Tensor Core precision, profiling tool landscape (Nsight Systems/Compute, PyTorch Profiler, rocprof), launch-overhead-bound as a third failure mode the roofline model alone doesn't reveal.
- Created concept: ml-compilers-and-runtimes — eager execution vs graph compilation, the torch.compile stack (TorchDynamo/AOTAutograd/TorchInductor), guards and recompilation, ONNX Runtime's execution-provider portability model, and an explicit comparison to XLA's static-shape requirement (cross-linking the existing distributed-training.md TPU/XLA section rather than re-explaining it).
- Created concept: ml-benchmarking-and-regression-detection — benchmark methodology (pinned variables, warm-up exclusion, statistical treatment), the metrics taxonomy (throughput/latency/utilization/cost, each pointing to where it's already defined in this wiki), statistically-aware regression gates vs fixed-percentage thresholds, bisection tooling, and the "one-off analysis vs durable platform capability" framing lifted directly from the JD's Senior-vs-Principal language.
- Created flashcards: microsoft-ai-frameworks-top15 (15 Q&A covering GPU performance, compilers/runtimes, benchmarking, system design, and Principal-level behavioral).
- Updated: index.md (3 new concepts, 1 new company, 1 new flashcard deck — all tagged #ResponsibleAI per explicit user request), INTERVIEW_NAVIGATOR.md (new path).
- Alignment note: the hand-typed Senior/Principal comparison table in the company page had drifted column borders on first pass (the same class of bug flagged earlier in solution-arch/concepts/ai-guardrails-and-safety.md's GUARDRAIL PIPELINE diagram) — rebuilt programmatically with a Python column-width script and verified every `│` lands in the same character column before considering the page done.
- Notes: tagged with #ResponsibleAI per explicit user instruction ("use the same tag as before") even though this role's content is performance/systems engineering, not governance — a deliberate cross-cutting tag choice by the user, not a semantic judgment made here. Deliberately cross-referenced rather than duplicated ml/concepts/distributed-training.md (parallelism, ZeRO, TPU/XLA) and ml/scenarios/llm-service-design.md (KV cache, continuous batching, memory-bandwidth-bound decode) — this ingest fills the GPU-kernel/compiler/benchmarking gap those pages don't cover, not a restatement of them.

## [2026-04-21] ingest | Initial ML knowledge base build
- Created: overview, index, log
- Created topics: supervised-learning, unsupervised-learning, deep-learning, model-evaluation, feature-engineering, nlp, ml-system-design
- Created concepts: bias-variance-tradeoff, gradient-descent, backpropagation, regularization, cross-validation, attention-mechanism, transformers, embeddings, loss-functions, activation-functions, batch-normalization, ensemble-methods, class-imbalance, precision-recall-auc, overfitting
- Created patterns: feature-engineering-patterns, model-selection, hyperparameter-tuning, transfer-learning, data-augmentation
- Created scenarios: model-not-converging, overfitting-diagnosis, class-imbalance-handling, low-accuracy-debugging, recommendation-system-design, search-ranking-design, fraud-detection-design
- Created flashcards: ml-concepts-anki, ml-scenarios-anki
- Created: mindmap
- Notes: Full initial build covering supervised/unsupervised/deep learning, NLP, and ML system design. Interview-focused with scenario-based flashcards and Anki-compatible cards.

## [2026-06-29] update | Google AI/ML interview prep expansion
- Created companies/: google-ml (process, ML coding, system design, Googleyness, napkin math, common mistakes)
- Created concepts: llm-fundamentals (pretraining, SFT, RLHF, KV cache, inference optimization), distributed-training (data/tensor/pipeline parallelism, ZeRO, FSDP, TPU), rag (dense retrieval, chunking, evaluation, failure modes), reinforcement-learning (policy gradient, PPO, bandits, RLHF)
- Created patterns: rlhf (3-phase pipeline, DPO alternative), rag-pattern (indexing + serving skeleton, system design)
- Created scenarios: youtube-recommendations (Google flagship, two-stage pipeline, multi-objective), google-ads-ctr (Wide & Deep, position bias, calibration), llm-service-design (KV cache, continuous batching, speculative decoding), content-moderation (multi-stage, human-in-the-loop, responsible AI)
- Created flashcards: google-ml-top20 (20 Q&A covering theory, system design, LLMs, responsible AI)
- Updated: ml/index.md (added companies section, 4 concepts, 2 patterns, 4 scenarios), INTERVIEW_NAVIGATOR.md (added Path D: Google AI/ML)
- Notes: Focused on Google-standard depth: first-principles ML theory, scale awareness (napkin math), responsible AI, and LLM/GenAI familiarity (Gemini context).
