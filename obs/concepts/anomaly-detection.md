# Anomaly Detection (ML-Based)

**Topic:** [[obs/topics/ai-for-observability]]
**Related:** [[obs/concepts/slo-burn-rate]], [[obs/concepts/cardinality]], [[ml/concepts/precision-recall-auc]], [[ml/concepts/class-imbalance]]

## What it is

Detecting "this metric/log signal is behaving unusually" without a human hand-setting a static threshold. The whole point is adapting to a signal's own normal behavior — including its seasonality and trend — rather than applying one fixed number that's correct for maybe a few hours a day.

## How it works

### Why static thresholds fail first — the motivating problem

```
"Alert if error rate > 5%" — this single number is wrong twice:
  - Too LOOSE at 3am on a Sunday, when normal traffic is a tenth of
    peak and even a small absolute number of errors is a much
    bigger PROPORTION of a smaller total — a real problem can hide
    under 5% while still being 10x the normal rate for that hour.
  - Too TIGHT during a Black Friday traffic spike, where a normal,
    healthy system's absolute error count rises even though the
    RATE is unchanged — false-positive paging exactly when the
    on-call least wants noise.

The signal that actually matters is "how far is this from what's
NORMAL FOR THIS TIME," not "how far is this from one fixed number."
```

### Statistical baselining — the simpler, often-sufficient first step

```
Moving average + standard deviation bands:
  baseline = rolling_mean(metric, window=7d)
  band     = baseline ± (k × rolling_stddev(metric, window=7d))
  Alert when the current value falls outside the band.

STL decomposition (Seasonal-Trend decomposition using Loess):
  Splits a time series into three components:
    Observed = Trend + Seasonal + Residual
  Anomaly detection runs on the RESIDUAL only, after removing
  known trend (slow growth) and seasonal (daily/weekly cycle)
  patterns — so a normal Monday-morning traffic ramp isn't
  mistaken for an anomaly just because it's higher than Sunday
  night.

This alone solves most of the false-positive problem from static
thresholds, and is far cheaper/more explainable than full ML — many
production systems stop here and never need the ML methods below.
```

### ML-based methods — for when statistical baselining isn't enough

```
Isolation Forest
  Unsupervised: randomly partitions the feature space; anomalies
  are points that get isolated in FEWER random splits than normal
  points (outliers are "easier to separate" from the bulk of the
  data). Works well on multi-dimensional signals (not just a
  single time series) without needing labeled anomaly examples —
  a major practical advantage, since labeled incident data is
  always scarce (see [[ml/concepts/class-imbalance]]: real
  incidents are a tiny minority class by definition).

Autoencoders
  Neural network trained to RECONSTRUCT normal input. Anomalies
  are inputs the model reconstructs POORLY (high reconstruction
  error), because the model only learned to compress/decompress
  patterns it saw during training on normal data. Useful for
  high-dimensional signals (e.g. many correlated metrics at once)
  where a simple per-metric threshold can't capture the
  relationship BETWEEN metrics.

Change-point detection
  Distinct question from "is this point anomalous" — instead asks
  "did the underlying DISTRIBUTION shift at this timestamp"
  (e.g. a deploy silently changed baseline latency permanently,
  not just a transient spike). Directly useful for correlating an
  anomaly to a deploy timestamp — see
  [[obs/concepts/automated-rca-correlation]].
```

### The precision/recall trade-off, applied to paging humans

```
This is the SAME classifier trade-off as
[[ml/concepts/precision-recall-auc]], with a concrete operational
cost on each side:

  High recall, low precision → pages on lots of false anomalies
    → alert fatigue (see [[obs/scenarios/alert-fatigue]]) → the
    exact failure mode ML anomaly detection was supposed to fix,
    reintroduced by an overly sensitive model instead of an overly
    tight static threshold

  High precision, low recall → misses real anomalies that don't
    look like the training examples → false sense of security,
    worse than a static threshold because the team may have
    REMOVED the manual threshold that would have caught it

Practical default: tune toward precision for PAGING alerts (an
anomaly detector's false page is exactly as costly as a bad static
threshold's false page), but toward recall for DASHBOARD/annotation
use cases (a false anomaly marker on a graph a human is already
looking at costs almost nothing to ignore).
```

