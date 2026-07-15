# Eval Harness — Gap Analysis (What's Missing to Actually Build It)

A review of the plan in [04-eval-harness-plan-langfuse-gcp.md](04-eval-harness-plan-langfuse-gcp.md) against what it takes to build and operate the harness. Grouped by how much each gap hurts if skipped.

## A. Critical gaps (will block or corrupt the harness)

### 1. Trace schema normalization (adapter layer)
Different agents (LangGraph, custom, ADK, …) log different trace shapes into Langfuse. Evaluators need one canonical `Trace / ToolCall` model, so the harness needs a **per-framework trace adapter** that maps raw Langfuse observations → the harness schema.
- Without it, every evaluator grows agent-specific parsing and reusability dies.
- The single most underestimated piece of the build.
- Location: `core/adapters/` (one adapter per agent framework, registered in the agent's YAML config).

### 2. PII redaction / anonymization
Real tickets carry names, emails, order IDs. Before traces become golden-dataset items or are sent to a judge LLM:
- Redaction step (Cloud DLP, or regex + NER pass) in both the promote-to-dataset path and the judge input path.
- Data retention policy for traces and datasets.
- **Compliance blocker if skipped.**

### 3. Judge engineering (beyond calibration)
- **Judge prompt versioning** — store rubrics in Langfuse Prompt Management; stamp `judge_model + rubric_version` into every score's metadata. (The plan says "pin it" but has no mechanism.)
- **Bias controls** — position bias, verbosity bias, self-preference. Randomize orderings; force structured JSON verdicts with reasoning-before-score.
- **Score caching / idempotency** — cache key = `(trace_id, evaluator, rubric_version)`. Never re-pay for a judge call; never double-write scores when the online worker retries.

### 4. Statistical rigor in the CI gate
"Fail if goal_success < 0.85" on a 60-item set is noise-driven — one flaky case flips the build.
- **k repeats per case (pass^k runner)** — non-determinism support built into the runner.
- **Bootstrap confidence intervals** when comparing a run to baseline; gate on significant regression, not raw point difference.
- **Baseline-run registry** — an explicit record of which run is the baseline, and who/what promotes a new baseline.

### 5. "Evals for the evals" — tests for the harness itself
- Unit tests for every deterministic evaluator.
- **Fixture traces** — golden traces with known correct scores; assert evaluators reproduce them.
- **Mock JudgeClient** so the harness test-suite runs in CI without Vertex spend.
- If evaluators are buggy, every downstream metric is fiction.

## B. Important gaps (needed for production operation)

### 6. Online worker reliability
- Retries with backoff; **dead-letter queue** for unscorable traces.
- Cursor/checkpoint recovery after crashes.
- Rate limiting against both Vertex AI and the Langfuse API; explicit concurrency model.
- **Eval-cost budgeting** — cap judge spend per day per agent; the eval pipeline has its own bill.

### 7. Score naming & metric-definition governance
- A `metrics.md` registry: exact score name, scale (0–1 vs 1–5), direction, level (output/trace/session), owner.
- Enforced naming conventions across agents — otherwise "faithfulness" on agent A isn't comparable to agent B and BigQuery dashboards break.

### 8. User-feedback ingestion path
The plan assumes thumbs-up/down arrive as Langfuse scores, but nothing designs the hookup: app → Langfuse SDK score call, with **trace_id propagation** from the agent response through to the UI.

### 9. Ground truth from the ticketing system (big win)
For ticket analysis, the *human-corrected final state* in Zendesk/ServiceNow (final category, priority, resolution) is free ground truth.
- Nightly backfill job joins agent predictions to eventual human outcomes → real accuracy with zero labeling effort.

### 10. Environment separation & access control
- Dev / staging / prod Langfuse projects so experiment noise never pollutes prod dashboards.
- Secret Manager wiring; least-privilege service accounts per pipeline.

### 11. Annotation program design
- Written labeling guidelines per score type.
- **Inter-annotator agreement** measurement; disagreement-resolution flow.
- Queues without guidelines produce inconsistent labels, which then miscalibrate the judge.

## C. Later-phase gaps (named in the plan but not designed)

### 12. Multi-turn simulated-user testing (τ-bench style)
Needs: a persona library, a simulated-user driver (LLM playing the customer), and conversation-termination logic.

### 13. Adversarial / red-team suite
Prompt-injection and jailbreak cases as a separate dataset with its own **pass-100%-required** gate.

### 14. Chaos / fault injection
A tool-mocking layer that injects timeouts/errors/corrupted responses during experiment mode, paired with recovery evaluators (borrowed from Strands Evals — see [03-strands-evals.md](03-strands-evals.md)).

### 15. Drift detection
Alert on input-distribution drift (ticket topics shifting) and score drift over time — not just per-run thresholds.

### 16. A/B / shadow evaluation online
Run challenger prompt/model on sampled traffic in shadow mode; compare scores before promotion.

### 17. Reporting surface & reproducibility
- HTML/markdown run report and PR-comment bot (named, not specced).
- **Run manifests** — config snapshot + dataset version + judge model/rubric version per run, written to GCS.

## Top 5 to fold into the build plan immediately

1. Trace adapters (A1)
2. PII redaction (A2)
3. Judge versioning + caching (A3)
4. pass^k + baseline registry (A4)
5. Harness self-tests (A5)

Everything else layers in after the first agent is onboarded.
