# Eval Frameworks for a Support / Ticket-Analysis Agent

Two kinds of tooling are needed: an **observability + tracing platform** (capture production runs) and an **eval harness** (run the golden ticket set offline and grade outputs).

## 1. Tracing + evaluation platforms (the core choice)

| Framework | Type | Why it fits a support agent |
|---|---|---|
| **Langfuse** | Open-source (self-hostable) or cloud | Full traces of every LLM/tool call, user feedback capture (thumbs on replies), LLM-as-judge evaluators on live traces, prompt versioning. Most popular OSS choice. |
| **LangSmith** | Managed (LangChain team) | Best-in-class trace UI, datasets + regression testing, online evaluators, annotation queues for human review. Works without LangChain. |
| **Braintrust** | Managed | Strong offline eval loops (Evals API), side-by-side diffing of prompt/model versions, good CI integration. |
| **Arize Phoenix** | Open-source | OpenTelemetry-based tracing, built-in evals (relevance, hallucination, toxicity), good for production drift monitoring. |
| **W&B Weave** | Managed | Traces + scorers + leaderboards per model version; good if already on Weights & Biases. |

## 2. Eval harnesses / grading libraries

- **DeepEval** (open-source, pytest-style) — easiest way to write agent evals as code. Built-in metrics that map to support use cases: answer relevancy, faithfulness/groundedness, hallucination, **task completion**, **tool correctness**, plus custom G-Eval rubrics ("was the reply empathetic and did it follow the refund policy?").
- **promptfoo** (open-source, config-driven) — YAML test cases; great for regression-testing classification tasks like ticket triage (category, priority, sentiment) with exact expected labels. Also does red-teaming / prompt-injection testing.
- **Ragas** — if the agent does RAG over a knowledge base / past tickets: grades retrieval quality (context precision/recall, faithfulness).
- **OpenAI Evals / MLflow LLM Evaluate** — usable, but the above are more actively used for agent workloads.

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
