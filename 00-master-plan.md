# Master Plan — Reusable Agent Eval Harness

The consolidated plan, merging all notes (01–08): fundamentals and metrics (01), framework selection (02–03), the core Langfuse + GCP design (04), the correctness gap analysis (05), production-scale requirements (06), system edges (07), and the Temporal durable-execution decision (08). This is the single build document; the numbered notes remain as background and rationale.

---

## 1. Goal & design principles

Build **one reusable eval harness** that measures and improves multiple agents (support agent, ticket-analysis agent, future ones) both offline (golden sets, CI) and online (sampled production traffic).

1. **One harness, many agents.** The harness is a Python package (`agent-evals`); each agent is a **YAML config + a trace adapter + optional custom evaluators** — never a fork.
2. **Langfuse is the system of record**: traces, golden datasets (Langfuse Datasets), human labels (Annotation Queues), and all metric results (Scores) live there, viewable per-trace in one UI.
3. **Same evaluators everywhere.** Offline and online pipelines call the identical evaluator library, so scores are comparable across modes and agents.
4. **Both levels of evaluation** (note 01): end-to-end *outcome* ("did it accomplish the task?") and *trajectory* ("did it behave well along the way?" — tool choice, efficiency, recovery).
5. **Durable, idempotent execution.** Long-running pipelines run as **Temporal workflows** (note 08) on the existing cluster; the score cache makes every judge call and score write idempotent.
6. **The harness is a production service**: it gets its own tests, SLOs, dashboards, budgets, and runbooks.

## 2. Stack (decided)

| Concern | Choice | Rationale |
|---|---|---|
| Tracing / datasets / annotation / scores | **Langfuse** | OSS/self-hostable, full trace capture, datasets + annotation queues + scores via API — portable across tiers (note 02) |
| Orchestration / durability | **Temporal** (existing cluster) | Resume-where-died runs, declarative retries, schedules, no hand-rolled DLQ/cursor code (note 08) |
| Compute | **Cloud Run** (Temporal workers + CI jobs), GCP | |
| Judge LLM | **Multi-provider** behind one `JudgeClient`: Vertex AI (Claude on Vertex / Gemini), Anthropic API (direct), OpenAI | Provider chosen per agent config; each has its own batch path (§7) |
| Warehouse / dashboards | **BigQuery** + Looker Studio | Long-horizon trends; Langfuse for per-trace drill-down |
| Secrets / artifacts | Secret Manager / GCS | |
| Methodology borrowed | τ-bench (simulated users, pass^k), Strands Evals (chaos + resilience evaluators, ActorSimulator concept) | notes 02–03 |

## 3. Architecture

```
                             ┌─────────────────────────────┐
   Agents (support,          │          Langfuse           │
   ticket-analysis) ────────▶│  traces │ datasets │ scores │◀── Annotation queues
        │        OTel/SDK    │         │ annotation queues │     (human review)
        │                    └────┬───────────▲────────────┘
        ▼                         │ fetch     │ write scores
  Pub/Sub (trace_id at    ┌───────┴───────────┴────────┐
  request completion)     │    agent-evals package     │
        │                 │ evaluators / adapters /    │
        ▼                 │ judge / redaction / cache  │
┌──────────────────┐      └───────────▲────────────────┘
│     Temporal     │                  │ called from activities
│  Schedules       │      ┌───────────┴────────────────┐
│  EvalRunWorkflow │─────▶│  Temporal workers          │──▶ Judge providers (§7):
│  TraceScoreWkfl  │      │  (Cloud Run service)       │     Vertex AI │ Anthropic
│  Canary/Shadow   │      └───────────┬────────────────┘     API │ OpenAI
└──────────────────┘                  │ export
        ▲                             ▼
   CI (GitHub Actions          BigQuery ──▶ Looker Studio /
   triggers run, blocks        (metrics     Grafana dashboards
   on result = PR gate)         warehouse)
```

Temporal replaces the original Cloud Scheduler / Pub/Sub-DLQ / Firestore-cursor design from note 04: Schedules do the nightly triggering, workflow state is the cursor, activity retry policies replace hand-rolled backoff, and failed workflows are the queryable dead-letter set. Rules that keep this safe (note 08): orchestration code only in `pipelines/`, pass IDs not payloads through workflow history, keep the score cache as the idempotency layer, activities-only for all I/O, chunked child workflows for large datasets.

