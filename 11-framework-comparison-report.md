# Agent Eval Frameworks — Detailed Comparison Report

Deep-dive companion to [02-eval-frameworks-comparison.md](02-eval-frameworks-comparison.md) (which is the short list). For every framework/library: what it actually is, **what we should take from it** (idea, feature, or pattern worth adopting even if we don't use the tool), pros, cons, and when it's the right pick. Ends with a decision matrix and the recommendation for our Langfuse + GCP + Temporal stack.

## Comparison criteria

Every tool below is judged against what a **multi-tool / multi-step / sub-agent** system needs (see [10](10-multi-tool-subagent-json-eval.md)):

1. **Trajectory awareness** — can it score tool selection, ordering, arguments, and multi-step behavior, or only final outputs?
2. **Structured-output grading** — schema validation + field-level scoring for nested JSON.
3. **LLM-judge tooling** — rubric support, calibration aids, judge cost control.
4. **Datasets + regression/CI** — golden sets, experiment diffing, gates.
5. **Tracing / online eval** — production trace capture, sampled live scoring.
6. **Deployment model** — self-host vs managed; data residency; lock-in.
7. **Maturity & momentum** — community, release cadence, risk of abandonment.

---

## Category A — Tracing + evaluation platforms

### A1. Langfuse *(our current choice)*

Open-source (self-hostable; MIT core, some enterprise-licensed features) or cloud. Tracing with a trace → observation (span/generation/event) model, prompt management with versioning, datasets, scores API, LLM-as-judge evaluators on live traces, user-feedback capture, sessions, OTel ingestion.

- **Take from it**: the **trace/observation data model** as the canonical schema for our adapters; the **scores API** (any score, any level — trace, span, session) as the metric sink; **prompt management** for judge rubric versioning ([05](05-harness-gaps.md) §A3).
- **Pros**: self-host = full data ownership (PII stays in our VPC); framework-agnostic SDKs; strong API for programmatic pipelines (exactly what a custom harness needs); large OSS community; cheap at volume; datasets + basic experiment comparison built in.
- **Cons**: **no built-in trajectory metrics** — tool-selection/ordering evaluators are ours to write; the offline experiment runner is thin vs. Braintrust/LangSmith (we built our own — `agent-evals/runner.py`); v3 self-host is real ops (ClickHouse + Redis + S3, not a single container); UI-configured judge evaluators are less flexible than code; annotation queues are younger than LangSmith's.
- **Pick when**: data residency matters, you want OSS + API-first, and you're willing to own the eval harness code. *(Us.)*

### A2. LangSmith

Managed platform from the LangChain team (self-host only at enterprise tier). Best-in-class trace UI, datasets + experiments with side-by-side diffs, online evaluators, **annotation queues** for human review, pairwise experiments, playground. **Framework-agnostic despite the name**: traces any stack via the `@traceable` decorator, SDK wrappers (`wrap_anthropic`, `wrap_openai`), or its OpenTelemetry endpoint — documented integrations include Google ADK, Gemini, Anthropic SDK, OpenAI Agents SDK, Vercel AI SDK, and Pydantic AI. LangChain/LangGraph are just the zero-config path; everything else is a one-line wrapper or standard OTel.

- **Take from it**: the **annotation queue workflow** (guidelines, assigned reviewers, agreement tracking) as the model for our human-labeling program ([05](05-harness-gaps.md) §B11); pairwise experiment UX for prompt A/B.
- **Pros**: most polished trace debugging UI for agents (great span-tree rendering of sub-agents); mature dataset/experiment loop; `openevals`/`agentevals` integrate natively; works with any agent framework/LLM SDK (ADK, Anthropic, Gemini, …); strong docs.
- **Cons**: managed-only unless enterprise (data leaves our VPC — PII problem); cost grows with trace volume; non-LangChain frameworks need explicit instrumentation (wrapper/decorator/OTel) vs. LangGraph's automatic tracing; scores/metrics less API-flexible than Langfuse for a fully custom harness.
- **Pick when**: managed is acceptable, team already uses LangChain/LangGraph, and human annotation at scale is a priority.

### A3. Braintrust

Managed eval-first platform. `Eval()` API (task + scorers over a dataset), the **autoevals** scorer library, side-by-side diffing of runs, CI integration, playground, prompt functions in TS/Python.

- **Take from it**: the **offline-eval developer experience** — one `Eval(dataset, task, scorers)` call, results diffed against the previous run automatically. Our `runner.py` + baseline registry should feel like this. Also **autoevals** as a reference for well-factored scorer implementations (JSON diff, numeric proximity, factuality).
- **Pros**: best-in-class iteration loop for offline evals; scorers-as-code; clean CI story; good caching of judge calls; fast diffing UX that engineers actually use.
- **Cons**: proprietary, eval-first — production **tracing/online story is weaker** than Langfuse/LangSmith; self-host only at enterprise tier; another vendor for data governance review; costs at scale.
- **Pick when**: the pain is offline experimentation velocity (prompt hill-climbing), not production observability.

### A4. Arize Phoenix

Open-source, **OpenTelemetry/OpenInference-native** tracing + evals. Built-in eval templates (hallucination, QA correctness, relevance, toxicity), datasets + experiments, embedding-drift analysis. Sister product Arize AX is the managed enterprise tier.

- **Take from it**: **OTel/OpenInference semantic conventions** as the wire format for our trace adapters — instrument once, portable to any OTel backend; drift-monitoring approach for input-distribution drift ([05](05-harness-gaps.md) §C15).
- **Pros**: truly OTel-standard (least instrumentation lock-in of any platform here); free self-host, lightweight (single container to start); solid RAG evals; good production drift tooling.
- **Cons**: eval library is RAG-centric — agent-trajectory metrics thin; annotation/human-review workflows weaker; feature tiering vs. Arize AX can be confusing; experiment diffing less polished than Braintrust/LangSmith.
- **Pick when**: OTel standardization is the priority, or you need drift monitoring alongside evals.

### A5. W&B Weave

Managed traces + evals from Weights & Biases. `@weave.op` decorators for tracing, `Evaluation` objects with scorers, leaderboards per model/prompt version, guardrails; ties into W&B experiment tracking.

- **Take from it**: **leaderboards per version** — a standing ranked view of prompt/model variants over the same dataset; nice pattern for our BigQuery dashboards.
- **Pros**: minimal-boilerplate instrumentation; good comparisons/leaderboards; natural if the org already runs W&B for ML training.
- **Cons**: managed-first; agent/trajectory-specific metrics limited; yet another platform if you're not already on W&B; weaker prompt management.
- **Pick when**: the team already lives in W&B.

### A6. Opik (Comet)

Open-source (Apache-2) + cloud. Tracing, heuristic + LLM-judge metrics (hallucination, moderation, G-Eval-style), datasets/experiments, playground, pytest integration, and an **agent optimizer** (automatic prompt improvement from eval results).

- **Take from it**: the **eval → prompt-optimizer loop** — closing the flywheel from scores back to prompt candidates automatically ([07](07-remaining-considerations.md) §E7).
- **Pros**: genuinely generous OSS feature set (closest OSS rival to Langfuse); fast release cadence; self-host is simpler than Langfuse v3; pytest-native CI.
- **Cons**: younger, smaller community than Langfuse; some features land cloud-first; trajectory metrics still basic; migration cost for us with no clear payoff over Langfuse.
- **Pick when**: starting fresh today and wanting one OSS tool covering tracing + evals + optimization.

### A7. Galileo

Managed enterprise platform. **Prebuilt agentic metrics out of the box** — tool selection quality, tool-argument quality, action advancement, action completion — plus **Luna**: small fine-tuned evaluator models that score at a fraction of LLM-judge cost. Runtime guardrails ("Protect") share definitions with offline metrics.

- **Take from it**: two ideas — (1) **small-model evaluators** for high-volume online scoring (our judge-cost budgets, [05](05-harness-gaps.md) §B6, could use a cheap Haiku-class screener before an expensive judge); (2) **shared metric definitions between guardrails and evals** ([07](07-remaining-considerations.md) §C4).
- **Pros**: the most complete out-of-box agent metric set of any platform; low-latency low-cost evaluators; enterprise support/compliance posture.
- **Cons**: proprietary and opaque (you can't inspect or version the metric internals — clashes with our metrics-governance principle); pricing enterprise-opaque; harness flexibility limited; strong lock-in.
- **Pick when**: enterprise wants agent metrics *now* with vendor support, and metric explainability is negotiable.

---

## Category B — Eval harnesses & grading libraries

### B1. DeepEval

OSS, pytest-style eval framework. Agent-relevant metrics: **task completion**, **tool correctness**, G-Eval (rubric-from-criteria judge), answer relevancy, faithfulness, hallucination; conversational metrics; dataset synthesizer; red-teaming via sibling DeepTeam; Confident AI cloud attached.

- **Take from it**: **G-Eval-style rubric generation** (criteria → judge prompt with chain-of-thought and score normalization) as a pattern for our `rubrics.py`; the pytest-marker ergonomics for CI.
- **Pros**: fastest path to code-first agent metrics; pytest = free CI integration; very active; broad metric catalog; dataset synthesis.
- **Cons**: steady product pressure toward Confident AI cloud; metric internals opinionated and occasionally changing across versions (pin it); its judges need the same calibration discipline as ours — the metric names give false comfort; trajectory support is task-level, not constraint-level.
- **Pick when**: you want ready-made metrics without building a harness — or as a scorer library *inside* a custom harness.

### B2. promptfoo

OSS, config-driven (YAML) CLI. Assertions (exact/contains/regex/`json_schema`/JS-or-Python custom/model-graded), test matrices across prompts × models, CI-friendly, web viewer, and the **strongest OSS red-team scanner** (injection, jailbreak, PII leakage, OWASP LLM Top-10 presets).

- **Take from it**: the **red-team plugin catalog** as the seed for our `redteam_support.jsonl` categories; declarative `json_schema` assertions for quick output checks.
- **Pros**: zero-code regression tests — non-engineers can add cases; ideal for classification-style tasks (triage labels); provider-agnostic; red-teaming is genuinely good; trivially CI-able.
- **Cons**: **fundamentally single-turn/output-oriented** — no real trajectory or sub-agent awareness; YAML sprawls past ~100 cases; complex graders end up as escape-hatch JS/Python anyway.
- **Pick when**: regression-testing prompts/classifiers, and for red-team scanning even if everything else is custom.

### B3. Ragas

OSS RAG-evaluation library: context precision/recall, faithfulness, answer relevancy; newer agent metrics (tool-call accuracy, agent-goal accuracy); knowledge-graph-based **test-set generation**.

- **Take from it**: retrieval metrics for the KB layer ([07](07-remaining-considerations.md) §B3); the KG-based synthetic test-set generator for densifying golden sets.
- **Pros**: de-facto standard vocabulary for RAG metrics; test-set generation is unique; research-grounded.
- **Cons**: RAG-centric — agent metrics are new and thin; judge-sensitivity: scores swing with the underlying judge model; API churn between versions.
- **Pick when**: the agent has a retrieval layer that needs its own metrics. Use for that layer only.

### B4. AgentEvals / openevals (LangChain)

Two small OSS libraries. `agentevals`: **trajectory match evaluators** — exact / unordered / **superset / subset** modes over OpenAI-format message lists — plus an LLM trajectory judge and LangGraph graph-trajectory variants. `openevals`: general prebuilt judges (correctness, conciseness, hallucination) and extraction helpers.

- **Take from it**: the **trajectory-match semantics** — superset/subset modes are exactly the must-call / may-call constraint matching from [10](10-multi-tool-subagent-json-eval.md) §2. Either import it directly into `evaluators/trajectory/` or copy the matching logic (it's small).
- **Pros**: laser-focused; tiny dependency; framework-agnostic input format; the only OSS library where trajectory comparison is the headline feature.
- **Cons**: narrow — no datasets, no runner, no platform (by design); young; tool-*argument* checking is shallow (presence/equality, not AST-style); LangChain-maintained, so LangSmith is the assumed home.
- **Pick when**: always — as a component, not a platform.

### B5. Vertex AI Gen AI Evaluation Service

Managed GCP eval service. **Prebuilt agent/trajectory metrics**: `trajectory_exact_match`, `trajectory_in_order_match`, `trajectory_any_order_match`, `trajectory_precision`, `trajectory_recall`, `single_tool_use`; pointwise + **pairwise** model-based metrics with custom rubrics; autorater with explanations; results land in Vertex/BigQuery.

- **Take from it**: since we're on GCP + Vertex judges already ([04](04-eval-harness-plan-langfuse-gcp.md)): use it as the **managed judge/trajectory backend inside our harness** rather than a competing harness — our runner orchestrates, Vertex scores. Pairwise autorating is the piece we haven't built (A/B prompt comparisons).
- **Pros**: zero eval infra; trajectory metrics prebuilt; pairwise comparisons; native BigQuery output (matches our dashboard plan); IAM/security already solved in our project.
- **Cons**: GCP lock-in; per-eval cost adds up at k-repeat volumes; custom metric flexibility bounded by their rubric format; batch-oriented (latency not designed for inline use); API surface has churned — pin SDK versions ([09](09-implementation-status-and-fine-print.md)).
- **Pick when**: already on GCP and wanting managed trajectory/judge scoring under a custom orchestrator. *(Strong candidate for us.)*

### B6. Inspect AI (UK AI Safety Institute)

OSS Python framework for rigorous evals: tasks / solvers / scorers, **sandboxed tool execution** (Docker), multi-turn agent support, first-class eval logs + viewer, large companion benchmark repo (`inspect_evals`).

- **Take from it**: **sandboxed agent execution** — run eval tasks against real (containerized) tools instead of mocks, so state-change assertions ("was the ticket actually updated?") are real; the reproducible **eval-log format** (every sample's full transcript, seed, config in one artifact) as the model for our run manifests ([05](05-harness-gaps.md) §C17).
- **Pros**: most rigorous execution/reproducibility model here; safety/red-team pedigree; genuinely agent-native (multi-turn, tools, sandboxes); government-backed, actively maintained.
- **Cons**: research-flavored ergonomics; no production tracing/online-eval story at all; its own task format = integration work with a Langfuse-centric pipeline; steeper learning curve.
- **Pick when**: high-stakes capability/safety suites where reproducibility beats convenience — or steal its sandbox pattern for our chaos/state-check tests.

### B7. TruLens

OSS "feedback functions" library (Snowflake-backed): groundedness, context relevance, answer relevance (the "RAG triad"), OTel-based tracing, simple dashboard.

- **Take from it**: the RAG-triad framing as a compact retrieval-quality summary; not much else beyond what we have.
- **Pros**: simple abstraction; OTel-aligned; fine for quick RAG scoring.
- **Cons**: momentum has slowed vs. peers; minimal agent/trajectory support; basic UI; overlaps entirely with Ragas + Phoenix.
- **Pick when**: rarely — Ragas or Phoenix covers the same ground with more momentum.

### B8. pydantic-evals

OSS eval library from the Pydantic team (pairs with Pydantic AI + Logfire). Typed `Case`/`Dataset`/`Evaluator` model, built-in `LLMJudge`, OTel span capture, YAML/JSON dataset serialization.

- **Take from it**: **typed, schema-first case definitions** — expected outputs as Pydantic models means nested-JSON field grading falls out of model comparison naturally; the cleanest pattern for [10](10-multi-tool-subagent-json-eval.md) §1's field-level scoring.
- **Pros**: excellent typed API; natural fit for nested-JSON grading; Logfire integration; Pydantic-team code quality.
- **Cons**: young, small ecosystem; happiest inside the Pydantic AI world; no platform/UI of its own; trajectory metrics absent.
- **Pick when**: building structured-output evals in a Pydantic-heavy codebase (ours is — worth borrowing the pattern even without adopting the lib).

### B9. MLflow (3.x) GenAI evaluation

OSS + Databricks. Traces, scorers incl. **guidelines-based judges** (plain-English rules → judge), evaluation harness, tight registry/lifecycle integration.

- **Take from it**: "guidelines" judges — expressing policy rules as short natural-language assertions, one judge call per rule (aligns with our many-binary-criteria rubric rule).
- **Pros**: one platform across classic ML + GenAI; solid if the org is Databricks-based; OSS core.
- **Cons**: heavyweight; agent/trajectory metrics thin; best experience assumes Databricks; slower-moving for agent workloads than dedicated tools.
- **Pick when**: the org already standardizes on MLflow/Databricks.

### B10. OpenAI Evals

The original OSS eval registry (YAML templates + completion functions). Now effectively **maintenance-mode** for community use; OpenAI's dashboard evals are the active successor, OpenAI-models-centric.

- **Take from it**: historical registry structure; little else today.
- **Pros**: simple; some reusable templates.
- **Cons**: stagnant; OpenAI-centric; no trajectory/agent awareness; better options everywhere above.
- **Pick when**: effectively never for new agent work.

### B11. Strands Evals (AWS) — see [03-strands-evals.md](03-strands-evals.md)

AWS's agent-eval SDK: evaluator library, **ActorSimulator** (LLM plays the user for multi-turn tests), **chaos testing** (fault injection into tools).

- **Take from it**: the **simulated-user driver** pattern (τ-bench-style, [05](05-harness-gaps.md) §C12) and **chaos injection** (already mirrored in our `chaos.py`).
- **Pros**: only SDK with simulator + chaos as first-class features; clean evaluator interfaces.
- **Cons**: young; AWS ecosystem gravity (we're GCP); small community. Full analysis in note 03.

---

## Category C — Benchmarks (methodology sources, not tools)

| Benchmark | What to take |
|---|---|
| **τ-bench / τ²-bench** | Simulated user + policy doc + tools; **pass^k** as the headline; policy-compliance grading. The blueprint for our multi-turn phase. |
| **BFCL** (Berkeley Function Calling Leaderboard) | **AST-based argument checking** — grade tool-call args structurally (right function, right params, values within tolerance) instead of string equality; "relevance detection" cases = our must-NOT-call near-misses ([10](10-multi-tool-subagent-json-eval.md) §4). |
| **GAIA** | Tasks with unambiguous, verifiable final answers — write golden cases whose outcomes are checkable by code, not judges. |
| **AgentBench / WebArena** | Environment-state verification: assert on the *world state* after the run (ticket updated, record created), not just the text output. |
| **SWE-bench** | Execution-based grading (tests pass/fail) — the extreme of code-checkable outcomes; also a caution on contamination hygiene ([06](06-production-scale.md)). |

---

## Decision matrix

Legend: ● strong / ◐ partial / ○ weak-or-absent. "Trajectory" = built-in tool-selection/ordering metrics.

| Tool | Trajectory | Nested-JSON grading | LLM judge | Datasets/CI | Tracing/online | Self-host | Red-team/sim | Maturity |
|---|---|---|---|---|---|---|---|---|
| Langfuse | ○ | ○ (via API) | ◐ | ◐ | ● | ● | ○ | ● |
| LangSmith | ◐ (agentevals) | ◐ | ● | ● | ● | ○ (ent.) | ◐ | ● |
| Braintrust | ◐ | ◐ (autoevals) | ● | ● | ◐ | ○ (ent.) | ○ | ● |
| Phoenix | ◐ | ○ | ◐ | ◐ | ● | ● | ○ | ● |
| Weave | ○ | ◐ | ◐ | ● | ◐ | ○ | ◐ | ◐ |
| Opik | ◐ | ◐ | ● | ● | ● | ● | ◐ | ◐ |
| Galileo | ● | ◐ | ● (Luna) | ◐ | ● | ○ | ◐ | ◐ |
| DeepEval | ◐ | ◐ | ● (G-Eval) | ● | ○ | ● | ● (DeepTeam) | ● |
| promptfoo | ○ | ◐ (json_schema) | ◐ | ● | ○ | ● | ● | ● |
| Ragas | ◐ | ○ | ◐ | ◐ | ○ | ● | ○ | ● |
| AgentEvals | ● | ○ | ◐ | ○ | ○ | ● | ○ | ◐ |
| Vertex AI Eval | ● | ◐ | ● (pairwise) | ◐ | ○ | ○ (GCP) | ○ | ● |
| Inspect AI | ● | ◐ | ◐ | ● | ○ | ● | ● | ● |
| TruLens | ○ | ○ | ◐ | ◐ | ◐ | ● | ○ | ◐ |
| pydantic-evals | ○ | ● | ◐ | ◐ | ◐ (Logfire) | ● | ○ | ◐ |
| MLflow | ○ | ◐ | ◐ | ● | ◐ | ● | ○ | ● |
| Strands Evals | ◐ | ○ | ◐ | ◐ | ○ | ● | ● (chaos+sim) | ◐ |

**The gap the matrix exposes:** no single tool is strong across trajectory + nested-JSON + judge + tracing + self-host. That's *why* the plan in [04](04-eval-harness-plan-langfuse-gcp.md) is a thin custom harness over a platform — the harness is the glue no vendor sells.

## One line to steal from each

| Source | The thing to adopt |
|---|---|
| Langfuse | Trace/score data model + prompt-managed rubrics (adopted) |
| LangSmith | Annotation-queue workflow design |
| Braintrust | `Eval()`-grade offline DX + auto-diff vs. baseline |
| Phoenix | OTel/OpenInference conventions in the adapter layer |
| Weave | Per-version leaderboards on dashboards |
| Opik | Eval-scores → prompt-optimizer flywheel |
| Galileo | Cheap small-model screeners before expensive judges; shared guardrail/eval definitions |
| DeepEval | G-Eval rubric-construction pattern |
| promptfoo | Red-team plugin taxonomy for the adversarial set |
| Ragas | Retrieval metrics + KG test-set generation for the KB layer |
| AgentEvals | Superset/subset trajectory matching (import it) |
| Vertex AI Eval | Managed trajectory metrics + pairwise autorater under our runner |
| Inspect AI | Sandboxed real-tool execution + reproducible eval-log artifacts |
| TruLens | RAG-triad framing (only) |
| pydantic-evals | Typed expected-output models for nested-JSON field grading |
| MLflow | Guidelines-based (one-rule-per-judge-call) policy checks |
| Strands | ActorSimulator + chaos injection patterns |
| τ-bench / BFCL | pass^k headline; AST argument grading; must-not-call cases |

## Recommendation for our stack

No change of platform — the combination stays **Langfuse (store/traces/prompts) + custom `agent-evals` harness + Temporal (pipelines) + Vertex (judges)**. Concretely, in order:

1. **Import `agentevals`** trajectory matchers into `evaluators/trajectory/` instead of maintaining our own matching logic.
2. **Pilot Vertex AI Gen AI Evaluation** as the scoring backend for trajectory metrics + pairwise prompt comparisons — managed replacement for code we'd otherwise write, already inside our cloud boundary.
3. **Adopt promptfoo for the red-team suite only** — its attack catalog is better than anything we'd hand-write.
4. **Copy patterns, not dependencies**, from Braintrust (runner DX), pydantic-evals (typed nested-JSON grading), Inspect (run manifests, sandboxed state checks), Galileo (cheap screener before judge), Strands (simulator/chaos — partly done).
5. **Re-evaluate Opik in ~6 months** as the main OSS platform risk/alternative to Langfuse; no migration unless Langfuse stalls.
