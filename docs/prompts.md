# PROMPTS.md
*Every prompt used anywhere in this system, in one place. If a prompt only
lives inside a Python string in some agent file, it will drift and nobody
will notice. Copy it here whenever it's written or changed, and keep
`prompts/` (the actual loaded-at-runtime files) as the executable source —
this doc is the human/AI-readable index of what exists and why.*

## Status
Phase 0 has **no LLM calls** (Document Agent and Temporal Agent are pure
rule evaluation). This file is a skeleton until Phase 1, when the Decision
Agent's system prompt is the first real entry. Don't let a session invent
LLM calls in Phase 0 agents — see `AGENTS.md` and `CODING_RULES.md`.

---

## Template for every prompt entry
```
### <Agent/Purpose name>
**File:** prompts/<filename>.txt (or wherever it's actually loaded from)
**Used by:** <agent name>
**Phase introduced:** N
**Purpose:** one line
**Prompt:**
    <the actual prompt text>
**Notes:** anything about why it's phrased this way, known failure modes,
citation requirements it enforces, etc.
```

---

## Planned prompts (Phase 1 onward — placeholders, not yet written)

### Decision Agent — system prompt
**Used by:** Decision Agent
**Phase introduced:** 1
**Purpose:** Synthesize upstream agent outputs into a final decision, with a
hard requirement that every claim cites a `decision_evidence`-eligible
source. Must explicitly refuse to state a compliance claim it cannot
attribute to retrieved evidence — no uncited assertions, ever (this is the
enforcement point for `PROJECT.md`'s "evidence citation, not vibes"
principle).
**Provider note:** runs against the Groq API (see `DECISIONS.md`). Groq
models may follow structured-output/citation instructions differently than
larger frontier models — the prompt and the citation validator (below) may
need more explicit formatting instructions and stricter post-hoc validation
than a first draft assumes. Test this early, don't assume it "just works."
**Status:** not yet written.

### RAG Agent — retrieval query construction
**Used by:** RAG Agent
**Phase introduced:** 1
**Purpose:** Turn case context into an effective FAISS query against the
regulatory corpus.
**Status:** not yet written.

### Citation validator — post-hoc check
**Used by:** Decision Agent (or a wrapper around it)
**Phase introduced:** 1
**Purpose:** Verify every claim in the Decision Agent's output actually maps
to a `CITED` evidence row before the transaction commits — a second pass
that can reject/retry the LLM's output if citations are missing or invalid.
**Status:** not yet written.

### Diff Engine — change severity/topic classification
**Used by:** Diff Engine (Loop 2)
**Phase introduced:** 3
**Purpose:** Classify a chunk-level regulatory diff by severity and topic so
the Impact Resolver knows how urgently to act.
**Status:** not yet written.

---

## Rules for this file
- Never let a prompt exist in code without an entry here.
- When a prompt changes materially (not a typo fix), add a `DECISIONS.md`
  entry — prompt changes can silently change decision behavior across the
  whole system and deserve the same scrutiny as a schema change.