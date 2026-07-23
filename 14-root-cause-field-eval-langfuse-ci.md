# 14. Scoring the Root-Cause Field Against Langfuse Closeout Metadata (CI Experiment)

How to evaluate **one critical free-text key** of the agent's JSON output —
`root_cause_description` — against a **reference** the reviewer stashed in the
Langfuse **dataset-item metadata** (the ticket's *closeout / disposition long
description*), score it as a Langfuse **experiment**, and gate it in **CI/CD**.

This is the concrete, single-field instance of the funnel in
[note 10 §1](10-multi-tool-subagent-json-eval.md) ("free-text fields → LLM judge
with a rubric scoped to *that field only*, reference-based") wired onto the
harness that already exists ([agent-evals repo](https://github.com/Srinath279/agent-evals)),
and gated exactly like [note 13 §4](13-annotation-queue-to-dataset-and-ci.md).

> Direction of travel:
> ```
> Langfuse Dataset (golden)                 agent under test
>   item.input      ── ticket ─────────────────▶  task_fn ──▶ output JSON
>   item.metadata.closeout_description                          { root_cause_description: "...",
>            │  (the human ground truth)                          category, disposition, ... }
>            ▼                                                        │  extract $.root_cause_description
>   Case.metadata[reference_field]  ─────────┐                       ▼
>                                            └──▶ root_cause_match (LLM judge, reference-based)
>                                                       │  Score(name="root_cause_match", 0..1)
>                                       ┌───────────────┼───────────────┐
>                                       ▼               ▼               ▼
>                          score_thresholds gate   Langfuse Score   experiment (dataset run) view
>                             (CI red/green)        on the trace     compare commit-over-commit
> ```

---

## 0. Where this fits the existing code

| Piece | Status today | This note adds |
|---|---|---|
| Load Langfuse dataset → `Case` (incl. `metadata`) | `LangfuseClient.load_dataset` ✅ ([langfuse_client.py:28](../agent-evals/src/agent_evals/core/langfuse_client.py#L28)) | nothing — `item.metadata` already lands in `Case.metadata`, so the closeout ref is already reachable |
| Field-scoped LLM judge, reference-based | ❌ (only `goal_success`, whole-output) | `root_cause_match` evaluator (§2) |
| Extract a nested key from the output JSON | partial (`exact_match` `field:` one level) | dotted-path extractor in the evaluator (§2) |
| Gate a metric in CI | `evals run --baselines` ✅ ([runner.py:308](../agent-evals/src/agent_evals/runner.py#L308)) | one `score_thresholds` row + GitHub Actions (§5) |
| Post scores to Langfuse | `run_offline(post_scores=True)` ✅ ([runner.py:355](../agent-evals/src/agent_evals/runner.py#L355)) | optional **dataset-run (experiment)** linking (§4) |

Two hard rules carry over unchanged:

- **Redaction before anything leaves the machine** (note 05 §2 / 09 §7). The
  closeout text and the agent field are production data; the evaluator wraps the
  judge payload in `redact_text` exactly like `goal_success`
  ([goal_success.py:76](../agent-evals/src/agent_evals/evaluators/llm_judge/goal_success.py#L76)).
- **Rubric text change ⇒ bump `rubric_version`** — it is part of the score-cache
  key and is stamped on every score (note 09 §8, metrics.md rules).

---

## 1. The data contract (what you already set up)

Each Langfuse dataset item carries, in **metadata**, the human closeout the
agent is graded against:

```jsonc
// a Langfuse dataset item
{
  "input":  { "subject": "...", "body": "...", "ticket_id": "INC-4821" },
  "expected_output": null,                       // optional; the ref lives in metadata
  "metadata": {
    "closeout_description": "Root cause was a stale DNS cache on the edge node; the record TTL had lapsed and the failover pointed at a decommissioned host.",
    "disposition": "resolved-infra",
    "expected_labels": { "category": "infra" }
  }
}
```

`LangfuseClient.load_dataset` copies the whole `item.metadata` dict into
`Case.metadata` ([langfuse_client.py:32-41](../agent-evals/src/agent_evals/core/langfuse_client.py#L32-L41)),
so **no client change is needed** — `case.metadata["closeout_description"]` is
already available to any evaluator. The agent's output is a JSON object whose
**critical key** is `root_cause_description`:

```jsonc
// task_fn(case.input).output  — the agent under test
{
  "root_cause_description": "A DNS TTL expiry caused failover to a dead host.",
  "category": "infra",
  "disposition": "resolved-infra",
  "recommended_action": "..."
}
```

We grade **only** `root_cause_description` against **only**
`metadata.closeout_description`. Everything else in the JSON (category,
disposition) is graded by the existing deterministic evaluators (`label_match`,
`json_schema_valid`) — this note is about the one free-text key that can't be
exact-matched.

> **Every graded item must carry the reference.** A judge with no reference
> can't grade, and the harness is fail-closed (a missing ref scores `0.0`, note
> 09 §6) — which would falsely fail the gate. So the dataset used for this
> metric must have `closeout_description` on **every** item. If you have a mixed
> dataset, either (a) split the root-cause items into their own dataset/config,
> or (b) use the `skip_if_missing` param (§2) to vacuously pass ref-less items.

---

## 2. Implement — the `root_cause_match` evaluator

New file `src/agent_evals/evaluators/llm_judge/root_cause_match.py`. It mirrors
`goal_success` (rubric resolution, redaction, judge stamp) but is **field-scoped
and reference-based**: it pulls one key from the output and one key from the
case metadata, and judges semantic agreement between them.

```python
"""root_cause_match — LLM judge, output level. Grades ONE critical free-text
key of the agent's JSON output (default `root_cause_description`) against a
human reference stored in the case metadata (default `closeout_description`,
the ticket's closeout / disposition long description).

Reference-based judging (note 10 §6): the judge is given the ground truth, so
it scores semantic agreement, not general plausibility. Payload is PII-redacted
before it reaches the judge (note 05 §2). Rubric changes MUST bump
rubric_version — it is part of the score-cache key."""

from __future__ import annotations

import json
from typing import Any, Optional

from agent_evals.core.evaluator import BaseEvaluator, register
from agent_evals.core.redaction import redact_text
from agent_evals.core.schemas import Case, Score, Trace

RUBRIC = """\
Compare the agent's ROOT CAUSE against the human REFERENCE root cause for the
same ticket. Score how well the agent identified the SAME underlying cause.

You are given:
- TICKET: the original ticket/context (for grounding; may be brief)
- REFERENCE: the human-written closeout / disposition describing the true root cause
- AGENT: the agent's root_cause_description field

Judge meaning, not wording. Paraphrases, added correct detail, and different
phrasing of the same cause are fine. Score down only when the agent names a
DIFFERENT cause, misses the operative cause, or asserts a cause the reference
contradicts.

Scoring guide:
- 1.0  same root cause; fully consistent with the reference (wording may differ)
- 0.7  correct core cause; minor omission or a small unsupported embellishment
- 0.4  partially right; identifies a contributing factor but misses the primary
       cause, or mixes the correct cause with an incorrect one
- 0.0  wrong cause, no cause given, or a cause the reference contradicts
       (a confident but incorrect root cause is always 0.0)
"""

RUBRIC_VERSION = "root_cause_match/v1"


def _output_as_obj(output: Any) -> Any:
    """The agent output may arrive as a dict or a JSON string (note: label_match
    does the same coercion)."""
    if isinstance(output, str):
        try:
            return json.loads(output)
        except (json.JSONDecodeError, ValueError):
            return output
    return output


def _dig(obj: Any, path: str) -> Any:
    """Dotted-path lookup into nested dicts, e.g. 'analysis.root_cause_description'.
    Returns None if any segment is missing or a non-dict is traversed."""
    cur = obj
    for seg in path.split("."):
        if not isinstance(cur, dict):
            return None
        cur = cur.get(seg)
    return cur


def _fmt(value: Any) -> str:
    if isinstance(value, (dict, list)):
        return json.dumps(value, ensure_ascii=False, default=str)
    return str(value)


@register
class RootCauseMatchEvaluator(BaseEvaluator):
    name = "root_cause_match"
    level = "output"
    requires_judge = True
    requires_case = True  # needs the reference from the golden case
    rubric_version = RUBRIC_VERSION
    # defaults used by `evals push-rubrics` to seed Prompt Management
    rubric_name = "evals/root_cause_match"
    rubric_text = RUBRIC

    def __init__(self, **params) -> None:
        super().__init__(**params)
        from agent_evals.core.rubrics import resolve_rubric

        # which key of the output JSON is the critical field, and where the
        # reference lives on the case — both configurable, sensible defaults
        self.output_field = params.get("output_field", "root_cause_description")
        self.reference_field = params.get("reference_field", "closeout_description")
        # ref-less item: 0.0 (fail-closed, default) vs 1.0 (vacuous pass)
        self.skip_if_missing = bool(params.get("skip_if_missing", False))

        self.rubric_text, self.rubric_version = resolve_rubric(
            params.get("rubric_name", self.rubric_name),
            RUBRIC,
            RUBRIC_VERSION,
            from_store=params.get(
                "rubric_from_store", params.get("rubric_from_langfuse", False)
            ),
            store=params.get("trace_store", "langfuse"),
        )

    def _reference(self, case: Case) -> Any:
        """Reference lives in case.metadata[reference_field]; fall back to the
        case's expected_output so either data layout works."""
        ref = case.metadata.get(self.reference_field) if case else None
        if ref is None and case is not None:
            ref = case.expected_output
        return ref

    def evaluate(self, trace: Trace, case: Optional[Case] = None) -> Score:
        reference = self._reference(case) if case else None
        if reference is None:
            # No ground truth to grade against — fail-closed unless told to skip.
            value = 1.0 if self.skip_if_missing else 0.0
            return Score(
                name=self.name, value=value, level=self.level,
                comment=f"no '{self.reference_field}' reference on case "
                        f"({'skipped' if self.skip_if_missing else 'fail-closed 0.0'})",
                metadata=self._judge_stamp(),
            )

        actual = _dig(_output_as_obj(trace.output), self.output_field)
        if actual is None or (isinstance(actual, str) and not actual.strip()):
            # Agent produced no root cause at all — a real miss, not a config error.
            return Score(
                name=self.name, value=0.0, level=self.level,
                comment=f"agent output has no '{self.output_field}' field",
                metadata=self._judge_stamp(),
            )

        payload = redact_text(
            f"TICKET:\n{_fmt(trace.input)}\n\n"
            f"REFERENCE:\n{_fmt(reference)}\n\n"
            f"AGENT:\n{_fmt(actual)}"
        )
        verdict = self.judge.verdict(self.rubric_text, payload)
        return Score(
            name=self.name,
            value=verdict.score,
            level=self.level,
            comment=verdict.reasoning,
            metadata=self._judge_stamp(),
        )
```

Register it so it loads with the other built-ins — add one line to
`src/agent_evals/evaluators/__init__.py` (next to the `goal_success` import):

```python
import agent_evals.evaluators.llm_judge.root_cause_match  # noqa: F401
```

**Why an LLM judge and not `exact_match`.** A closeout narrative and the agent's
root cause will never be string-equal even when they mean the same thing —
`exact_match` would score every case 0. Semantic agreement against the reference
is exactly the reference-based free-text case from note 10 §1 step 2 / §6.

---

## 3. Register the metric + wire the config

### 3.1 — `metrics.md` (governance rule: same PR as the evaluator)

Add one row to the registry ([metrics.md](../agent-evals/metrics.md)):

| name | scale | direction | level | type | owner | definition |
|---|---|---|---|---|---|---|
| `root_cause_match` | 0–1 | higher better | output | LLM judge | platform | Semantic agreement between the agent's `root_cause_description` (configurable `output_field`) and the human closeout reference in the case metadata (`reference_field`, default `closeout_description`). Wrong/absent cause = 0.0. Rubric: `root_cause_match/v1`. |

### 3.2 — Agent config

A config that reads the **Langfuse** golden dataset and grades the field. Point
`dataset:` at your Langfuse dataset name (not `local_dataset`), and set a real
judge:

```yaml
# configs/rca_agent.yaml
agent: rca-agent
trace_store: langfuse
trace_adapter: langfuse-generic
task_fn: your_pkg.rca_agent:invoke     # task_fn(case.input) -> Trace | dict
dataset: rca-golden-v1                  # the Langfuse dataset with closeout metadata

repeats: 3                             # k for pass^k (judge is non-deterministic)

judge:
  provider: anthropic                  # mock for local dry-runs; anthropic/vertex in CI
  model: claude-opus-4-8
  daily_budget_usd: 25

evaluators:
  - name: root_cause_match
    output_field: root_cause_description       # dotted path ok: analysis.root_cause_description
    reference_field: closeout_description       # key in the dataset item metadata
    # skip_if_missing: false                    # default: ref-less items fail-closed
    # rubric_from_store: true                   # pull rubric text from Prompt Management
  - name: json_schema_valid                     # the output contract still applies
    schema:
      type: object
      required: [root_cause_description, category, disposition]
      properties:
        root_cause_description: {type: string}
        category: {type: string}
        disposition: {type: string}
  - name: label_match                           # deterministic keys graded separately
    fields: [category, disposition]

score_thresholds:                     # the CI gate
  root_cause_match: 0.80              # start conservative; raise as the judge is trusted
  json_schema_valid: 1.0
  label_match: 0.90
```

`repeats: 3` matters here — a judge score is non-deterministic, so pass^k (all 3
repeats clear the bar) is the reliability floor you actually ship on (note 10 §5).
`expected_labels` on each case still comes from `metadata.expected_labels`
(load_dataset already maps it), so `label_match` keeps working alongside.

---

## 4. Score it as a Langfuse **experiment** (dataset run)

Two layers, and you only strictly need the first for CI.

### 4.1 — Harness-native (already works): `--post-scores`

`evals run --post-scores` posts every `root_cause_match` score to Langfuse via
`store.post_score` ([runner.py:355-361](../agent-evals/src/agent_evals/runner.py#L355-L361)),
stamped with `judge_provider` / `judge_model` / `rubric_version`. That gives you
the scores in Langfuse and the local run artifacts (`scores.jsonl`,
`manifest.json`, `report.md`) — enough to gate. But those scores attach to the
runner's **deterministic trace IDs** (`{run_id}/{case_id}/r{k}`), not to a
Langfuse **dataset run**, so they won't show in the side-by-side *Experiments*
comparison view.

### 4.2 — Link a dataset run (the Experiments UI)

To compare commit-over-commit in Langfuse's *Experiments* tab, each run must be
a **dataset run**: link every executed item to a trace under a run name, then
attach the score. This is the one new bit of store I/O — add it to
`LangfuseClient` (best-effort, same policy as `enqueue_annotation`; SDK method
names vary by version, so pin per note 09 §3 and confirm):

```python
# src/agent_evals/core/langfuse_client.py
def link_experiment(self, run_name: str, item_id: str, trace_id: str,
                    scores: list[Score]) -> bool:
    """Attach one graded dataset item to a Langfuse dataset run so it appears in
    the Experiments view. Best-effort: returns False on any SDK/version mismatch
    rather than failing the eval run."""
    try:
        item = self._lf.api.dataset_items.get(id=item_id)  # or cache from load_dataset
        item.link(trace_id=trace_id, run_name=run_name)     # SDK: DatasetItemClient.link
        for s in scores:
            self.post_score(s)
        return True
    except Exception:
        return False
```

Drive it from the CLI with an opt-in `--experiment <name>` flag so the default
path stays unchanged. Name the run after the commit so runs are comparable:

```bash
uv run evals run --config configs/rca_agent.yaml \
  --post-scores --experiment "ci-${GITHUB_SHA::7}" \
  --baselines baselines/ --out runs/ci
```

Then in Langfuse → your dataset → **Experiments**, each CI commit is a column
and `root_cause_match` is a row: regressions on the critical field are visible
per item, not just as one aggregate number. (The `case_id` for a Langfuse
dataset is the item `id` — [langfuse_client.py:35](../agent-evals/src/agent_evals/core/langfuse_client.py#L35) —
so the runner already has the `item_id` needed to link.)

> If you don't want to touch the client yet, ship **4.1 only** — the gate works
> off the local run's exit code regardless. Add 4.2 when you want the visual
> experiment comparison in the Langfuse UI.

---

## 5. Run it in CI/CD

The gate mechanism is unchanged from note 13 §4: `evals run --baselines` returns
**exit 1** when `root_cause_match`'s mean drops below its threshold **or**
regresses significantly vs. the committed baseline ([runner.py:307-335](../agent-evals/src/agent_evals/runner.py#L307-L335)),
so the PR check goes red with no extra YAML logic.

`agent-evals/.github/workflows/rca-eval-gate.yml`:

```yaml
name: rca-eval-gate
on:
  pull_request:
    paths:
      - "configs/rca_agent.yaml"
      - "src/agent_evals/evaluators/llm_judge/root_cause_match.py"
      - "agents/rca/**"          # the agent's own prompt/tool code
  workflow_dispatch: {}

jobs:
  root-cause-eval:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    env:
      LANGFUSE_HOST:       ${{ secrets.LANGFUSE_HOST }}
      LANGFUSE_PUBLIC_KEY: ${{ secrets.LANGFUSE_PUBLIC_KEY }}
      LANGFUSE_SECRET_KEY: ${{ secrets.LANGFUSE_SECRET_KEY }}
      ANTHROPIC_API_KEY:   ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }          # baselines/ history for the regression gate
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --extra langfuse --extra anthropic

      # Fail closed: if the judge or Langfuse is unreachable, the gate fails
      # (note 09 §6) — a green check must mean the field was actually graded.
      - name: Grade root_cause_description vs closeout reference
        run: |
          uv run evals run \
            --config configs/rca_agent.yaml \
            --post-scores --experiment "ci-${GITHUB_SHA::7}" \
            --baselines baselines/ \
            --out runs/ci-${{ github.run_id }}

      - name: Publish report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: rca-eval-${{ github.run_id }}
          path: runs/ci-${{ github.run_id }}/
```

**Cost/flake control in CI** (the one difference from a deterministic gate):

- The judge costs tokens and is non-deterministic. The **score cache**
  ([runner.py:104-123](../agent-evals/src/agent_evals/runner.py#L104-L123)) keys on
  `(trace_id, evaluator, rubric_version, judge_provider, judge_model)`, so
  identical retries never re-pay — but a new commit mints new trace IDs, so
  budget for `n_cases × repeats` judge calls per run. Keep the dataset tight
  (note 10 §4) and set `judge.daily_budget_usd` as the kill switch.
- `repeats: 3` + the mean gate absorbs single-run judge jitter; pin the judge
  model (don't float to "latest") so the bar doesn't move under you.

### 5.1 — Promote a baseline after an intentional win

When a PR legitimately improves `root_cause_match`, refresh the baseline so the
next PR compares against the new bar (identical to note 13 §4.4):

```bash
uv run evals run --config configs/rca_agent.yaml --out runs/candidate
uv run evals promote-baseline --config configs/rca_agent.yaml --run runs/candidate
git add baselines/rca-agent.json && git commit -m "bump rca eval baseline"
```

Hold the dataset version fixed while comparing (note 13 §5): a "gain" measured
on a new dataset version may just be an easier dataset. Re-baseline on any
dataset bump before trusting a diff.

---

## 6. Test — without Langfuse, without spending judge tokens

Use `MockJudge` for the evaluator logic and a fake reference — the whole point of
the judge abstraction ([judge.py:45](../agent-evals/src/agent_evals/core/judge.py#L45)).

`tests/test_root_cause_match.py`:

```python
from agent_evals.core.judge import MockJudge, Verdict
from agent_evals.core.schemas import Case, Trace
from agent_evals.evaluators.llm_judge.root_cause_match import RootCauseMatchEvaluator


def _ev(**params):
    ev = RootCauseMatchEvaluator(**params)
    # echo score so we can assert routing; capture the payload the judge saw
    ev._seen = {}
    def fn(rubric, payload):
        ev._seen["payload"] = payload
        return Verdict(reasoning="ok", score=0.9)
    ev.judge = MockJudge(verdict_fn=fn)
    return ev


def _case(**md):
    return Case(case_id="c1", input={"body": "outage"}, metadata=md)


def test_grades_field_against_metadata_reference():
    ev = _ev()
    trace = Trace(trace_id="t1", output={"root_cause_description": "DNS TTL expiry"})
    score = ev.evaluate(trace, _case(closeout_description="stale DNS cache / TTL lapsed"))
    assert score.value == 0.9
    assert "DNS TTL expiry" in ev._seen["payload"]          # agent field in payload
    assert "stale DNS cache" in ev._seen["payload"]         # reference in payload
    assert score.metadata["rubric_version"] == "root_cause_match/v1"


def test_missing_field_is_zero_without_calling_judge():
    ev = _ev()
    trace = Trace(trace_id="t1", output={"category": "infra"})   # no root cause
    score = ev.evaluate(trace, _case(closeout_description="x"))
    assert score.value == 0.0
    assert ev.judge.calls == 0                                # never paid the judge


def test_missing_reference_fail_closed_by_default():
    assert _ev().evaluate(
        Trace(trace_id="t1", output={"root_cause_description": "x"}), _case()
    ).value == 0.0


def test_missing_reference_can_skip():
    assert _ev(skip_if_missing=True).evaluate(
        Trace(trace_id="t1", output={"root_cause_description": "x"}), _case()
    ).value == 1.0


def test_reference_falls_back_to_expected_output():
    ev = _ev()
    case = Case(case_id="c1", input={}, expected_output="the real cause")
    ev.evaluate(Trace(trace_id="t1", output={"root_cause_description": "y"}), case)
    assert "the real cause" in ev._seen["payload"]


def test_pii_redacted_before_judge():
    ev = _ev()
    trace = Trace(trace_id="t1",
                  output={"root_cause_description": "email jane@acme.com"})
    ev.evaluate(trace, _case(closeout_description="call 415-555-1212"))
    assert "@acme.com" not in ev._seen["payload"]            # email redacted
    assert "555-1212" not in ev._seen["payload"]             # phone redacted


def test_dotted_output_field():
    ev = _ev(output_field="analysis.root_cause_description")
    trace = Trace(trace_id="t1",
                  output={"analysis": {"root_cause_description": "nested cause"}})
    ev.evaluate(trace, _case(closeout_description="x"))
    assert "nested cause" in ev._seen["payload"]
```

Run the suite:

```bash
cd agent-evals
uv run pytest tests/test_root_cause_match.py -q     # the new tests
uv run pytest -q                                    # full suite must stay green
```

### 6.1 — Dry-run against real Langfuse (manual, once)

Confirm the dataset metadata actually carries the reference before wiring CI —
run with the mock judge so it costs nothing, and eyeball the report:

```bash
export LANGFUSE_PUBLIC_KEY=... LANGFUSE_SECRET_KEY=... LANGFUSE_HOST=...
# temporarily set judge.provider: mock in the config, then:
uv run evals run --config configs/rca_agent.yaml --out runs/dryrun
# inspect runs/dryrun/rca-agent-*/report.md — every case should show a
# root_cause_match score and NO "no 'closeout_description' reference" comments
```

If any case logs the missing-reference comment, that item lacks the closeout
metadata — fix the dataset (§1) before trusting the gate.

### 6.2 — Calibrate the judge before it gates (recommended)

`root_cause_match` gates a subjective field, so validate it against human labels
before trusting the number (note 10 §6, note 04 §4a). Hand-label 50–100 items
(root cause correct: yes/no), then join judge vs. human by `trace_id` with the
existing tool and require kappa > 0.8 (note 13 §2.5):

```bash
uv run evals calibrate --scores runs/dryrun/rca-agent-*/scores.jsonl \
  --labels human_root_cause_labels.jsonl --metric root_cause_match
```

Until it clears, keep the threshold conservative (0.80) and treat the metric as
advisory rather than merge-blocking.

---

## 7. Recap

1. The reviewer already put the closeout / disposition long description in each
   Langfuse **dataset-item metadata**; `load_dataset` surfaces it as
   `Case.metadata[reference_field]` — **no client change**.
2. **`root_cause_match`** (new LLM-judge evaluator) extracts the critical output
   key `root_cause_description` (dotted path ok), redacts, and scores **semantic
   agreement** against that reference; missing cause = 0.0, missing reference =
   fail-closed 0.0 (or vacuous pass with `skip_if_missing`).
3. Wire it in the agent config against the **Langfuse dataset**, add its
   `metrics.md` row (same PR) and one `score_thresholds` entry.
4. `evals run --post-scores` puts the score in Langfuse; add `--experiment`
   (dataset-run linking, §4.2) for the commit-over-commit **Experiments** view.
5. **CI/CD**: `evals run --baselines` on every PR that touches the agent/config/
   evaluator — exit 1 on threshold breach or significant regression turns the
   required check red. Fail-closed if the judge/Langfuse is unreachable.

**New code to write:** `evaluators/llm_judge/root_cause_match.py` (+ one import
line), the `metrics.md` row, `configs/rca_agent.yaml`, `tests/test_root_cause_match.py`,
one GitHub Actions workflow, and — only for the visual Experiments view —
`LangfuseClient.link_experiment` + a `--experiment` CLI flag. Everything else
(dataset loading, judge, redaction, score cache, baseline/regression gating,
`evals calibrate`) already exists.
