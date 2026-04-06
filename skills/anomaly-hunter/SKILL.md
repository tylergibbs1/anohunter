---
name: anomaly-hunter
description: Autonomous experiment loop that searches for surprises in deterministic systems. Compares actual outputs against a predictor, logs anomalies, maps boundaries, generates hypotheses via discriminating experiments, and produces discovery reports. Triggers on "anomaly hunt," "find anomalies," "hunt for surprises," "find where the model breaks," "discover unexpected behavior," "explore parameter space," "discovery mode," "explain this anomaly," "hypothesis testing," "is this already known." For optimization loops, see auto-research.
metadata:
  version: 2.0.0
---

# Anomaly Hunter

Autonomous experiment loop that explores for surprises, not optimization. Finds inputs where a system diverges from its predictor, maps boundaries, and investigates why.

## Before Hunting

Gather this context (ask if not provided):

1. **Target system** — Accepts parameterized inputs, returns measurable outputs, runs in minutes.
2. **Predictor** — Model of expected behavior to compare against.
3. **Input space** — Parameters, types, and valid ranges.
4. **Prior knowledge** — What's known to work or fail? Where has nobody looked?
5. **Discovery mode** — Should confirmed anomalies trigger hypothesis generation? (default: yes)

---

## Core Principles

- **Explore, don't optimize.** Map terrain. Balance ~60% exploitation (boundary mapping near anomalies) with ~40% exploration (new regions).
- **Surprise over magnitude.** A 0.3% deviation where predictions are usually within 0.01% beats a 10% deviation in a noisy region. Use contextual scoring.
- **Characterize, don't just detect.** Map the boundary, check reproducibility (3x minimum), write it up for a domain expert.
- **The predictor sets the bar.** If everything is anomalous, the predictor is too inaccurate. If nothing is, the model is complete.

---

## The program.md

Human's control surface. Must exist in the working directory before the hunt begins.

### Required sections

```markdown
# Research directive: [title]

## Target system
[Path, invocation command, output format.]

## Predictor
[Path or description of expected-behavior model.]

## Input parameters
[Parameter name, type, valid range, units.]

## What I already know
[Known behaviors, validated regions.]

## What I'm curious about
[Research questions. Regions of interest.]

## Constraints
[Max runtime. Forbidden inputs. Exploration budget rules.]

## Discovery mode behavior (optional)
[When to enter discovery mode. Literature search scope. Hypothesis rules.]
```

---

## Experiment Lifecycle

### Single experiment

```
1. Propose experiment (hypothesis + inputs + rationale)
2. Run target → actual_output
3. Run predictor → predicted_output
4. Score surprise (contextual weighted, see references/scoring-methods.md)
5. Score > threshold → log as ANOMALY → decide: probe boundary or move on
6. Score ≤ threshold → log as CONFIRMED EXPECTED, continue
```

### Anomaly investigation

```
1. Confirm: rerun 3x. Variance >5% → flag non-deterministic, move on.
2. Marginal sweep: ±10% each parameter, identify which matter.
3. Boundary find: bisect between anomaly and non-anomaly points.
4. Characterize: sharp cliff or gradual drift?
5. Write anomaly report → return to exploration (enforce budget).
```

For detailed investigation protocols: [references/implementation-guide.md](references/implementation-guide.md)

---

## Discovery Mode

Triggered after an anomaly is confirmed reproducible with boundary mapped. Full protocol: [references/discovery-mode.md](references/discovery-mode.md)

```
1. LITERATURE CHECK — search external sources first. If known, log and move on.
2. HYPOTHESES — 3-5 specific, testable, grounded candidates.
   Bad: "interaction effect." Good: "upstream wake low-pressure at 2-3 diameters
   creates reverse flow suppressing downstream shedding."
3. DISCRIMINATING EXPERIMENTS — for each hypothesis pair, find inputs that
   produce different predicted outcomes. Log predictions BEFORE running.
4. RUN & UPDATE — track belief state. Kill hypotheses the data rejects.
5. REPORT — surviving mechanism, honest ambiguity, or "open problem."
```

Never claim discovery without a literature check. Never hide disconfirming experiments.

---

## Scoring

**Default**: contextual weighted surprise — deviation normalized by local predictor accuracy.