## Complexity

Not algorithmic in the classic sense, but real operational cost: any ML-based detector needs periodic retraining as normal behavior drifts (a service's baseline six months ago may no longer be its baseline today — the same training-serving skew concern as [[mlops/concepts/training-serving-skew]], applied to an anomaly model instead of a product model), and needs a labeled or semi-labeled feedback loop (did responders confirm this was a real anomaly?) to actually improve precision over time.

## When to use

```
✅ High-volume, seasonal metrics where maintaining per-service
   static thresholds by hand doesn't scale (hundreds+ of services)
✅ Multi-dimensional signals where the anomaly is in the
   RELATIONSHIP between metrics, not any single one crossing a line
✅ As an ADDITIONAL signal layered on top of existing SLO burn-rate
   alerting (see [[obs/concepts/slo-burn-rate]]), not a wholesale
   replacement — burn-rate alerting on a defined SLI is still the
   right primary paging mechanism for a service with a real SLO

❌ Low-volume or genuinely non-seasonal metrics, where a simple
   static threshold is already accurate and an ML model adds
   retraining/drift overhead for no real precision gain
❌ As the ONLY layer for a critical service's core paging alert,
   without an SLO-based backstop — an anomaly detector can silently
   drift and stop firing on real problems if retrained on a
   period that included the degraded behavior as "normal"
```

## Common interview angles

```
Q: "Why not just lower the static threshold to catch more
    incidents?"
A: A single global threshold can't be simultaneously right for
   low-traffic and high-traffic periods — tightening it to catch
   overnight anomalies guarantees false pages during traffic
   spikes, and loosening it to survive traffic spikes guarantees
   missed overnight anomalies. The fix is comparing against a
   TIME-AWARE baseline (STL decomposition or ML), not a tighter
   fixed number.

Q: "Your team deployed an ML anomaly detector and on-call
    complains it's noisier than the old static thresholds. What
    happened?"
A: Almost certainly a precision problem — the detector is tuned
   toward recall (catching everything that LOOKS unusual) without
   enough weight on precision (only paging on things that matter).
   Fix: retune the operating point, and/or route low-confidence
   anomalies to a dashboard annotation instead of a page — reserve
   paging for high-confidence anomalies only.

Q: "How would you detect an anomaly without any labeled examples
    of past incidents?"
A: Unsupervised methods — Isolation Forest or an autoencoder's
   reconstruction error — don't require labeled anomalies at all,
   only a corpus of (mostly) normal data to learn "normal" from.
   This matters because labeled incident data is always scarce by
   definition (see [[ml/concepts/class-imbalance]]) — a supervised
   approach would be starved of positive examples.

Q: "What's the difference between an anomaly and a change point?"
A: An anomaly is a single point (or short window) that deviates
   from the established baseline, often transient. A change point
   is a shift in the UNDERLYING DISTRIBUTION itself — the new
   normal is genuinely different going forward (e.g. after a
   deploy). Treating a change point as a transient anomaly means
   the detector keeps "flagging" the new normal indefinitely until
   it retrains; correctly identifying it as a change point lets you
   immediately correlate it to a deploy timestamp instead.
```

## Examples

```
STL-based alert rule (conceptual):
  residual = observed - trend - seasonal_component(day_of_week, hour_of_day)
  baseline_stddev = rolling_stddev(residual, window=14d)
  alert if abs(residual) > 3 × baseline_stddev
    AND confidence_score > 0.8   (route lower-confidence to dashboard only)
```

## Sources
- [[obs/topics/ai-for-observability]]
- [[obs/concepts/slo-burn-rate]]
- [[ml/concepts/precision-recall-auc]]
- [[ml/concepts/class-imbalance]]
- [[obs/scenarios/alert-fatigue]]
