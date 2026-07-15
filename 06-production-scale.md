# Eval Harness — Production-Scale Requirements

[05-harness-gaps.md](05-harness-gaps.md) covered *correctness* of the harness. This note covers what changes when the harness runs at **production scale** — high trace volume, many agents, many teams, long time horizons.

## A. Throughput & cost at scale

### 1. Batch judge inference
At thousands of traces/day, per-trace judge calls are slow and expensive.
- Use **Vertex AI batch prediction** for offline/backfill scoring.
- Reserve real-time judge calls for the online sampled path only.

### 2. Langfuse at volume
- REST pagination won't survive high trace volume — use Langfuse's **bulk/blob export to GCS** as the offline data path.
- Size self-hosted ClickHouse / ingestion for peak trace rate.
- **Trace retention/TTL** with cold archive to GCS/BigQuery.

### 3. FinOps
- Per-agent eval-cost attribution dashboards.
- Unit-economics guardrail: eval spend stays under ~10% of the agent's own inference spend.

## B. The harness as a platform (multi-agent, multi-team)

### 4. Onboarding kit
- Template agent config.
- **Adapter conformance test**: "run these fixture checks against your traces; pass = onboardable."
- Self-serve docs. This is what makes agent #3 and #4 cheap to onboard.

### 5. Versioned harness releases
- Semver the `agent-evals` package; backwards-compat policy for score names.
- Defined **re-score / epoch strategy** when evaluator logic changes: either backfill history or mark a metric epoch so dashboards don't show fake regressions.

### 6. Eval-config as code with review gates
- Rubric, threshold, and sampling changes go through PRs with CODEOWNERS.
- A silent threshold edit is how quality gates rot.

## C. Operating the pipeline itself

### 7. SLOs + observability for the harness
- e.g., "95% of sampled traces scored within 15 minutes"; scoring-coverage %; judge error rate.
- The harness gets its own dashboards, alerts, and runbooks — monitored like any production service.

### 8. Graceful degradation
- Judge model down/rate-limited → deterministic metrics keep flowing; judge work queues and catches up.
- Gaps are *visible* (coverage metric drops), never silently missing scores.

### 9. Regression response playbook
- A score drop maps to a defined action: who's notified, rollback-vs-investigate thresholds, and how a prompt rollback is actually executed.

## D. Keeping the eval trustworthy over time

### 10. Golden-set hygiene at scale
- Dedup; difficulty stratification.
- **Coverage mapping**: does the golden set's topic distribution still match production tickets? Refresh quarterly.

### 11. Contamination control
- A **held-out set** never used during prompt iteration.
- If the team tunes prompts against the golden set, scores inflate while production quality doesn't.

### 12. Meta-evaluation cadence
- Scheduled judge re-calibration against fresh human labels (not just once at setup) — ticket mix and judge models drift.

### 13. Failure taxonomy + clustering
- Auto-cluster failing traces by root cause: wrong tool, bad retrieval, policy miss, hallucination.
- Turns a dropping score from *"something broke"* into *"here's what to fix"* — the piece that makes evals a debugging tool, not just a scoreboard.

## E. Security & compliance at scale

### 14. Audit + data governance
- Audit logs of who ran evals and who accessed traces.
- Data-residency review for judge calls; DPA coverage for the judge model.
- VPC Service Controls perimeter around Vertex + Langfuse if tickets are sensitive.

## Priority order for production hardening

Once the core harness (notes 04 + 05) works for one agent:
1. Harness SLOs + graceful degradation (C7, C8) — trust in the pipeline.
2. Batch judging + Langfuse bulk export (A1, A2) — survive volume.
3. Held-out set + golden-set coverage mapping (D10, D11) — trust in the numbers.
4. Failure clustering (D13) — make the numbers actionable.
5. Onboarding kit + versioned releases (B4, B5) — scale to more agents.
6. FinOps, audit, playbooks (A3, E14, C9) — organizational maturity.
