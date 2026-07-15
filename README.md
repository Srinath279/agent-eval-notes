# Agent Evaluation — Notes

Notes on evaluating AI agents (support / ticket-analysis use cases), eval frameworks, and the plan for a reusable eval harness on Langfuse + GCP.

## Index

1. [Agent Evaluation Fundamentals](01-agent-eval-fundamentals.md) — what agent evals are, the two levels of evaluation, full metrics catalog, what data to log, how to run evals.
2. [Eval Frameworks Comparison](02-eval-frameworks-comparison.md) — Langfuse, LangSmith, Braintrust, Phoenix, DeepEval, promptfoo, Ragas, τ-bench; recommended stack for a support/ticket agent.
3. [Strands Evals Deep Dive](03-strands-evals.md) — AWS Strands Evals SDK: evaluators, ActorSimulator, chaos testing; advantages and trade-offs vs. other frameworks.
4. [Reusable Eval Harness Plan (Langfuse + GCP)](04-eval-harness-plan-langfuse-gcp.md) — architecture, reusable package design, offline + online pipelines, golden data & annotation workflow, metrics, rollout phases.
5. [Harness Gap Analysis](05-harness-gaps.md) — critical pieces missing from the plan: trace adapters, PII redaction, judge versioning/caching, statistical rigor (pass^k, baselines), harness self-tests, ops reliability, governance, and later-phase items (red-team, chaos, drift, shadow A/B).
