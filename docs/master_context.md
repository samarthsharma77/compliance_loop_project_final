# MASTER_CONTEXT.md
*The onboarding file. Every new session/model reads this FIRST, before
touching anything else.*

---

You are joining ComplianceLoop, an AI-powered compliance decisioning
platform for NBFCs, being built solo on free-tier cloud credits using
Cursor and Claude.

## Read these, in order, before writing any code

1. `docs/PROJECT.md` — what this system is, its non-negotiable principles,
   the actual tech stack.
2. `docs/ARCHITECTURE.md` — the data flow, both loops (hot path decisioning,
   cold path drift), the folder structure, local infra substitutions.
3. `docs/AGENTS.md` — every agent, what it does, what phase it belongs to,
   what evidence it must emit.
4. `docs/DATABASE.md` — the schema, and why `decision_evidence` is the
   hinge of the entire system.
5. `docs/DECISIONS.md` — why things are the way they are. Read this fully;
   it prevents you from "helpfully" undoing a deliberate tradeoff.
6. `docs/ROADMAP.md` — what phase we're in, what's done, what's next.
7. `docs/SESSION_LOG.md` — read at minimum the LAST entry. It tells you
   exactly where the previous session left off and what to do next.
8. `docs/TODO.md` — the current, present-tense task list.
9. `docs/CODING_RULES.md` — behavioral rules. Treat violations as bugs.
10. `docs/API.md` and `docs/PROMPTS.md` — reference as needed for the
    specific endpoint or prompt you're touching.

The full original specification is `docs/ComplianceLoop_Production_Blueprint_v2.md`
— when anything in the files above seems to conflict with it, the blueprint
wins, and that conflict should become a `DECISIONS.md` entry to resolve it.

## After reading, before writing any code

**Summarize your understanding in your own words**: what phase we're in,
what the immediate task is, and what you must NOT touch (per `TODO.md`'s
scope and `CODING_RULES.md`'s rule 9 — don't build ahead of the current
phase). If anything is ambiguous, ask instead of assuming.

## Standing rules
- Do not modify architecture, schema, API shapes, or prompts unless
  explicitly asked, or unless it's clearly and narrowly inside the current
  task's scope.
- If unsure, ask instead of assuming.
- Preserve existing coding style and conventions (`CODING_RULES.md`).
- At the end of the session, generate a handoff and append it to
  `SESSION_LOG.md`, and update `TODO.md` — see `BOOTSTRAP_PROMPT.md`.