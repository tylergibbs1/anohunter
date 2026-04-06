# Scoring Methods

Deep guide on quantifying how surprising an experimental result is. The scorer is the core judgment engine — if it's miscalibrated, the agent either drowns in false positives or misses real anomalies.

---

## Principles

### Surprise, Not Magnitude

A 0.3% deviation where predictions are usually within 0.01% is a 30σ event. A 10% deviation where predictions are routinely off by 8% is a 1.25σ event. The first is vastly more interesting even though the raw deviation is smaller.

### Context Is Everything

The same raw deviation means different things in different input regions. A scorer that ignores regional predictor accuracy will either:
- Flag everything in noisy regions (too many false positives)
- Miss subtle anomalies in well-predicted regions (false negatives)

### Multi-Output Systems Need Multi-Dimensional Scoring

If the target system returns 5 output values, an anomaly might appear in only one of them. Score each output dimension independently, then combine.

---

## Scoring Methods

### 1. Absolute Deviation

```
score = |actual - predicted|
```

**Pros**: Dead simple. No historical data needed. Works for first experiment.
**Cons**: No context. A deviation of 0.5 means nothing without knowing the typical scale.
**When to use**: First 5-10 experiments before you have enough history for statistical methods. Quick sanity check.

### 2. Relative Deviation

```
score = |actual - predicted| / |predicted|
```

**Pros**: Scale-invariant. A 5% deviation is a 5% deviation regardless of units.
**Cons**: Fails when predicted ≈ 0 (division by near-zero). No context about typical variance.
**When to use**: When output values span multiple orders of magnitude and you need a quick normalized metric.

**Division-by-zero guard**: Use `max(|predicted|, epsilon)` where epsilon is a small fraction of the typical output range.

### 3. Statistical Surprise (Z-Score)

```
regional_errors = [|actual_i - predicted_i| for i in nearby_past_experiments]
mean_error = mean(regional_errors)
std_error = stdev(regional_errors)
score = (|actual - predicted| - mean_error) / std_error
```

**Pros**: Adapts to local predictor accuracy. Properly calibrated — 2σ means "this is unusual for this region."
**Cons**: Needs 5+ experiments in the region to be meaningful. Can be gamed if all past experiments in the region were also anomalous (inflated mean_error).
**When to use**: Primary scorer after initial exploration phase (20+ experiments total, 5+ in region).

**Regional definition**: Define "nearby" as experiments within 10% of parameter range on each dimension. If fewer than 5 neighbors exist, expand to 20%.

### 4. Contextual Weighted Surprise

The recommended default. Combines z-score with predictor accuracy weighting:

```python
def contextual_surprise(actual, predicted, regional_history):
    """
    Higher score = more surprising relative to how good the predictor
    usually is in this region.
    """
    deviation = abs(actual - predicted)
    
    if len(regional_history) < 3:
        # Not enough context — fall back to relative deviation
        return deviation / max(abs(predicted), 1e-10)
    
    past_deviations = [abs(a - p) for a, p in regional_history]
    local_mean = statistics.mean(past_deviations)
    local_std = statistics.stdev(past_deviations)
    
    if local_std < 1e-10:
        # Predictor has been perfect in this region
        # Any deviation is extremely surprising
        return deviation / max(local_mean, 1e-10) * 10
    
    z_score = (deviation - local_mean) / local_std
    
    # Weight by predictor accuracy: surprises in accurate regions score higher
    accuracy_weight = 1.0 / max(local_mean, 1e-10)
    # Normalize accuracy weight to [1, 5] range
    accuracy_weight = min(max(accuracy_weight, 1.0), 5.0)
    
    return z_score * accuracy_weight
```

**Why accuracy weighting**: If the predictor is typically within 0.1% in a region, even a 1% deviation is worth investigating. If the predictor is typically within 15%, a 1% deviation is noise.

### 5. Multi-Dimensional Scoring

For systems with multiple output values:

```python
def multi_dim_surprise(actual_dict, predicted_dict, regional_history_dict):
    """
    Score each output dimension independently, report the worst-case.
    """
    scores = {}
    for key in actual_dict:
        regional_history = [(h[key] for h in regional_history_dict[key])]
        scores[key] = contextual_surprise(
            actual_dict[key], 
            predicted_dict[key], 
            regional_history
        )
    
    return {
        "max_score": max(scores.values()),
        "max_dimension": max(scores, key=scores.get),
        "all_scores": scores
    }
```

