# Implementation Guide

Step-by-step guide for the agent to execute an anomaly hunt from start to finish. This is the operational playbook.

---

## Phase 0: Setup and Validation

Before any experiments run, validate the entire pipeline.

### Step 1: Read and parse program.md

```
1. Read program.md from working directory
2. Extract:
   - Target system command and expected output format
   - Predictor command and expected output format
   - Parameter names, types, and valid ranges
   - Known behaviors and prior results
   - Research questions
   - Constraints (time limits, forbidden regions, budget rules)
3. If any required section is missing, ask the human before proceeding
```

### Step 2: Validate target system

```bash
# Create a test config with mid-range values for all parameters
echo '{"param1": <midpoint>, "param2": <midpoint>, ...}' > test_config.json

# Run target system
<target_command> --config test_config.json > target_output.json

# Verify output schema matches documentation
# Check: does it return the expected keys? Are values numeric? Is it JSON?
```

If the target system fails on mid-range inputs, stop and report to the human. Don't proceed with a broken target.

### Step 3: Validate predictor

```bash
# Run predictor with same test config
<predictor_command> --config test_config.json > predictor_output.json

# Verify:
# 1. Output schema matches target system schema (same keys)
# 2. Values are in the same ballpark (not off by orders of magnitude)
# 3. Predictor completes within time constraints
```

### Step 4: Validate scoring

```
1. Compute surprise score for the test experiment
2. Verify score is a finite number (not NaN, not infinity)
3. Score should be low for this baseline test (mid-range inputs usually boring)
```

### Step 5: Create directory structure

```bash
mkdir -p anomaly-log discovery-log experiment-log reports
echo '[]' > experiment-log/experiments.jsonl  # Initialize empty
```

---

## Phase 1: Initial Survey (Experiments 1-30)

Goal: broad coverage of the input space. No deep investigation yet.

### Strategy

Use the agent's domain knowledge + systematic sampling to pick 20-30 well-distributed points across the input space.

```
For the first 30 experiments:
1. Divide each parameter's range into 5 equal segments
2. Ensure each segment is sampled at least once for each parameter
3. Let the agent use domain knowledge to pick specific values within segments
4. Use relative deviation scoring (not enough history for statistical methods)
5. Set threshold at 2.0σ (generous — accept false positives)
6. Log all results but don't enter boundary mapping mode yet
7. After 30 experiments, review anomaly candidates and prioritize
```

### What to record per experiment

```json
{
  "id": 1,
  "timestamp": "2026-04-06T03:00:00Z",
  "type": "exploration",
  "hypothesis": "Baseline test at mid-range values, expect predictor to match well",
  "inputs": {"reynolds": 5000, "grid": 200, "obstacle": "circle", "...": "..."},
  "actual": {"drag_coeff": 1.23, "lift_coeff": 0.045, "shed_freq": 12.3},
  "predicted": {"drag_coeff": 1.21, "lift_coeff": 0.042, "shed_freq": 12.1},
  "surprise_score": 0.8,
  "anomaly": false,
  "notes": "As expected, good agreement at moderate Reynolds number"
}
```

---

## Phase 2: Steady-State Exploration (Experiments 31+)

Goal: systematic search with anomaly investigation.

### Per-experiment workflow

```
1. Review state:
   - How many anomalies found so far?
   - What regions are well-explored vs. sparse?
   - What does program.md say to investigate?
   - Am I within exploitation budget?

2. Decide experiment type:
   - If exploitation budget allows and there's an uninvestigated anomaly:
     → Boundary mapping (exploitation)
   - If budget exhausted or no anomalies to investigate:
     → New region (exploration)

3. Write experiment spec:
   HYPOTHESIS: [what I expect and why]
   INPUTS: [exact parameters]
   RATIONALE: [why this experiment now]

4. Execute:
   - Write inputs to config file
   - Run target system, capture output
   - Run predictor, capture output
   - Compute surprise score

5. Record:
   - Append to experiments.jsonl
   - If anomaly: create anomaly-NNN.md, enter investigation sub-loop
   - If not anomaly: update coverage map in summary.md

6. Repeat
```

### Anomaly investigation sub-loop

When surprise score exceeds threshold:

```
1. CONFIRM: Rerun exact same inputs 3 times
   - If results vary by >5%: flag as non-deterministic, note variance, return to exploration
   - If consistent: proceed

2. MARGINAL SWEEP: For each parameter:
   - Run at +10% and -10% (holding others constant)
   - Record: does anomaly persist, strengthen, or disappear?
   - Identify which parameters affect the anomaly

3. BOUNDARY FIND: For each sensitive parameter:
   - Use bisection between anomaly point and nearest non-anomaly point
   - Find boundary to within 2% of parameter range
   - Record: sharp transition or gradual?

4. INTERACTION CHECK (if 2+ sensitive parameters):
   - Vary two sensitive parameters simultaneously
   - Check if effects are additive or interactive

5. WRITE REPORT: Create anomaly-NNN.md with full data

6. CHECK BUDGET: If over exploitation limit, return to exploration
```

