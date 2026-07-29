# ROADMAP.md
*So no session has to guess what's done, in progress, or next.*

Last updated: 2026-07-29 — project kickoff, no code written yet.

## Done
*(nothing yet — repo not initialized)*

## In Progress
- Phase 0: Foundations — repo skeleton + project brain being set up.

## Upcoming — full phase list

### Phase 0 — Foundations
Core data model (`decisions`, `decision_evidence`, `outbox_events`), Document
Agent, Temporal Agent, sync MVP via `POST /v1/decide`, audit-before-response
transactional outbox pattern.
**Exit criterion:** a real application can be decisioned end-to-end with a
persisted, citable audit trail.

### Phase 1 — Full decision pipeline
Sanctions Agent, Transaction Agent, RAG Agent (FAISS v1), Decision Agent
(LLM synthesis with citation enforcement).
**Exit criterion:** decisions are LLM-synthesized and every claim traces to
a `decision_evidence` row.

### Phase 2 — Human review & calibration
Temporal `review_workflow`, calibration engine, observability
(Prometheus/Grafana/OTel).
**Exit criterion:** low-confidence decisions route to a human reviewer, and
the calibration engine tracks how often the model's confidence matches
reality.

### Phase 3 — Regulatory Drift Loop (detect & alert)
Sentinel, Diff Engine, Impact Resolver, Affected Decision Set (ADS)
generation.
**Exit criterion:** a changed RBI circular is detected and correctly
resolved to the exact set of past decisions that cited it — no false
positives/negatives in the golden regression set.

### Phase 4 — Auto-recompute, human-confirms
Recompute Orchestrator, canary execution, `recompute_workflow`.
**Exit criterion:** affected decisions are automatically recomputed and
queued for human confirmation, without a full pipeline re-run.

### Phase 5 — Scale hardening
Multi-tenant support, load/chaos testing, DR drills, real KMS signing
pipeline, move to real cloud infra (Kubernetes, managed Kafka/Temporal,
multi-AZ Postgres).
**Note:** this phase's operational maturity items (24/7 on-call, true
multi-AZ failover, chaos testing at 10x peak) are not fully achievable solo
— see `DECISIONS.md` and `PROJECT.md` for what's explicitly deferred and why.

### Phase 6 — Full closed-loop autonomy
Fully automated confirm-and-close for recomputed decisions.
**Note:** the code for this phase is buildable; its exit criterion (a
sustained, measured low false-flip rate over a real observation period)
requires real production traffic and can't be proven solo in dev. Build it,
but don't declare it "done" until there's real usage data.

---

## How to update this file
At the end of every phase (not every session), move completed items from
"In Progress" to "Done" and pull the next phase's items into "In Progress."
Don't mark something "Done" until its exit criterion is actually met and
tested — see `SESSION_LOG.md` for session-by-session granularity instead.