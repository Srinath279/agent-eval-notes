# Plan — Reusable Eval Harness on Langfuse + GCP

A reusable eval harness across multiple agents/use-cases, with **Langfuse as the system of record** for traces, golden datasets, annotations, and scores, running on **GCP**.

## 1. Design principles

- **One harness, many agents.** The harness is a Python package (`agent-evals`); each agent (support agent, ticket-analysis agent, future ones) is just a **config file + optional custom evaluators**, never a fork of the pipeline.
- **Langfuse is the source of truth**: traces in, golden datasets stored as Langfuse Datasets, human labels via Annotation Queues, all metric results written back as Langfuse **Scores** — everything viewable per-trace in one UI.
- **Same evaluators run everywhere.** Offline (golden set, CI) and online (sampled production traces) pipelines call the identical evaluator library, so scores are comparable.

## 2. Architecture

```
                        ┌─────────────────────────────┐
   Agents (support,     │          Langfuse           │
   ticket-analysis) ───▶│  traces │ datasets │ scores │◀─── Annotation queues
        │               │         │ annotation queues │      (human review)
        │ OTel/SDK      └────┬───────────▲────────────┘
        ▼                    │ fetch     │ write scores
┌───────────────┐     ┌──────┴───────────┴──────┐
│  ONLINE PIPE  │     │   agent-evals package    │
│ Pub/Sub +     │────▶│  evaluators / runners /  │◀────┌──────────────────┐
│ Cloud Run     │     │  dataset & trace clients │     │  OFFLINE PIPE     │
│ worker        │     └──────────┬───────────────┘     │ Cloud Run Job /   │
└───────────────┘                │ export              │ Scheduler + CI    │
                                 ▼                     └──────────────────┘
                          BigQuery (metrics warehouse)
                                 │
                          Looker Studio / Grafana dashboards
```

### GCP service mapping

| Concern | GCP service |
|---|---|
| Offline eval runs (nightly + on-demand) | **Cloud Run Jobs** triggered by **Cloud Scheduler** and by CI (Cloud Build / GitHub Actions) |
| Online eval workers | **Cloud Run service** consuming **Pub/Sub** (or a polling worker without eventing) |
| Judge LLM | **Vertex AI** (Claude on Vertex or Gemini) — one `JudgeClient` abstraction so it's swappable |
| Metrics warehouse | **BigQuery** (Langfuse scores exported for long-term trends/slicing) |
| Dashboards | **Looker Studio** on BigQuery + Langfuse's built-in dashboards |
| Secrets (Langfuse keys, judge keys) | **Secret Manager** |
| Eval artifacts (reports, run manifests) | **GCS bucket** |

## 3. Reusable package — repo structure

```
agent-evals/
├── core/
│   ├── evaluator.py        # BaseEvaluator: score(trace|case) -> Score(name, value, comment)
│   ├── judge.py            # JudgeClient (Vertex AI / configurable model)
│   │                       #   + rubric versioning (Langfuse Prompt Mgmt), bias controls,
│   │                       #   + score cache keyed (trace_id, evaluator, rubric_version)
│   ├── langfuse_client.py  # fetch traces, datasets, post scores, dataset runs
│   ├── schemas.py          # Trace, ToolCall, Case, Score pydantic models
│   ├── adapters/           # per-framework trace adapters: raw Langfuse observations
│   │                       #   -> canonical Trace schema (langgraph.py, custom.py, ...)
│   └── redaction.py        # PII redaction (Cloud DLP / regex+NER) before datasets & judges
├── evaluators/
│   ├── deterministic/      # exact_match, json_schema_valid, label_match,
│   │                       #   tool_called, latency_threshold, cost_threshold
│   ├── llm_judge/          # helpfulness, faithfulness/groundedness, policy_compliance,
│   │                       #   tone, hallucinated_success, rubric (generic G-Eval style)
│   ├── trajectory/         # tool_selection, tool_param_accuracy, redundant_calls,
│   │                       #   loop_detection, recovery_after_error, steps_efficiency
│   └── session/            # goal_success, escalation_correctness, instruction_adherence
├── pipelines/
│   ├── offline_run.py      # golden-set experiment runner (Langfuse dataset runs)
│   │                       #   + k repeats per case (pass^k), bootstrap CIs vs baseline
│   ├── online_worker.py    # sampled production-trace scorer
│   │                       #   + retries/backoff, DLQ, checkpoint recovery, rate limits
│   ├── backfill_labels.py  # join agent predictions to human-corrected ticket outcomes
│   │                       #   (Zendesk/ServiceNow final category/priority = free ground truth)
│   └── bq_export.py        # scores -> BigQuery
├── baselines/              # baseline-run registry: which run is baseline, promotion record
├── tests/                  # "evals for the evals": unit tests per evaluator,
│                           #   fixture traces with known scores, mock JudgeClient (no spend)
├── configs/
│   ├── support_agent.yaml
│   └── ticket_analysis_agent.yaml
├── metrics.md              # score registry: name, scale, direction, level, owner
│                           #   (enforced naming so scores are comparable across agents)
└── cli.py                  # `evals run --config support_agent.yaml --mode offline`
```

