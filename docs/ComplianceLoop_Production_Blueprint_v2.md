# ComplianceLoop — Production Implementation Blueprint
### An AI-Native Compliance Decisioning Operating System for NBFC Lending (RBI + DPDP era)
 
**Document type:** Architecture & implementation blueprint
**Audience:** Founding engineering team, compliance/legal stakeholders, infra/SRE
**Status:** v1.0 — ready for phased build
 
---
 
## 0. How to read this document
 
This is written as a build blueprint, not a pitch deck. Every section either (a) specifies a concrete technology and *why it was chosen over alternatives*, or (b) specifies a data model / algorithm precisely enough that an engineer could implement it without further clarification. Where regulatory specifics (exact RBI circular numbers, exact DPDP Rule clauses) are referenced, they are referenced **structurally** (i.e., "the KYC Master Direction," "the Digital Lending Guidelines," "DPDP notice-and-consent obligations") rather than by clause number — your compliance/legal function must bind the system to the *current, dated* text of each instrument, because regulatory text changes (which is precisely the problem this system is built to survive). Treat every "RBI-XXX" or "clause 7.2(b)" reference below as an *illustrative placeholder* for real citation IDs your system will mint at ingestion time.
 
Two loops are described:
 
- **Loop 1 — Decision & Review Loop** (your existing design): real-time multi-agent decisioning + human-in-the-loop review + calibration.
- **Loop 2 — Regulatory Drift & Selective Recompute Loop** (the new capability you asked for): detects regulatory change, computes exactly which past decisions are affected, and selectively re-runs only those, under audit.
Everything downstream (data model, tech stack, latency, scaling) is designed so these two loops **never contend for the same resources** — this is the single most important architectural decision in this document, and it's called out repeatedly because it's the thing most teams get wrong (a regulatory re-run storm should never be able to slow down a live loan decision).
 
---
 
## 1. Design principles (non-negotiable)
 
1. **Audit-before-response.** No decision is returned to a caller until it is durably persisted. Latency budget accounts for this; it is not an afterthought bolted on later.
2. **Evidence, not vibes.** Every sub-verdict an agent produces must cite the specific piece of evidence (document, clause ID, rule version, list version) it used. This is what makes Loop 2 possible — you cannot selectively re-run "only affected users" unless you know, per decision, precisely which regulatory atoms were load-bearing in that decision.
3. **Hot path and cold path are physically isolated.** Real-time decisioning (Loop 1) and regulatory recompute (Loop 2) run on separate compute pools, separate queues, separate rate budgets. A 50,000-case recompute triggered by one circular must never add a millisecond of p99 latency to a live application.
4. **Determinism over cleverness.** The pipeline graph (LangGraph) is a fixed, versioned DAG. LLMs are used *inside* nodes for judgment tasks (classification, synthesis, rationale) but the *routing* is deterministic code, not an LLM deciding what to do next. This is what makes the system auditable and testable.
5. **No silent recomputation.** Every re-run triggered by Loop 2 is traceable to a specific regulatory change event, and every case selected for re-run is traceable to a specific query over stored evidence — never a blanket "re-run everyone."
6. **Confidence is calibrated, not asserted.** The number the Decision Agent outputs is only useful if it's periodically checked against reviewer overrides and adjusted — this is Loop 1's calibration engine, extended in Loop 2 to learn from regulation-triggered flips too.
---
 
## 2. System overview
 
```
                              ┌─────────────────────────────────────────────┐
                              │                CLIENTS / LOS                │
                              │   (NBFC loan origination system, portal)    │
                              └───────────────────────┬───────────────────────┘
                                                       │ POST /v1/decide  (sync, p99 target < 6s)
                                                       ▼
 ┌─────────────────────────── HOT PATH  (real-time, autoscaled, isolated pool) ───────────────────────────┐
 │                                                                                                          │
 │   FastAPI Gateway ──► LangGraph Orchestrator (deterministic DAG)                                        │
 │                          │                                                                              │
 │        ┌─────────────────┼──────────────┬───────────────┬────────────────┐                             │
 │        ▼                 ▼              ▼               ▼                ▼                             │
 │   Document Agent   Sanctions Agent  Temporal Agent  Transaction Agent   RAG Agent                       │
 │   (KYC artifacts)  (PAN/watchlist)  (validity/exp.) (FOIR/feasibility) (RBI corpus, FAISS)               │
 │        └─────────────────┴──────────────┴───────────────┴────────────────┘                             │
 │                                        │  (fan-in, all evidence + citations)                            │
 │                                        ▼                                                                │
 │                              Decision Agent (LLM synthesis)                                             │
 │                          → verdict (APPROVE/REVIEW/REJECT) + confidence + cited evidence                │
 │                                        │                                                                │
 │                                        ▼                                                                │
 │                    Transactional Outbox Write (Postgres: decision + evidence + outbox row, 1 txn)        │
 │                                        │ (commit = response gate)                                       │
 │                                        ▼                                                                │
 │                                   HTTP 200 response                                                     │
 └───────────────────────────────────────┬─────────────────────────────────────────────────────────────────┘
                                          │ CDC (Debezium) reads outbox → Kafka
                                          ▼
                     ┌────────────────────────────────────────────────────────┐
                     │                     KAFKA BACKBONE                      │
                     │  topics: decisions.made, decisions.reviewed,            │
                     │          regulatory.changed, recompute.requested,       │
                     │          recompute.completed                            │
                     └───┬───────────────┬───────────────────┬────────────────┘
                         ▼               ▼                   ▼
              Human Review Service   Calibration Engine   Observability Pipeline
              (Loop 1 close)         (Loop 1 close)        (OTel/Prometheus/Loki)
 
 ┌───────────────────────── COLD PATH (Loop 2, rate-limited, separate pool) ─────────────────────────┐
 │                                                                                                     │
 │  Sentinel Agent  ──►  Diff & Change-Classification Engine  ──►  Regulatory Change Event (RCE)       │
 │  (polls RBI site,        (chunk-level semantic diff,              │                                 │
 │   sanctions lists,        severity + topic tagging)                ▼                                │
 │   DPDP notifications)                                     Impact Resolver                            │
 │                                                            (queries Decision Evidence Store,          │
 │                                                             applies per-topic recompute policy)       │
 │                                                                     │                                 │
 │                                                                     ▼                                 │
 │                                                     Affected Decision Set (ADS) — signed manifest     │
 │                                                                     │                                 │
 │                                                                     ▼                                 │
 │                                              Recompute Orchestrator (Temporal workflows,               │
 │                                              throttled, canary-first, idempotent per (decision,RCE))  │
 │                                                                     │                                 │
 │                                          ┌──────────────────────────┴───────────────┐                │
 │                                          ▼                                          ▼                │
 │                              Confirmed (no material change)              Flipped verdict / conf. Δ   │
 │                              → logged, closed                             → REVIEW queue + notify     │
 │                                                                                                        │
 │  Meta-Audit Ledger: every step above is hash-chained + KMS-signed, independent of Loop-1 audit table   │
 └────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```
 
