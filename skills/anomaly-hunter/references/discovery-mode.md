# Discovery Mode

Deep guide for the hypothetico-deductive loop that kicks in after an anomaly is confirmed reproducible and its boundary mapped. This is where anomaly hunting becomes science.

---

## When to Enter Discovery Mode

Discovery mode triggers when ALL of these are true:

1. An anomaly has been detected (surprise score above threshold)
2. Reproducibility confirmed (3 runs within 5% of each other)
3. Boundary mapped (sensitive parameters identified, transitions characterized)
4. Discovery mode is enabled in program.md (default: yes)

Do NOT enter discovery mode for:
- Non-reproducible anomalies (variance > 20% across runs)
- Anomalies clearly caused by degenerate inputs (edge of valid range)
- Anomalies the human has already flagged as "known, skip" in program.md

---

## Step 1: Literature Check

This is the most important step. Without it, the agent generates excited reports about well-known phenomena.

### What to search

For each anomaly, construct search queries from:
- The specific observable (e.g., "drag reduction," "frequency shift," "convergence failure")
- The system type (e.g., "tandem cylinders," "neural network," "finite element")
- The input regime (e.g., "Reynolds number 10000-15000," "high learning rate," "fine grid")
- The phenomenon's shape (e.g., "sharp transition," "hysteresis," "bistability")

### Where to search

Use web search tools to check:
1. Google Scholar / arxiv — for academic papers
2. GitHub issues — for known software behaviors
3. Stack Overflow / relevant forums — for practitioner knowledge
4. The target system's own documentation — for documented edge cases
5. Wikipedia — for named phenomena

### How to report

Always document:
```markdown
### Literature check
**Searched**: [exact queries used]
**Sources checked**: [list of sources]
**Found**: [what was found, with citations]
**Assessment**: 
  - "Known phenomenon: [name], described in [citation]" → mark as known, move on
  - "Partially known: phenomenon class matches [citation] but specific 
    [parameter/regime/behavior] appears under-characterized" → proceed to hypotheses
  - "No matches found in [N] searches across [sources]" → proceed to hypotheses
```

### Decision tree

```
Literature check results:
  |
  ├── Phenomenon is well-documented with a name
  │   → Log the reference, mark anomaly as "known phenomenon"
  │   → Add to anomaly report: "This is [name], see [citation]"
  │   → Return to exploration. Do NOT claim discovery.
  │
  ├── Phenomenon class is known but specific behavior is novel
  │   → Log what's known and what's new
  │   → Proceed to hypothesis generation, framing hypotheses
  │     in context of existing literature
  │
  └── No matches found
      → Note what was searched (scope the "novelty" claim)
      → Proceed to hypothesis generation
      → Include caveat: "searched [X] but may have missed relevant work"
```

---

## Step 2: Hypothesis Generation

### Requirements for each hypothesis

Every hypothesis must be:

1. **Specific enough to make predictions**: "The upstream wake's turbulent transition at Re~10,000 suppresses downstream vortex shedding" — not "there's an interaction."
2. **Testable within the system's input space**: You can design an experiment to distinguish it from alternatives.
3. **Grounded in domain knowledge**: Based on actual physics/math/CS principles, not vague hand-waving.
4. **Distinct from other hypotheses**: Each must predict at least one different outcome.

### Generating good hypotheses

```
1. Start with the boundary data: what parameters matter, and what's the transition shape?
2. For each sensitive parameter, ask: what physical/mathematical mechanism
   would produce a transition at this value?
3. Consider categories:
   - Physical mechanism (real phenomenon the predictor doesn't model)
   - Numerical artifact (discretization, convergence, floating point)
   - Predictor limitation (the predictor's model is too simplified here)
   - Interaction effect (multiple parameters combining non-linearly)
4. Generate 3-5 candidates. Include at least one "boring" explanation
   (numerical artifact, predictor bug) alongside the interesting ones.
```

### Format