**Why max, not mean**: An anomaly in one dimension is still an anomaly even if all other dimensions are normal. Mean scoring would dilute a genuine single-dimension surprise.

### 6. Information-Theoretic Surprise

```
score = -log(P(actual | predicted))
```

Where P is the probability of observing `actual` given the predictor's output, assuming the predictor's error follows a distribution estimated from history.

**Pros**: Theoretically principled. Handles non-Gaussian error distributions.
**Cons**: Requires enough data to estimate the error distribution. More complex to implement.
**When to use**: Advanced use case with 100+ experiments and a good understanding of error distribution shape.

**Practical implementation**:
```python
from scipy import stats

def info_surprise(actual, predicted, regional_errors):
    """Fit a distribution to historical errors, score new observation."""
    # Fit normal distribution (or use KDE for non-Gaussian)
    mu, sigma = stats.norm.fit(regional_errors)
    current_error = abs(actual - predicted)
    # Negative log-likelihood
    return -stats.norm.logpdf(current_error, mu, sigma)
```

---

## Threshold Selection

### Adaptive thresholds

Don't use a fixed threshold for the entire hunt. Adapt based on phase:

| Phase | Threshold | Rationale |
|---|---|---|
| Initial survey (< 20 experiments) | 2.0σ | Cast a wide net, accept false positives |
| Steady-state exploration | 2.5σ | Balance signal and noise |
| Boundary mapping | 1.5σ | Catch the edges of anomaly regions |
| Confirmation runs | 0σ (log everything) | During reproducibility checks, record all data |

### Dynamic threshold based on false positive rate

If more than 30% of experiments are flagged as anomalies, the threshold is too low (or the predictor is too bad). If fewer than 2% are flagged after 50+ experiments, the threshold might be too high.

Target: 5-15% anomaly rate during steady-state exploration. Adjust threshold to stay in this range.

---

## Regional Definition

"Nearby" experiments for computing local statistics. Two approaches:

### Fixed-radius neighborhood

```python
def get_neighbors(point, all_experiments, radius=0.1):
    """
    radius: fraction of parameter range 
    (0.1 = experiments within 10% on each dimension)
    """
    neighbors = []
    for exp in all_experiments:
        distances = [
            abs(point[p] - exp[p]) / param_range[p] 
            for p in parameters
        ]
        if max(distances) <= radius:
            neighbors.append(exp)
    return neighbors
```

### K-nearest neighbors

```python
def get_k_nearest(point, all_experiments, k=10):
    """Always returns k neighbors, regardless of distance."""
    distances = [
        sum((point[p] - exp[p])**2 / param_range[p]**2 for p in parameters)**0.5
        for exp in all_experiments
    ]
    sorted_indices = sorted(range(len(distances)), key=lambda i: distances[i])
    return [all_experiments[i] for i in sorted_indices[:k]]
```

**Recommendation**: Use K-nearest with k=10 as default. Fixed-radius can return 0 neighbors in sparse regions.

---

## Handling Edge Cases

### Predictor returns NaN or error

Score as maximum surprise (infinity). Log as "predictor failure" rather than "anomaly." These are worth investigating — the predictor can't handle this input, which may itself be informative.

### Target system returns NaN or error

Score as maximum surprise. Log as "target failure." Check if inputs are within valid ranges. If yes, this is a real anomaly (the system crashes on valid input). If no, the agent proposed invalid inputs — fix the input validation.

### Identical actual and predicted

Score = 0. This is the least surprising possible result. Log as "perfect match" in the boring-experiments summary.

### First experiment (no history)

Use absolute or relative deviation. Set a generous threshold. The first few experiments are calibrating the scorer, not finding anomalies.

---

## Combining Scores Across Runs

When checking reproducibility (3 runs of same inputs):

```python
def reproducibility_score(runs):
    """
    Returns (mean_surprise, variance_across_runs).
    High mean + low variance = real anomaly.
    High mean + high variance = numerical instability.
    Low mean + low variance = boring.
    """
    scores = [score_surprise(run) for run in runs]
    return statistics.mean(scores), statistics.variance(scores)
```

| Mean surprise | Variance | Interpretation |
|---|---|---|
| High | Low | Real, reproducible anomaly |
| High | High | Numerical instability or non-determinism |
| Low | Low | Expected behavior, well-predicted |
| Low | High | Noisy system, predictor matches on average |
