# Decision — Temporal for Durable Execution of the Harness

**Decision: use the existing Temporal cluster** as the execution backbone for the harness's long-running pipelines. The generic argument against Temporal (a whole new platform to operate before the harness scores a single trace) doesn't apply — the infra already exists. What remains is the fit, which is strong: gap items in [05-harness-gaps.md](05-harness-gaps.md) §6 (retries, backoff, DLQ, checkpoint recovery, rate limits) are Temporal's native feature set.

## What Temporal replaces in the [note 04](04-eval-harness-plan-langfuse-gcp.md) plan

| Plan component | Before | With Temporal |
|---|---|---|
| Nightly scheduling | Cloud Scheduler → Cloud Run Job | **Temporal Schedules** (per agent config) |
| Offline run recovery | re-run job + score cache skips done work | **Workflow resumes exactly where it died** |
| Online worker retries/backoff | hand-rolled in `online_worker.py` | **Activity retry policies** (declarative) |
| Dead-letter queue | Pub/Sub DLQ | failed activities/workflows retained & queryable in Temporal — no separate DLQ to build |
| Cursor/checkpoint (Firestore/GCS) | custom | **workflow state** is the cursor |
| Rate limiting vs Vertex/Langfuse | custom limiter | worker task-queue concurrency + activity slots |

Compute stays on GCP: Temporal workers run as a Cloud Run service (or GKE) importing the same `agent-evals` package. Nothing about Langfuse-as-source-of-truth, BigQuery export, or the evaluator library changes.

## Workflow design

- **`EvalRunWorkflow(agent_config, dataset_version, run_id)`** — offline experiment mode. Fans out per dataset item × k repeats; activities: `invoke_agent` (task_fn), `run_evaluator` (one per evaluator, judge calls inside), `post_score`. Per-activity retry policies (aggressive for judge 429s, none for deterministic evaluators — they only fail on bugs). Emits the run report artifact at the end.
- **`TraceScoreWorkflow(trace_id, agent_config)`** — online mode. One short workflow per sampled trace, started by the Pub/Sub subscriber (or a polling workflow that owns the Langfuse cursor in its state). Cheap-tier evaluators only, then score write-back; below-threshold traces trigger an `enqueue_annotation` activity.
- **CI gate**: CI triggers `EvalRunWorkflow` and blocks on the result (workflow update/query), so PR checks and nightly runs share one code path.
- Later phases land naturally: canary/shadow comparison ([07 §A1](07-remaining-considerations.md)) is a durable-timer workflow that waits on a production score window then signals promote/rollback; simulated-user multi-turn sessions and chaos schedules are long-lived workflows — Temporal's sweet spot.

## Rules to not get burned

1. **Orchestration only.** Evaluators, adapters, redaction, judge client stay pure library code called from activities. No Temporal imports outside `pipelines/`. The harness must stay runnable locally via `cli.py` without a cluster.
2. **Pass IDs, not payloads.** Workflow history has ~2 MB event / 50k event limits — pass `trace_id`/`item_id` through workflows and fetch the actual trace inside activities. Never route full traces or judge transcripts through workflow state.
3. **Keep the score cache anyway.** Activities are at-least-once; the `(trace_id, evaluator, rubric_version)` cache ([05 §A3](05-harness-gaps.md)) remains the idempotency layer so retries never double-spend judge calls or double-write scores.
4. **Determinism discipline.** No Langfuse/Vertex calls, clocks, or randomness in workflow code — activities only. Randomized k-repeat seeds come from workflow-safe APIs or activity results.
5. **Large datasets → batch children.** Fan out in chunks (child workflow per ~50 cases) or `continue_as_new` to keep histories small as golden sets grow.
6. **Budget enforcement.** The per-agent `daily_budget_usd` cap becomes a counter the judge activity checks — a workflow-visible kill switch instead of best-effort accounting.

## What this does NOT change

The build order from [07](07-remaining-considerations.md) stands: schemas → adapter → redaction → 3 evaluators → pass^k runner. Temporal is *how the runner executes*, not a new phase — the first vertical slice can even run the workflow on a local `temporal server start-dev` and point workers at the real cluster later.
