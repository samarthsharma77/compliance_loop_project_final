# API.md
*Endpoint-level reference. Update this alongside the code that implements it — never let it drift.*

## Conventions
- All endpoints are async FastAPI routes.
- All mutating endpoints accept an `Idempotency-Key` header; replaying the
  same key with the same body must not create duplicate rows.
- All responses are only returned after their underlying DB transaction
  commits (audit-before-response — see `PROJECT.md` non-negotiables).
- Auth: not yet implemented (Phase 0 has none — add before any real data
  touches this system; track as a `TODO.md` item, don't silently ship
  without it).

---

## `POST /v1/decide`
**Phase:** 0
**Status:** not yet implemented

**Purpose:** Submit an application for compliance decisioning.

**Request headers:**
```
Idempotency-Key: <client-generated UUID>
Content-Type: application/json
```

**Request body (Phase 0 scope — Document + Temporal agents only):**
```json
{
  "application_id": "string",
  "documents": [
    {
      "document_type": "string",
      "storage_ref": "string",
      "issue_date": "YYYY-MM-DD"
    }
  ]
}
```

**Response 200:**
```json
{
  "decision_id": "uuid",
  "application_id": "string",
  "status": "APPROVED | REJECTED | REFERRED",
  "confidence": 0.0,
  "evidence": [
    {
      "evidence_type": "string",
      "evidence_id": "string",
      "evidence_version": "string",
      "role": "CITED"
    }
  ],
  "created_at": "ISO-8601 timestamp"
}
```

**Response 4xx/5xx:** standard FastAPI validation errors; on internal
failure, no partial decision is ever persisted (transaction rolls back
entirely — no half-written audit trail).

**Note on Phase 0 vs later phases:** in Phase 0 this endpoint only runs
Document Agent + Temporal Agent (no LLM call, no real "decision" synthesis
yet — the join node just aggregates both agents' verdicts). From Phase 1
onward, the same endpoint runs the full agent DAG including the Decision
Agent's LLM-synthesized verdict. The request/response shape is designed not
to need a breaking change between phases — extend, don't rename.

---

## Planned endpoints (not yet built — listed so scope isn't lost)

| Endpoint | Phase | Purpose |
|---|---|---|
| `GET /v1/decisions/{id}` | 0/1 | Retrieve a decision + its full evidence trail |
| `POST /v1/decisions/{id}/review` | 2 | Human reviewer submits a verdict on a referred decision |
| `GET /v1/regulatory-drift/events` | 3 | List detected regulatory changes and their ADS |
| `POST /v1/regulatory-drift/{event_id}/recompute` | 4 | Trigger recompute for an ADS (or confirm auto-triggered recompute) |
| `GET /v1/health` | 0 | Liveness/readiness for orchestration |

---

## Rule for this file
**Never rename an existing endpoint or change its response shape without a
corresponding entry in `DECISIONS.md` explaining why**, and without checking
whether it's a breaking change for any consumer. See `CODING_RULES.md`.