---
 
## 3. Loop 1 — the real-time decision pipeline
 
### 3.1 Agent specification
 
| Agent | Responsibility | Inputs | Output (must include) | Latency budget (p95) | Failure mode |
|---|---|---|---|---|---|
| **Document Agent** | Validates presence, format, and internal consistency of mandatory KYC artifacts (PAN, address proof, income proof, entity docs for non-individuals) | Uploaded docs + KYC checklist ruleset version | pass/fail per artifact, missing-artifact list, `ruleset_version` used | 300 ms | Missing doc → routes straight to REJECT-pending-resubmission, never REVIEW |
| **Sanctions Agent** | Screens applicant identifiers (PAN, name, DOB, entity CIN) against watchlists (RBI defaulter lists, UN/MHA lists, internal negative list) | Applicant identifiers + `sanctions_list_version` | match/no-match, match confidence, `list_version` cited | 200 ms | Any ambiguous fuzzy match → forced REVIEW, never auto-approve |
| **Temporal Agent** | Validity windows: document expiry, KYC re-verification due dates, offer validity | Document metadata, today's date | expired/valid flags, days-to-expiry | 50 ms | Pure rule evaluation, no external I/O — cheapest, fastest agent |
| **Transaction Agent** | FOIR (Fixed Obligation to Income Ratio) and repayment feasibility | Bureau data, declared income, existing obligations, `foir_ruleset_version` | FOIR value, feasibility verdict, threshold used | 400 ms | Bureau timeout → REVIEW with reason "bureau unavailable," never silently skip the check |
| **RAG Agent** | Retrieves the specific RBI/DPDP clauses relevant to this application's risk factors (loan type, digital lending flag, data-sharing flag) | Application risk tags, FAISS index (`corpus_index_version`) | Top-k clauses **with clause IDs**, relevance scores | 500 ms | Index unavailable → falls back to last-known-good index (see §7.4), never fails open with no regulatory context |
| **Decision Agent** | Synthesizes all signals into verdict + confidence + human-readable rationale that **explicitly cites** the clause IDs / rule versions it relied on | All agent outputs | `verdict`, `confidence (0–1)`, `rationale`, `cited_evidence[]` (structured, not free text) | 2000 ms | Any agent error/timeout upstream → Decision Agent is *forced* into REVIEW, confidence capped, never allowed to APPROVE/REJECT on partial evidence |
 
**Fan-out/fan-in in LangGraph:** Document, Sanctions, Temporal, Transaction, and RAG agents have no dependency on each other and run as **parallel branches** in the graph; the Decision Agent node has an explicit `join` dependency on all five. This turns the pipeline's latency from *sum of five agents* into *max of five agents* — the single largest lever on hot-path latency.
 
**Why the citation requirement matters beyond explainability:** the Decision Agent's output schema (enforced via structured/tool-call output, not free text) forces it to emit `cited_evidence: [{type: "clause"|"rule"|"list", id, version}]`. This is validated by code before the decision is persisted — the set of cited IDs must be a subset of the IDs actually retrieved by upstream agents (prevents the LLM from hallucinating a citation). This same array is what Loop 2 indexes on to find affected decisions later — it is not extra plumbing for explainability alone, it is the load-bearing data structure for the whole regulatory-drift capability.
 
### 3.2 Audit-before-response (transactional outbox pattern)
 
The naive approach — write to Postgres, then publish to Kafka — has a well-known failure mode: the DB commit succeeds but the Kafka publish fails (or vice versa), leaving the system in an inconsistent state exactly when you need consistency most (an audit trail). ComplianceLoop instead:
 
1. Writes `decisions`, `decision_evidence`, and a row in an `outbox_events` table **in one Postgres transaction**.
2. Returns HTTP 200 only after that transaction commits (this is the audit-before-response guarantee — the record exists even if every downstream service is down).
3. **Debezium** (CDC) tails the Postgres WAL and publishes `outbox_events` rows to Kafka reliably, decoupled from the request path — this is what feeds the human-review service, calibration engine, and observability pipeline, none of which can slow down the hot path because they're not in it.
### 3.3 Human review loop (existing, formalized as a durable workflow)
 
Each application that lands in REVIEW is modeled as a **Temporal workflow execution**, not a row with a status column that a cron job polls. Reasons:
- Temporal gives you durable "wait for human input" semantics with timeouts/escalation for free (e.g., auto-escalate to a senior reviewer after 4 business hours).
- Every state transition is automatically part of Temporal's own replayable history — a second, independent audit trail for free.
- The same workflow engine is reused for Loop 2's recompute jobs (see §4.5), so you only operate one workflow system, not two.
### 3.4 Calibration engine
 
A scheduled job (not part of the hot path) that:
- Pulls `(decision.confidence, decision.verdict, reviewer.final_outcome)` tuples from the last N days.
- Computes calibration error (e.g., Expected Calibration Error / reliability diagrams) per verdict bucket and per risk segment.
- Adjusts **threshold configuration** (the confidence cutoffs that route to REVIEW vs auto-decide) stored in a config service (not code) — a threshold change is a config deploy, not a redeploy.
- Every threshold change is itself audit-logged (old value, new value, triggering evidence, approver) — thresholds are a compliance-sensitive control surface and are treated with the same rigor as the regulatory corpus.
---
 
## 4. Loop 2 — Regulatory Drift & Selective Recompute Loop (the new capability)
 
This is the mechanism you described: an agent that watches for guideline changes, figures out exactly what changed, and re-runs *only* the decisions that are actually affected — under its own audit trail. Below is a complete design.
 
### 4.1 Design goals
 
- **Precision over recall in blast radius** — better to under-select slightly (with a documented, auditable policy for why) than to trigger a mass re-run that swamps reviewers and cold-path infra.
- **No hot-path contention**, ever — enforced architecturally (separate pools), not by convention.
- **Full non-repudiation** — a regulator asking "why was this closed case reopened" gets a complete, signed answer chain: change → evidence query → selection → recompute → outcome.
- **Graduated trust** — the loop ships in "detect and alert" mode first, then "auto-recompute, human-confirm flips" mode, then (only once proven) "fully automated confirm-if-unchanged."
### 4.2 Component breakdown
 
