# AGENTS.md
*Every agent, described the same way, so a new session doesn't have to infer behavior from code.*

All agents implement the shared interface in `agents/base.py`:
`async run(input) -> AgentResult`, where `AgentResult` carries `status`,
`output`, and a list of `evidence` entries (`type`, `id`, `version`, `role`).
No agent is exempt from emitting evidence, even ones that don't call an LLM.

---

## Document Agent
**Phase:** 0
**Responsibility:** Validate presence and format of required KYC artifacts
against a checklist ruleset.
**Input:** Applicant's uploaded documents (references, not raw bytes — bytes
live in MinIO/S3).
**Output:** Structured JSON — per-document status (present/missing/malformed),
overall completeness verdict.
**Uses:** Ruleset version lookup (mocked in Phase 0, no LLM call).
**Evidence emitted:** ruleset version cited for each check.
**Status:** not yet implemented.

## Temporal Agent
**Phase:** 0
**Responsibility:** Check document expiry / validity windows against today's
date (e.g., ID proof not older than N months per current RBI KYC norms).
**Input:** Document metadata (issue date, type).
**Output:** Structured JSON — per-document validity verdict.
**Uses:** Pure rule evaluation. No I/O, no LLM.
**Evidence emitted:** the specific expiry rule/window applied.
**Status:** not yet implemented.
**Naming note:** "Temporal Agent" here means *time/date validity checking*,
unrelated to the Temporal workflow engine used elsewhere in this project.
Don't conflate the two when reading logs or code.

## Sanctions Agent
**Phase:** 1
**Responsibility:** Screen applicant identity against sanctions/watchlists
(mocked list in dev — no real bureau/sanctions API access yet).
**Input:** Applicant identity fields.
**Output:** Match/no-match verdict, confidence score.
**Uses:** Fuzzy name matching against a fixture dataset.
**Evidence emitted:** watchlist version/source, matched record if any.
**Status:** not yet implemented.

## Transaction Agent
**Phase:** 1
**Responsibility:** Analyze transaction/bureau data for risk signals (mocked
bureau data in dev).
**Input:** Transaction history / bureau report reference.
**Output:** Risk signal summary, flagged patterns.
**Uses:** Rule-based checks initially; may incorporate an LLM pass later for
narrative summarization — evidence discipline applies either way.
**Evidence emitted:** specific transaction records or bureau fields cited.
**Status:** not yet implemented.

## RAG Agent
**Phase:** 1
**Responsibility:** Retrieve relevant RBI circular clauses for the case at
hand from the regulatory corpus.
**Input:** Case context (application type, applicant category, flagged
issues from other agents).
**Output:** Ranked list of candidate clauses with similarity scores.
**Uses:** FAISS local vector index over regulatory text chunks embedded with
`sentence-transformers` (`BAAI/bge-small-en-v1.5`, local CPU, no API key —
see `DECISIONS.md`).
**Evidence emitted:** every retrieved chunk is logged with `role=RETRIEVED`;
only chunks the Decision Agent actually uses get promoted to `role=CITED`.
**Status:** not yet implemented.

## Decision Agent
**Phase:** 1
**Responsibility:** Synthesize outputs from all other agents into a single
compliance decision, citing the specific evidence (documents, clauses,
transaction records) that justifies it.
**Input:** All upstream agent outputs + RAG Agent's retrieved clauses.
**Output:** Final decision (approve/reject/refer-to-human), confidence score,
citation list.
**Uses:** Groq API (see `DECISIONS.md` for why this replaced Anthropic).
Must fail closed — if it cannot produce a citation for a claim, it cannot
make that claim part of the decision. Citation-enforcement behavior must be
verified against Groq's actual output patterns, not assumed from prior
provider experience.
**Evidence emitted:** promotes RAG-retrieved evidence to `CITED` where
actually used; this is the row that closes the loop for §1's "evidence
citation, not vibes" principle.
**Status:** not yet implemented.

---

## Loop 2 agents (cold path — Phase 3+)

### Sentinel
**Phase:** 3
**Responsibility:** Poll RBI/regulatory sources for new or changed documents.
**Output:** New/changed-document events.
**Status:** not yet implemented.

### Diff Engine
**Phase:** 3
**Responsibility:** Chunk-level diff between old and new circular versions;
classify severity and topic of each change.
**Output:** Structured diff with severity/topic tags per changed chunk.
**Status:** not yet implemented.

### Impact Resolver
**Phase:** 3
**Responsibility:** Query `decision_evidence` for every past decision that
cited a now-changed clause; produce the Affected Decision Set (ADS).
**Output:** ADS — list of decision IDs + which evidence made them affected.
**Status:** not yet implemented.

### Recompute Orchestrator
**Phase:** 4
**Responsibility:** Selectively re-run only the affected agents (not the
whole pipeline) for each decision in the ADS; canary a subset before wider
rollout; route results through human confirmation (Temporal
`recompute_workflow`).
**Status:** not yet implemented.

---

## Adding a new agent — checklist
1. Implement `agents/base.py`'s `Agent` interface.
2. Every claim in `output` that isn't a raw pass-through of input must have a
   corresponding `decision_evidence` row.
3. Add an entry to this file before or alongside the code — don't let an
   agent exist in code without a description here.
4. Register it in `orchestrator/graph.py`; note whether it's parallel
   fan-out or depends on another agent's output first.