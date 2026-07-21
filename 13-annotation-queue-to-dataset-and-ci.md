# 13. Annotation Queue → Dataset → CI Evaluation

How a reviewer's labels in a Langfuse **annotation queue** become a versioned
**golden dataset**, and how the CI evaluator runs against that dataset to gate a
release. This closes the loop the harness already opens: [note 09](09-implementation-status-and-fine-print.md)
and the online pipeline push below-threshold traces *into* a queue
(`LangfuseClient.enqueue_annotation`); this note builds the **drain** direction
— queue → cases → dataset — plus the CI wiring.

> Direction of travel:
> ```
> online pipeline ──enqueue_annotation()──▶ Langfuse annotation queue
>                                                   │  reviewer labels in UI
>                                                   ▼
>        promote-annotations  ◀──fetch_annotations()── (completed items)
>                │  map → Case, redact, version
>                ▼
>        seed_dataset()  ──▶ Langfuse Dataset  ──▶  CI: evals run  ──▶ gate
> ```

---

## 0. Where this fits the existing code

| Piece | Status today | This note adds |
|---|---|---|
| Push trace to queue | `enqueue_annotation()` ✅ | — |
| Fetch completed annotations | ❌ | `fetch_annotations()` — all human score dimensions, incremental (§2.5–2.6) |
| Capture human eval across dimensions | partial (single `user_feedback`) | multi-dimension queue → trace Scores + calibration set (§2.5) |
| Map annotation → `Case` | ❌ | `annotations_to_cases()` pipeline fn |
| Redact + seed dataset | `seed_dataset()` ✅ | reuse it, add `promote-annotations` CLI |
| Run evaluator in CI | `evals run --baselines` ✅ | GitHub Actions workflow |

Everything below respects the repo's two hard rules:
- **Redaction happens before anything leaves the machine** (note 05 §2 / 09 §7)
  — the CLI redacts `Case.input` / `expected_output` before `seed_dataset`.
- **The store is the only thing that talks to Langfuse** (`core/store.py`).
  New queue I/O goes on the `TraceStore` contract, not in the pipeline.

---

## 1. What the reviewer does (no code)

1. A trace lands in the queue `<agent>-review` — either automatically
   (online pipeline, threshold failure) or manually ("Add to annotation queue"
   in the Langfuse UI).
2. The reviewer opens the queue and, per trace, records:
   - a score on **each configured dimension** (correctness, goal_success,
     faithfulness, policy_compliance, … — see §2.5), and
   - optionally the **corrected expected output** and **expected labels**
     (category/priority for the support agent).
3. They mark the item **Completed**.

Those completed items are exactly the ground truth we want to harvest: (a) new
golden items, and (b) the judge-calibration set (note 04 §4a).

---

## 2. Implement — step by step

### Step 2.1 — Add `fetch_annotations()` to the store contract

`src/agent_evals/core/store.py`, on the `TraceStore` ABC (next to
`enqueue_annotation`):

```python
@abstractmethod
def fetch_annotations(
    self, queue_name: str, status: str = "COMPLETED", since: str | None = None
) -> list[dict]:
    """Drain reviewed items from an annotation queue. `since` is an optional
    watermark (completedAt/updatedAt) — items at or before it are skipped, so
    repeat runs fetch only newly reviewed traces (see §2.6). Returns one dict
    per completed item, including every human score dimension (§2.5):
        {trace_id, scores: {dim: value}, correct, expected_output,
         expected_labels, comment, completed_at}
    Best-effort like enqueue: an unsupported queue API returns []."""
```

Give it a default no-op on any store that doesn't implement it yet so the
harness still imports clean:

```python
def fetch_annotations(
    self, queue_name: str, status: str = "COMPLETED", since: str | None = None
) -> list[dict]:
    return []
```

### Step 2.2 — Implement it on `LangfuseClient`

`src/agent_evals/core/langfuse_client.py`. Mirror the low-level `api` access
already used by `enqueue_annotation`. Langfuse exposes queue items and the
annotations (scores + comments) a reviewer attached to each trace:

```python
def fetch_annotations(self, queue_name: str, status: str = "COMPLETED") -> list[dict]:
    """Drain reviewed items. Queue APIs vary by Langfuse version/tier, so this
    is best-effort: on any failure return [] rather than raising (same policy
    as enqueue_annotation)."""
    try:
        api = self._lf.api
        queues = api.annotation_queues.list_queues()
        queue = next(q for q in queues.data if q.name == queue_name)
        items = api.annotation_queues.list_queue_items(queue_id=queue.id)
    except Exception:
        return []

    out: list[dict] = []
    for item in items.data:
        if getattr(item, "status", None) != status:
            continue
        trace_id = item.object_id
        # A reviewer's labels arrive as trace-level scores/comments.
        annotations = api.annotation_queues.get_item_annotations(
            queue_id=queue.id, item_id=item.id
        ) if hasattr(api.annotation_queues, "get_item_annotations") else []
        labels, correct, comment = {}, None, ""
        for ann in getattr(annotations, "data", annotations) or []:
            name, value = ann.get("name"), ann.get("value")
            if name == "correct":
                correct = bool(value)
            elif name == "comment" or ann.get("comment"):
                comment = ann.get("comment") or comment
            else:
                labels[name] = value
        # The corrected output the reviewer typed (config field name may vary).
        raw = self.fetch_trace_raw(trace_id)
        expected_output = (raw["trace"].get("metadata") or {}).get(
            "reviewer_expected_output"
        ) or raw["trace"].get("output")
        out.append({
            "trace_id": trace_id,
            "correct": correct,
            "expected_output": expected_output,
            "expected_labels": labels,
            "comment": comment,
        })
    return out
```

> The exact attribute names (`list_queue_items`, `get_item_annotations`) differ
> across Langfuse SDK versions. Pin the SDK (note 09 §3) and confirm against
> your version; the `try/except → []` keeps a mismatch from breaking a run.

### Step 2.3 — Map annotations → `Case` (pure library code)

New file `src/agent_evals/pipelines/annotations.py`. Pure, testable, no I/O —
same discipline as `online.py`:

```python
"""Annotation-queue harvest (note 13): reviewed traces → golden Cases.
Pure mapping; the CLI owns fetching, redaction, and seeding."""

from __future__ import annotations

from agent_evals.core.schemas import Case


def annotations_to_cases(
    annotations: list[dict], *, keep_incorrect: bool = False
) -> list[Case]:
    """One completed annotation → one golden Case.

    By default only reviewer-confirmed-correct traces become golden items
    (their output is a trustworthy expected_output). Set keep_incorrect=True
    to also harvest failures as regression fixtures — but only when the
    reviewer supplied a corrected expected_output."""
    cases: list[Case] = []
    for a in annotations:
        correct = a.get("correct")
        expected = a.get("expected_output")
        if correct is False and not (keep_incorrect and expected is not None):
            continue
        cases.append(Case(
            case_id=f"annot-{a['trace_id']}",
            input=a.get("input"),  # filled from the trace when None (see CLI)
            expected_output=expected,
            expected_labels=a.get("expected_labels", {}),
            reference=a.get("comment") or None,
            metadata={
                "source_trace_id": a["trace_id"],
                "source": "annotation_queue",
                "reviewer_correct": correct,
            },
        ))
    return cases
```

### Step 2.4 — Add the `promote-annotations` CLI command

`src/agent_evals/cli.py`. It drains → maps → **redacts** → seeds a dataset,
reusing the exact redaction + `seed_dataset` path that `seed-dataset` already
uses. Register the subparser next to `seed-dataset`:

```python
promote_parser = sub.add_parser(
    "promote-annotations",
    help="drain a Langfuse annotation queue into a (versioned) golden dataset",
)
promote_parser.add_argument("--config", required=True)
promote_parser.add_argument("--queue", default=None,
                            help="queue name (default: <agent>-review)")
promote_parser.add_argument("--name", default=None,
                            help="target dataset name (default: config dataset)")
promote_parser.add_argument("--keep-incorrect", action="store_true",
                            help="also harvest reviewer-corrected failures")
promote_parser.add_argument("--dry-run", action="store_true",
                            help="print the cases; do not write to the store")
```

Handler (place with the other `if args.command == ...` blocks):