### Per-agent config is the only thing that varies

```yaml
# configs/support_agent.yaml
agent: support-agent
trace_adapter: langgraph              # maps this agent's trace shape -> canonical schema
langfuse_dataset: support-agent-golden-v1
task_fn: agents.support.invoke        # for online-style dataset runs in CI
trace_filter: {tags: ["support-agent"], environment: "prod"}
online_sample_rate: 0.10
repeats: 3                            # k runs per case for pass^k / variance
judge:
  model: claude-sonnet-5              # pinned; recorded on every score
  rubric_versions: pinned             # rubrics from Langfuse Prompt Management
  daily_budget_usd: 25                # eval-cost cap for this agent
evaluators:
  - name: goal_success           # LLM judge, session level
  - name: policy_compliance
    rubric_prompt: prompts/support_policy_rubric.txt
  - name: faithfulness           # grounded in KB/tool results
  - name: tool_selection
  - name: label_match            # deterministic — ticket category/priority
    fields: [category, priority]
  - name: latency_threshold      # p95 < 20s
    threshold_ms: 20000
  - name: cost_threshold
    max_usd: 0.15
score_thresholds:                # CI gate
  goal_success: 0.85
  policy_compliance: 0.95
```

## 4. Offline pipeline (Langfuse traces + golden data)

### 4a. Golden dataset lifecycle — lives in Langfuse Datasets

