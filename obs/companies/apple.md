# Apple Observability Interview Prep

**Related:** [[obs/overview]], [[sre/companies/apple]]

## Apple's Observability Constraints

Apple's observability is shaped by a fundamental constraint that no other FAANG faces at the same intensity: **privacy by default**. Every observability decision — what to collect, where to store it, who can query it — is filtered through Apple's privacy values.

This creates unique engineering challenges that Apple interviewers test explicitly:
- How do you observe a system without collecting user data?
- How do you debug user-reported issues without having their data?
- How do you build a monitoring system for iCloud that respects Apple's privacy commitments?

---

## Apple's Observability Architecture Principles

### 1. Aggregate, Not Individual
Apple collects aggregate metrics (population-level statistics), not individual user data.

```
✗ Wrong: Log that user "john@icloud.com" had a 3.4s upload on 2026-04-22
✓ Right: 99th percentile upload duration increased from 1.2s to 3.4s across US region
```

The implication: Apple's observability systems are built around **differential privacy** and **aggregate statistics** rather than raw event logs with user identifiers.

### 2. On-Device First
Apple pushes computation to the device where possible:
- **MetricKit:** App performance metrics computed and aggregated on-device, then batched and uploaded in a privacy-preserving way.
- **Crash reports:** Symbolicated on-device before upload. Apple's servers never see raw stack traces with memory addresses.
- **ML inference:** On-device ML (Core ML) means user data never leaves the device.

### 3. Server-Side: Metadata Only
For server-side services (iCloud, App Store, Apple Music):
- Log **metadata** (duration, error code, region, timestamp) but not **content** (file name, message body, search query).
- Replace user-identifying fields with anonymous tokens or ephemeral identifiers.
- Per-user encryption: even if someone queries the storage system, data is encrypted with user-specific keys.

---

## MetricKit (iOS/macOS App Observability)

MetricKit is Apple's framework for collecting performance metrics from iOS/macOS apps without user data.

### What MetricKit Collects
```swift
class MetricsHandler: NSObject, MXMetricManagerSubscriber {
    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            // Histogram of app launch duration
            let launchMetrics = payload.applicationLaunchMetrics
            print("Average cold launch: \(launchMetrics?.histogrammedTimeToFirstDraw)")

            // CPU time per process
            let cpuMetrics = payload.cpuMetrics
            print("CPU time: \(cpuMetrics?.cumulativeCPUTime)")

            // Memory footprint histogram
            let memoryMetrics = payload.memoryMetrics
            print("Peak memory: \(memoryMetrics?.peakMemoryUsage)")

            // Hang rate (main thread blocked > 250ms)
            let hangMetrics = payload.hangDiagnostics  // from MXDiagnosticPayload
        }
    }
}
```

**Privacy design:** MetricKit aggregates data on-device over 24 hours and reports histograms, not raw events. Apple's servers receive population statistics, not per-user data points.

### Key MetricKit Metrics for Interviews
| Metric | Meaning | Interview context |
|---|---|---|
| Time to First Draw (TTFD) | Cold launch latency | App startup SLO |
| Hang rate | % time main thread blocked >250ms | UI responsiveness SLO |
| CPU time | Per-category CPU consumption | Battery/performance |
| Memory footprint histogram | Distribution of memory usage | OOM/crash rate |
| Disk write rate | bytes written per day | Storage efficiency |

---

## Privacy-First Observability Design Patterns

### Pattern 1: Differential Privacy for Aggregate Analytics
Add calibrated noise to aggregate statistics so that the presence or absence of any individual user cannot be inferred:

```python
import numpy as np

def dp_average(values: list[float], epsilon: float = 1.0, sensitivity: float = 1.0) -> float:
    """
    Compute average with Laplace noise for differential privacy.
    epsilon: privacy budget (smaller = more private, less accurate)
    sensitivity: max change in average from one user's data point
    """
    true_average = sum(values) / len(values)
    noise = np.random.laplace(0, sensitivity / epsilon)
    return true_average + noise
```

