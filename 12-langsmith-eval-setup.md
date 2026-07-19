# LangSmith Eval Setup — Full Evaluator Surface

How to set up and test evals on LangSmith end-to-end, exercising every evaluator capability it supports: custom code evaluators, LLM-as-judge (openevals), trajectory evaluators (agentevals), trace-aware evaluators, summary evaluators, pairwise/comparative evals, pytest integration, and online evaluators. Ends with how this plugs into our harness's `trace_store: langsmith` backend ([09](09-implementation-status-and-fine-print.md)).

## 1. Setup

```bash
pip install -U langsmith openevals agentevals
# openevals  = LLM-as-judge helpers + prebuilt rubric prompts
# agentevals = trajectory evaluators (works on OpenAI-format message lists, no LangChain needed)
```

```bash
export LANGSMITH_API_KEY="lsv2_..."           # smith.langchain.com → Settings → API Keys
export LANGSMITH_TRACING=true                 # auto-trace anything wrapped/instrumented
export LANGSMITH_PROJECT="loom-support-agent" # tracing project name (optional)
```

Same `LANGSMITH_API_KEY` env var the harness's `LangSmithClient` reads; for the harness path also `pip install 'agent-evals[langsmith]'`.

## 2. Create the golden dataset

```python
from langsmith import Client

client = Client()
ds = client.create_dataset("loom-golden-tickets", description="50 anonymized support tickets")

client.create_examples(
    dataset_id=ds.id,
    examples=[
        {
            "inputs": {"ticket": "I was double-charged for my subscription"},
            "outputs": {"expected_output": "refund issued", "category": "billing", "priority": "high"},
            "metadata": {"expected_tools": ["lookup_customer", "issue_refund"], "split": "triage"},
        },
        # ...
    ],
)
```

- Datasets are **versioned** automatically — every edit creates a new version you can pin an experiment to.
- Examples can be tagged into **splits** (`triage`, `rag`, `hard`) and evaluated per-split.
- This shape matches what the harness's `load_dataset()` expects: `expected_output` in outputs, `expected_labels` / `expected_tools` in metadata.

## 3. Target function + `evaluate()`

```python
from langsmith import traceable

@traceable  # captures the full trace incl. tool-call child runs
def target(inputs: dict) -> dict:
    reply, tool_calls = run_my_agent(inputs["ticket"])
    return {"answer": reply, "tool_calls": tool_calls}

results = client.evaluate(
    target,
    data="loom-golden-tickets",          # or client.list_examples(dataset_name=..., splits=["triage"])
    evaluators=[...],                    # row-level (§4a–d)
    summary_evaluators=[...],            # dataset-level (§4e)
    experiment_prefix="claude-refund-prompt-v3",
    num_repetitions=3,                   # each example run 3× — pass^k-style consistency ([10](10-multi-tool-subagent-json-eval.md))
    max_concurrency=4,
    metadata={"prompt_version": "v3"},   # filterable in the UI
)
```

`client.aevaluate(...)` for async targets. Each call creates an **experiment** on the dataset; the UI gives side-by-side comparison and regression highlighting across experiments.

## 4. The full evaluator surface

### a) Custom code evaluators (row-level)

Plain functions; args matched by name (`inputs`, `outputs`, `reference_outputs`). Return bool, float, dict, or a **list of dicts** to emit multiple scores from one evaluator:

```python
def triage_exact(outputs: dict, reference_outputs: dict) -> list[dict]:
    return [
        {"key": "category_match", "score": outputs.get("category") == reference_outputs.get("category")},
        {"key": "priority_match", "score": outputs.get("priority") == reference_outputs.get("priority")},
    ]
```

### b) LLM-as-judge (openevals)

Prebuilt rubric prompts plus custom ones; supports constrained continuous scores and few-shot examples:

```python
from openevals.llm import create_llm_as_judge
from openevals.prompts import CORRECTNESS_PROMPT, HALLUCINATION_PROMPT

correctness = create_llm_as_judge(
    prompt=CORRECTNESS_PROMPT, model="anthropic:claude-sonnet-5", feedback_key="correctness",
)

policy_judge = create_llm_as_judge(
    prompt="Does the reply follow the refund policy: no refunds promised over $500 without approval?\n"
           "<reply>{outputs}</reply>",
    choices=[0.0, 0.5, 1.0],             # constrained score set
    feedback_key="policy_compliance",
)
```

