# DATABASE.md
*Schema-level reference. Source of truth is the Alembic migrations in
`db/migrations/versions/` — this file explains intent, the migrations are
the literal ground truth if they ever disagree.*

## Engine
Postgres (async via SQLAlchemy 2.0 + asyncpg). Local via Docker Compose for
Phase 0–4; Supabase/Neon free tier or equivalent for an always-on instance
when needed (see `DECISIONS.md`).

## Phase 0 tables

### `decisions`
The record of a single compliance decision.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID, PK | |
| `application_id` | text | external reference |
| `status` | enum (`APPROVED`, `REJECTED`, `REFERRED`) | |
| `confidence` | float | |
| `ruleset_version` | text | which checklist/rules version produced this |
| `created_at` | timestamptz | |
| `idempotency_key` | text, unique | enforces the no-duplicate-on-retry guarantee |

### `decision_evidence`
The hinge table — every claim a decision makes must trace back here. See
`ARCHITECTURE.md` for why this table is the whole system's foundation.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID, PK | |
| `decision_id` | UUID, FK → decisions.id | |
| `evidence_type` | text | e.g. `document`, `regulatory_clause`, `transaction_record` |
| `evidence_id` | text | reference to the source record/chunk |
| `evidence_version` | text | version of the evidence at time of citation — critical for Loop 2 |
| `evidence_role` | enum (`RETRIEVED`, `CITED`) | RETRIEVED = considered; CITED = actually used to justify the decision |
| `created_at` | timestamptz | |

**Index (non-negotiable, built in Phase 0):**
```sql
CREATE INDEX ON decision_evidence (evidence_type, evidence_id, evidence_role)
INCLUDE (decision_id);
```
This is what makes the Impact Resolver's blast-radius query fast at scale —
"which decisions cited clause X" needs to be an index lookup, not a table
scan, from day one.

### `outbox_events`
Transactional outbox — written in the *same transaction* as `decisions` and
`decision_evidence`, then picked up by Debezium CDC and published to
Redpanda/Kafka.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID, PK | |
| `aggregate_type` | text | e.g. `decision` |
| `aggregate_id` | UUID | e.g. the decision id |
| `event_type` | text | e.g. `decision.created` |
| `payload` | jsonb | |
| `created_at` | timestamptz | |

---

## Planned tables (later phases — listed so scope isn't lost)

| Table | Phase | Purpose |
|---|---|---|
| `regulatory_documents` | 1 | Ingested RBI circulars, chunked + embedded, versioned |
| `review_tasks` | 2 | Human review queue, linked to Temporal `review_workflow` |
| `calibration_records` | 2 | Tracks predicted confidence vs. actual reviewer outcome |
| `drift_events` | 3 | Sentinel/Diff Engine output — a detected regulatory change |
| `affected_decision_sets` | 3 | Impact Resolver output — ADS per drift event |
| `recompute_runs` | 4 | Recompute Orchestrator's canary/rollout tracking |

---

## Migration rules
- Every schema change goes through Alembic. No manual `ALTER TABLE` against
  the dev database that isn't captured in a migration file.
- Never change an existing column's type or drop a column without a
  `DECISIONS.md` entry explaining why and what reads/writes it.
- `decision_evidence` and `outbox_events` are structurally frozen unless the
  blueprint's own audit/outbox pattern changes — these are the two tables
  every downstream guarantee depends on.