**a) Sentinel Agent (change ingestion)**
- Scheduled + event-driven ingestion of: RBI Master Directions/circulars/notifications, DPDP Rules and MeitY clarifications, sanctions/watchlist updates, internal policy documents.
- Each fetched artifact is content-hashed (SHA-256). A new **version** is only created when the hash differs from the last stored version for that source — this alone eliminates most false-positive "changes" (re-publication of identical PDFs, etc.).
- Raw artifacts land in versioned, immutable object storage (S3 with Object Lock / WORM); metadata (source, version, hash, fetched_at, effective_date) lands in Postgres.
**b) Diff & Change-Classification Engine**
- Both the previous and new version of a document are chunked identically (same chunking strategy used for the FAISS corpus) so chunks are comparable 1:1 where possible.
- For each chunk pairing: compute cosine similarity between old and new embeddings.
  - similarity ≥ 0.98 → **unchanged**, no event.
  - 0.92 ≤ similarity < 0.98 → **EDITORIAL** (wording/formatting only) — logged, no recompute triggered by default.
  - similarity < 0.92 → **MODIFIED**, escalate to an LLM classifier.
- Chunks present only in the new version → **ADDED**; chunks present only in the old version → **REMOVED**.
- The LLM classifier (structured output) is prompted to classify MODIFIED chunks into `{severity: MAJOR|MINOR, topic_tags: [KYC, FOIR, SANCTIONS, CONSENT, RETENTION, DIGITAL_LENDING, ...], plain_english_summary}`. It does **not** decide whether to trigger a recompute — that's a deterministic downstream policy lookup, not an LLM decision (principle #4).
- Output: a **Regulatory Change Event (RCE)** — `{change_id, source_doc_id, old_clause_id, new_clause_id, change_type, severity, topic_tags, effective_date, diff_text}` — written to Postgres and published to `regulatory.changed` on Kafka.
**c) Impact Resolver ("blast radius calculator")**
This is the piece that makes "re-run only affected users" possible, and it depends entirely on the evidence-citation discipline from §3.1. On receiving an RCE:
 
1. Look up the **Recompute Policy** for the RCE's topic tags (see table below).
2. Query the Decision Evidence Store for every decision that **cited** (not merely retrieved) the changed clause/rule/list version, filtered by the policy's applicability scope and lookback window.
3. Produce an **Affected Decision Set (ADS)**: a signed, immutable manifest — `{change_id, generated_at, policy_applied, decision_ids[], count}` — stored before any recompute work begins. This manifest *is* the audit answer to "why did you touch this case."
Example query (illustrative schema, detailed in §5):
 
```sql
SELECT DISTINCT d.decision_id, d.application_id, d.applicant_id,
                d.decision_status, d.decided_at, d.account_state
FROM decision_evidence de
JOIN decisions d ON d.decision_id = de.decision_id
WHERE de.evidence_type = 'clause'
  AND de.evidence_id = 'RBI-MD-NBFC-2026-04:Clause-7.2(b)'
  AND de.evidence_role = 'CITED'                     -- not just retrieved-and-discarded
  AND d.decision_status IN ('APPROVED','REJECTED')   -- already-reviewed cases are out of scope
  AND d.account_state = ANY (:applicability_scope)    -- e.g. {'ACTIVE','PENDING_DISBURSEMENT'}
  AND d.decided_at >= :lookback_cutoff
ORDER BY d.decided_at DESC;
```
 
**Recompute policy table (per topic tag) — this is where business/legal judgment is encoded, explicitly, as data:**
 
| Topic tag | Applicability scope | Lookback window | Trigger severity | Rationale |
|---|---|---|---|---|
| SANCTIONS | ALL active accounts, incl. disbursed | Unlimited (ongoing screening obligation) | MAJOR + MINOR | Sanctions/AML screening is a continuous obligation, not a point-in-time check |
| KYC / DOCUMENT | Open/pending decisions only | 180 days | MAJOR only | Re-underwriting a closed, repaid loan on a document-format change has no legal or business value |
| FOIR / TRANSACTION | Open/pending + active loans within lookback | 365 days | MAJOR only | Threshold changes can matter to live exposure, not to closed accounts |
| CONSENT / RETENTION (DPDP) | All accounts with active data holding | Unlimited while data is retained | MAJOR + MINOR | Data-handling obligations persist as long as data is held, independent of loan status |
| DIGITAL_LENDING / GOVERNANCE | Open/pending decisions only | 90 days | MAJOR only | Governance/process clauses affect future decisioning posture more than past outcomes |
 
This table is itself version-controlled and change-audited (it is compliance-owned configuration, reviewed with legal, not a hardcoded constant).
 
**d) Recompute Orchestrator**
- Each entry in the ADS becomes a **Temporal workflow** with idempotency key `(decision_id, change_id)` — Temporal guarantees this workflow runs to completion exactly once even under retries/crashes.
- **Selective agent replay, not full pipeline replay**: the orchestrator inspects which agent(s) actually depend on the changed evidence (from `decision_evidence.agent_name`) and re-runs *only* those agent(s) against the *original, immutable* application inputs (pulled from the Application Evidence Store, never re-fetched from mutable external sources unless that specific source is what changed). Other agents' outputs are replayed from stored provenance. This is a major latency/cost optimization: a FOIR-threshold change re-runs only the Transaction Agent and Decision Agent, not Document/Sanctions/Temporal.
- **Canary-first execution**: the orchestrator processes a random 1–2% sample of the ADS first, holds it for a compliance/reviewer spot-check, and only proceeds to the full batch once the sample's flip-rate looks sane (guards against a bad diff classification triggering thousands of spurious flips).
- **Throttled, cold-path-isolated worker pool**: recompute jobs run at a configurable fraction of hot-path capacity (e.g., never more than 10% of total decision-engine throughput), enforced via a token bucket, on a physically separate Kubernetes node pool with its own priority class — a recompute backlog can grow long, but it can never starve live traffic.
**e) Outcome handling**
- New sub-verdict vs. stored old sub-verdict:
  - **Same verdict, confidence delta below threshold** → logged as "confirmed, no material change," workflow closes automatically, no reviewer involved.
  - **Verdict changes, or confidence crosses a re-review threshold** → routed into the same REVIEW workflow type as Loop 1, tagged `trigger_source = regulatory_change`, with an auto-generated diff explanation ("This case moved from APPROVE to REVIEW because RBI clause 7.2(b) changed on 2026-04-17; the new clause requires an additional income-verification artifact this application does not have on file.") Reviewer sees old vs. new evidence side by side.
  - If the applicant/customer must be notified per your compliance posture on automated-decision changes affecting them, a notification event is emitted on `recompute.completed` for the notification service to handle — this is a business/legal decision to configure per topic tag, not hardcoded.
