# Agent Evaluation — Fundamentals & Metrics

Evaluating an agent is different from evaluating a plain LLM call. A chatbot gives one answer you can grade; an agent takes a **multi-step trajectory** — it plans, calls tools, reacts to results, and eventually produces an outcome. So you need to evaluate both **the final outcome** and **the path it took to get there**.

## 1. The two levels of evaluation

- **End-to-end (outcome) evals** — "Did the agent accomplish the task?" Define a task with a known expected result and check whether the agent got there, regardless of how.
- **Trajectory (process) evals** — "Did the agent behave well along the way?" Two agents can both succeed, but one did it in 4 steps for $0.02 and the other took 40 steps, called the wrong tools, and cost $1.50.

## 2. Metrics to capture

### Task success / quality
- **Task success rate (pass rate)** — % of eval tasks completed correctly. The headline metric.
- **pass^k / consistency** — run the same task k times; does it succeed *every* time? Agents are non-deterministic, so single-run pass rate overstates reliability.
- **Partial credit / rubric score** — for non-binary tasks, score against a rubric (often LLM-as-judge).
- **Answer correctness / groundedness** — is the final answer factually supported by what the tools actually returned (no hallucinated results)?

### Tool-use metrics
- **Tool selection accuracy** — did it pick the right tool for the step?
- **Tool call validity** — % of tool calls with correct schema/arguments.
- **Tool error rate** — how often tool calls fail, and whether the agent **recovers** or loops/gives up.
- **Redundant calls** — same tool called with same args repeatedly (a classic sign of a confused agent).

### Efficiency / cost
- **Steps (turns) per task** — vs. an expected optimal number.
- **Tokens per task** — input + output, and cache hit rate if using prompt caching.
- **Cost per task ($)** and **latency per task** (end-to-end wall clock + per-step latency).
- **Loop / stall detection** — % of runs that hit max-iterations without finishing.

### Trajectory quality
- **Plan quality** — was the decomposition sensible? (usually LLM-judge scored)
- **Progress rate** — does each step move toward the goal, or does it wander?
- **Recovery behavior** — after a failed step, does it adapt or repeat the same mistake?
- **Instruction adherence** — did it respect constraints ("don't modify prod", "only use these tools", output format)?

### Safety / trust
- **Harmful or unauthorized actions** — did it try something outside its permitted scope?
- **Hallucinated tool results or claims** — reporting success when the tool actually failed is the most dangerous agent failure mode.
- **Escalation behavior** — does it ask the user when it should, instead of guessing on destructive actions?
- **Prompt-injection resistance** — does content retrieved from tools hijack its instructions?

### Operational (online/production) metrics
- **Human intervention rate** — how often a person had to take over or correct it.
- **User feedback** — thumbs up/down, edit rate of agent output, task abandonment.
- **Regression tracking** — pass rate per version, so a prompt/model change that breaks something is visible.

## 3. What data to log (full traces)

You can't compute any metric without **full traces**. Capture per run:

1. **Task ID + input** — the user request and any initial context.
2. **Every LLM call** — prompt, response, model, tokens in/out, latency.
3. **Every tool call** — tool name, arguments, result, success/failure, duration.
4. **The agent's reasoning/plan** (if exposed) — needed for trajectory judging.
5. **Final output** and a **termination reason** (completed / max steps / error / user abort).
6. **Metadata** — model version, prompt version, timestamp, run ID, seed/config — so results can be sliced by version and failures reproduced.

## 4. How to run the evals

- **Build a golden dataset**: 30–200 representative tasks with expected outcomes. Start small; grow it from real production failures — every bug becomes a test case.
- **Graders**: *code-based checks* wherever the outcome is verifiable (file exists, API state changed, exact answer matches); *LLM-as-judge with a rubric* for fuzzy quality (helpfulness, plan quality). Spot-check the judge against human labels so you trust it.
- **Run each task multiple times** (non-determinism); track mean and worst-case.
- **Automate in CI** — any prompt/tool/model change reruns the suite; compare pass rate / cost / latency against baseline.
- **Close the loop with production**: sample live traces, judge them offline, promote interesting failures into the golden set.

## TL;DR — minimal starter set

If you capture only five things: **task success rate**, **tool-call error rate**, **steps per task**, **cost + latency per task**, and **full traces of every run** (so any other metric can be added retroactively). Everything else builds on top.