## 4. The `agent-evals` package

```
agent-evals/
├── core/
│   ├── evaluator.py        # BaseEvaluator: score(trace|case) -> Score(name, value, comment)
│   ├── judge.py            # JudgeClient interface + provider adapters
│   │                       #   (vertex.py, anthropic.py, openai.py) + rubric versioning
│   │                       #   + bias controls + score cache keyed
│   │                       #   (trace_id, evaluator, rubric_version, judge_provider, judge_model)
│   ├── langfuse_client.py  # traces, datasets, scores, dataset runs
│   ├── schemas.py          # canonical Trace / ToolCall / Case / Score models
│   ├── adapters/           # per-framework: raw Langfuse observations -> canonical Trace
│   └── redaction.py        # PII redaction (Cloud DLP / regex+NER) before datasets & judges
├── evaluators/
│   ├── deterministic/      # exact_match, json_schema_valid, label_match, tool_called,
│   │                       #   state_equals, latency_threshold, cost_threshold
│   ├── llm_judge/          # goal_success, helpfulness, faithfulness, policy_compliance,
│   │                       #   tone, hallucinated_success, refusal, rubric (G-Eval style)
│   ├── trajectory/         # tool_selection, tool_param_accuracy, redundant_calls,
│   │                       #   loop_detection, recovery_after_error, steps_efficiency
│   ├── session/            # goal_success (multi-turn), escalation_correctness,
│   │                       #   instruction_adherence, handoff_summary_quality
│   └── rag/                # context_precision/recall, kb_freshness (phase 5)
├── pipelines/              # the ONLY module that imports Temporal
│   ├── workflows.py        # EvalRunWorkflow, TraceScoreWorkflow, CanaryCompareWorkflow
│   ├── activities.py       # invoke_agent, run_evaluator, post_score, enqueue_annotation
│   ├── backfill_labels.py  # join predictions to human-corrected ticket outcomes
│   └── bq_export.py        # scores -> BigQuery (nightly)
├── baselines/              # baseline-run registry + promotion records
├── tests/                  # evals-for-the-evals: fixture traces w/ known scores,
│                           #   unit test per evaluator, mock JudgeClient (no spend)
├── configs/                # one YAML per agent (the only thing that varies per agent)
├── metrics.md              # score registry: name, scale, direction, level, owner
└── cli.py                  # `evals run --config support_agent.yaml --mode offline`
                            #   runs locally WITHOUT Temporal — must always work
```

**Per-agent config** (the reusability contract — onboarding agent N must be only this file + an adapter):

```yaml
agent: support-agent
trace_adapter: langgraph
langfuse_dataset: support-agent-golden-v1
task_fn: agents.support.invoke
trace_filter: {tags: ["support-agent"], environment: "prod"}
online_sample_rate: 0.10
repeats: 3                          # k for pass^k
judge:
  provider: anthropic               # anthropic | vertex | openai
  model: claude-opus-4-8            # pinned; provider+model stamped on every score
  fallback:                         # optional second judge if primary is down/rate-limited
    provider: vertex
    model: claude-opus-4-8          # Claude on Vertex uses the same bare model IDs
  rubric_versions: pinned           # from Langfuse Prompt Management
  daily_budget_usd: 25              # enforced by the judge activity
evaluators: [goal_success, policy_compliance, faithfulness, tool_selection,
             label_match, latency_threshold, cost_threshold]
score_thresholds: {goal_success: 0.85, policy_compliance: 0.95}
```

## 5. Metrics catalog & governance

All captured as Langfuse Scores; full taxonomy in note 01. Categories: **outcome** (goal success, label correctness, pass^k, rubric score) · **quality** (faithfulness, policy compliance, helpfulness, tone) · **tool use** (selection, params, error rate, recovery, redundancy/loops) · **efficiency** (steps, tokens, cost, latency, cache-hit — free from trace metadata) · **safety** (harmfulness, inappropriate refusal, **hallucinated success**, instruction adherence, injection flags) · **human/ops** (annotations, user feedback, escalation rate, judge-vs-human agreement).

