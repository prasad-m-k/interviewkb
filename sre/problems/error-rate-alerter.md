# Error Rate Alerter — Sliding Window Log Monitor

**Difficulty:** Medium
**Topic:** [[sre/topics/linux-cli]]
**Pattern:** Sliding Window / Deque
**Companies:** [[sre/companies/google]], [[sre/companies/meta]], [[sre/companies/apple]]

## Problem

Write a script that **tails a log file in real time** and fires an alert when the error rate exceeds a threshold within a sliding time window.

Requirements:
- Read log lines from stdin or a file (simulating `tail -f`).
- Each line contains a timestamp (ISO-8601 or Unix epoch) and a log level (`INFO`, `WARN`, `ERROR`).
- Alert when: `error_count / total_count > threshold` within the last `window_seconds`.
- After alerting, suppress repeated alerts until the rate drops below threshold (hysteresis).

Example log format:
```
2026-04-22T10:00:00 INFO  GET /api/users 200
2026-04-22T10:00:01 ERROR GET /api/orders 500
2026-04-22T10:00:01 ERROR GET /api/orders 500
2026-04-22T10:00:02 INFO  GET /api/users 200
```

## Why This Is Tricky

- A **fixed window** (count errors in the current minute) is simpler but creates boundary effects: 59 errors at 10:00:59 and 59 errors at 10:01:01 = 118 errors in 2 seconds, but a fixed-window counter shows 59 each.
- A **sliding window** requires evicting old events. A deque (double-ended queue) lets you append new events and pop expired ones in O(1) amortized.
- **Hysteresis** (not alerting repeatedly once already alerting) is always asked as a follow-up.

## Approach

1. Read lines one at a time (streaming).
2. Parse the timestamp from each line.
3. Maintain two deques: one for all events, one for error events.
4. On each new line, evict entries older than `window_seconds` from both deques.
5. Compute the current error rate.
6. If the rate exceeds the threshold and we are not already in an alerted state, fire an alert.
7. If the rate drops below the threshold and we are in an alerted state, reset the alert state.

## Solution (Python)

```python
import sys
from collections import deque
from datetime import datetime, timezone

def parse_timestamp(token: str) -> float:
    """Parse ISO-8601 or Unix epoch string to Unix timestamp (float)."""
    try:
        return float(token)
    except ValueError:
        dt = datetime.fromisoformat(token)
        if dt.tzinfo is None:
            dt = dt.replace(tzinfo=timezone.utc)
        return dt.timestamp()

def monitor(source, window_seconds: int = 60, threshold: float = 0.1):
    """
    Fire an alert when error_count/total_count > threshold
    within the last window_seconds seconds.
    """
    all_events = deque()    # stores timestamps of all log lines
    error_events = deque()  # stores timestamps of ERROR lines
    alerting = False

    for line in source:
        line = line.rstrip('\n')
        if not line:
            continue

        parts = line.split(maxsplit=2)
        if len(parts) < 2:
            continue

        try:
            ts = parse_timestamp(parts[0])
        except (ValueError, IndexError):
            continue

        level = parts[1].upper() if len(parts) > 1 else ""
        is_error = level == "ERROR"

        # Append new event
        all_events.append(ts)
        if is_error:
            error_events.append(ts)

        # Evict events outside the sliding window
        cutoff = ts - window_seconds
        while all_events and all_events[0] < cutoff:
            all_events.popleft()
        while error_events and error_events[0] < cutoff:
            error_events.popleft()

        # Compute error rate
        total = len(all_events)
        errors = len(error_events)
        rate = errors / total if total > 0 else 0.0

        # State machine with hysteresis
        if not alerting and rate > threshold:
            alerting = True
            print(f"ALERT: error rate {rate:.1%} exceeds {threshold:.0%} "
                  f"({errors}/{total} errors in last {window_seconds}s) "
                  f"at ts={ts}", flush=True)
        elif alerting and rate <= threshold:
            alerting = False
            print(f"RESOLVED: error rate dropped to {rate:.1%} at ts={ts}", flush=True)

if __name__ == "__main__":
    window = int(sys.argv[1]) if len(sys.argv) > 1 else 60
    pct    = float(sys.argv[2]) if len(sys.argv) > 2 else 0.1
    monitor(sys.stdin, window_seconds=window, threshold=pct)
```

## Usage

```bash
# Simulate real-time log monitoring
tail -f /var/log/app.log | python alerter.py 60 0.1

# Test with a generated stream
python generate_test_logs.py | python alerter.py 10 0.2
```

## Bash / Shell Approach

For a quick one-shot check (not streaming):
```bash
# Error rate in last 60 seconds across an existing log
NOW=$(date +%s)
WINDOW=60
awk -v now="$NOW" -v win="$WINDOW" '
  { total++ }
  /ERROR/ { errors++ }
  END {
    if (total > 0) printf "Error rate: %.1f%% (%d/%d)\n", errors/total*100, errors, total
  }
' /var/log/app.log
```

For real streaming with alerting, the Python version is required.

## Complexity

- **Time:** O(1) amortized per line (deque eviction is O(1) amortized).
- **Space:** O(W × R) where W = window in seconds, R = max events per second.

## Key Insight

A deque gives O(1) amortized sliding window eviction because events are in chronological order — once the front is expired, it stays expired. Never use a list with `list.pop(0)` (O(N)) or re-scan the entire list on every event.

## Variants / Follow-ups

- "What if timestamps are out of order?" → Add a sort buffer (heap) for late-arriving events with a bounded delay. Out-of-order timestamp events older than the window get discarded.
- "What if you want p99 latency alerts instead of error rate?" → Replace the boolean `is_error` with the latency value, maintain a sorted structure or approximate with a histogram (t-digest).
- "How would you make this distributed?" → Ship events to a stream (Kafka/Pub-Sub), use a stream processing framework (Flink, Spark Streaming) to aggregate per window. Local state becomes partitioned state.
- "Add per-endpoint breakdown" → Key the deques on `(endpoint, level)` — bloom filter or LRU-capped dict to avoid memory explosion.

## Sources

- [[sre/companies/google]]
- [[sre/companies/meta]]