---

## Phase 2.5: Discovery Mode (per confirmed anomaly)

Triggered after boundary mapping completes on a confirmed anomaly, if discovery mode is enabled.

### Pre-requisites

Before entering discovery mode for anomaly #NNN:
- Reproducibility: confirmed (3 runs within 5%)
- Boundary: mapped (sensitive parameters identified)
- Budget: exploitation budget has room for 5-15 more experiments

### Literature check workflow

```
1. Construct search queries:
   - "[observable] [system type]" (e.g., "drag reduction tandem cylinders")
   - "[observable] [input regime]" (e.g., "drag reduction Reynolds 10000")
   - "[transition shape] [system type]" (e.g., "sharp transition cylinder wake")

2. Search using web search tools:
   - Run 3-5 different queries
   - Check arxiv, google scholar, github issues, target system docs

3. For each relevant result:
   - Read the abstract/summary
   - Check if the described phenomenon matches your anomaly
   - Note: same observable? Same input regime? Same mechanism?

4. Record all searches and results in the discovery report
   (even negative results — "searched X, found nothing relevant")

5. Decision:
   - Known phenomenon → log reference, mark as known, exit discovery mode
   - Partially known → proceed, framing hypotheses in context of literature
   - Not found → proceed with novelty caveat
```

### Hypothesis generation workflow

```
1. Review the boundary data:
   - Which parameters are sensitive?
   - What's the transition shape (sharp/gradual)?
   - Where exactly does it occur?

2. For each sensitive parameter, reason about mechanisms:
   - What physical/mathematical process changes at this value?
   - What does the predictor assume that might break here?

3. Generate 3-5 hypotheses:
   - At least 1 "interesting" (real phenomenon)
   - At least 1 "boring" (numerical artifact, predictor bug)
   - Each must predict a specific different outcome for a testable input

4. Write hypotheses with predictions in the experiment log
```

### Discriminating experiment workflow

```
1. For each pair of surviving hypotheses:
   - Design an input where they predict different outcomes
   - Record both predictions BEFORE running

2. Run the experiment:
   - Record actual output
   - Compare to each hypothesis's prediction
   - Update belief state

3. After each experiment:
   - Can any hypothesis be eliminated? (prediction directly contradicted)
   - Can any hypothesis be weakened? (result inconsistent but not definitive)
   - Is one hypothesis dominant? (>80% belief after 3+ supporting experiments)

4. Stop when:
   - One hypothesis dominates, OR
   - All distinguishing experiments have been run, OR
   - Discovery budget exhausted

5. Write discovery report (discovery-log/discovery-NNN.md)
6. Git commit the discovery report
```

### Experiment log entries for discovery mode

```json
{"id": 150, "type": "discovery_literature", "anomaly_ref": "017", "searches": ["tandem cylinder drag reduction", "..."], "findings": "Zdravkovich 1987 describes...", "assessment": "partially_known"}
{"id": 151, "type": "discovery_discriminating", "anomaly_ref": "017", "distinguishes": ["H1", "H3"], "predictions": {"H1": "drag ~0.74 at 400x400", "H3": "drag ~1.21 at 400x400"}, "actual": {"drag_coeff": 0.76}, "interpretation": "H1 supported, H3 weakened"}
```

---

## Phase 3: Report Generation

After the experiment session ends (time limit, count limit, or human interruption).

### Nightly summary template

```markdown
# Anomaly hunt: [date]

## Session stats
- Experiments run: [N]
- Duration: [start] to [end]
- Anomalies found: [N] (scores: [comma-separated list])
- Confirmed reproducible: [N]
- Input space coverage: ~[X]% of defined ranges
- New regions explored: [list]

## Top anomalies (ranked by surprise score)

### 1. [Short descriptive name] ([score]σ)
- **Inputs**: [key parameters]
- **Expected**: [key predicted values]
- **Actual**: [key actual values]
- **Boundary**: [parameter ranges where anomaly occurs]
- **Reproducibility**: [N/N within X%]
- **Agent hypothesis**: [1-2 sentences]
- **Full report**: anomaly-log/anomaly-NNN.md

### 2. [...]

## Regions confirmed as expected
[Where predictions held up. Saves the human from re-exploring these.]
- [Region description]: [N] experiments, all within [X]σ

## Suggested next directions
[Agent's recommendations for program.md updates]
1. [Direction 1 and why]
2. [Direction 2 and why]
3. [Direction 3 and why]
```