```markdown
### Hypotheses

1. **[Short name]**: [Specific mechanistic claim]. Predicts: [what this 
   hypothesis says will happen in a specific test].

2. **[Short name]**: [Specific mechanistic claim]. Predicts: [different 
   outcome for same or related test].

3. **[Short name]**: [Specific mechanistic claim]. Predicts: [yet another 
   different outcome].

4. **Null: numerical artifact**: [Specific claim about what numerical 
   effect could produce this]. Predicts: [effect disappears/changes with 
   grid refinement, different solver settings, etc.]
```

### Common hypothesis categories by system type

| System type | Common "interesting" hypotheses | Common "boring" hypotheses |
|---|---|---|
| Physics simulation | Phase transition, mode coupling, resonance, turbulence onset | Grid dependence, time step sensitivity, boundary effects |
| ML model | Distribution shift, feature interaction, memorization | Label noise, data leakage, preprocessing artifact |
| Compiler | Optimization interaction, undefined behavior exploitation | Flag ordering sensitivity, version-specific behavior |
| Distributed system | Contention, queueing, clock skew | GC pause, connection pool exhaustion, log buffer |

---

## Step 3: Discriminating Experiment Design

This is the hardest part. The goal: find inputs where hypothesis A predicts outcome X and hypothesis B predicts outcome Y.

### The key question

For each pair of hypotheses, ask:

> "What input would produce different results depending on which hypothesis is true?"

If you can't answer this for a pair, the hypotheses aren't distinct enough. Refine them until they make different predictions.

### Design protocol

```
For hypotheses H1, H2, H3:
  
  Experiment D1 (distinguishes H1 from H2):
    Input: [specific parameters]
    H1 predicts: [outcome A]
    H2 predicts: [outcome B]
    If result ≈ A: H1 supported, H2 weakened
    If result ≈ B: H2 supported, H1 weakened
  
  Experiment D2 (distinguishes H1 from H3):
    Input: [specific parameters]
    H1 predicts: [outcome C]
    H3 predicts: [outcome D]
    ...
  
  Experiment D3 (distinguishes H2 from H3):
    ...
```

### Critical rule: predict BEFORE running

Write down what each hypothesis predicts for the discriminating experiment BEFORE running it. This prevents retroactive "well, hypothesis 2 would have predicted that." Log the predictions in the experiment record.

```json
{
  "id": 156,
  "type": "discovery_discriminating",
  "anomaly_ref": "017",
  "distinguishes": ["H1", "H3"],
  "predictions": {
    "H1": "drag coefficient should remain low (~0.74) at Re=12400 with 400x400 grid",
    "H3": "drag coefficient should increase toward predicted value (~1.21) at finer grid"
  },
  "actual": {"drag_coeff": 0.76},
  "interpretation": "H1 supported (grid-independent), H3 weakened (not a resolution artifact)"
}
```

### How many discriminating experiments?

- Minimum: one per pair of hypotheses that remain alive. For N hypotheses, that's N*(N-1)/2 pairs.
- Typical: 5-15 total experiments per anomaly investigation.
- Maximum: governed by exploitation budget in program.md.

---

## Step 4: Belief State Tracking

After each discriminating experiment, update a belief state:

```markdown
### Belief state (after experiment #158)

| Hypothesis | Status | Key evidence |
|---|---|---|
| H1: Wake turbulent transition | 70% — SUPPORTED | Grid-independent (#156), Re-threshold matches (#157) |
| H2: Predictor spacing correction error | 10% — WEAKENED | Effect persists with corrected spacing model (#155) |
| H3: Numerical resolution artifact | 5% — NEAR ELIMINATED | Effect persists at 400x400 and 500x500 (#156) |
| H4: Solver-specific discretization | 15% — NOT TESTED | Would require a second solver to rule out |
```

### When to stop

Stop the discovery investigation when:

1. **One hypothesis dominates** (>80% belief) and at least 3 discriminating experiments support it
2. **Multiple hypotheses survive** but you've exhausted the experiments that can distinguish them within the available input space — report the ambiguity
3. **All hypotheses eliminated** — the anomaly is weirder than expected, flag for human
4. **Discovery budget exhausted** — report current state and return to exploration

