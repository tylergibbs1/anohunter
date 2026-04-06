# Anomaly Hunter

An autonomous experiment loop that doesn't optimize — it searches for surprises.

You give it a system with predictable behavior and a model of what that behavior should be. The agent finds inputs where reality diverges from prediction, maps the boundaries of those divergences, investigates why, and writes up what it found.

The output isn't a better model. It's a list of "you should look at this" with evidence.

## How it works

```
1. You write a program.md defining the target system, predictor, and input space
2. The agent proposes experiments (hypothesis + inputs + rationale)
3. Target system and predictor both run with those inputs
4. A surprise scorer quantifies deviation from expected behavior
5. Anomalies get logged, boundary-mapped, and investigated
6. Discovery mode kicks in: literature check → hypotheses → discriminating experiments
7. You read the morning report and redirect
```

The agent balances **exploration** (~40% — trying new input regions) with **exploitation** (~60% — probing deeper around known anomalies). It's not hill-climbing. It's terrain-mapping.

## What's in this repo

This is a [Claude Code Skill](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills/overview) — a structured set of instructions that Claude loads when you ask it to hunt for anomalies.

```
skills/anomaly-hunter/
├── SKILL.md                          # Main instructions (loaded when triggered)
├── references/
│   ├── discovery-mode.md             # Hypothesis generation and testing protocol
│   ├── exploration-strategies.md     # Sampling, boundary finding, budget enforcement
│   ├── implementation-guide.md       # Step-by-step operational playbook
│   └── scoring-methods.md            # Surprise scoring implementations
└── evals/
    └── evals.json                    # 8 evaluation scenarios
```

## Quick start

1. Clone this repo (or copy `skills/anomaly-hunter/` into your project's skills directory)
2. Write a `program.md` in your working directory defining your target system, predictor, input parameters, and what you're curious about
3. Ask Claude to run an anomaly hunt

### Example program.md

```markdown
# Research directive: anomaly hunting in [your system]

## Target system
`run_simulation.py --config experiment.json`
Returns JSON with output metrics.

## Predictor
`predict.py --config experiment.json`
Returns same JSON schema with expected values.

## Input parameters
- param_a: float, 0.0–1.0
- param_b: int, 1–1000
- param_c: enum [low, medium, high]

## What I already know
- System matches predictions well for param_a < 0.5
- Nobody has tested param_b > 500

## What I'm curious about
- Are there parameter combinations where the prediction breaks down?
- What happens at the boundary of param_a ≈ 0.5?

## Constraints
- Each experiment must complete in under 60 seconds
- No more than 10 consecutive boundary probes before returning to exploration
```

## The two modes

### Anomaly hunting

Find where predictions fail. Map boundaries. Check reproducibility. The core loop runs experiments, scores surprise (deviation normalized by local predictor accuracy), and logs anything above threshold.

### Discovery mode

Triggered after an anomaly is confirmed and boundary-mapped. The agent:

1. **Checks the literature** — has anyone described this before? If yes, logs the reference and moves on. This prevents enthusiastic rediscovery of well-known phenomena.
2. **Generates hypotheses** — 3-5 specific, testable candidates. "Interaction effect" is not a hypothesis. "Upstream wake low-pressure at 2-3 diameters creates reverse flow" is.
3. **Designs discriminating experiments** — for each hypothesis pair, finds inputs that would produce different outcomes. Logs predictions *before* running.
4. **Reports honestly** — a defended mechanism, genuine ambiguity, or "this is weirder than I can explain."

## What you do

1. **Write the program.md** — define the territory, say what you're curious about, set constraints
2. **Read the morning report** — judge which anomalies are real vs. artifacts (requires your domain expertise)
3. **Update program.md** — redirect based on findings
4. **Review discovery reports** — the agent proposes mechanisms, you judge whether they're correct

## Success criteria

**Tier 1** (first week): Finds a reproducible anomaly you didn't know about. Boundary mapping is accurate enough to form a hypothesis.

**Tier 2** (first month): Literature checks catch known phenomena. Discovery reports produce mechanisms a domain expert calls plausible. Agent honestly says "I don't know" when warranted.

**Tier 3** (aspirational): A finding survives expert review and independent reproduction.

## License

MIT
