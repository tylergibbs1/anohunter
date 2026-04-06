# Exploration Strategies

Deep guide on how the agent should navigate input space to maximize discovery while avoiding redundant experiments.

---

## The Exploration-Exploitation Tradeoff

This is not the standard reinforcement learning tradeoff. In RL, exploitation means "do the thing that maximizes reward." Here, exploitation means "probe deeper around a known anomaly to understand it." The goal of exploitation is characterization, not optimization.

### Budget enforcement

The program.md should specify an exploration budget. Default recommendation:

```
- 60% exploitation: boundary mapping, reproducibility checks, parameter sweeps near anomalies
- 40% exploration: completely new input regions
- Never exceed 10 consecutive exploitation experiments without returning to exploration
- Never exceed 5 consecutive exploration experiments without checking if any new anomaly deserves investigation
```

### How to track the budget

Maintain a running count in the experiment log:

```jsonl
{"id": 43, "type": "exploration", "region": "high-Re-triangles", ...}
{"id": 44, "type": "exploitation", "anomaly_ref": "007", "param_varied": "reynolds", ...}
{"id": 45, "type": "exploitation", "anomaly_ref": "007", "param_varied": "reynolds", ...}
```

After each experiment, check: am I over budget on exploitation? If yes, force exploration next.

---

## Exploration Strategies

### 1. Latin Hypercube Sampling (LHS)

For initial survey of the input space. Divides each parameter's range into N equal intervals and samples one point per interval in each dimension, ensuring even coverage.

**When to use**: First 20-30 experiments. You know nothing about the landscape.

**Implementation**:
```python
from scipy.stats import qmc

sampler = qmc.LatinHypercube(d=len(parameters))
samples = sampler.random(n=30)
# Scale to actual parameter ranges
for i, param in enumerate(parameters):
    samples[:, i] = param.min + samples[:, i] * (param.max - param.min)
```

**Agent version** (no scipy needed): The agent can manually divide each parameter range into N segments and ensure each segment is visited once. Less mathematically optimal than true LHS but achieves the goal of coverage.

### 2. Grid Sweep Along Axes

Pick one parameter, hold everything else at a "boring" baseline, and sweep that parameter across its full range with 10-20 points. Repeat for each parameter.

**When to use**: After LHS, to understand the marginal effect of each parameter in isolation.

**Advantage**: If a parameter has a critical threshold (e.g., Reynolds number where turbulence begins), this finds it.

**Disadvantage**: Misses interaction effects entirely.

### 3. Anomaly Neighborhood Search

When an anomaly is found at input X, explore a hypersphere around X:

```
For each parameter p_i in X:
    Run experiment with p_i + delta, all others held constant
    Run experiment with p_i - delta, all others held constant
    If anomaly persists: increase delta (expand boundary)
    If anomaly disappears: record boundary point
```

**Delta sizing**: Start with 5% of the parameter's range. If the anomaly boundary is found within 2 steps, switch to 1% for finer resolution.

### 4. Bisection for Boundary Finding

When you know the anomaly exists at point A but not at point B:

```
1. Midpoint M = (A + B) / 2
2. Run experiment at M
3. If anomaly at M: boundary is between M and B → recurse(M, B)
4. If no anomaly at M: boundary is between A and M → recurse(A, M)
5. Stop when |A - B| < resolution_threshold
```

**When to use**: After neighborhood search identifies the approximate boundary. Bisection gives precise boundary points.

### 5. Interaction Probing

After marginal effects are mapped, probe parameter interactions:

```
1. Pick two parameters that each have known anomalies when varied alone
2. Vary both simultaneously
3. Check if combined effect is additive (independent) or non-linear (interaction)
```

**How to detect interaction**: If anomaly(p1_high, p2_high) ≠ anomaly(p1_high) + anomaly(p2_high), there's an interaction.

### 6. Random Jump

Periodically (every 15-20 experiments), pick a completely random point in the input space that is far from all previously sampled points. This prevents the agent from getting trapped in one region of the space.