### What "eliminated" means

A hypothesis is eliminated when a discriminating experiment produces a result that directly contradicts its prediction. Not "slightly different from what it predicted" — directly contradicts.

A hypothesis is "weakened" when results are inconsistent with it but don't definitively rule it out (could be noise, could be an edge case of the hypothesis).

---

## Step 5: Writing the Discovery Report

### Structure

```markdown
## Discovery report: [descriptive name]

**Based on**: Anomaly #[N] (see anomaly-[N].md for raw data)
**Discovery experiments**: #[M] through #[P] ([count] experiments)

### Literature check
**Searched**: [exact queries]
**Sources**: [list]
**Found**: [results with citations]
**Assessment**: [known / partially known / apparently novel]

### Hypotheses tested

1. **[Name]** — [STATUS] 
   Evidence: experiment #[N] showed [result], which [supports/contradicts] 
   this because [reason].

2. **[Name]** — [STATUS]
   Evidence: ...

3. ...

### Current best story
[The surviving explanation with the evidence chain. Or honest uncertainty:
"hypotheses A and C both survive, I couldn't distinguish them because 
[specific limitation]."]

### What would close this
[Specific experiments, tools, or data that would resolve remaining ambiguity.
Be concrete: "Run the same configuration in OpenFOAM" not "use a different solver."]

### Honest caveats
- [How many experiments support the conclusion — is it enough?]
- [What the literature search covered and what it might have missed]
- [Limitations of the agent's domain knowledge]
- [Experiments that didn't fit the preferred story]

**Commit**: [hash]
```

### What success looks like

One of these outcomes counts:

1. **Correct mechanism identified**: The agent's surviving hypothesis turns out to be correct when a domain expert reviews it. Rare but possible.

2. **Focused human attention**: The agent ruled out obvious explanations and honestly flagged "I don't know why this happens" — focusing the human on something genuinely puzzling instead of noise.

3. **False discovery prevented**: The agent found the phenomenon is already known, prevented a false discovery claim, and saved the human from embarrassment. Most common and still valuable.

### The failure mode to watch for

The agent constructs a plausible-sounding mechanistic story that fits the data but is actually wrong. LLMs are very good at generating plausible stories. The literature check and "predict before running" rule both help, but don't eliminate this risk. The final safeguard is always the human reading the report.

---

## Integration with the Core Loop

Discovery mode is a sub-loop within the main anomaly investigation:

```
Anomaly detected
  → Reproducibility check (3 runs)
    → Boundary mapping
      → Discovery mode (if enabled):
        → Literature check
        → Hypothesis generation
        → Discriminating experiments
        → Discovery report
      → Return to exploration
```

Budget accounting: discovery mode experiments count toward the exploitation budget. If the total investigation (boundary mapping + discovery) exceeds the exploitation budget, pause and return to exploration. Discovery can resume later.

---

## Examples of Good vs. Bad Discovery Work

### Good: specific, testable, honest

```
Hypothesis: "The drag reduction occurs because turbulent transition in the 
upstream wake (onset at Re~10,000 for this geometry) produces higher-frequency 
fluctuations that disrupt the downstream cylinder's regular shedding pattern."

Discriminating experiment: "If this is correct, the effect should disappear 
when I add artificial turbulence to the inlet at Re=8,000 (below transition). 
H1 predicts: effect appears. H2 (predictor bug) predicts: no change."

Result: "Effect appeared at Re=8,000 with turbulent inlet. H1 supported."

Caveat: "Only 3 experiments support this. The mechanism is consistent with 
Zdravkovich (1987) for the phenomenon class but the Re-threshold coupling 
has not been independently verified."
```

### Bad: vague, unfalsifiable, overclaimed

```
Hypothesis: "There's some kind of interaction between the cylinders."

Experiment: "Run it at different Reynolds numbers."

Result: "The effect changes. This confirms the interaction hypothesis."

Conclusion: "We have discovered a new fluid dynamics phenomenon."
```

The first is science. The second is confabulation with a lab coat.