1. **Seed**: curate 50–100 real anonymized tickets → `langfuse.create_dataset_item(input=ticket, expected_output=..., metadata={type: "triage"|"reply"})`.
2. **Grow from production**: any interesting/failed prod trace → "add to dataset" in the UI, or `evals promote-trace <trace_id> --dataset ...`.
3. **Human ground truth via Annotation Queues**: push sampled traces and judge-disagreement cases into a Langfuse annotation queue; reviewers label correctness/policy-compliance in the Langfuse UI. Annotations become:
   - (a) expected outputs for new golden items, and
   - (b) the calibration set for validating the LLM judge (measure judge-vs-human agreement, e.g. Cohen's kappa — target >0.8 before trusting a judge metric).
4. **Version datasets** (`-v1`, `-v2`) so runs are comparable over time.

### 4b. Two offline modes, same runner

- **Experiment mode (CI / pre-release)**: iterate dataset items → invoke the agent live via `task_fn` → each execution linked as a Langfuse **dataset run** → all evaluators score it → scores posted to the run. Compare runs side-by-side in Langfuse (prompt v3 vs v4, model A vs B).
- **Trace-replay mode (regression on history)**: fetch existing prod traces via the Langfuse API (filtered by tag/time window), run evaluators over stored trajectories without re-invoking the agent. Cheap; catches judge/rubric changes and scores backfills.

### 4c. Scheduling & gating

- **Cloud Scheduler → Cloud Run Job** nightly per agent config.
- **CI gate**: any prompt/model/tool change runs the golden-set experiment; the run fails if any `score_thresholds` regress vs. the baseline run. Report artifact (markdown/HTML) written to GCS and linked in the PR.

## 5. Online pipeline (production traces)

1. **Ingestion**: Cloud Run worker polls Langfuse for new traces matching each agent's `trace_filter` (cursor in Firestore/GCS), or agents publish `trace_id` to Pub/Sub at request completion.
2. **Sampling**: `online_sample_rate` (e.g. 10% random) **+ 100% of traces with errors, negative user feedback, or latency/cost outliers** — always score the suspicious ones.
3. **Scoring**: run the *cheap-tier* evaluator subset online (deterministic checks + 2–3 key judges: goal_success, faithfulness, harmfulness) to control judge cost; full suite stays offline.
4. **Write back**: `langfuse.score(trace_id=..., name=..., value=..., comment=judge_reasoning)` — scores appear on the trace in the Langfuse UI.
5. **Feedback loop**: traces scoring below threshold are auto-pushed to the **annotation queue** and become golden-set candidates.
6. **Alerting**: worker emits metrics to Cloud Monitoring; alert if rolling goal_success drops or hallucinated_success fires at all.

## 6. Metrics catalog (all captured as Langfuse Scores)

- **Outcome**: goal/task success, correctness vs. expected label (triage), pass^k on golden set, partial-credit rubric score.
- **Quality (LLM judge)**: faithfulness/groundedness, policy compliance, helpfulness, tone, relevance, coherence.
- **Tool use**: tool selection accuracy, parameter accuracy, tool error rate, recovery-after-error, redundant/looping calls.
- **Efficiency (numeric, from trace metadata — free)**: steps per task, tokens in/out, cost USD, end-to-end + per-step latency, cache hit rate.
- **Safety**: harmfulness, inappropriate refusal, hallucinated success ("claimed the ticket was updated but no tool call succeeded"), instruction adherence, prompt-injection flags.
- **Human/ops**: annotation labels, user feedback scores (app thumbs posted as scores), escalation/human-takeover rate, judge-vs-human agreement.

Export all scores nightly to **BigQuery** (`bq_export.py`) for long-horizon trends, per-agent/per-version slicing, and Looker dashboards — Langfuse for per-trace drill-down, BigQuery for aggregates.

## 7. Rollout phases

1. **Week 1–2 — foundation**: `core/` (schemas, **trace adapter for the first agent**, **PII redaction**) + deterministic evaluators + Langfuse client; **harness self-tests** (fixture traces, mock judge) from day one; seed support-agent golden dataset in Langfuse; offline experiment runner working locally.
2. **Week 3–4 — judges + CI**: LLM-judge evaluators on Vertex AI with **rubric versioning and score caching**, calibrated against a small human-labeled set from annotation queues (with **written labeling guidelines + inter-annotator agreement**); CI gate on the golden set using **k repeats + bootstrap CIs against a registered baseline run**; nightly Cloud Run Job. Separate **dev/staging/prod Langfuse projects** before anything touches prod data.
3. **Week 5–6 — online**: sampling worker (retries, DLQ, checkpoints, rate limits, **daily judge-cost budget**) + score write-back + low-score → annotation-queue loop; **user-feedback ingestion** (app posts thumbs as Langfuse scores with propagated trace_id); **ticket-system ground-truth backfill** (human-corrected category/priority joined to agent predictions); Cloud Monitoring alerts.
4. **Week 7+ — scale out**: onboard ticket-analysis agent (should be *only* a new YAML + a trace adapter + maybe 1–2 custom evaluators — that's the test that the harness is truly reusable); BigQuery export + dashboards + **drift detection**; then simulated-user multi-turn tests (τ-bench-style), **red-team/prompt-injection suite** (pass-100% gate), **chaos/fault injection**, and **shadow A/B evaluation** as new pipeline modes.

See [05-harness-gaps.md](05-harness-gaps.md) for the full gap analysis behind these additions.

## Cautions

- **Pin and record judge model + rubric versions on every score** — a judge upgrade silently shifts all metrics; rerun the golden set on any judge change.
- **Check Langfuse tier features** — annotation queues / managed evaluators availability varies by tier; self-hosted OSS covers datasets/scores/queues fine, and this design only needs API-level features, so it stays portable.