```python
if args.command == "promote-annotations":
    from agent_evals.core.redaction import redact
    from agent_evals.core.store import get_store
    from agent_evals.pipelines.annotations import annotations_to_cases

    store = get_store(cfg.trace_store)
    queue = args.queue or f"{cfg.agent}-review"
    annotations = store.fetch_annotations(queue)

    # backfill each case's input from its source trace
    for a in annotations:
        try:
            a["input"] = store.fetch_trace_raw(a["trace_id"])["trace"].get("input")
        except Exception:
            a["input"] = None

    cases = annotations_to_cases(annotations, keep_incorrect=args.keep_incorrect)
    if not cases:
        print(f"no promotable annotations in queue '{queue}'")
        return 0

    for case in cases:                       # redact BEFORE anything leaves
        case.input = redact(case.input)
        case.expected_output = redact(case.expected_output)

    name = args.name or cfg.dataset
    if args.dry_run:
        for c in cases:
            print(f"  {c.case_id}  labels={c.expected_labels}")
        print(f"[dry-run] {len(cases)} cases would seed '{name}'")
        return 0

    n = store.seed_dataset(name, cases)
    _writeback_human_scores(store, annotations, out_dir=args.out)  # §2.5
    store.flush()
    print(f"promoted {n} annotated cases from '{queue}' -> {cfg.trace_store} dataset '{name}'")
    return 0
```

`_writeback_human_scores` posts the per-dimension human scores back to each trace
and emits the calibration set; it's defined in §2.5 and called here so the handler
is complete. (`--dry-run` returns before both `seed_dataset` and the write-back.)

### Step 2.5 — Capture human evaluation across *dimensions*, not one boolean