Governance (notes 05 §7, 06 §5–6):
- `metrics.md` is the enforced **score registry** — exact name, scale, direction, level, owner. "faithfulness" must mean the same thing on every agent or cross-agent dashboards are fiction.
- **Eval-config as code**: rubric, threshold, and sampling changes go through PRs with CODEOWNERS. A silent threshold edit is how quality gates rot.
- Semver the package; defined **re-score/epoch strategy** when evaluator logic changes (backfill or mark a metric epoch — never show fake regressions).

## 6. Data strategy

- **Golden datasets in Langfuse**, versioned (`-v1`, `-v2`): seed with 50–100 real anonymized tickets; grow from production ("add to dataset" on any interesting/failed trace); densify with synthetic paraphrase/edge cases (note 07 §6).
- **PII redaction before anything leaves the trace store** (note 05 §2 — compliance blocker): redaction pass (Cloud DLP or regex+NER) in both the promote-to-dataset path and the judge input path; retention policy on traces and datasets.
- **Annotation program** (note 05 §11): written labeling guidelines per score type, inter-annotator agreement measurement, disagreement-resolution flow. Queues without guidelines miscalibrate the judge.
- **Free ground truth**: nightly Temporal job joins agent predictions to human-corrected final ticket state (Zendesk/ServiceNow category/priority/resolution) — real accuracy, zero labeling effort (note 05 §9).
- **User feedback path designed, not assumed**: app posts thumbs as Langfuse scores with **trace_id propagated** from the agent response through to the UI.
- **Contamination control** (note 06 §11): a **held-out set** never used during prompt iteration; refresh and coverage-map the golden set quarterly against the production topic mix.

## 7. Judge engineering

**Multi-provider by design.** The `JudgeClient` is an interface with one adapter per provider — `vertex` (Claude on Vertex or Gemini), `anthropic` (direct API), `openai` — selected in the agent's `judge.provider` config. Evaluators never import a provider SDK; they call `JudgeClient.score(rubric, payload) -> StructuredVerdict`.

