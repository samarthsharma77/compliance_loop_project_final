# BOOTSTRAP_PROMPT.md
*Paste this into Cursor (or any fresh Claude session) FIRST, every time you
start a new session or switch accounts/models. It's the standing "wake up
and get context" instruction.*

---

```
You are continuing development of ComplianceLoop.

Before making any changes:

1. Read docs/MASTER_CONTEXT.md.
2. Read docs/PROJECT.md.
3. Read docs/ARCHITECTURE.md.
4. Read docs/AGENTS.md.
5. Read docs/DATABASE.md.
6. Read docs/DECISIONS.md.
7. Read docs/ROADMAP.md.
8. Read docs/SESSION_LOG.md — at minimum the most recent entry.
9. Read docs/TODO.md.
10. Read docs/CODING_RULES.md.

After reading, summarize the current system and the current task in your
own words before writing any code.

Rules for this session:
- Do not modify architecture, database schema, API shapes, or prompts
  unless explicitly asked, or unless it's clearly and narrowly inside the
  current task's scope per TODO.md.
- Do not build ahead of the current phase in ROADMAP.md, even if it seems
  efficient to "do it while you're in there."
- If unsure about anything, ask instead of assuming.
- Preserve existing coding style and conventions.
- Every new agent, endpoint, or table gets a corresponding doc update in
  the same session — not deferred to later.

At the end of this session, generate a session handoff and append it to
docs/SESSION_LOG.md using the template at the top of that file. Include:
- Summary of what was worked on
- Files modified
- Any architecture/schema/prompt changes (and whether DECISIONS.md was
  updated accordingly)
- Pending issues / anything left broken or half-done
- Things the next session should NOT modify without asking
- Next steps, and the literal prompt to paste to resume

Also update docs/TODO.md to reflect the real current state before ending
the session.
```

---

## When to use this
- Every time you open a fresh Cursor Agent/Composer session.
- Every time you switch Cursor accounts or Claude models.
- Any time a session seems to have lost context or is about to start a
  meaningfully new chunk of work.

## Why this matters
Without this, a new session either re-derives the architecture from
scratch (slow, and prone to drifting from the blueprint) or worse, silently
assumes something that was already deliberately decided against — see
`docs/DECISIONS.md` for exactly the kind of tradeoffs that get accidentally
"fixed" by a session that skipped this step.