---

## Experiment Log Format

### experiments.jsonl

One JSON object per line. Append-only.

```json
{"id": 1, "timestamp": "...", "type": "exploration", "hypothesis": "...", "inputs": {...}, "actual": {...}, "predicted": {...}, "surprise_score": 0.8, "anomaly": false, "notes": "..."}
{"id": 2, "timestamp": "...", "type": "exploration", "hypothesis": "...", "inputs": {...}, "actual": {...}, "predicted": {...}, "surprise_score": 3.2, "anomaly": true, "anomaly_id": "001", "notes": "..."}
{"id": 3, "timestamp": "...", "type": "exploitation", "anomaly_ref": "001", "param_varied": "reynolds", "hypothesis": "...", "inputs": {...}, "actual": {...}, "predicted": {...}, "surprise_score": 2.9, "anomaly": true, "notes": "boundary probe"}
```

### summary.md

Updated after every 10 experiments. Contains:

```markdown
# Exploration Summary

## Coverage (updated after experiment #[N])

| Parameter | Range | Explored segments | Sparse segments | Unexplored |
|---|---|---|---|---|
| reynolds | 100-100000 | 100-5000 | 5000-20000 | 20000-100000 |
| grid | 50-500 | 50-300 | 300-400 | 400-500 |
| ... | ... | ... | ... | ... |

## Known anomalies
- #001: [short description] at [key params], [score]σ, [status: confirmed/investigating/resolved]
- #002: ...

## Boring regions (predictions hold)
- Low-Re single obstacle: 23 experiments, max deviation 0.8σ
- ...
```

---

## Error Handling

### Target system crashes

```
1. Log as experiment with "target_error" status
2. Record the error message
3. Check: are the inputs within valid ranges per program.md?
   - If yes: this is a real finding (system crashes on valid input) — log as anomaly type "crash"
   - If no: agent proposed invalid inputs — fix input generation
4. Do NOT retry with same inputs unless investigating the crash boundary
```

### Predictor crashes

```
1. Log as experiment with "predictor_error" status
2. The predictor can't handle this input region — worth noting
3. Score as maximum surprise (the predictor's model breaks down here)
4. Note this as a predictor limitation, not necessarily a target anomaly
```

### Target system takes too long

```
1. If experiment exceeds time limit from program.md: kill the process
2. Log as "timeout" with the inputs that caused it
3. Slow regions are often interesting (phase transitions, convergence issues)
4. Note the timeout and move on — don't block the entire hunt
```

### Outputs don't match schema

```
1. If target output is missing expected keys: log and skip
2. If values are non-numeric (strings, nulls): log and skip
3. If output is valid but unexpected format: try to parse, log any issues
4. After 3 consecutive schema errors: stop and ask the human
```

---

## Git Integration

Each anomaly should be committed with full reproducibility context:

```bash
# When an anomaly is confirmed (after reproducibility check)
git add anomaly-log/anomaly-NNN.md
git add experiment-log/experiments.jsonl
git commit -m "anomaly #NNN: [short description] at [key params] ([score]σ)

Reproducible across 3 runs within [X]%.
See anomaly-log/anomaly-NNN.md for full boundary mapping."

# When discovery mode completes on an anomaly
git add discovery-log/discovery-NNN.md
git add experiment-log/experiments.jsonl
git commit -m "discovery #NNN: [short description]

[one-line summary: mechanism identified / ambiguity reported / flagged for human]
Based on anomaly #NNN. See discovery-log/discovery-NNN.md."
```

This creates a git history where each commit is a discovery. You can `git log` to see the timeline of findings.

---

## Configuration Defaults

When program.md doesn't specify, use these defaults:

| Setting | Default | Notes |
|---|---|---|
| Scoring method | Contextual weighted surprise | Falls back to relative deviation for first 10 experiments |
| Anomaly threshold | 2.5σ (2.0σ for first 20 experiments) | Adjust if anomaly rate is outside 5-15% |
| Reproducibility runs | 3 | Minimum to confirm an anomaly |
| Reproducibility tolerance | 5% | Max variance across runs to call it reproducible |
| Exploitation budget | 60% | Max fraction of experiments on boundary mapping |
| Max consecutive exploitation | 10 | Force exploration after this many boundary probes |
| Max consecutive exploration | 5 | Check for uninvestigated anomalies after this many |
| Boundary resolution | 2% of parameter range | Stop bisecting when closer than this |
| Neighbor radius | K=10 nearest | For regional statistics |
| Summary update frequency | Every 10 experiments | Update summary.md |
| Nightly report | End of session | Generate reports/nightly-YYYY-MM-DD.md |
