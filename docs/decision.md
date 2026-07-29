# DECISIONS.md
*The most important file in this repo. Every time something changes,
write an entry here BEFORE moving on. This is how a new session knows WHY
things are the way they are, not just what they are.*

**Format:**
```
## YYYY-MM-DD — <short title>
**Decision:** what changed
**Reason:** why
**Impact:** what else had to change / what to watch out for
```

---

## 2026-07-29 — Use Redpanda instead of managed Kafka for local dev
**Decision:** Run Redpanda (Kafka-API compatible) in Docker Compose instead of
AWS MSK/Confluent Cloud.
**Reason:** Solo developer, free-tier cloud credits only. Redpanda is a
single container, free, and speaks the same Kafka protocol.
**Impact:** All producer/consumer code should target the standard Kafka
client API, not a Redpanda-specific one — this keeps the swap to a managed
service at deploy time a pure config change (`KAFKA_BOOTSTRAP_SERVERS`), not
a code change.

## 2026-07-29 — Self-host Temporal instead of Temporal Cloud
**Decision:** Run Temporal server via Docker Compose locally.
**Reason:** Same solo/free-tier constraint. Temporal Cloud costs money;
self-hosted Temporal uses the identical SDK.
**Impact:** No code impact — only the connection target (`TEMPORAL_HOST`)
changes if this later moves to Temporal Cloud. Operational maturity
(multi-region Temporal) is explicitly deferred to Phase 5+.

## 2026-07-29 — Local Postgres instead of Aurora multi-AZ
**Decision:** Postgres runs in Docker locally for Phases 0–4; move to
Supabase/Neon free tier only when an always-on instance is needed (e.g., for
a demo or early pilot).
**Reason:** Multi-AZ failover is not achievable or meaningfully testable
solo; ACID transaction guarantees and the outbox pattern work identically on
a single local instance.
**Impact:** DR/failover testing is explicitly out of scope until Phase 5.
This is a deferred capability, not a silently dropped one — see
`ROADMAP.md` Phase 5.

## 2026-07-29 — MinIO instead of AWS S3 for local dev
**Decision:** Object storage (regulatory documents, application documents)
uses MinIO locally, S3-compatible API.
**Reason:** Free, S3 SDK-compatible, no cloud account needed for Phase 0–4
development.
**Impact:** Code must use the standard S3 SDK (boto3 or equivalent), not a
MinIO-specific client, so the endpoint swap to real S3 at deploy is
config-only.

## 2026-07-29 — Local ED25519 keypair instead of Cloud KMS for signing
**Decision:** Decision/audit signing uses a locally generated ED25519
keypair in dev; real Cloud KMS integration is deferred to deploy time.
**Reason:** Cloud KMS requires a provisioned cloud account and has real
cost; the *signing logic and Merkle-chain logic* are identical regardless of
which signer backend is behind `security/signer.py`.
**Impact:** `security/signer.py` must be written as an interface with a
swappable implementation from day one — never hardcode local-key assumptions
into calling code. This is a hard rule, see `CODING_RULES.md`.

## 2026-07-29 — Kubernetes deferred to Phase 5; Docker Compose for Phases 0–4
**Decision:** No Kubernetes manifests written or used until Phase 5.
**Reason:** Multi-node-pool isolation and cluster ops are not solo/free-tier
achievable in a way that's meaningfully different from Docker Compose's
resource limits for a single-developer local environment.
**Impact:** `infra/k8s/` stays empty (with a README stub) until Phase 5.
Don't let Cursor scaffold Kubernetes manifests before then — flag it if it
tries.

## 2026-07-29 — Standardize on Anthropic Claude, not OpenAI
**Decision:** All LLM calls (Decision Agent, future RAG synthesis, etc.) use
the Anthropic API (Claude Sonnet/Haiku), not OpenAI.
**Reason:** Project owner's stated preference and existing tooling (Claude
Code, Cursor + Claude workflow).
**Impact:** `ANTHROPIC_API_KEY` is the one required real external secret from
Phase 1 onward. Use Haiku during dev/testing to control cost; reserve Sonnet
for cases where citation-quality reasoning matters most.

---

*Add new entries above this line, most recent at the top of the log or
bottom — pick one convention and stay consistent. (This file currently lists
oldest-first as decisions were made; keep appending downward.)*