# Strands Evals — Deep Dive & Comparison

**Strands Evals** (`pip install strands-agents-evals`, from AWS's Strands Agents team) is an *agent-native* eval framework in ways most eval libraries aren't.

## Core concepts

- **Cases** — test scenarios with input + optional expected output/trajectory + metadata.
- **Experiments** — collections of cases bundled with evaluators.
- **Evaluators** — mostly LLM-as-judge (Bedrock with Claude as the default judge).
- **Task functions** — bridge enabling two modes:
  - **Online eval**: invoke the live agent (development / CI-CD).
  - **Offline eval**: grade historical traces from logs/databases (production monitoring).

## Built-in evaluators (~25, by category)

- **Response quality**: OutputEvaluator (custom rubrics), Helpfulness, Faithfulness, Correctness, Coherence, Conciseness, ResponseRelevance.
- **Safety & compliance**: Harmfulness, Refusal (inappropriate refusals), Stereotyping, InstructionFollowing.
- **Tool usage**: ToolSelectionAccuracy, ToolParameterAccuracy.
- **Conversation flow**: Trajectory (sequence of actions and tool-usage patterns), Interactions.
- **Goal achievement**: GoalSuccessRate (end-to-end task completion).
- **Resilience**: FailureCommunication (error-message clarity), PartialCompletion (progress despite failures), RecoveryStrategy (recovery quality when tools fail).
- **Multimodal**: output/quality/correctness/faithfulness/instruction-following variants for image/document tasks.
- **Deterministic (code-based, no judge)**: Equals, Contains, StartsWith, ToolCalled, StateEquals.
- **Custom** — extend the base Evaluator class.

**Evaluation levels**: OUTPUT_LEVEL (single response), TRACE_LEVEL (turn-by-turn), SESSION_LEVEL (full conversation) — the same experiment can grade all three at once.

## Advantages over LangSmith / Langfuse / DeepEval

1. **ActorSimulator — built-in simulated users.** Generates realistic user personas (personality, expertise, communication style, goal) and runs multi-turn conversations against the agent automatically. τ-bench methodology, productized. For a support agent this is the killer feature: "frustrated customer demanding a refund outside policy" test conversations without hand-scripting.
2. **ExperimentGenerator** — auto-generates test cases and rubrics from a high-level agent description. Solves the golden-set cold-start problem.
3. **Hierarchical evaluation levels** — output/trace/session grading built in; other harnesses make you bolt this together.
4. **Chaos testing + resilience evaluators (unique).** Deterministic fault injection (tool timeouts, network errors, corrupted responses) paired with RecoveryStrategy / PartialCompletion / FailureCommunication evaluators. For a support agent whose CRM/ticketing API will flake in production, tests "does it recover gracefully or tell the customer something false."
5. **Failure detection + root-cause analysis** and **red-team evaluation** (adversarial attack strategies) in the same package.

## Trade-offs

- **Eval harness, not an observability platform** — no trace UI, dashboards, prompt versioning, or user-feedback capture. Pair with Langfuse/LangSmith/Phoenix (Strands agents emit OpenTelemetry, so this pairs cleanly).
- **Ecosystem gravity is AWS/Bedrock** — defaults assume Bedrock as judge; works best with the Strands Agents SDK. Offline trace eval via task functions can wrap other agents, but less turnkey than DeepEval's framework-agnostic approach.
- **Younger project, smaller community** than LangSmith/DeepEval/promptfoo — fewer examples, faster-moving APIs.

## Bottom line for a support/ticket agent

- On **AWS/Bedrock (especially Strands Agents SDK)**: Strands Evals is arguably the best fit — simulated-customer conversations, goal-success scoring, tool-accuracy checks, chaos testing cover exactly the failure modes of support agents. Add Langfuse or CloudWatch/OTel for production tracing.
- **Not on AWS / different framework**: DeepEval + Langfuse remains the more neutral stack; borrow Strands' ideas (simulated users, fault injection) manually.

## Sources

- [AWS blog: Evaluating AI agents for production — a practical guide to Strands Evals](https://aws.amazon.com/blogs/machine-learning/evaluating-ai-agents-for-production-a-practical-guide-to-strands-evals/)
- [Strands Evals — Evaluators docs](https://strandsagents.com/docs/user-guide/evals-sdk/evaluators/)
- [strands-agents/evals on GitHub](https://github.com/strands-agents/evals/)
- [strands-agents-evals on PyPI](https://pypi.org/project/strands-agents-evals/)
- [DeepEval's Strands integration](https://deepeval.com/integrations/frameworks/strands)
