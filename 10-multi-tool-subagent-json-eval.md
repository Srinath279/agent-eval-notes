# Evaluating a Multi-Tool / Sub-Agent System with Nested-JSON Output

Fills the methodology gaps left by [01-agent-eval-fundamentals.md](01-agent-eval-fundamentals.md): how to grade a **nested JSON final output**, how to match trajectories when **many paths are valid**, how to evaluate **sub-agents and handoffs**, and how to design the dataset when the agent has **~10 tools**. Most of this already has code in `agent-evals/` — mapped inline.

## 1. Grading a nested-JSON final output (funnel + partial credit)

Never grade the whole blob with one exact-match — one wrong leaf zeros a mostly-correct answer and tells you nothing about *which* field failed. Grade as a **funnel**, cheap → expensive:

1. **Parse + JSON Schema validation** (deterministic, free). Required keys, types, enums, nested shapes. Fail here → score 0, skip all further grading (don't spend judge tokens on malformed output). → `evaluators/deterministic/json_schema_valid.py`
2. **Field-level scoring by JSONPath**, each field graded by its type:
   - **Deterministic fields** (IDs, enums, booleans, labels): exact match per path, e.g. `$.customer.tier == "gold"`. → `exact_match.py`, `label_match.py`
   - **Numeric fields**: tolerance bands (`abs(actual − expected) < ε`), never equality. → `thresholds.py`
   - **Lists**: decide *per field* whether order matters. Order-insensitive → compare as sets/multisets and score precision/recall/F1 over items, not pass/fail.
   - **Free-text fields** (summaries, explanations): LLM judge with a rubric scoped to **that field only** ("is `$.resolution.summary` faithful to the ticket?"), not the whole output.
3. **Weighted aggregation.** Fields are not equal — a wrong `refund_amount` outweighs a mediocre `summary`. Emit a weighted total **plus the per-field breakdown**, so regressions read "accuracy on `$.actions[].type` dropped 12%", not "score went down".

## 2. Trajectory matching when multiple paths are valid

With 10 tools and several steps there is almost never one golden sequence. **Do not exact-match a reference trajectory** — express expectations as **constraints**:

- **must-call**: `lookup_customer` appears somewhere in the run.
- **must-not-call**: `issue_refund` never fires on a non-refund case.
- **ordering**: `lookup_customer` **before** `issue_refund` (partial order, not full sequence).
- **argument checks** on the calls that do happen.

Score tool use as **precision/recall vs. the necessary-tool set**: recall = every required tool was called; precision = no unnecessary calls. → `evaluators/trajectory/tool_selection.py`, `deterministic/tool_called.py`

(Vertex AI's Gen AI eval service ships the same spectrum as prebuilt metrics: exact / in-order / any-order / precision / recall — see [02](02-eval-frameworks-comparison.md).)

## 3. Sub-agent (hierarchical) evaluation

End-to-end score alone can't attribute failure in a composed system. Evaluate three things separately:

1. **Component evals per sub-agent** — each sub-agent gets its **own small dataset** and is run in isolation. When e2e fails, component evals say *which* agent broke; without them every failure is just "the system got worse".
2. **Orchestrator routing + handoff quality** — did the parent pick the right sub-agent, and did it pass **sufficient context** in the handoff? Under-specified handoff prompts are a huge failure class: the sub-agent fails through no fault of its own. Also check the parent correctly **integrates** the sub-agent's result rather than ignoring or distorting it.
3. **Trace as a span tree, not a flat list** — each sub-agent invocation is a nested span containing its own tool-call spans (this is what the `core/adapters/` layer must preserve). That lets every trajectory metric from §2 be computed **per sub-agent** ("researcher is efficient, writer loops").

**Failure-attribution order** when e2e fails: which sub-agent's span diverged → that sub-agent's tool calls → **the handoff prompt it received**. Roughly half the time the bug is in the handoff, not the sub-agent.

## 4. Dataset design for N tools: the coverage matrix

Random sampling over-covers popular tools and never exercises rare ones. Build a matrix — for **each** of the 10 tools include cases where it:

- **(a) must be used** (happy path),
- **(b) must NOT be used** — a *near-miss* where a similar tool is correct; this is where tool-confusion bugs live,
- **(c) errors out** — paired with recovery scoring (`trajectory/recovery.py` + fault injection from `chaos.py`).

Plus **multi-tool cases** requiring 3+ tools in sequence, and the red-team set. ~20 cases per tool + ~30 multi-step cases catches most regressions.

## 5. pass@k vs pass^k (complement to 01/05)

Run each case k times (k = 3–5) and report **both**:

- **pass@k** — succeeds *at least once* in k runs → the **capability ceiling** ("can it do this at all?").
- **pass^k** — succeeds *every* time → the **reliability floor** (the number that matters for production).

A large gap between them = the agent *can* do the task but is inconsistent → usually a prompt/decoding problem, not a capability problem.

## 6. Judge inputs (two additions to the judge rules in [05](05-harness-gaps.md))

- **Give the judge the ground truth** where it exists (golden JSON, reference facts) — reference-based judging is far more reliable than judging from general knowledge.
- **Give the judge the trajectory** when scoring process quality — it cannot assess "did the agent use tools sensibly" from the final JSON alone.
- Validate rubric quality with **judge–human agreement (Cohen's κ)** on 50–100 hand-labeled outputs before trusting any judge number; prefer many binary criteria over one 1–10 scale.

## 7. The per-case scorecard (what one eval run should emit)

```
case_042:
  output:     schema=pass, fields=0.91 (miss: $.actions[1].amount), summary_judge=4/5
  trajectory: tools P=1.0 R=0.83 (missed: verify_identity), steps=7/budget 10, recovery=n/a
  subagents:  router=correct, billing_agent=pass, handoff_context=complete
  e2e:        goal_success=pass
```

Aggregate across the dataset with CIs, compare to baseline, gate in CI, sample online — per notes 04–06.

**TL;DR:** schema-validate then field-score the nested JSON with partial credit; constraint-check the trajectory instead of exact-matching it; evaluate each sub-agent in isolation *plus* the handoffs; give judges the reference and the trajectory; run everything k times — a single end-to-end score says *that* something broke, never *what*.