**f) Meta-audit (audit of the audit loop)**
Every step above — fetch, diff, classify, policy lookup, ADS generation, each workflow's execution, each outcome — is written to an **independent, hash-chained, KMS-signed ledger** (see §9.2), separate from (but cross-referenced with) the Loop-1 decision audit trail. A regulator query "why was this closed case reopened on this date" is answered by walking exactly one chain: RCE → ADS membership proof → workflow execution record → outcome.
 
### 4.3 Why this is feasible at NBFC scale (worked example)
 
Say a Master Direction amendment changes an income-verification requirement (topic tag: KYC/DOCUMENT). Sentinel detects the change within hours of publication (poll interval configurable, default hourly for RBI/sanctions sources, daily for DPDP/MeitY sources — sanctions sources may also support push/webhook where available). Diff engine classifies it MAJOR, topic KYC. Impact Resolver applies the KYC policy: open/pending accounts only, 180-day lookback. For a mid-size NBFC processing ~100k applications/month, this typically narrows a "changed clause" event down from hundreds of thousands of historical decisions to a few hundred or low thousands of genuinely open, still-relevant cases — the actual recompute workload is small, bursty, and cheap, which is exactly why the selective-targeting step is the core innovation here rather than an optimization detail.
 
### 4.4 Interaction with calibration
 
Reviewer outcomes on regulation-triggered flips feed the same calibration engine as Loop 1, tagged by `trigger_source`. Over time this lets the calibration engine learn, e.g., "MINOR severity FOIR clarifications historically flip verdicts <2% of the time" — informing whether MINOR severity should ever auto-trigger recompute for that topic, which is itself a governance decision reviewed periodically, not silently self-tuned.
 
---
 
## 5. Data model (core tables)
 
```
regulatory_documents(doc_id, source, version, content_hash, s3_uri, fetched_at, effective_date)
regulatory_clauses(clause_id, doc_id, doc_version, chunk_text, embedding_ref, topic_tags[], superseded_by)
 
sanctions_lists(list_id, source, version, s3_uri, effective_date, content_hash)
 
decisions(decision_id, application_id, applicant_id, verdict, confidence,
          decided_at, account_state, ruleset_versions jsonb, pipeline_version)
 
decision_evidence(decision_id, agent_name, evidence_type,      -- 'clause'|'rule'|'list'|'document'
                   evidence_id, evidence_version, evidence_role, -- 'RETRIEVED'|'CITED'
                   created_at)
   -- indexed on (evidence_type, evidence_id, evidence_role) for Impact Resolver queries
   -- indexed on (decision_id) for per-decision provenance lookups
 
outbox_events(event_id, aggregate_type, aggregate_id, payload jsonb, created_at, published_at)
 
regulatory_change_events(change_id, source_doc_id, old_clause_id, new_clause_id,
                          change_type, severity, topic_tags[], effective_date,
                          diff_text, detected_at)
 
recompute_policies(topic_tag, applicability_scope[], lookback_days, trigger_severity[],
                    version, approved_by, approved_at)
 
affected_decision_sets(ads_id, change_id, policy_version, generated_at,
                        decision_ids[], count, manifest_hash, kms_signature)
 
recompute_workflows(workflow_id, decision_id, change_id, status,
                     old_verdict, new_verdict, old_confidence, new_confidence,
                     outcome, started_at, completed_at)
 
audit_ledger(seq_no, entity_type, entity_id, event_payload_hash, prev_hash, this_hash,
             kms_signature, written_at)   -- append-only, hash-chained, one per Loop-1 decision
meta_audit_ledger(seq_no, ..., same shape ...)  -- append-only, hash-chained, for Loop 2 steps
```
 
The single most important index for the whole regulatory-drift capability is:
`CREATE INDEX ON decision_evidence (evidence_type, evidence_id, evidence_role) INCLUDE (decision_id);`
— this turns "which decisions used this exact clause version" into an O(log n) lookup instead of a table scan, which matters once you have tens of millions of historical decisions.
 
---
 
## 6. Full technology stack
 
| Layer | Technology | Why this, specifically |
|---|---|---|
| Agent orchestration | **LangGraph** | Deterministic DAG with explicit parallel fan-out/fan-in; graph is versioned and testable independent of the LLM calls inside nodes |
| Long-running/durable workflows | **Temporal** (self-hosted cluster or Temporal Cloud) | Human-in-loop waits, retries, timeouts, and replay history come free; reused for both Loop 1 review and Loop 2 recompute so you operate one workflow engine, not two |
| LLM reasoning | **Claude Sonnet 5** for decision synthesis & change classification; **Claude Haiku 4.5** for cheap high-volume tasks (document completeness pre-checks, simple screening) | Structured/tool-call output enforces the citation schema; prompt caching on the (large, mostly-static) regulatory system context cuts both cost and latency on repeated calls |
| Embeddings | **Sentence-transformers** (e.g., a strong open-weight bi-encoder such as BGE/E5-class model) | Runs on owned infra (cost-predictable at high volume), swappable without vendor lock-in |
| Vector store | **FAISS**, sharded by regulatory domain (lending / DPDP / sanctions-notes), served behind a retrieval microservice | Matches your stated design; sharding + a routing layer solves FAISS's single-node/in-memory scaling limit; double-buffered index versions enable the zero-downtime swap you described |
| Primary datastore | **PostgreSQL** (managed, e.g. Aurora/Cloud SQL, with read replicas) | ACID transactions are required for the audit-before-response guarantee; mature JSONB + GIN indexing handles the evidence tables well |
| Change data capture | **Debezium** on Postgres WAL → Kafka | Solves the dual-write problem (outbox pattern) reliably |
| Event backbone | **Apache Kafka** (managed: MSK/Confluent) | Durable, replayable, multi-consumer fan-out for decisions, reviews, regulatory events, and recompute events without coupling services |
| Cache | **Redis** | Hot-path sanctions-list lookups, embedding cache, idempotency/dedup keys |
| Object storage | **S3-compatible, with Object Lock (WORM)** | Immutable storage for regulatory documents and audit bundle exports — a real non-repudiation mechanism, not a marketing claim |
| Signing / KMS | **Cloud KMS asymmetric signing** (e.g., AWS KMS) | Signs daily Merkle roots of the audit ledger; avoids rolling your own crypto |
| API layer | **FastAPI** (Python), stateless | Async I/O suits the fan-out agent pattern; horizontal autoscaling on stateless pods is straightforward |
| Container orchestration | **Kubernetes**, with **separate node pools / priority classes** for hot path vs. cold path (Loop 2) | This is the physical enforcement of "recompute never starves live traffic" |
| IaC / deployment | **Terraform** + **ArgoCD** (GitOps), canary/shadow deploys for agent & prompt version changes | High-stakes decision logic changes get shadow-mode validation before promotion, same pattern already used for FAISS index swaps |
| Observability | **OpenTelemetry** (tracing per agent span) → **Prometheus/Grafana** (metrics) → **Loki/ELK** (structured logs) → PagerDuty/Opsgenie (alerting) | Standard, vendor-neutral, and gives per-agent latency/error breakdown needed for the SLOs in §7 |
| Drift/calibration monitoring | Custom service (PSI/KS-test on confidence distributions) + reliability diagrams, or an off-the-shelf ML-monitoring library | Confidence drift is a compliance signal, not just an ML-ops nicety, per your design |
| Data residency / region | Single-region primary (India region, e.g. `ap-south-1`), multi-AZ HA | NBFC/DPDP data-handling expectations favor keeping principal data in-region; confirm current cross-border transfer rules with legal before assuming any specific hard localization mandate |
 
