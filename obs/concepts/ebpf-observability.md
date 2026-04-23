# eBPF Observability

**Topic:** [[obs/topics/tracing]], [[obs/topics/metrics]]
**Related:** [[obs/concepts/opentelemetry]], [[sre/concepts/process-signals]]

## What is eBPF?

**eBPF (extended Berkeley Packet Filter)** is a Linux kernel technology that allows programs to run sandboxed code in the kernel without changing kernel source code or loading kernel modules.

For observability, eBPF enables **zero-instrumentation** monitoring — attach to kernel and user-space functions, intercept system calls, network packets, and function calls — all without modifying application code.

```
Traditional observability: App code → OTel SDK → backend
eBPF observability:        Kernel hooks → eBPF program → backend
                           (no code changes in the app)
```

---

## How eBPF Works

```
┌──────────────────────────────────────────────────────────────────┐
│  User Space                                                       │
│  App process ─► syscall ─► kernel hook ─► eBPF program           │
│  (unmodified)              (kprobe)       (sandboxed BPF bytecode)│
└──────────────────────────────────────────────────────────────────┘
│  Kernel Space                                                     │
│  eBPF programs attach to:                                         │
│  - kprobes/kretprobes (kernel function entry/exit)               │
│  - uprobes (user-space function entry/exit)                      │
│  - tracepoints (static kernel instrumentation points)            │
│  - XDP (eXpress Data Path) — network at NIC level                │
│  - socket filters (packet inspection)                            │
│  - perf events (hardware performance counters)                   │
└──────────────────────────────────────────────────────────────────┘
```

**Safety guarantees:** The eBPF verifier (kernel) checks all programs before loading:
- No loops that could run forever
- No out-of-bounds memory access
- No kernel panic possible
- Programs run to completion quickly

---

## eBPF Observability Use Cases

### 1. Network Observability (Cilium / Hubble)
```bash
# Hubble: observe L4/L7 traffic between pods without a service mesh
hubble observe --namespace prod --protocol http

# Output: every HTTP request, response code, latency, source/dest pod
May 10 10:34:21.432  checkout → payment    HTTP/1.1 POST /charge  200  12ms
May 10 10:34:21.891  checkout → inventory  HTTP/1.1 GET  /stock   200   3ms
May 10 10:34:22.012  checkout → payment    HTTP/1.1 POST /charge  500  3421ms ← error!
```

**What it replaces:** Service mesh sidecar (Envoy/Istio) for network metrics. eBPF observes at the socket level — no sidecar, no mTLS overhead.

### 2. CPU Profiling (Continuous Profiling)
```bash
# Parca / Pyroscope: continuous flame graph without code changes
# Attaches to perf events and stack traces at sampling rate

# Example: find what's consuming CPU in a Go service
# eBPF samples stack traces at 100Hz, aggregates into flame graph
```

**Tools:** Parca, Pyroscope, Google Cloud Profiler, Datadog Continuous Profiler.

**What it reveals:** Which function is using the most CPU, even in compiled languages (Go, C++, Rust) without modifying the binary. Works by reading DWARF debug symbols.

### 3. System Call Tracing
```python
# BCC (BPF Compiler Collection) script: trace slow disk I/O
from bcc import BPF

bpf_program = """
#include <uapi/linux/ptrace.h>
#include <linux/blkdev.h>

BPF_HISTOGRAM(dist);

int trace_req_start(struct pt_regs *ctx, struct request *req) {
    u64 ts = bpf_ktime_get_ns();
    start.update(&req, &ts);
    return 0;
}
"""

# Equivalent shell one-liner using bpftrace:
# bpftrace -e 'kprobe:blk_account_io_start { @start[arg0] = nsecs; }
#              kretprobe:blk_account_io_done /@start[arg0]/ {
#                @usecs = hist((nsecs - @start[arg0]) / 1000); delete(@start[arg0]);
#              }'
```

### 4. Zero-Instrumentation Distributed Tracing
Tools like **Odigos**, **Keyval**, and **Groundcover** use eBPF uprobes to inject trace context into network calls automatically:

```bash
# Odigos: auto-instrument any service without code changes
kubectl apply -f https://github.com/keyval-dev/odigos/releases/latest/download/install.yaml

# Odigos detects running services, attaches eBPF uprobes, generates OTel traces
# Works for: Go, Java, Python, Node.js, .NET
# No SDK needed in the application
```

---

## Key eBPF Tools

| Tool | Purpose | Layer |
|---|---|---|
| **Cilium** | Network policy + observability | L3/L4/L7 |
| **Hubble** | Network flow visibility (Cilium component) | L4/L7 |
| **Falco** | Security event detection | Syscall |
| **Pyroscope** | Continuous CPU profiling | User-space |
| **Parca** | Continuous CPU + memory profiling | User-space |
| **bpftrace** | Ad-hoc kernel tracing scripting | Kernel |
| **BCC** | BPF Compiler Collection; Python/C | Kernel + user |
| **Pixie** | Auto-instrumented metrics + traces for K8s | L4/L7 + user |
| **Odigos** | Auto OTel via eBPF | User-space |

---

## eBPF vs. Sidecar (Istio/Envoy)

| | eBPF (Cilium/Hubble) | Sidecar (Istio/Envoy) |
|---|---|---|
| Latency overhead | ~microseconds | ~1–5ms per request (extra hop) |
| Resource overhead | Kernel-level, minimal | Each pod: 50–100MB RAM, 0.1 CPU |
| mTLS | eBPF-based mTLS (newer) | Native mTLS via Envoy |
| L7 visibility | HTTP, gRPC, Kafka | HTTP, gRPC, Kafka, AMQP |
| Deployment | DaemonSet (one per node) | Sidecar (one per pod) |
| Maturity | Newer, growing fast | Mature, battle-tested |

**Current recommendation:** Cilium + Hubble for network observability (replacing Istio for observability use case). Istio still valuable for advanced traffic management and mTLS at scale.

---

## Interview Questions

**Q: What is eBPF and why is it relevant for observability?**
A: eBPF lets you run sandboxed programs in the Linux kernel in response to events (system calls, network packets, function calls) without modifying application code or loading kernel modules. For observability, this means you can instrument any application — regardless of language or framework — without requiring developers to add OTel SDK calls. You get metrics, traces, and security events from kernel-level hooks.

**Q: What is the difference between kprobes and uprobes?**
A: Kprobes attach to kernel function entry/exit — they fire for any process hitting that kernel code (e.g., `open()`, `connect()`). Uprobes attach to user-space function entry/exit — they fire for a specific process's function (e.g., the `http.Get` function in a Go binary). Uprobes require knowing the function offset in the binary; kprobes are kernel-wide.

**Q: How does Cilium use eBPF for networking?**
A: Cilium replaces kube-proxy's iptables rules with eBPF programs loaded into the kernel. This gives: (1) faster packet forwarding (eBPF in the kernel vs. iptables traversal), (2) L7 visibility (eBPF can inspect HTTP/gRPC headers at the socket level), (3) network policies enforced at kernel speed. Hubble, Cilium's observability layer, provides a real-time view of all network flows.

**Q: Can eBPF crash the kernel?**
A: In theory, no — the eBPF verifier rejects any program that could cause memory corruption, infinite loops, or kernel panics. In practice, bugs in the verifier or BPF programs themselves have caused issues historically, but the safety model is much stronger than kernel modules. Production eBPF tools (Cilium, Falco) are widely used at Google/Meta/Apple scale.

## Sources
- [[obs/concepts/opentelemetry]]
- [[obs/topics/tracing]]
