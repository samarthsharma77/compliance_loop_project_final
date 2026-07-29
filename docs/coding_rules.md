# CODING_RULES.md
*Behavioral rules for any model/session working on this codebase. Read
before writing code; violating these should be treated as a bug, not a
style preference.*

## Architectural rules (non-negotiable — see PROJECT.md)
1. **Never return an HTTP response before the underlying transaction
   commits.** Audit-before-response is the system's core guarantee.
2. **Never let an agent make a claim without a corresponding
   `decision_evidence` row.** If it can't cite it, it can't claim it.
3. **Every new agent must implement `agents/base.py`'s `Agent` interface.**
   No ad hoc agent shapes.
4. **`security/signer.py` stays an interface**, never hardcode assumptions
   that only work with the local dev keypair — it must be swappable for a
   real KMS-backed implementation without touching calling code.

## Change-management rules
5. **Never rename an existing API endpoint or change its response shape**
   without a `DECISIONS.md` entry and a check for breaking-change impact.
6. **Never change the database schema without an Alembic migration.** No
   manual `ALTER TABLE` against dev DBs that isn't captured in a migration.
7. **Never change a prompt materially without a `DECISIONS.md` entry** —
   prompt changes can silently change decision behavior system-wide.
8. **No infrastructure or new tables beyond what's in the blueprint or
   already-scoped phase work, without flagging it to the project owner
   first.** If unsure whether something is in scope, ask — don't assume.
9. **Don't build ahead of the current phase.** If working on Phase 0, don't
   scaffold Phase 3 code "while we're at it." See `ROADMAP.md`.

## Code style
10. Python 3.11+, async everywhere it's plausible (FastAPI routes,
    SQLAlchemy 2.0 async sessions, agent `run()` methods).
11. Pydantic v2 for all request/response schemas and structured agent I/O.
12. Logging via `loguru`, not the stdlib `logging` module directly.
13. All LLM prompts live in `prompts/` as loadable files, indexed in
    `docs/PROMPTS.md` — never inline a prompt as a bare string buried in
    agent logic where it won't be discovered or reused.
14. No business logic inside FastAPI routers — routers call orchestrator/
    service functions; keep routers thin (request validation → call →
    response shaping only).
15. Every agent, endpoint, and table gets a corresponding entry in
    `AGENTS.md` / `API.md` / `DATABASE.md` — code and docs land in the same
    commit, not as a follow-up "I'll document it later."

## Testing rules
16. Every new endpoint gets an integration test asserting the
    audit-before-response guarantee (row exists in DB after response
    returns) and idempotency behavior where applicable.
17. Golden regression set (`tests/golden_regression_set/`) is never edited
    to make a failing test pass — if a legitimate behavior change makes an
    old golden case wrong, that's a `DECISIONS.md`-worthy conversation, not
    a quiet test edit.

## Session hygiene
18. Commit with meaningful, conventional messages: `feat:`, `fix:`,
    `refactor:`, `docs:`, `test:`.
19. At the end of every session, generate a handoff and append it to
    `docs/SESSION_LOG.md` (see `docs/BOOTSTRAP_PROMPT.md` for the exact
    ask). Update `docs/TODO.md` to reflect reality before stopping.
20. If a session notices these rules are wrong or outdated, say so and
    propose an edit — don't silently work around a stale rule.