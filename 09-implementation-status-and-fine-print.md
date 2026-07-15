# Implementation Status & Production Fine Print

**Code-complete through the phases** at [Srinath279/agent-evals](https://github.com/Srinath279/agent-evals) (v0.3.0, 50 tests):

- **Phase 0–1**: schemas, `langfuse-generic` adapter + conformance kit, PII redaction, 13 evaluators (deterministic / trajectory / judge / safety / resilience), pass^k runner + fail-closed gate, deterministic trace IDs for idempotency, `evals seed-dataset` (redacted Langfuse upload).
- **Phase 2**: Anthropic/Vertex/OpenAI judges, FallbackJudge (truthful stamping), BudgetedJudge (daily cap kill switch), baseline registry + bootstrap-CI regression gating, trace-replay mode, `evals calibrate` (kappa/pearson vs human labels).
- **Phase 3**: online sampling (rate + 100% suspicious), cheap-tier `score_online`, `TraceScoreWorkflow`, annotation-queue push w/ graceful fallback, `post_user_feedback`, BigQuery export.
- **Phase 4/5 code parts**: onboarding template + adapter conformance test, failure clustering in reports, red-team suite (`gate_mode: all`, `forbidden_content` — demo catches a live priority injection), `ChaosInjector` + `recovery_after_error`.

**Remaining = infra wiring** (user-side): Langfuse projects/keys, real golden set seeding, Cloud Run Temporal workers + Schedules, Pub/Sub starter, BigQuery dataset + dashboards, CI gate wiring. Later code: batch judging APIs, simulated users, drift detection, canary/shadow workflows.

This note records the **small production-scale details discovered while building** that no earlier note captures. Each one bites at scale if forgotten.

## Fine print discovered during implementation

### 1. Idempotency depends on deterministic trace IDs
The score cache is keyed on `trace_id` — but if `task_fn` mints a random trace ID per invocation (as real agents do), a retried Temporal activity re-invokes the agent AND re-pays the judge; the cache never hits.
- **Rule**: in experiment mode, the runner must derive/pin a deterministic ID per `(run_id, case_id, repeat)` for caching, or the activity must record the produced trace_id in workflow state before scoring.
- Also: **agent invocation itself is not idempotent** — a retried `score_case` re-runs the agent (side effects: real ticket updates!). Experiment-mode task_fns must target sandboxed/stubbed tools, never production systems.

### 2. The sqlite score cache is single-node
Fine locally; **Cloud Run workers share no filesystem**. At scale the idempotency layer must move to a shared store (Cloud SQL / Firestore / Memorystore) — or use "does this score already exist in Langfuse?" as the dedupe check, making Langfuse the cache.

### 3. Pin the Langfuse SDK — v2→v3 API drift is real
`langfuse.score()` became `create_score()` in v3 (the client wraps both, but silently). Pin the SDK version in prod and treat SDK upgrades like judge upgrades: run the test suite + a golden-set smoke run first.

### 4. Temporal history budget includes activity *results*
`score_case` returns a compact dict deliberately. As the evaluator count grows (25+ evaluators × 100 cases × k), even compact results add up against the ~2 MB / 50k-event history limits — move aggregation into child workflows per chunk, and never return comments/reasoning through workflow state (fetch from Langfuse for reports).

### 5. Activities reload config + dataset per case
Acceptable at Phase 0 scale; at volume it's N² I/O against Langfuse. Fix in Phase 2: fetch single dataset items by ID, and cache the parsed config per worker process.

### 6. The gate is fail-closed — keep it that way
A thresholded metric that never produced a score **fails the gate** (implemented and tested). Recording as explicit policy: a broken evaluator must break the build, not silently drop its metric.

### 7. Redaction pattern ordering is load-bearing
Found live: the greedy phone regex consumed fragments of IP addresses until specific patterns were ordered first. Consequences:
- redaction needs its **own test suite** (it has one) and any Cloud DLP swap must pass the same tests behind the same function signatures;
- never add a pattern without a test proving it doesn't shadow the others.

### 8. Rubric-version continuity across the Prompt-Management migration
Rubrics are code constants today (`goal_success/v1`). Phase 2 moves them to Langfuse Prompt Management — the **rubric_version strings must carry over unchanged**, or every cache key invalidates and dashboards show a fake epoch break. Same string, new storage.

### 9. `daily_budget_usd` exists but is NOT enforced yet
The config field is plumbed through; enforcement lands in the Phase 3 judge activity. Until then nothing stops a runaway judge loop — don't point an expensive judge at a big dataset before Phase 3, or add a hard cap in CI (limit k × cases).

### 10. The task_fn contract is the onboarding contract
`task_fn(case_input) -> Trace` (dicts coerced). This plus a trace-adapter conformance test (pattern exists in `tests/test_adapter.py`) is what the note-06 onboarding kit should formalize.

## Plan-vs-code deltas (deliberate Phase 0 simplifications)
- Trace-replay mode: not yet (Phase 2) — experiment mode only.
- Bootstrap CIs / baseline registry: gate is raw thresholds for now (Phase 2).
- Vertex/OpenAI judges + fallback judge: raise clear "Phase 2" errors.
- Online `TraceScoreWorkflow`, annotation-queue push, BQ export: Phase 3.
- pass^k defined as "all thresholded metrics met on every repeat" — revisit when partial-credit metrics get their own pass definitions.