**Distance metric**: Normalize all parameters to [0,1], compute Euclidean distance to nearest sampled point. Pick a point that maximizes this distance (or pick randomly and reject if too close to existing samples).

---

## Exploitation Strategies

### Boundary Mapping Protocol

When an anomaly is detected, the full characterization protocol is:

```
Phase 1: Confirm (1-3 experiments)
  - Rerun the exact same inputs 3 times
  - If results vary by more than 5%: flag as non-deterministic, note variance
  - If results are consistent: proceed to boundary mapping

Phase 2: Marginal boundaries (2*N experiments, where N = number of parameters)
  - For each parameter: probe ±10% from anomaly point
  - Record which parameters affect the anomaly and which don't

Phase 3: Sensitive parameter sweep (5-10 experiments per sensitive parameter)
  - For parameters that affect the anomaly: sweep with finer resolution
  - Use bisection to find precise boundary points

Phase 4: Interaction check (optional, 4-8 experiments)
  - If multiple parameters affect the anomaly:
    vary two simultaneously to check for interaction effects

Phase 5: Write-up
  - Produce anomaly report with all boundary data
  - Include table of boundary probes
  - Note sharp vs. gradual transitions
  - Return to exploration
```

### When to Stop Investigating an Anomaly

Stop boundary mapping when:
- All parameter boundaries have been located to within 2% of parameter range
- Exploitation budget is exhausted
- The anomaly turns out to be non-reproducible (variance > 20% across 3 runs)
- The anomaly is clearly a degenerate input (edge of valid range, numerical instability)

---

## Experiment Selection Heuristics

The agent should use these heuristics when deciding what to try next:

### Diversity score

Before running an experiment, compute how different it is from all previous experiments:

```
diversity = min(distance(proposed, past_experiment) for past_experiment in log)
```

If diversity is below a threshold (e.g., 0.05 in normalized space), reject the experiment and propose something more different.

### Information value

Prefer experiments that resolve uncertainty:
- A point halfway between a known anomaly and a known boring region: high information value (is the boundary here?)
- A point very close to a previous experiment: low information value (we already know what happens here)
- A point in a completely unexplored region: medium information value (could be interesting, could be boring, but we don't know)

### Domain-informed priors

The agent has domain knowledge from its training. Use it:
- "Phase transitions in physics often occur at dimensionless numbers near 1"
- "Machine learning models tend to fail on out-of-distribution inputs near the boundary of the training set"
- "Numerical methods often have stability thresholds related to grid spacing and time step"

These priors should inform where to explore, not where to avoid. If domain knowledge says "transitions happen near Re=1000," that's a good place to look — not a place to skip because "we already know about it."

---

## Tracking Explored Space

### Coverage map

Maintain a discretized coverage map of the input space:

```json
{
  "reynolds_number": {
    "100-1000": {"experiments": 12, "anomalies": 0, "status": "well-explored"},
    "1000-5000": {"experiments": 8, "anomalies": 1, "status": "partially-explored"},
    "5000-20000": {"experiments": 3, "anomalies": 2, "status": "sparse"},
    "20000-100000": {"experiments": 0, "anomalies": 0, "status": "unexplored"}
  }
}
```

Update after each experiment. Use this to guide exploration toward unexplored regions.

### Staleness detection

If a region was explored more than 50 experiments ago and the predictor has been updated since, mark it as "stale" and re-probe with a few samples to check if the anomaly landscape has changed.

---

## Multi-Dimensional Considerations

With many parameters, exhaustive exploration is impossible. Prioritize:

1. **Parameters the human flagged as interesting** in program.md
2. **Parameters with known physical significance** (Reynolds number > grid resolution)
3. **Parameters that showed anomalies in marginal sweeps**
4. **Interaction effects between parameters from groups 2 and 3**

Leave low-priority parameters at baseline values most of the time. Only vary them in exploratory random jumps.