---
 
## 7. Latency strategy
 
### 7.1 Hot-path budget (target: p95 < 3s, p99 < 6s, end-to-end, excluding human review wait)
 
| Stage | p95 target | Technique |
|---|---|---|
| Parallel agent fan-out (Document, Sanctions, Temporal, Transaction, RAG) | ≤ 500 ms (bounded by the slowest branch, RAG) | Concurrent execution, not sequential; each agent has its own budget from §3.1 |
| Decision synthesis (LLM) | ≤ 2000 ms | Prompt caching for static system/regulatory context; streaming not needed since output must be validated before persisting |
| Audit-before-response write | ≤ 50 ms | Single local Postgres transaction commit; async replication happens after |
| **Total (server-side)** | **≤ 3s p95** | |
 
### 7.2 Concrete techniques
- **Parallelize independent agents** (§3.1) — the largest single latency win.
- **Prompt caching** on the regulatory system prompt/context block passed to the Decision Agent — this block is large and mostly static between requests; caching it avoids re-processing the same tokens on every call.
- **In-memory sanctions list** (refreshed on a schedule, served from Redis/local memory) instead of a DB round-trip per screening.
- **Pure-code temporal checks** — no LLM call needed for expiry/date arithmetic; keep this agent LLM-free entirely.
- **FAISS ANN search tuned for recall/latency tradeoff** appropriate to corpus size (IVF+PQ style indexing once the corpus grows beyond what a flat index can serve in-budget).
- **Bureau/external data timeouts** are hard-capped; a slow external dependency degrades to REVIEW rather than blocking the response indefinitely (never let an external system's latency become your system's outage).
### 7.3 Cold-path (Loop 2) latency posture
Loop 2 is explicitly **not** latency-sensitive in the same sense — a regulatory change doesn't need to fully resolve in milliseconds. Its SLA is expressed differently:
- **Sanctions-topic changes**: begin re-screening within minutes to a few hours of detection (this one *is* time-sensitive, for AML/CFT reasons).
- **KYC/FOIR/governance-topic changes**: begin recompute within a business day, complete within a configurable window (e.g., 3–5 business days for the full ADS), explicitly to allow canary validation and to keep the cold-path throttle from ever pressuring the hot path.
### 7.4 Safe index/version swaps (as you specified, generalized)
- FAISS indexes are built in the background as `index_v(n+1)`, validated against a fixed retrieval-regression test set, then atomically swapped via a pointer/alias — traffic never sees a partially-built index. The previous two versions stay warm for instant rollback.
- The same double-buffer-and-swap pattern is applied to **agent/prompt versions**: a new Decision Agent prompt or model version runs in **shadow mode** (processes live traffic in parallel, outputs logged but not returned to callers) until its outputs are validated against the golden regression set, then promoted via canary (§13).
### 7.5 No cold starts on the hot path
 
Every model the hot path depends on — the embedding model behind RAG retrieval, and any self-hosted classification/reranking model — is loaded at process start and health-gated before the pod is added to the load-balancer's ready pool, not loaded lazily on first request:
 
```
Bad:   Request → Load model → Inference → Unload           (cold start on the critical path)
Good:  Server starts → Model loaded → Readiness probe passes → Ready → Request → Inference
```
 
Kubernetes readiness probes enforce this: a pod that hasn't finished loading its model never receives traffic. This applies to the RAG Agent's embedding model and to any self-hosted reranker; it does not apply to the Decision/Sentinel LLM calls, which go to a hosted API and have no local load step. Cold starts on a locally-hosted model are exactly the kind of latency spike that can silently blow the p95 budget in §7.1, so warm-pool sizing is treated as a first-class capacity parameter, not an incidental deployment detail.
 
### 7.6 Queue slow document processing off the synchronous path
 
The Document Agent's job in the hot path (§3.1) is bounded at 300ms — that budget covers presence/format/consistency checks against already-extracted fields, not full OCR of a scanned artifact, which can run 15–30s for a multi-page scanned income proof or entity document. Forcing the caller to block on that would blow the entire p95 budget on its own, so OCR is explicitly out of the synchronous `/v1/decide` path:
 
```
Upload → Queue (Redis/Kafka-backed) → OCR Worker pool → Extract fields → Store to Application
Evidence Store → publish `document.extracted` → LangGraph workflow resumes from the
Document Agent node with extracted fields available
```
 
Concretely: if an application includes documents that need OCR, `/v1/decide` either (a) returns a `PENDING_EXTRACTION` status immediately if OCR isn't yet complete, with the client polling or subscribing to a webhook, or (b) — more commonly — OCR happens at upload time, ahead of and decoupled from the decision call, so by the time `/v1/decide` is invoked the extracted fields already exist and the Document Agent's 300ms budget only covers validation, not extraction. Either way, no caller ever blocks synchronously on an OCR worker. This also smooths traffic spikes: a burst of document uploads queues into the OCR worker pool and drains at the pool's sustained throughput, rather than each upload holding open an API connection for 20+ seconds.
 
