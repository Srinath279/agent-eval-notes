# Eval Harness — Remaining Considerations (Edges of the System)

Final layer beyond [05-harness-gaps.md](05-harness-gaps.md) (correctness) and [06-production-scale.md](06-production-scale.md) (scale): the places where the harness meets the rest of the system — deployment, composed agents, runtime guardrails, data strategy, and governance evidence.

## A. Deployment integration (harness meets CD)

### 1. Eval gates wired into the deploy pipeline itself
The CI gate checks *code changes*; nothing yet connects eval runs to *releases*.
- Eval run as a required check in Cloud Deploy / GitHub environments.
- **Canary rollout**: new prompt/model gets a traffic slice; its online scores are compared to the incumbent before full promotion.
- **Auto-rollback triggers** tied to online score thresholds — score → deployment decision is currently a human step.

## B. Evaluating composed systems

### 2. Multi-agent / handoff evaluation
Once the support agent routes to sub-agents or escalates to humans:
- Routing-correctness metrics.
- **Step-level failure attribution** in chains — which component caused the miss.
- **Handoff-summary quality** — is the context passed to the human agent complete and accurate? A big real-world support metric.

### 3. RAG-layer metrics
Agents retrieve from a KB, but the harness has no retrieval evaluators:
- Context precision / recall (Ragas-style).
- **KB freshness checks** — a faithful answer grounded in a stale KB article is still wrong; no current metric catches it.

## C. Runtime quality layer

### 4. Guardrails vs. evals
The same policy-compliance / harmfulness logic should exist at two enforcement points:
- **Post-hoc scoring** (the harness) — measurement.
- **Inline pre-send guardrail** — blocks a bad reply before the customer sees it.
- Share rubric definitions between both so they never diverge.

## D. Coverage & data

### 5. Multilingual evaluation
If tickets arrive in multiple languages:
- Per-locale golden subsets.
- Judge accuracy validated *per language* — judges degrade non-uniformly across languages.

### 6. Synthetic case generation
Paraphrase / edge-case augmentation to densify the golden set beyond what production has surfaced yet.

## E. Closing the loop

### 7. The data flywheel
Scored + human-annotated traces are training data:
- Feed into fine-tuning sets or automated prompt optimization (DSPy-style).
- Upgrades the harness from a measurement tool to an **improvement engine** — the end-state purpose of the whole system.

## F. Business continuity & evidence

### 8. Backup / DR for Langfuse data
Golden datasets, annotations, and score history are business-critical assets — backup/restore and a recovery plan like any production database.

### 9. Audit-ready eval documentation
Model cards and eval-coverage reports per agent for AI-governance / regulatory needs (EU AI Act-style obligations for customer-facing AI).

### 10. Stakeholder scorecards + shared judge-quota management
- Weekly per-agent quality reports for non-engineers; exec-level trendlines.
- Priority queues when many agents' eval runs compete for the same Vertex quota.

---

## The checklist is now complete — the risk is execution

Notes 04–07 cover design comprehensively: core plan, correctness gaps, scale, and system edges. Adding more checklist items now has diminishing returns; the remaining risk is that nothing is built.

**Recommended path**: build the minimal vertical slice first —
schemas → one trace adapter → PII redaction → 3 evaluators (1 deterministic, 1 judge, 1 trajectory) → pass^k offline runner against a 20-item golden set in Langfuse — for **one agent**, end to end. Then layer in notes 05–07 by their priority orders as the system earns real usage.