**When Apple uses this:** Aggregate analytics (e.g., "which features are most used?") are computed with differential privacy so that even aggregate reports don't reveal individual user behavior.

### Pattern 2: Anonymized Request IDs
Instead of logging `user_id`, generate an ephemeral, per-session anonymous token:

```python
import hashlib
import os

def get_request_token(user_id: str, session_id: str, salt: bytes) -> str:
    """
    Generate a per-session anonymous token. 
    Cannot be reversed to user_id without the salt (which rotates daily).
    """
    return hashlib.blake2b(
        f"{user_id}:{session_id}".encode(),
        key=salt,
        digest_size=16
    ).hexdigest()

# The salt rotates every 24 hours — old tokens become unlinkable
# Even if the token is logged, it cannot be correlated across sessions
```

### Pattern 3: Metadata-Only Logging
```python
# ✗ Bad: includes user content
logger.info(f"User {user_id} searched for '{query}' — {result_count} results in {latency_ms}ms")

# ✓ Good: only metadata
logger.info("search_completed", extra={
    "result_count": result_count,
    "latency_ms": latency_ms,
    "region": region,
    "is_cached": is_cached,
    "request_token": request_token,  # anonymous, rotates daily
    "trace_id": trace_id,
})
```

---

## Apple Observability Interview Questions

### Design Questions
- **"Design a logging system for iCloud Photos that helps debug latency issues without storing photo metadata."**
  → Response: Log operation type (upload/download/sync), duration histogram, error codes, region, device OS version, file size buckets (1KB/10KB/1MB/10MB+), but never file names, content hashes, or user identifiers. Correlation via anonymous session token (rotates daily). For debugging specific reports: let users voluntarily opt in to enhanced diagnostics (sysdiagnose-style) that are deleted after 24 hours.

- **"Design an alerting system for Apple Music that detects streaming quality degradation."**
  → Collect: buffering ratio (rebuffers/sec), stream start latency, bit rate achieved vs. requested — all as aggregate histograms per CDN region. Alert on population-level SLO: "p95 stream start latency > 2s in US region." Never log what song a user was streaming or that a specific user experienced degradation.

- **"How do you build crash reporting that is privacy-preserving?"**
  → On-device symbolication (convert memory addresses to function names using debug symbols). Upload only the symbolicated stack trace (function names + line numbers), not the raw binary crash report. Strip any user data from the crashed memory pages. Aggregate identical crashes by signature (hash of stack trace) — report count per signature, not individual instances.

### Coding Questions (Apple-Specific)
- **"Parse a MetricKit histogram and calculate p99 latency."**

```python
def p99_from_histogram(histogram: dict[tuple[float, float], int]) -> float:
    """
    MetricKit reports histograms as {(bucket_start, bucket_end): count}.
    Compute p99 by finding the bucket containing the 99th percentile count.
    """
    total = sum(histogram.values())
    target = total * 0.99
    cumulative = 0
    for (low, high), count in sorted(histogram.items()):
        cumulative += count
        if cumulative >= target:
            # Interpolate within bucket
            fraction = (target - (cumulative - count)) / count
            return low + (high - low) * fraction
    return max(high for (low, high) in histogram)
```

---

## Behavioral: Observability and Privacy Alignment

Apple interviewers look for this mindset:

> "I'd design the logging system to collect the minimum data necessary to answer operational questions. For latency debugging, I need duration and error codes — not the content of the request. For regional analysis, I need region and OS version — not user identity. The design question is always 'what's the minimum I need to know to diagnose this?' not 'what can I collect?'"

This is fundamentally different from the Google/Meta mindset (collect everything, query later) — and Apple interviewers specifically probe for it.

## Sources
- [[sre/companies/apple]]
- [[sre/sources/apple-interview-wiki]]