### 7.7 Streaming upload progress
 
For large PDF/scan uploads, the client receives incremental progress (percentage or byte-chunk acknowledgments) rather than a single blocking call that only resolves at 100%. This is a perceived-latency fix, not a throughput one — it doesn't change when OCR starts, but it means the applicant-facing UI never looks frozen during a multi-MB upload, which matters operationally because it reduces duplicate-submission retries from anxious end users hammering the upload button.
 
### 7.8 Timeouts and circuit breakers as first-class latency controls
 
Every external dependency the hot path touches has both a hard timeout and a circuit breaker, not just a timeout in isolation:
 
- **Bureau API, sanctions data source, LLM provider**: each call is timeout-capped (e.g., LLM decision-synthesis call capped at the 2000ms budget from §7.1); a timeout does not retry indefinitely, it degrades immediately per the failure-mode column in §3.1's agent table (forced REVIEW, never a silent stall).
- **Circuit breaker per dependency**: after a configured run of consecutive failures (e.g., 5) against a given dependency, the breaker opens for a cooldown window; while open, calls to that dependency are skipped entirely rather than re-attempted and timed out one by one, and the affected agent falls back to its documented degraded behavior (e.g., RAG Agent falls back to the last-known-good FAISS index per §7.4; Decision Agent, if the LLM provider's breaker is open, cannot synthesize a verdict and the case is forced to REVIEW with reason `llm_provider_unavailable`).
- This is what keeps a third-party outage from becoming *ComplianceLoop's* outage: the system degrades gracefully into a higher REVIEW rate rather than collapsing into timeouts and 5xxs.
---
 
## 8. Scalability strategy
 
### 8.1 Capacity model (illustrative — recalibrate to actual NBFC volume)
Assume a mid/large NBFC processing 50k–200k applications/month → average 2–8 requests/sec, with peaks (month-end, promotional campaigns) up to 10x average. Hot-path infra is sized and autoscaled (Kubernetes HPA on queue depth + p95 latency, not just CPU) for the peak, not the average — e.g. gateway pods scaling from a steady-state 2 replicas to 8+ as request rate climbs from ~10 req/s to ~100 req/s, then back down as demand falls, so idle capacity isn't paid for outside campaign/month-end windows.
 
### 8.2 Physical isolation of Loop 1 and Loop 2
- Separate Kubernetes node pools/namespaces with distinct resource quotas and priority classes; Loop 2 workers are explicitly lower-priority and preemptible.
- Separate Kafka topics/partitioning, and separate consumer groups, so a recompute backlog is a queue-depth metric on its own dashboard, never a shared-resource contention problem.
- A token-bucket rate limiter caps Loop 2's consumption of shared downstream dependencies (e.g., the bureau API, the LLM provider's rate limits) to a configured fraction of total capacity, so a large regulatory event can never exhaust the quota the hot path needs.
### 8.3 Horizontal scaling points
- **FastAPI gateway + LangGraph workers**: stateless, scale on request rate.
- **FAISS retrieval service**: scale by sharding across regulatory domains; each shard independently horizontally replicated for read throughput.
- **Kafka**: partition by `applicant_id`/tenant for per-user ordering while allowing cross-user parallelism.
- **Postgres**: read replicas for the Impact Resolver's evidence queries (these are read-heavy, bursty, and should not compete with hot-path write transactions); consider a dedicated replica specifically for Loop 2 analytical queries.
- **Temporal**: scales by adding worker pools per task queue; Loop 1 review workflows and Loop 2 recompute workflows run on separate task queues for the same isolation reason as everywhere else in this design.
### 8.4 Multi-tenancy (if ComplianceLoop serves multiple NBFC clients)
Tenant isolation at the data layer (row-level security or schema-per-tenant in Postgres, tenant-prefixed Kafka topics or headers), with recompute policies and regulatory corpora scoped per tenant where their applicable regulation set differs (e.g., different classes of NBFC have different applicable Master Directions).
 
### 8.5 Event-driven decoupling of secondary work
 
The response path stays minimal by construction — `/v1/decide` only ever waits on the agent fan-out, decision synthesis, and the audit-before-response write (§3.2):
 
```
Instead of:  API → Workflow → Audit → Notification → Analytics → Monitoring → Response  (caller waits on all of it)
ComplianceLoop:  API → Workflow → Decision → Audit-before-response write → Response
                 then, via Debezium/Kafka (§3.2): Decision Created →
                 independent consumers handle Notification / Analytics / Monitoring / (Loop 2 indexing)
```
 
This isn't new relative to §2/§3.2's outbox pattern — it's the same mechanism, called out here explicitly because it's the reason adding a new downstream consumer (say, a new analytics pipeline, or a future regulator-reporting feed) never requires touching the hot path or re-budgeting its latency: it just subscribes to `decisions.made`.
 
### 8.6 Scale each agent independently
 
Agents have materially different load profiles, so they are separate deployments with independent HPA policies, not one monolithic "agent service":
 
| Agent | Illustrative load | Scaling behavior |
|---|---|---|
| Document Agent | ~200 req/s | Scales first and most aggressively under upload-heavy traffic (e.g., 3 → 10 pods) |
| Sanctions Agent | ~20 req/s | Scales moderately; also protected by the in-memory list cache from §7.2 |
| Temporal Agent | High, but sub-millisecond CPU cost (pure rule eval, no I/O) | Rarely needs more than a couple of replicas for redundancy, not throughput |
| Transaction Agent | Bounded by bureau API's own rate limits, not by ComplianceLoop's compute | Scaling replicas beyond the bureau's own throughput ceiling doesn't help — this is where a token-bucket client-side limiter matters more than pod count |
| RAG Agent | ~5 req/s, but each call is FAISS-search + embed | Scales least aggressively; sharded per regulatory domain (§8.3) rather than blindly replicated |
 
A spike in document-upload traffic scales Document pods without touching Sanctions or RAG capacity, and vice versa — this is the practical payoff of the fan-out agents being independent LangGraph nodes with independent deployments rather than one process handling all five.
 
### 8.7 Load balancing
 
The FastAPI gateway sits behind a load balancer distributing across multiple stateless gateway pods (no session affinity required, since idempotency keys and request state live in Postgres/Redis, not in-memory) — no single gateway instance is a bottleneck or a single point of failure:
 
