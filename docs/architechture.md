# ARCHITECTURE.md
*The system's shape. Store it here so no session has to re-derive it.*

## High-level data flow

```
Applicant / Case Data
        │
        ▼
   FastAPI Gateway  (api/routers/decide.py)
        │  idempotency key check
        ▼
   LangGraph Orchestrator  (orchestrator/graph.py)
        │
   ┌────┴─────────────────────────────┐
   │        Parallel agent fan-out     │
   ▼            ▼            ▼          ▼
Document     Temporal    Sanctions   Transaction    RAG Agent
 Agent        Agent        Agent       Agent       (regulatory
   │            │            │           │          retrieval)
   └────┬───────┴─────┬──────┴───────────┘
        ▼
   Decision Agent  (LLM synthesis, MUST cite evidence)
        │
        ▼
   ONE Postgres transaction:
     - decisions (row)
     - decision_evidence (rows, one per citation)
     - outbox_events (row)
        │  commit
        ▼
   HTTP response returned  ← only after commit (audit-before-response)
        │
        ▼
   Debezium (CDC on outbox_events) → Redpanda topic → downstream consumers
   (calibration engine, notification service, Loop 2 impact indexing)
```

## Loop 2 — Regulatory Drift Loop (cold path, Phase 3+)

```
RBI website / circular sources
        │
        ▼
   Sentinel  (loop2/sentinel)  — polls sources, detects new/changed documents
        │
        ▼
   Diff Engine  (loop2/diff_engine)  — chunk-level diff, severity + topic classification
        │
        ▼
   Impact Resolver  (loop2/impact_resolver)  — queries decision_evidence to find
        │                                       every past decision that cited the
        │                                       now-changed clause (blast radius)
        │                                       → generates an Affected Decision Set (ADS)
        ▼
   Recompute Orchestrator  (loop2/recompute_orchestrator)  — selectively replays
        │                                                     only the affected agents
        │                                                     for ADS decisions, canary-tests
        ▼
   Temporal review_workflow  — human confirms recomputed decisions before they replace originals
```

## Why decision_evidence is the hinge of the whole system
`decision_evidence` is what makes Loop 2 possible at all. Every agent, when it
uses a regulatory clause or a document field to justify part of a decision,
writes a row here (`evidence_type`, `evidence_id`, `evidence_version`,
`evidence_role` = RETRIEVED or CITED). When a regulation changes, the Impact
Resolver's entire job is a lookup against this table — "which decisions cited
clause X, version Y." If this table is incomplete or agents skip writing to
it, Loop 2 cannot function no matter how good the drift detection is. This is
why Phase 0 builds this table and its indexing before any LLM agent exists.

## Folder structure
See `docs/PROJECT.md`'s tech stack table for the "what," and the repo root
`README.md` for the authoritative live folder tree (kept in sync as the repo
grows — don't duplicate the full tree in two places).

Top-level shape:
```
complianceloop/
├── agents/            one subfolder per agent, each implements agents/base.py's Agent interface
├── orchestrator/       LangGraph DAG
├── api/                FastAPI app, routers, schemas, middleware
├── db/                 SQLAlchemy models, Alembic migrations, evidence_store helpers
├── workflows/           Temporal workflows (review, recompute)
├── loop2/               Sentinel, Diff Engine, Impact Resolver, Recompute Orchestrator
├── calibration/          scheduled calibration engine (Phase 2)
├── security/             KMS signer interface, DPDP consent (Phase 3)
├── observability/         OpenTelemetry + Prometheus
├── infra/                docker-compose, k8s (Phase 5+), terraform (Phase 5+)
├── tests/                unit, integration, golden regression set
├── scripts/               dev fixtures/seed data
└── docs/                  this project brain
```

## Local infra substitutions (vs. the blueprint's cloud-scale versions)
See `DECISIONS.md` for the full reasoning on each of these — this table is
just the current state:

| Blueprint component | Local dev implementation |
|---|---|
| Managed Kafka (MSK/Confluent) | Redpanda, single Docker container |
| Temporal Cloud | Self-hosted Temporal, Docker Compose |
| Aurora Postgres multi-AZ | Local Postgres → later Supabase/Neon free tier |
| AWS S3 + Object Lock | MinIO |
| Cloud KMS | Local ED25519 keypair (dev only) |
| Kubernetes multi-node-pool | Docker Compose; `kind`/`k3d` single-node when namespace isolation is needed |

## API surface
See `API.md` for endpoint-level detail.

## Data model
See `DATABASE.md` for table-level detail.

## Diagrams
Ascii diagrams above are the canonical version for now. If this file grows a
proper diagramming need, generate diagrams via the Visualizer/draw.io and
store exported SVGs under `docs/diagrams/`, referenced from here — don't
let diagrams drift out of sync with this file's text description.