### c) Trajectory evaluators (agentevals)

Grades the *sequence of tool calls* — maps directly to `expected_tools` and the constraint-style matching in [10](10-multi-tool-subagent-json-eval.md):

```python
from agentevals.trajectory.match import create_trajectory_match_evaluator
from agentevals.trajectory.llm import create_trajectory_llm_as_judge, TRAJECTORY_ACCURACY_PROMPT

tool_order = create_trajectory_match_evaluator(
    trajectory_match_mode="superset",    # exact | unordered | subset | superset
)
trajectory_judge = create_trajectory_llm_as_judge(
    prompt=TRAJECTORY_ACCURACY_PROMPT, model="anthropic:claude-sonnet-5",
)
```

### d) Trace-aware evaluators

Declare `run` and `example` params to receive the full Run tree — walk `run.child_runs` to check intermediate steps (retrieved docs, tool arguments) rather than only final output:

```python
def used_refund_tool(run, example) -> dict:
    tool_runs = [r.name for r in (run.child_runs or []) if r.run_type == "tool"]
    return {"key": "called_issue_refund", "score": "issue_refund" in tool_runs}
```

### e) Summary evaluators (dataset-level)

Computed once over the whole experiment — for aggregates a per-row score can't express (precision/recall over triage labels, pass-rate gates):

```python
def triage_accuracy(inputs: list[dict], outputs: list[dict], reference_outputs: list[dict]) -> dict:
    hits = sum(o.get("category") == r.get("category") for o, r in zip(outputs, reference_outputs))
    return {"key": "triage_accuracy", "score": hits / len(outputs)}
```

### f) Pairwise / comparative evaluators

Compare two existing experiments head-to-head (prompt A vs prompt B) instead of scoring in isolation; results appear in the comparison view with win rates:

```python
from langsmith import evaluate

evaluate(
    ("experiment-A-name-or-id", "experiment-B-name-or-id"),
    evaluators=[ranked_preference],  # receives inputs + both outputs, returns which won
)
```

openevals also ships pairwise judge helpers.

### g) Pytest integration (CI)

Evals as tests — run with plain `pytest`, each test logs as a tracked experiment case (`LANGSMITH_TEST_SUITE` groups them):

```python
import pytest
from langsmith import testing as t

@pytest.mark.langsmith
def test_billing_ticket():
    t.log_inputs({"ticket": "double charged"})
    out = target({"ticket": "double charged"})
    t.log_outputs(out)
    t.log_feedback(key="has_apology", score="sorry" in out["answer"].lower())
    assert out["category"] == "billing"
```

### h) Online evaluators (production)

Configured in the UI on a tracing project (Rules → Add rule → LLM-as-judge or code evaluator): sample e.g. 10% of live traces, auto-score (hallucination, policy compliance), route low scorers to an **annotation queue** for human review. This is the LangSmith-native equivalent of the harness's `online_sample_rate` + annotation-queue degradation path.

## 5. Testing the setup

1. Auth smoke test: `python -c "from langsmith import Client; print(Client().info)"`.
2. Run `evaluate()` with just `triage_exact` on a 3-example dataset — confirm the experiment appears under the dataset in the UI with per-row scores.
3. Add the judges and `num_repetitions=3`; check score variance across repetitions in the experiment view.
4. Wire into CI: the pytest route, or a script that reads aggregate feedback stats off the experiment and fails the build below a threshold.
5. Harness path: point an agent config at `trace_store: langsmith` with `dataset: loom-golden-tickets` and run the evaluator — scores land as feedback on runs via `post_score`, tagged `rubric_version=<v>` ([09 §8](09-implementation-status-and-fine-print.md)).

## 6. Caveat vs. the Langfuse-first plan

LangSmith is managed-only for practical purposes (self-hosting is enterprise-tier). If data residency drove the Langfuse choice in [04](04-eval-harness-plan-langfuse-gcp.md), keep LangSmith to offline golden-set evals over anonymized tickets, with Langfuse remaining the production trace store.