```
                    Load Balancer
                 /       |        \
            Gateway-1 Gateway-2 Gateway-3
```
 
The same pattern applies one layer down to each agent deployment in §8.6.
 
### 8.8 Database query discipline
 
Postgres is the hot path's only synchronous write dependency (§3.2), so query discipline there is a direct latency lever, not just a hygiene concern:
 
- No `SELECT *` on hot-path queries — only the columns a given agent or the gateway actually needs.
- Indexes purpose-built for the actual query patterns — most importantly the `decision_evidence (evidence_type, evidence_id, evidence_role)` index from §5, which is what keeps Impact Resolver queries out of full table scans as history grows.
- Prepared statements and a connection pooler (e.g., PgBouncer) in front of Postgres, since connection setup overhead is otherwise a meaningful fraction of a sub-100ms query budget under load.
- Pagination on any endpoint that lists decisions/audit rows for a human reviewer or a regulator export — never an unbounded result set.
- Read replicas: the hot path writes to primary only; Impact Resolver's evidence queries and any reviewer-dashboard reads are steered to a replica (already specified in §8.3) so read-heavy Loop 2/reporting traffic can never contend with hot-path write latency.
### 8.9 Storage separated by purpose
 
Postgres is not used as a catch-all — each data shape goes to the store built for it:
 
| Data | Store | Why |
|---|---|---|
| Applications, decisions, evidence, audit ledger | PostgreSQL | ACID transactions, required for audit-before-response (§3.2) |
| Session/idempotency state | Redis | Sub-millisecond reads, natural TTL expiry |
| Raw uploaded documents, regulatory PDFs | S3-compatible object storage (WORM/Object Lock) | Cheap, immutable, high-throughput blob storage — not what a relational DB is built for |
| Audit search/export (regulator queries across large date ranges) | Postgres, optionally mirrored to Elasticsearch for full-text/faceted search | Postgres remains system of record; Elasticsearch (if added) is a read-optimized derivative, never the source of truth |
| Regulatory embeddings | FAISS | Purpose-built ANN index, not a relational table |
| Metrics | Prometheus | Purpose-built time-series store for the SLO dashboards in §12 |
 
### 8.10 Rate limiting
 
