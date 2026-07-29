# PROJECT.md
*Read this first, every session. This file changes rarely.*

## Project Name
ComplianceLoop

## Goal
An AI-powered compliance decisioning platform for NBFCs (Non-Banking Financial
Companies) in India. Given a loan/onboarding application, it produces a
compliant, auditable, evidence-cited decision — and keeps that decision
correct over time as RBI regulations change, without waiting for a human to
notice the regulation moved.

## Who this is for
NBFC compliance/risk teams who currently do this manually and cannot prove,
after the fact, exactly which regulation and which document justified a
decision made six months ago.

## Non-negotiable principles (never compromise these, regardless of phase or infra)
1. **Audit-before-response.** A decision is never returned to a caller before
   the decision + its evidence + its outbox event are durably committed in one
   transaction. No exceptions, no "we'll backfill the audit trail later."
2. **Evidence citation, not vibes.** Every decision must cite the specific
   document, clause, or data point that justified it. An LLM saying "this
   looks compliant" with no citation is not a valid decision.
3. **Deterministic, replayable pipeline.** Given the same inputs and the same
   ruleset version, the same decision must be reproducible. Agents are not
   free to be creatively inconsistent.
4. **Human-in-the-loop until earned otherwise.** Full autonomy (Phase 6) is
   only appropriate after a sustained, measured low false-flip rate — not
   assumed on day one.
5. **Regulatory drift is a first-class citizen, not an afterthought.** The
   system must detect when RBI circulars change and know which past decisions
   are now potentially stale (Loop 2), not just decide well on day one.

## Two loops, one system
- **Loop 1 (hot path):** applicant comes in → multi-agent decisioning →
  auditable decision out.
- **Loop 2 (cold path):** regulation changes → system detects drift → figures
  out blast radius → recomputes affected decisions → human confirms.

## Core objectives
- Document ingestion & KYC validation
- RBI circular understanding (regulatory RAG)
- Multi-agent reasoning (Document, Temporal, Sanctions, Transaction, RAG,
  Decision agents)
- Risk scoring & decisioning
- Immutable, citable audit trail
- Human approval / review workflow
- Regulatory drift detection and selective recompute

## Tech stack (actual, not aspirational)

| Layer | Technology | Notes |
|---|---|---|
| API | FastAPI (async) | |
| Orchestration | LangGraph | agent DAG |
| Durable workflows | Temporal (self-hosted via Docker locally) | review + recompute workflows |
| Primary DB | Postgres | source of truth, outbox pattern |
| Vector search | FAISS (local) | swap to pgvector/managed later if needed |
| Streaming | Redpanda (Kafka-API compatible) | CDC via Debezium |
| Cache | Redis | |
| Object storage | MinIO (S3-compatible) | swap to real S3 on deploy |
| LLM | Groq API | free/low-cost inference — see DECISIONS.md for why this replaced Anthropic |
| Embeddings | Sentence Transformers, `BAAI/bge-small-en-v1.5`, local | runs on CPU, no API key, no cost — see DECISIONS.md |
| Frontend | React | not built in Phase 0 |
| Containerization | Docker / Docker Compose | Kubernetes deferred to Phase 5 |
| Signing | Local ED25519 keypair (dev) → Cloud KMS (deploy) | |

## Solo/free-tier context
Built solo, on free-tier cloud credits, primarily in Cursor. LLM inference
via Groq and embeddings via a local Sentence Transformers model were chosen
specifically to keep ongoing running cost near zero — see `DECISIONS.md`.
See
`ARCHITECTURE.md` for which managed services are substituted with
self-hosted equivalents during local development, and `DECISIONS.md` for the
reasoning behind each substitution.

## Source of truth
The full system specification lives in
`docs/ComplianceLoop_Production_Blueprint_v2.md`. This file and the rest of
the project brain summarize and index it — when in doubt, the blueprint wins.