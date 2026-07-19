# Eval Frameworks for a Support / Ticket-Analysis Agent

Two kinds of tooling are needed: an **observability + tracing platform** (capture production runs) and an **eval harness** (run the golden ticket set offline and grade outputs).

## 1. Tracing + evaluation platforms (the core choice)

| Framework | Type | Why it fits a support agent |
|---|---|---|
| **Langfuse** | Open-source (self-hostable) or cloud | Full traces of every LLM/tool call, user feedback capture (thumbs on replies), LLM-as-judge evaluators on live traces, prompt versioning. Most popular OSS choice. |
| **LangSmith** | Managed (LangChain team) | Best-in-class trace UI, datasets + regression testing, online evaluators, annotation queues for human review. Framework-agnostic — traces any stack (Google ADK, Anthropic SDK, Gemini, OpenAI Agents SDK, …) via wrappers/decorators or OpenTelemetry; LangChain/LangGraph is just the zero-config path. |
| **Braintrust** | Managed | Strong offline eval loops (Evals API), side-by-side diffing of prompt/model versions, good CI integration. |
| **Arize Phoenix** | Open-source | OpenTelemetry-based tracing, built-in evals (relevance, hallucination, toxicity), good for production drift monitoring. |
| **W&B Weave** | Managed | Traces + scorers + leaderboards per model version; good if already on Weights & Biases. |
| **Opik (Comet)** | Open-source or cloud | Tracing + eval metrics + playground; fast-growing OSS alternative to Langfuse. |
| **Galileo** | Managed | Agent-specific metrics out of the box (tool selection quality, action advancement/completion); enterprise-leaning. |

## 2. Eval harnesses / grading libraries

- **DeepEval** (open-source, pytest-style) — easiest way to write agent evals as code. Built-in metrics that map to support use cases: answer relevancy, faithfulness/groundedness, hallucination, **task completion**, **tool correctness**, plus custom G-Eval rubrics ("was the reply empathetic and did it follow the refund policy?").
- **promptfoo** (open-source, config-driven) — YAML test cases; great for regression-testing classification tasks like ticket triage (category, priority, sentiment) with exact expected labels. Also does red-teaming / prompt-injection testing.
- **Ragas** — if the agent does RAG over a knowledge base / past tickets: grades retrieval quality (context precision/recall, faithfulness).
- **AgentEvals / openevals (LangChain)** — small OSS library purpose-built for **trajectory evaluation**: exact / unordered / superset / subset trajectory match plus an LLM trajectory judge. Directly implements the constraint-style matching in [10](10-multi-tool-subagent-json-eval.md); works on plain OpenAI-format message lists, no LangChain required.
- **Vertex AI Gen AI Evaluation Service** — managed eval on GCP with **prebuilt agent/trajectory metrics** (`trajectory_exact_match`, `trajectory_in_order_match`, `trajectory_any_order_match`, `trajectory_precision/recall`, `single_tool_use`) plus rubric-based judges. Worth first-class consideration given the GCP + Vertex judge stack in [04](04-eval-harness-plan-langfuse-gcp.md).
- **Inspect AI (UK AI Safety Institute)** — OSS Python framework for rigorous agent evals: solvers, scorers, **sandboxed tool execution**, multi-step agent tasks. Strong for safety/red-team suites.
- **TruLens** — OSS "feedback functions" over traces (groundedness, relevance); OpenTelemetry-based.
- **pydantic-evals** — lightweight, typed eval library from the Pydantic team; pairs naturally with a Pydantic-schema harness for **nested-JSON output checks**.
- **OpenAI Evals / MLflow LLM Evaluate** — usable (MLflow 3 added agent/trace evals), but the above are more actively used for agent workloads.

## 2b. Benchmarks to borrow methodology from

- **BFCL (Berkeley Function Calling Leaderboard)** — *the* reference for **tool/function-calling accuracy**: single, parallel, and multi-turn tool calls, plus relevance detection (knowing when *not* to call a tool). Its AST-based argument checking is the model for grading tool-call arguments against 10 tools.
- **GAIA / AgentBench / WebArena** — general multi-step agent benchmarks; useful for methodology (verifiable end states), not directly for a support agent.
- **τ-bench / τ²-bench** — see §3 below.

## 3. Support-agent-specific benchmark: τ-bench (tau-bench)

From Sierra — *the* reference benchmark for customer-service agents. Simulates a user + a policy document + tools (e.g., airline/retail support with refund rules) and measures whether the agent follows policy across multi-turn conversations, including **pass^k consistency**.

Even without running it directly, copy its methodology:
- Simulated-user conversations
- Policy-compliance checks
- Repeated runs per scenario (consistency, not just single-run pass)

## 4. Recommended stack for a support/ticket agent

1. **Langfuse** (or LangSmith if managed preferred) — trace every production ticket run, collect agent-reply feedback.
2. **DeepEval** — offline suite over a golden set of 50–100 real anonymized tickets:
   - *Triage tasks* (category, priority, routing) → exact-match grading (promptfoo also works well here).
   - *Reply generation* → G-Eval rubrics: policy compliance, groundedness in the KB, tone, no over-promising (refunds/SLAs).
   - *Tool use* (CRM lookups, ticket updates) → tool-correctness metric + "did it actually update the ticket" state checks.
3. **τ-bench-style simulated-user harness** later, once basics are solid — multi-turn conversations are where support agents actually fail.

Wire the offline suite into CI so every prompt/model change reruns the golden set; pipe real production failures from Langfuse back into the dataset.