| Phase | Threshold |
|---|---|
| Early (< 20 experiments) | 2.0σ |
| Steady-state (20+) | 2.5σ |
| Boundary mapping | 1.5σ |

Target 5-15% anomaly rate. If above 30%, predictor is too inaccurate or threshold too low.

For scoring methods and implementation: [references/scoring-methods.md](references/scoring-methods.md)

---

## File Structure

```
working-directory/
├── program.md                    # Human's research directive
├── anomaly-log/anomaly-NNN.md    # Per-anomaly reports
├── discovery-log/discovery-NNN.md # Discovery reports
├── experiment-log/
│   ├── experiments.jsonl         # All experiments (one JSON per line)
│   └── summary.md               # Coverage map, known anomalies
└── reports/nightly-YYYY-MM-DD.md # End-of-session summary
```

---

## Output Formats

### Anomaly report (anomaly-log/anomaly-NNN.md)

```markdown
## Anomaly #[N]
**Found**: [timestamp] (experiment #[M]) | **Score**: [X]σ

### Inputs
[Full parameter set]

### Results
**Expected**: [predictor output] | **Actual**: [target output]

### Boundary mapping
[Which parameters matter, transition points, sharp vs gradual]

### Reproducibility
[N/N runs within X%]

### Agent notes
[Hypothesis, references to domain knowledge]
```

### Discovery report (discovery-log/discovery-NNN.md)

```markdown
## Discovery report: [name]
**Based on**: Anomaly #[N]

### Literature check
[What was searched, what was found]

### Hypotheses tested
1. [Hypothesis] — [ELIMINATED/SUPPORTED] by experiment #[N]

### Current best story
[Surviving explanation or honest uncertainty]

### What would close this
[Specific next steps]

### Honest caveats
[Limitations, sample sizes, search scope]
```

### Nightly summary (reports/nightly-YYYY-MM-DD.md)

```markdown
# Anomaly hunt: [date]
## Stats
[Experiments run, anomalies found, coverage, discoveries]

## Top anomalies
[Ranked with 2-3 sentence summaries]

## Regions confirmed boring
[Where predictions held]

## Suggested next directions
[Recommendations for program.md updates]
```

---

## Core Loop

```
1. Read program.md, validate target and predictor systems
2. Initial survey: 20-30 experiments across parameter space
3. Steady-state loop:
   a. Read state (past experiments, anomalies, program.md)
   b. Decide: explore new region or investigate anomaly (60/40)
   c. Propose and run experiment
   d. Score and log
   e. If anomaly: investigation sub-loop → boundary mapping
   f. If confirmed + mapped + discovery enabled: discovery sub-loop
   g. Update summary every 10 experiments
4. Generate nightly report
```

For exploration strategies: [references/exploration-strategies.md](references/exploration-strategies.md)

---

## Failure Modes

| Problem | Fix |
|---|---|
| Everything anomalous | Tighten predictor or use contextual scoring |
| Nothing anomalous | Model is complete — move on |
| Stuck in one region | Enforce exploration budget |
| Anomalies are bugs | Reproducibility check (3x) |
| Degenerate high scores | Constrain input space in program.md |
| Vague hypotheses | Require "X causes Y at Z" specificity |
| Claims known phenomena | Literature check is mandatory and first |
| Hides disconfirming data | Require all experiments in discovery report |

---

## What the Human Does

1. **Write program.md** — define territory, curiosity, constraints
2. **Read morning report** — judge which anomalies are real (requires domain expertise)
3. **Update program.md** — redirect based on findings
4. **Review discovery reports** — judge whether proposed mechanisms are correct

---

## Success Criteria

**Tier 1** (first week): Finds a reproducible anomaly the human didn't know about. Boundary mapping sufficient to form a hypothesis.

**Tier 2** (first month): Literature check catches ≥80% of known phenomena. At least one discovery report a domain expert calls "plausible." Agent honestly reports "I don't know" when warranted.

**Tier 3** (aspirational): A finding survives expert review and independent reproduction. Most runs won't hit this — the value is Tier 1 and 2 happening reliably.

---

## Related Skills

- **auto-research**: Optimization loops (maximize a metric, not explore for surprises)
- **designing-agents**: Architecture patterns for building custom hunting agents