The shape shown so far reduces a review to `correct: bool` + a labels bag — which
wastes what annotation queues are *for*. A Langfuse queue has a **score config**:
several named dimensions (categorical or numeric), and the reviewer scores the
trace on **each**. That's exactly "annotate a trace with scores across
dimensions," and every dimension name must already exist in
[metrics.md](../agent-evals/metrics.md) (the enforced registry — human and judge
scores share the namespace so they're comparable).

Configure the queue's dimensions to mirror the judge metrics you want to trust,
e.g. for the support agent:

| dimension | scale | what the reviewer judges |
|---|---|---|
| `goal_success` | 0–1 | did the agent accomplish the user's goal |
| `faithfulness` | 0–1 | grounded in tool evidence, no fabrication |
| `policy_compliance` | 0/1 | followed refund/PII policy |
| `label_match` | 0/1 | category/priority correct |
| `correct` | 0/1 | overall accept as golden |

**Generalize `fetch_annotations` to return every dimension** — replace the single
`correct` field with a `scores` map (keep `correct` as one recognized key). Which
names are classification *labels* vs. score *dimensions* comes from the config's
`label_match` evaluator (`fields: [category, priority]`), not a magic constant:

```python
label_fields = set(_label_match_fields(cfg))   # from the config's label_match spec
labels, scores, comment = {}, {}, ""
for ann in getattr(annotations, "data", annotations) or []:
    name, value = ann.get("name"), ann.get("value")
    if ann.get("comment"):
        comment = ann["comment"]
    if name in label_fields:
        labels[name] = value               # -> expected_labels (golden)
    elif value is not None:
        scores[name] = float(value)        # -> human score dimension
out.append({
    "trace_id": trace_id,
    "scores": scores,                      # {goal_success: 1.0, faithfulness: 0.0, ...}
    "correct": bool(scores.get("correct", 1.0)),
    "expected_output": expected_output,
    "expected_labels": labels,
    "comment": comment,
    "completed_at": completed_at,
})
```

The harvested dimensions feed **two** consumers, both inside the one
`_writeback_human_scores` helper the CLI handler (§2.4) calls after seeding:

```python
def _writeback_human_scores(store, annotations, out_dir):
    import json, os
    from agent_evals.core.schemas import Score

    # 1. Human record in Langfuse — one Score per dimension on the REAL trace,
    #    so the UI shows human vs. judge side by side.
    for a in annotations:
        for dim, val in a["scores"].items():
            store.post_score(Score(
                name=dim, value=val, trace_id=a["trace_id"], level="trace",
                comment=a["comment"],
                metadata={"source": "human_annotation", "source_trace_id": a["trace_id"]},
            ))
    # 2. Calibration set — one JSONL row per (trace, dimension) for `evals calibrate`.
    with open(os.path.join(out_dir, "human_labels.jsonl"), "w") as f:
        for a in annotations:
            for dim, val in a["scores"].items():
                f.write(json.dumps(
                    {"trace_id": a["trace_id"], "name": dim, "value": val}) + "\n")
```

(`post_score` routes on `source_trace_id`, so these land on the real production
trace, not a synthetic id — [langfuse_client.py:45](../agent-evals/src/agent_evals/core/langfuse_client.py#L45).)

**Calibrating a judge against these labels.** `evals calibrate` joins judge scores
to human labels **by `trace_id`** ([cli.py](../agent-evals/src/agent_evals/cli.py)
`_calibrate`) — so the judge scores must be on the *same production traces* the
humans reviewed. An offline golden run won't do: its scores are keyed by
deterministic runner ids that never overlap the production trace-ids. Get matching
judge scores by **replaying those exact traces through the judge**, then join:

```bash
# 1. Export the annotated production traces to adapter-shaped JSONL (their real IDs)
# 2. Re-score them with the full judge suite:
evals run --config configs/support_agent.yaml --mode replay \
  --traces runs/promote/annotated_traces.jsonl --out runs/calib
# 3. Judge vs. human, per dimension (target kappa > 0.8 before it may gate — §7):
evals calibrate --scores runs/calib/scores.jsonl \
  --labels runs/promote/human_labels.jsonl --metric faithfulness
evals calibrate ... --metric goal_success
```

> Alternatively the online pipeline already scored these traces in production;
> if you export those scores by `trace_id` you can skip the replay.

**Reviewer agreement.** When more than one reviewer labels the same trace, don't
average silently — compute inter-annotator agreement first (the same `cohens_kappa`
in `core/stats.py`). Low human-human agreement on a dimension means the *rubric* is
ambiguous; fix that before trusting either humans or the judge on it. Aggregate to
one value per (trace, dimension) only once agreement is acceptable.

So one review yields human evaluation **three** ways: dimension scores written back
to the trace (UI), the same scores exported for per-dimension judge calibration, and
the `correct` dimension + corrected output promoted as a golden `Case`. Add each
dimension to `metrics.md` in the same PR (governance rule), and never plot human and
judge series together without a marked epoch (metrics.md rules — judge scores are
judge-specific measurements).

### Step 2.6 — Fetch only *newly* reviewed items (incremental drain)

⚠️ The `fetch_annotations` above returns **every** `COMPLETED` item, and
`seed_dataset` (existing code) calls `create_dataset_item` with **no `id`** — so
running `promote-annotations` a second time would reprocess every annotation and
create **duplicate** dataset items. A completed item stays completed, so status
alone can't tell "new" from "already promoted." Two defenses, matching the repo's
existing idempotency idioms (the `ScoreCache`, note 08 cursors):

**(a) Idempotent seeding — the safety net.** Give each dataset item a
deterministic `id` derived from its source trace, so a re-seed is an *upsert*,
not a duplicate. Patch `LangfuseClient.seed_dataset`:

```python
self._lf.create_dataset_item(
    id=case.metadata.get("source_trace_id") and f"annot-{case.metadata['source_trace_id']}",
    dataset_name=name,
    input=case.input,
    expected_output=case.expected_output,
    metadata={...},   # unchanged
)
```

Langfuse upserts dataset items on `id`, so re-promoting the same (or a
re-labeled) annotation updates in place. This alone makes the pipeline safe to
re-run; it just still does redundant *work*.

**(b) A cursor — so you only *fetch* new work.** Persist a watermark between
runs and pass it to `fetch_annotations`. Langfuse queue items carry a
`completedAt`/`updatedAt` timestamp; keep the max seen so far in a tiny sqlite
(same shape as `ScoreCache`):

```python
def fetch_annotations(self, queue_name, status="COMPLETED", since=None):
    ...
    for item in items.data:
        if getattr(item, "status", None) != status:
            continue
        completed_at = getattr(item, "completed_at", None) or getattr(item, "updated_at", None)
        if since and completed_at and completed_at <= since:
            continue          # already promoted on a previous run
        ...
        out.append({..., "completed_at": completed_at})
    return out
```

CLI side — load the cursor before, advance it after a successful seed:

```python
import sqlite3, os
cur_db = os.path.join(args.out if hasattr(args, "out") else "runs", "annot_cursor.sqlite3")
conn = sqlite3.connect(cur_db)
conn.execute("CREATE TABLE IF NOT EXISTS cursor(queue TEXT PRIMARY KEY, ts TEXT)")
since = (conn.execute("SELECT ts FROM cursor WHERE queue=?", (queue,)).fetchone() or [None])[0]

annotations = store.fetch_annotations(queue, since=since)
...                                   # map, redact, seed as before
newest = max((a.get("completed_at") for a in annotations if a.get("completed_at")), default=since)
if newest and not args.dry_run:
    conn.execute("INSERT OR REPLACE INTO cursor(queue, ts) VALUES (?, ?)", (queue, newest))
    conn.commit()
```

Guidance: **ship (a) always** — it's the correctness guarantee and costs
nothing. Add **(b)** once the queue is large enough that re-scanning it every run
is wasteful. In CI, commit the cursor db to a cache/artifact (or store the
watermark as a Langfuse dataset-metadata field) so it survives across runs;
without persistence you fall back to "(a) makes re-scan harmless, just slower."
A re-opened/re-labeled item legitimately gets a newer `completedAt`, so the
cursor re-promotes it and (a) upserts it — the correction lands. ✅

### Step 2.7 — Version the dataset (don't overwrite)

`seed_dataset` extends an existing dataset. To keep runs comparable over time
(note 04 §4a step 4), promote into a **new version** name and repoint the config:

```bash
evals promote-annotations --config configs/support_agent.yaml \
  --name support-agent-golden-v2
# then bump `dataset:` in the config to -v2 in the same PR
```

---

## 3. Test — step by step (in the eval notes / repo)

The whole point of the store abstraction is that this is testable **without
Langfuse**. Use a fake store; assert on the mapping and the redaction ordering.

### Step 3.1 — Unit test the pure mapping

`tests/test_annotations.py`:

```python
from agent_evals.pipelines.annotations import annotations_to_cases


def _ann(**kw):
    base = {"trace_id": "t1", "correct": True, "expected_output": "route to billing",
            "expected_labels": {"category": "billing"}, "comment": "ok"}
    base.update(kw); return base


def test_correct_annotations_become_cases():
    cases = annotations_to_cases([_ann()])
    assert len(cases) == 1
    c = cases[0]
    assert c.case_id == "annot-t1"
    assert c.expected_labels == {"category": "billing"}
    assert c.metadata["source"] == "annotation_queue"


def test_incorrect_dropped_by_default():
    assert annotations_to_cases([_ann(correct=False)]) == []


def test_incorrect_kept_only_with_corrected_output():
    kept = annotations_to_cases(
        [_ann(correct=False, expected_output="corrected")], keep_incorrect=True)
    assert len(kept) == 1
    assert annotations_to_cases(
        [_ann(correct=False, expected_output=None)], keep_incorrect=True) == []
```

### Step 3.2 — Test drain + redaction with a fake store

Prove PII never survives promotion (the fail-condition of note 05 §2):

```python
from agent_evals.core.schemas import Case
from agent_evals.core.store import TraceStore, register_store


@register_store
class FakeStore(TraceStore):
    name = "fake"
    seeded: list[Case] = []

    def fetch_annotations(self, queue_name, status="COMPLETED", since=None):
        return [{"trace_id": "t1", "correct": True,
                 "scores": {"correct": 1.0, "faithfulness": 0.0},
                 "expected_output": "email me at jane@acme.com",
                 "expected_labels": {"category": "billing"}, "comment": "",
                 "completed_at": "2026-01-01T00:00:00Z"}]

    def post_score(self, score):            # capture the human write-back
        pass

    def fetch_trace_raw(self, trace_id):
        return {"trace": {"input": "call me at 415-555-1212"}, "observations": []}

    def seed_dataset(self, name, cases):
        FakeStore.seeded = cases; return len(cases)

    # remaining abstract methods: raise NotImplementedError / return None
```

Then run the CLI handler (`main(["promote-annotations", "--config", ...])`
against a config with `trace_store: fake`) and assert:

```python
assert "@" not in FakeStore.seeded[0].expected_output   # email redacted
assert "555" not in str(FakeStore.seeded[0].input)      # phone redacted
```

### Step 3.3 — Adapter conformance (once, per agent)

Before trusting `fetch_trace_raw` for a new agent, run the existing check
(from the template header):

```python
from agent_evals.testing import assert_adapter_conformance
assert_adapter_conformance("langfuse-generic")
```

### Step 3.4 — Run the suite

```bash
cd agent-evals
uv run pytest tests/test_annotations.py -q      # the new tests
uv run pytest -q                                # full suite must stay green
```

### Step 3.5 — Dry-run against real Langfuse (manual, once)

```bash
export LANGFUSE_PUBLIC_KEY=... LANGFUSE_SECRET_KEY=... LANGFUSE_HOST=...
evals promote-annotations --config configs/support_agent.yaml --dry-run
```

Confirm the printed cases have the reviewer's labels and no raw PII, then drop
`--dry-run`.

---

## 4. Run the evaluator in CI/CD

The evaluator already gates via `evals run --baselines` (regression gating,
[cli.py](../agent-evals/src/agent_evals/cli.py) → exit code 1 on `gate: FAILED`).
CI just needs to call it against the promoted dataset on every prompt/model/tool
change (note 04 §4b).

### 4.1 — Two jobs, two triggers

| Job | Trigger | Command | Gate |
|---|---|---|---|
| **promote** | manual / nightly cron | `evals promote-annotations` | grows the golden set |
| **eval-gate** | every PR touching agent/prompt/config | `evals run --baselines baselines/` | blocks merge on regression |

Keep promotion **out** of the PR gate — you don't want a PR's result to depend
on whoever labeled the queue that morning. Promote on a schedule, review the
dataset diff, then let PRs run against the frozen `-vN` dataset.

### 4.2 — GitHub Actions workflow

`agent-evals/.github/workflows/eval-gate.yml`:

```yaml
name: eval-gate
on:
  pull_request:
    paths:
      - "configs/**"
      - "src/agent_evals/**"
      - "agents/**"          # the agent's own prompt/tool code
  workflow_dispatch: {}

jobs:
  golden-eval:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    env:
      LANGFUSE_HOST:        ${{ secrets.LANGFUSE_HOST }}
      LANGFUSE_PUBLIC_KEY:  ${{ secrets.LANGFUSE_PUBLIC_KEY }}
      LANGFUSE_SECRET_KEY:  ${{ secrets.LANGFUSE_SECRET_KEY }}
      ANTHROPIC_API_KEY:    ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }          # baselines/ history for regression gate
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --extra langfuse

      # Fail closed: if the judge/store is unreachable, the gate fails (note 09 §6)
      - name: Run golden-set evaluation
        run: |
          uv run evals run \
            --config configs/support_agent.yaml \
            --baselines baselines/ \
            --out runs/ci-${{ github.run_id }}

      - name: Publish report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: eval-report-${{ github.run_id }}
          path: runs/ci-${{ github.run_id }}/
```

`evals run` returns **exit 1** when any threshold regresses vs. the baseline or a
`score_thresholds` fails, so the job — and the PR's required check — goes red
automatically. No extra gate logic in YAML.

### 4.3 — Promotion workflow (scheduled)

`agent-evals/.github/workflows/promote-annotations.yml`:

```yaml
name: promote-annotations
on:
  schedule: [{ cron: "0 6 * * 1" }]       # Mondays 06:00 UTC
  workflow_dispatch:
    inputs:
      dataset_name: { description: "target dataset (e.g. support-agent-golden-v3)", required: true }

jobs:
  promote:
    runs-on: ubuntu-latest
    env:
      LANGFUSE_HOST:       ${{ secrets.LANGFUSE_HOST }}
      LANGFUSE_PUBLIC_KEY: ${{ secrets.LANGFUSE_PUBLIC_KEY }}
      LANGFUSE_SECRET_KEY: ${{ secrets.LANGFUSE_SECRET_KEY }}
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --extra langfuse
      - name: Drain queue into new dataset version
        run: |
          uv run evals promote-annotations \
            --config configs/support_agent.yaml \
            --name "${{ inputs.dataset_name || 'support-agent-golden-nightly' }}"
      # Optional: open a PR bumping `dataset:` to the new version so the change
      # to what CI grades is itself reviewed.
```

### 4.4 — Promote a new baseline after an intentional improvement

When a PR *legitimately* moves a metric, refresh the baseline so the next PR
compares against the new bar:

```bash
evals run --config configs/support_agent.yaml --out runs/candidate
evals promote-baseline --config configs/support_agent.yaml --run runs/candidate
git add baselines/support-agent.json && git commit -m "bump eval baseline"
```

---

## 5. Progression *and* regression — both directions of the gate

The baseline comparison is a two-sided bootstrap CI, so both directions come from
the *same* number — but the harness only wires the regression side today
([baselines.py](../agent-evals/src/agent_evals/core/baselines.py) `compare_to_baseline`
flags `regression = hi < 0`). Close the loop by surfacing the symmetric verdict.

**Regression (covered).** `regression = hi < 0` — the CI for (current − baseline)
lies **entirely below zero**. `evals run --baselines` fails the gate; CI goes red.
This runs on **every** gated metric, every PR — the default safety net.

**Progression (add the twin verdict).** `improvement = lo > 0` — the CI lies
**entirely above zero**: a statistically significant *gain*, not point noise. One
line in `compare_to_baseline`:

```python
comparison[metric] = {
    "mean_diff": mean(cur) - mean(base),
    "ci": [lo, hi],
    "regression": hi < 0,
    "improvement": lo > 0,          # NEW: significant gain, not just "not worse"
    "baseline_run": baseline.get("run_id"),
}
```

and surface it in the CLI summary (today it prints only `REGRESSION`/`ok`, so a
real gain reads as "ok" — indistinguishable from no change):

```python
tag = ("REGRESSION" if cmp["regression"]
       else "IMPROVEMENT" if cmp["improvement"] else "no-change")
```

Now every run classifies each metric as **improved / regressed / flat** — three
outcomes, both tails tested.

**Two gate modes, one comparison:**

| Mode | Rule | Use |
|---|---|---|
| **Regression gate** (default) | fail if **any** metric `regression` | every PR — never ship a backslide |
| **Progression gate** (opt-in) | fail unless the **named** metric `improvement` | a fix/tuning PR must *prove* it moved its target metric up |

A PR titled "improve faithfulness" should have to demonstrate it — otherwise it's
an untested claim. Wire it as an opt-in flag, e.g.
`evals run --require-improvement faithfulness`, red unless that metric's CI clears
zero, while the regression gate still guards everything else.

**Progression must hold the dataset fixed.** `mean_diff` isolates the *agent* only
when the golden set is constant. Bumping to a new dataset version (§2.7) makes the
old baseline non-comparable — its metrics were measured on different items. So on
every version bump, **re-baseline on the new dataset before trusting a diff**
(run the *current* agent on `-v2`, `promote-baseline`, then future PRs compare on
`-v2`). Otherwise a "gain" may just be an easier dataset. This is the one coupling
between the flywheel (§2.7) and the gate.

**Progression over time (trend).** The baseline registry stamps `promoted_at` on
each promotion, so a metric's baseline history *is* the release-over-release
progression curve. Chart mean per promoted baseline to see whether the agent
trends up across releases — not just PR-vs-PR. (Never join across judge/rubric
epochs without a marker — metrics.md rules.)

---

## 6. Recap — the closed flywheel

1. Online pipeline / UI → `enqueue_annotation` → **queue**.
2. Reviewer labels correctness + corrected output → **Completed**.
3. `evals promote-annotations` → `fetch_annotations(since=cursor)` (only newly
   reviewed items) → `annotations_to_cases` → **redact** → `seed_dataset`
   (idempotent upsert on `annot-<trace_id>`) → **versioned Langfuse Dataset**.
4. CI `evals run --baselines` grades the agent on that dataset every PR;
   red on regression.
5. The **per-dimension** human scores (goal_success, faithfulness,
   policy_compliance, …) are written back as trace Scores *and* exported as a
   calibration set, so `evals calibrate --metric <dim>` validates each judge
   dimension (kappa > 0.8) before it's trusted to gate (note 04 §4a, §7).

**New code to write:** `fetch_annotations` (contract + Langfuse impl, multi-dimension
+ `since` cursor), an idempotent `id` in `seed_dataset`, `pipelines/annotations.py`,
the `promote-annotations` CLI command + `_writeback_human_scores` helper, the
`improvement` verdict + `--require-improvement` progression gate (§5),
`tests/test_annotations.py`, and two GitHub Actions workflows. Everything else
(redaction, `seed_dataset`, `evals run` regression gating, replay mode,
`evals calibrate`, baselines) already exists.