Every external-facing endpoint is rate-limited per tenant/API key using a Redis-backed token bucket, independent of the internal Loop 2 throttle described in §8.2 (which limits Loop 2's *internal* consumption of shared dependencies, not external callers):
 
```
One tenant → burst of requests → limiter → within budget → processed
                                          → over budget → HTTP 429
```
 
This protects the hot path from a single misbehaving integration (a retry loop on the LOS side, for instance) from degrading service for every other tenant.
 
### 8.11 Batch operations where the workload is naturally batchable
 
Sanctions screening for a bulk upload (e.g., a portfolio re-screen, or a Loop 2 SANCTIONS-topic recompute batch) uses a single bulk screening call against the watchlist service rather than one call per applicant — this is the same principle Loop 2's canary-then-full-batch execution (§4.2d) already depends on, made explicit here because it's a general pattern, not something specific to recompute: network round-trip overhead dominates at small-payload scale, so batching is a real throughput win, not just tidiness.
 
---
 
## 9. Reliability & consistency
 
### 9.1 Idempotency everywhere
- Every hot-path request carries an idempotency key so retried client calls don't produce duplicate decisions.
- Every Loop 2 recompute workflow is keyed by `(decision_id, change_id)` — Temporal guarantees exactly-once execution for that key even across crashes/retries.
### 9.2 Immutable, non-repudiable audit
- Decision and evidence rows are never updated in place; corrections are new rows referencing the old one.
- A scheduled job computes a Merkle root over each day's audit ledger rows and signs it with KMS — this gives you a cheap, defensible non-repudiation mechanism without needing to reach for heavier "blockchain" infrastructure, which would add operational complexity without adding real guarantees here.
- The meta-audit ledger for Loop 2 is structurally identical but logically separate, so a regulator's question about *why a change was detected and acted on* doesn't get mixed up with the question about *why a specific decision was made*.
### 9.3 Failure handling
- Any upstream agent error/timeout forces the Decision Agent into REVIEW (never a silent default to APPROVE).
- Per-dependency circuit breakers (§7.8) sit in front of every external call (LLM provider, bureau API, sanctions source); an open breaker skips the call outright instead of letting each request re-discover the outage via its own timeout — this is what keeps a third-party outage from turning into cascading hot-path latency.
- Dead-letter queues on every Kafka consumer for events that fail processing after retry, with alerting — nothing is silently dropped.
- Disaster recovery: multi-AZ Postgres with automated failover; periodic backup restoration drills; Kafka replication factor ≥ 3.
---
 
## 10. Security & DPDP-aligned privacy controls
 
- **Data minimization at the access layer**: a Consent/Purpose service issues purpose-bound access tickets (e.g., `purpose=KYC_VERIFICATION`); the data access proxy only returns fields tagged for that purpose to a given agent — agents cannot over-fetch by construction, not by policy alone.
- **Consent ledger**: `consent_id, data_principal_id, purpose, scope, granted_at, withdrawal_status` — checked before any personal-data pull; withdrawal is enforced going forward, with retained-data handling reconciled against your legal retention obligations (a "logical erasure with legal-hold flag" pattern, since regulatory retention duties and a data principal's request can be in tension — this needs explicit legal sign-off on the reconciliation rule, not an engineering assumption).
- **Encryption**: at rest (KMS-managed keys), in transit (mTLS between internal services), field-level encryption/tokenization for sensitive identifiers (PAN, any Aadhaar-linked reference — never store raw Aadhaar).
- **Access auditing**: every internal read of personal data is itself logged (who, when, which record, under which purpose ticket) — this is both an RBI vendor-accountability expectation and a DPDP accountability-principle control.
- **Breach-response readiness**: the same audit ledger infrastructure gives you the "what data, which principals, when" answer a breach-notification obligation requires, without needing a separate forensic system.
---
 
## 11. RBI alignment controls
 
- **Accountability stays with the regulated entity**: every automated decision has a named accountable reviewer role for REVIEW-routed cases, and every Loop 2 recompute batch has a named compliance approver for the recompute policy applied — the system supports human accountability, it doesn't replace it, consistent with expectations that vendor/agent involvement doesn't dilute the NBFC's own responsibility.
- **Governance over model/agent changes**: shadow-mode validation and canary promotion (§7.4, §13) exist specifically so that a change to decisioning logic is itself a governed, auditable event, not a silent deploy.
- **Digital lending accountability**: the RAG Agent's citation-enforced retrieval means every decision has a traceable link to the specific guidance it was evaluated against, at the version in force on that date — useful both for internal model governance reviews and for responding to regulator queries about a specific cohort of decisions.
- **KYC and prudential controls**: encoded as versioned rulesets (not hardcoded logic) so their evolution is itself tracked the same way regulatory text is tracked.
*(Confirm the exact current text of the applicable Master Directions, Digital Lending Guidelines, and KYC Master Direction with your compliance/legal function — this document encodes the control pattern, not a specific dated clause set.)*
 
---
 
## 12. Observability & governance
 
- **Per-agent tracing** (OpenTelemetry spans) gives latency and error attribution down to a single agent call within a single decision — essential for diagnosing which stage is threatening the SLO in §7.
- **Confidence drift dashboard**: rolling distribution of Decision Agent confidence vs. actual reviewer-override rate, segmented by risk category and by `trigger_source` (organic vs. regulation-triggered) — this is the calibration engine's primary input and also a standing governance artifact.
- **Loop 2 dashboards**: change-detection latency (publication → RCE), ADS size per change, canary flip-rate, full-batch flip-rate, recompute backlog depth per topic tag, and — critically — a hot-path SLO panel that proves Loop 2 activity never correlates with hot-path latency regressions.
- **Alerting**: SLO burn-rate alerts on hot-path latency/error rate; backlog-depth alerts on Loop 2 (a stalled recompute queue on a MAJOR sanctions-topic change is a compliance risk, not just an ops nuisance, and should page accordingly).
---
 
## 13. Testing & safe rollout strategy
 
- **Golden regression set**: a curated, versioned set of past applications with known-correct outcomes plus deliberately hard edge cases (ambiguous sanctions matches, borderline FOIR, expiring documents) — every agent/prompt/model version change is run against this set before promotion.
- **Shadow mode**: new decision-logic versions process live traffic in parallel without affecting real responses; outputs are diffed against the current production version before any promotion decision.
- **Canary promotion**: gradual traffic ramp (e.g., 1% → 10% → 50% → 100%) with automatic rollback triggers on error-rate or flip-rate anomalies.
- **Loop 2 specific tests**: 
  - Unit tests on the diff/classification thresholds against a labeled corpus of real historical circular revisions (to validate the 0.92/0.98 similarity cutoffs are sane, not arbitrary).
  - Chaos test: simulate a large-blast-radius regulatory change and verify the token-bucket throttle holds and hot-path latency is unaffected.
  - Replay test: use Temporal's deterministic replay to verify a recompute workflow produces the same outcome given the same inputs, across code deploys.
---
 
## 14. Phased delivery roadmap
 
| Phase | Scope | Exit criteria |
|---|---|---|
| **0 — Foundations** | Core data model (decisions, decision_evidence, audit_ledger), Document + Temporal agents, synchronous MVP, audit-before-response write path | A real application can be decisioned end-to-end with a persisted, citable audit trail |
| **1 — Full decision pipeline** | Sanctions, Transaction, RAG agents; Decision Agent synthesis with enforced citation schema; FAISS corpus v1 | Golden regression set passes; hot-path p95 within budget under load test |
| **2 — Human review & calibration** | Temporal-based review workflows, calibration engine v0 (manual threshold review), observability stack | Reviewer override rate tracked; SLO dashboards live |
| **3 — Regulatory Drift Loop, "detect & alert only"** | Sentinel Agent, diff/classification engine, Impact Resolver producing ADS manifests — **no automatic recompute yet**, compliance team reviews ADS manually | ADS accuracy validated against a set of known historical circular changes with hand-labeled correct affected sets |
| **4 — Regulatory Drift Loop, "auto-recompute, human-confirms-flips"** | Recompute Orchestrator live, canary-first execution, auto-close on "confirmed unchanged," human review on flips | Canary flip-rate consistently sane across several real regulatory events; zero hot-path SLO regressions observed during recompute batches |
| **5 — Scale hardening** | Multi-tenant isolation (if applicable), chaos/load testing at peak-times-10x, DR failover drills, full meta-audit signing pipeline | DR drill passes RTO/RPO targets; load test sustains peak volume at target latency for a sustained window |
| **6 — Full closed-loop autonomy (optional, only once trusted)** | Fully automated confirm-and-close for low-severity, high-confidence recompute outcomes, with periodic sampled audits | Sustained low false-flip rate over a defined observation period, signed off by compliance |
 
---
 
## 15. Risk register (selected, high-signal items)
 
| Risk | Mitigation |
|---|---|
| LLM cites a clause it didn't actually retrieve (hallucinated citation) | Code-level validation: cited evidence IDs must be a subset of retrieved evidence IDs before a decision can be persisted; violation forces REVIEW |
| A misclassified "MAJOR" change triggers a mass, unnecessary recompute | Canary-first execution (1–2% sample validated before full batch); severity thresholds tuned against a labeled historical dataset, not guessed |
| Regulatory recompute backlog competes with live traffic under load | Physical pool isolation + token-bucket throttling (§8.2), enforced in infra, not just in application logic |
| A regulatory source change is missed because the source has no reliable feed | Multiple ingestion channels per source (scheduled poll + manual upload path for compliance-team-sourced documents), with a "last successfully ingested" staleness alert |
| Confidence scores drift out of calibration silently | Scheduled calibration job with drift-monitoring dashboards and alerting, not a one-time tuning exercise |
| Data-principal erasure request conflicts with regulatory retention duty | Explicit "logical erasure + legal-hold flag" pattern, with the reconciliation rule owned and signed off by legal, not assumed by engineering |
| Vendor/agent involvement is read by a regulator as diluting NBFC accountability | Every automated pathway has a named accountable human role at the review and recompute-policy-approval points; this is documented as an operating control, not just a technical one |
 
---
 
## 16. Summary
 
The core idea you added — an agent that watches regulatory sources, figures out precisely what changed, and re-runs only the decisions that relied on the changed material — is feasible **specifically because** of a discipline the rest of the system already needs anyway: every decision must record, per agent, exactly which regulatory atoms it *used* (not just retrieved). That evidence-citation requirement is what turns "re-run everyone" into "re-run exactly these N cases, and here is the signed proof of why." Combined with physical isolation between the real-time hot path and the recompute cold path, and a graduated rollout (detect-only → human-confirmed → limited autonomy), this closes the loop you described — regulation → decisioning → audit → continuous learning — without putting live lending latency or system stability at risk at any point.