| Provider | Client / auth | Model IDs (judge tiers) | Batch path (offline/backfill) |
|---|---|---|---|
| `anthropic` | `anthropic.Anthropic()`, API key in Secret Manager | `claude-opus-4-8` (primary), `claude-sonnet-5`, `claude-haiku-4-5` (cheap tier) | **Anthropic Message Batches API** — 50% price, ≤100k requests/batch |
| `vertex` | `AnthropicVertex(project_id, region)` for Claude, or Gemini SDK; GCP ADC — no API key | Claude on Vertex uses the **same bare IDs** (`claude-opus-4-8`, `claude-sonnet-5`); Gemini IDs per Vertex catalog | **Vertex batch prediction** (Anthropic's Batches API is *not* available on Vertex) |
| `openai` | OpenAI SDK, API key in Secret Manager | per OpenAI catalog | OpenAI Batch API |

Rules that keep multi-provider sane:
- **Version everything, including the provider**: rubrics in Langfuse Prompt Management; `judge_provider + judge_model + rubric_version` stamped in every score's metadata and in the cache key. Any judge/rubric/provider change → rerun golden set; never mix epochs silently.
- **Scores are judge-specific, not interchangeable.** "faithfulness by claude-opus-4-8" and "faithfulness by gpt-x" are different measurements. Calibrate each provider+model against the human-labeled set independently (κ > 0.8 per judge), and never aggregate across judges in one dashboard series without a marked epoch.
- **Structured verdicts everywhere**: force JSON with reasoning-before-score on every provider (Anthropic: `output_config.format` structured outputs; equivalents on Gemini/OpenAI) so parsing never depends on the provider's prose style.
- **Self-preference bias**: never judge an agent with the same model family that powers it if an alternative is calibrated — this is a first-class reason to keep two providers live.
- **Bias controls**: randomized orderings (position bias), watch verbosity bias.
- **Fallback judge**: the optional `judge.fallback` config lets the worker degrade to a second provider on outage/rate-limit — scores written by the fallback carry its own provider/model stamp and count against the same daily budget.
- **Calibration is a cadence, not a setup step**: re-measure judge-vs-human agreement on a schedule with fresh labels (note 06 §12).
- **Cost discipline**: score cache (never re-pay), per-agent daily budget enforced in the judge activity, the provider-appropriate batch path above for offline/backfill volume, cheap-tier subset online.

## 8. Statistical rigor & release gating

- **pass^k runner**: k repeats per case, built into `EvalRunWorkflow`; track mean and worst-case (agents are non-deterministic — single-run pass rate overstates reliability).
- **Bootstrap confidence intervals** vs the baseline run; the CI gate fails on *significant regression*, not raw point differences on a 60-item set.
- **Baseline-run registry**: explicit record of the current baseline and who/what promotes a new one.
- **Deploy integration** (note 07 §A): the eval run is a required check in the deploy pipeline, not just CI; **canary rollout** compares the challenger's online scores to the incumbent before promotion (`CanaryCompareWorkflow` — durable timer over the score window); **auto-rollback triggers** tied to online score thresholds.
- Report artifact (markdown/HTML) + **run manifest** (config snapshot, dataset version, judge provider/model/rubric versions) to GCS, linked in the PR.

## 9. Pipelines (all on Temporal)

### Offline — `EvalRunWorkflow(agent_config, dataset_version, run_id)`
Two modes, one runner:
- **Experiment mode** (CI / pre-release / nightly Schedule): iterate dataset items × k repeats → `invoke_agent` activity → Langfuse dataset run → all evaluators → scores posted to the run; compare runs side-by-side in Langfuse.
- **Trace-replay mode**: re-score stored prod trajectories without re-invoking the agent — cheap regression for judge/rubric changes and backfills.
Fan-out in child workflows of ~50 cases; per-activity retry policies (aggressive for judge 429s; none for deterministic evaluators).

### Online — `TraceScoreWorkflow(trace_id, agent_config)`
1. Agents publish `trace_id` on completion (Pub/Sub → start workflow), or a polling workflow owns the Langfuse cursor in its state.
2. Sample `online_sample_rate` **+ 100% of traces with errors, negative feedback, or latency/cost outliers**.
3. Cheap-tier evaluators only (deterministic + 2–3 key judges: goal_success, faithfulness, harmfulness).
4. Scores written back to the trace; below-threshold traces auto-pushed to the **annotation queue** → golden-set candidates. This is the flywheel's intake.

### Later modes (phase 5, designed in notes 05 §C / 07)
- **Simulated-user multi-turn testing** (τ-bench style): persona library + LLM-as-customer driver + termination logic — long-lived workflows.
- **Red-team suite**: prompt-injection/jailbreak dataset with its own **pass-100%-required** gate.
- **Chaos/fault injection**: tool-mocking layer injecting timeouts/errors/corruption in experiment mode + recovery evaluators (RecoveryStrategy / PartialCompletion / FailureCommunication, borrowed from Strands).
- **Shadow A/B**: challenger prompt/model scored on sampled traffic before promotion.

## 10. Operating the harness

- **SLOs + observability** (note 06 §7): "95% of sampled traces scored within 15 min"; scoring-coverage %; judge error rate. Own dashboards, alerts, runbooks.
- **Graceful degradation** (§8): judge down → deterministic metrics keep flowing, judge work queues (Temporal makes this natural) and catches up; gaps are *visible* (coverage drops), never silent.
- **Regression response playbook** (§9): a score drop maps to a defined action — who's notified, rollback-vs-investigate thresholds, how a prompt rollback actually executes.
- **FinOps** (§3): per-agent eval-cost attribution; guardrail — eval spend < ~10% of the agent's own inference spend; shared judge-quota priority queues across agents.
- **Failure taxonomy + clustering** (§13): auto-cluster failing traces by root cause (wrong tool / bad retrieval / policy miss / hallucination) — turns "score dropped" into "here's what to fix."
- **Drift detection** (note 05 §15): input-distribution drift (ticket topics) and score drift over time, not just per-run thresholds.

## 11. Platform, security & the edges

- **Environments**: dev / staging / prod Langfuse projects; Secret Manager; least-privilege service accounts per pipeline.
- **Onboarding kit** (note 06 §4): template config + **adapter conformance test** ("pass these fixture checks = onboardable") + self-serve docs. Agent #3 must cost a YAML, an adapter, and maybe 1–2 evaluators.
- **Langfuse at volume** (note 06 §2): bulk/blob export to GCS as the offline data path (REST pagination won't survive volume); size ClickHouse for peak; trace TTL + cold archive.
- **Guardrails vs evals** (note 07 §4): the same policy/harm rubrics run post-hoc (harness) and inline (pre-send guardrail) — shared definitions so they never diverge.
- **Multilingual** (note 07 §5): per-locale golden subsets; judge accuracy validated per language.
- **Multi-agent/handoff eval** (note 07 §2): routing correctness, step-level failure attribution, handoff-summary quality — once agents compose.
- **Audit & compliance** (note 06 §14, 07 §9): audit logs for eval runs and trace access; data-residency/DPA review for judge calls; VPC-SC perimeter if tickets are sensitive; model cards + eval-coverage reports per agent (EU-AI-Act-style evidence).
- **Backup/DR** (note 07 §8): golden datasets, annotations, and score history are business-critical — backup/restore plan like any prod database.
- **The flywheel** (note 07 §7): scored + annotated traces feed fine-tuning / prompt optimization (DSPy-style) — the harness's end state is an improvement engine, not a scoreboard.

## 12. Consolidated roadmap

Gap items from notes 05–07 are folded into the phase where they must land, not deferred to a cleanup pass.

**Phase 0 — vertical slice (the real start; note 07's recommendation)**
Schemas → **one trace adapter** → **PII redaction** → 3 evaluators (1 deterministic, 1 judge, 1 trajectory) → pass^k runner against a 20-item golden set in Langfuse, for **one agent**, end to end. Runs via `cli.py` locally and as `EvalRunWorkflow` on `temporal server start-dev`. **Harness self-tests (fixture traces + mock judge) from day one.**

**Phase 1 (≈ wk 1–2) — foundation**
Full deterministic evaluator set; Langfuse client; seed support-agent golden set (50–100 redacted tickets); Temporal workers on Cloud Run pointed at the real cluster; dev/staging/prod Langfuse projects + Secret Manager before anything touches prod data.

**Phase 2 (≈ wk 3–4) — judges + gating**
LLM-judge evaluators with **rubric versioning + score caching + bias controls**; annotation queue with **written guidelines + inter-annotator agreement**; judge calibration (κ > 0.8); CI gate = golden-set run with **k repeats + bootstrap CIs vs the registered baseline**; nightly Temporal Schedule; run manifests + report artifacts to GCS.

**Phase 3 (≈ wk 5–6) — online**
`TraceScoreWorkflow` + sampling (incl. 100% of suspicious traces); daily judge budgets; score write-back; low-score → annotation-queue loop; **user-feedback ingestion** (trace_id propagation); **ticket-system ground-truth backfill**; Cloud Monitoring alerts; nightly BigQuery export.

**Phase 4 — scale-out & trust**
Onboard agent #2 (**the reusability test**: YAML + adapter only); onboarding kit + adapter conformance tests; harness SLOs + graceful degradation; batch judging (provider-appropriate path, §7) + Langfuse bulk export; **held-out set + golden-set coverage mapping**; failure clustering; Looker dashboards; drift detection; semver + epoch strategy; FinOps attribution.

**Phase 5 — advanced modes & edges**
Simulated-user testing; red-team suite (pass-100% gate); chaos injection + resilience evaluators; canary/shadow deploy gates + auto-rollback; RAG metrics + KB freshness; multilingual subsets; guardrail/rubric sharing; scheduled judge re-calibration; audit docs, DR, stakeholder scorecards; the data flywheel.

## 13. Standing cautions

- **Pin and record judge provider + model + rubric version on every score** — a silent judge upgrade shifts every metric, and scores from different judges are different measurements.
- **The adapter layer is the most underestimated piece** (note 05 §1) — without one canonical Trace schema, reusability dies in agent-specific parsing.
- **If evaluators are buggy, every downstream metric is fiction** — the harness's own test suite is not optional.
- **Don't tune prompts against the golden set** — that's what the held-out set is for.
- **The checklist is complete; the risk is execution.** Build Phase 0 before adding anything to this document.
