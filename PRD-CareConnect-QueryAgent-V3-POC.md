# Product Requirements Document
## CareConnect Query Agent V3 — Proof of Concept

| Field | Value |
|---|---|
| Document Version | 1.1 |
| Status | Draft — Post-ARB Revision |
| Product | CareConnect Query Agent |
| Release | V3 POC |
| POC Phase 1 Target | 50 care managers |
| POC Phase 2 Target | 200 care managers |
| Prepared By | Engineering |
| Last Updated | 2026-06-23 |

---

## 1. Executive Summary

CareConnect care managers currently query member records and care gap analytics through a conversational AI agent (V2) that costs approximately $7,968 per year at 500 conversations per day and relies on Claude Sonnet via an LMS gateway. The V3 POC replaces the underlying LLM with Gemini Flash via Google AI API, introduces Google AI context caching to reduce per-request token costs by 75%, and upgrades the query cache from exact-match to semantic similarity using pgvector — enabling paraphrase matching and raising expected cache hit rate from 30% to 50%.

The projected **LLM spend** after the migration is ~$432/year, a 94% reduction against V2 (LLM-only comparison; V2's $7,968 was also LLM-only). Full system cost on the existing CVS infrastructure is ~$3,000–$4,056/year (Scenario A in `COST-ESTIMATION-V3.md`), driven by GKE compute that V2 also paid; the LLM line item is what shrinks by 94%. Accuracy is preserved and made structurally robust: an eval gate blocks any deployment that falls below 80% on the golden test set, and a runtime fallback escalates low-confidence queries to Gemini 1.5 Pro automatically.

This document defines the requirements, acceptance criteria, user stories, and delivery timeline for the V3 POC.

---

## 2. Problem Statement

### 2.1 Cost Problem

V2 uses Claude Sonnet for both source routing and NL-to-query translation. Sonnet is priced at a rate that makes the system expensive at scale. At the projected 500 conversations per day, V2 costs $7,968 per year — $0.0615 per cache miss — solely from LLM token spend. There is no context caching in V2, meaning the full schema (system prompt + schema definition + few-shot examples, approximately 15,000 tokens) is paid on every call.

### 2.2 Cache Effectiveness Problem

V2 uses an exact-hash cache keyed on the normalized question and schema version. This means that "how many open care gaps do I have?" and "what is my total count of open care gaps?" each pay the full LLM cost despite being semantically identical. The cache hit rate sits at approximately 30% as a result. Raising this to 50% through semantic matching would alone reduce monthly spend by 20% before any model switch.

### 2.3 Architecture Rigidity Problem

V2 uses a hardcoded function pipeline with no crash recovery and no composability interface. It cannot be orchestrated by a future Supervisor agent, cannot be called as a LangChain tool by a Notification or Escalation agent, and cannot recover mid-pipeline from an infrastructure failure. This is a ceiling on the multi-agent roadmap.

### 2.4 Infrastructure Fragmentation Problem

V2 has the LangGraph agent on Cloud Run and the MCP execution server on GKE, connected via a VPC serverless connector. This split adds operational complexity, requires a separate Cloud Scheduler job for cache warm-up, and means the agent and MCP server cannot communicate over cluster-internal DNS. V3 consolidates both onto GKE, removing Cloud Run and Cloud Scheduler entirely.

---

## 3. Goals

### 3.1 Primary Goals (POC must validate all of these)

1. Reduce LLM cost per request by ≥90% compared to V2 while maintaining ≥80% accuracy on the golden test set.
2. Raise semantic cache hit rate from ~30% (V2 exact-match) to ≥50% using pgvector similarity matching.
3. Validate that Gemini Flash — with context caching, few-shot examples, and runtime fallback to Gemini 1.5 Pro — can match V2 accuracy on the existing golden test set.
4. Demonstrate crash recovery from any pipeline node using LangGraph + PostgreSQL checkpointer.
5. Deliver the first working multi-agent interface (`as_subgraph`, `as_tool`) for future agent composition.

### 3.2 Secondary Goals (validated post-POC)

6. Confirm the architecture scales to 200 concurrent users without latency regression beyond the p95 budget.
7. Produce the `tier_2_escalation_rate` SLO baseline for future production alerting.
8. Validate K8s CronJob as a replacement for Cloud Scheduler for context cache warm-up.

### 3.3 Non-Goals

- Full production rollout is out of scope for this POC.
- Multi-agent orchestration (Notification, Escalation, Analytics, Supervisor agents) is out of scope; only the interface contracts are delivered.
- MongoDB Atlas migration or schema changes are out of scope; V3 connects to the existing Atlas cluster.
- UI or portal changes are out of scope; the POC exposes the same API contract as V2.

---

## 4. Background

### 4.1 Version History

V1 used Gemini Pro with four LLM calls per request (query planning, routing, translation, validation) and a Redis exact-match cache. Annual cost at projected volume was ~$18,000, with $0.1318 per cache miss.

V2 replaced V1's redundant LLM calls with a tighter two-call pipeline (route + translate) and migrated to Claude Sonnet via the CVS LMS Gateway. Cost dropped to ~$7,968 per year, but the model choice and the absence of context caching left significant cost reduction on the table.

V3 makes five changes simultaneously: switches to Gemini Flash, adds Google AI context caching for both MQL and SQL translator prompts, upgrades the query cache to semantic similarity via pgvector, restructures the orchestration from a hardcoded function sequence to a LangGraph named-node pipeline, and consolidates all compute onto GKE.

### 4.2 Why Now

The existing Cloud SQL instance already runs PostgreSQL, and enabling the pgvector extension costs nothing — the semantic cache upgrade has no new infrastructure dependency. The Gemini Flash and Google AI context caching APIs are already in use elsewhere at CVS Health, meaning no new vendor approval is needed. The engineering team has the LangGraph and LangChain expertise from V2. The design is complete and the migration path is staged, making this a low-risk execution exercise.

---

## 5. Solution Overview

### 5.1 Architecture Summary

The V3 agent is a structured LangGraph pipeline deployed as a GKE Deployment. It processes natural language questions from care managers through a fixed sequence of named nodes. Two nodes call LLMs (source router and translator, both Gemini Flash). All other nodes are deterministic — no LLM reasoning overhead between steps.

The pipeline runs inside a single GKE cluster alongside the MCP Server, which handles all database execution (MongoDB and BigQuery) using Workload Identity for credential-free authentication. A Kubernetes CronJob runs the context cache warm-up hourly. The two services communicate over cluster-internal K8s Service DNS, eliminating the VPC connector and reducing inter-service latency.

### 5.2 Request Flow

A care manager submits a natural language question. The input security gate (deterministic) runs first — always — checking scope, detecting injection attempts, and stripping PII. The semantic cache is checked next: exact hash first, then vector similarity against stored embeddings. A cache hit routes directly to execution, skipping all LLM calls. A cache miss proceeds to the source router (LLM call 1 — Gemini Flash classifies MongoDB vs BigQuery), then the translator (LLM call 2 — Gemini Flash generates MQL or SQL against cached schema context), then the accuracy check, then the four-stage query validator. After execution, the output security gate runs — always — scrubbing PHI, capping rows, confirming scope, and checking Bounds Registry values. State and audit records are written asynchronously after the response is yielded.

### 5.3 Key Design Decisions

The structured pipeline (not a ReAct agent loop) eliminates 4–5 LLM reasoning calls per request with no loss in capability for a well-understood pipeline. Security gates are LangGraph nodes sitting outside the LLM path — the model cannot influence, skip, or route around them. The query stored in cache is a template with `:user_id` and `:caseload_scope` as bind parameters, substituted at execution time by the MCP executor — enabling cross-user cache reuse without data leakage. When Gemini Flash returns low confidence (routing confidence below 0.7, UNCERTAIN verdict, or a failed repair loop), the system automatically escalates that single translation to Gemini 1.5 Pro using the same ChatGoogleGenerativeAI interface — no provider hop, no extra auth surface.

---

## 6. Functional Requirements

### 6.1 Natural Language Query

The system must accept a natural language question from an authenticated care manager and return a structured answer derived from either MongoDB (member records) or BigQuery (analytics aggregates), based on the semantic content of the question.

The system must handle at minimum five intent types: count, filter, trend, ranking, and lookup. The source router must correctly classify the data source for ≥85% of golden test set questions to maintain overall accuracy above the 80% deployment gate.

### 6.2 Semantic Cache

The cache must implement two-tier lookup: exact hash match on normalized question plus schema version first (zero embedding cost), followed by pgvector cosine similarity against stored embeddings if the exact match misses. A similarity threshold of 0.95 is required. The cache must use HNSW indexing (not ivfflat) given the projected row count of approximately 250 rows under 24-hour TTL at 500 conversations per day.

Cache entries must be invalidated on every schema version or Bounds Registry version bump, using a DELETE statement that selectively removes rows whose schema_version or bounds_version does not match the current version. This must run as a pre-deploy Kubernetes Job (spec.parallelism: 1) that completes before the rolling update begins. Running the DELETE as a pod startup hook would allow old pods to continue serving stale queries during the rolling update window — the pre-deploy Job approach guarantees the DELETE finishes before any new pod receives traffic.

Question embeddings must be pre-computed in the load_context node and stored in AgentState.question_embedding. The semantic_cache_node performs only the vector similarity lookup against the pre-computed embedding — it does not call the embedding API. Pre-computing the embedding in load_context ensures the embedding call latency is not on the cache-hit critical path and keeps the semantic cache lookup step deterministic and fast.

### 6.3 Security Gates

The input security gate must run before any other node and must run on every request including cache hits. It performs three checks in order:

1. **Scope classification** — deterministic regex/keyword classifier against the CareConnect domain vocabulary (no LLM call). Out-of-scope questions are refused before any further processing.
2. **PII strip** — deterministic regex-based removal of obvious member identifiers (name patterns, MRN format, DOB) from the question text *before* the question can be sent to any downstream LLM call. This is the first PHI control.
3. **Injection detection** — Gemini Flash classifier call with a hardened, short prompt (~500 tokens in, ~50 tokens out). Probabilistic, not deterministic; the planned future replacement is a local classifier (Meta `prompt-guard-86m` or equivalent) running in-process, which would make this step deterministic. For the POC, the Flash classifier is acceptable provided the false-negative rate on the golden injection test set stays at or below 1%; this threshold replaces the prior "Zero false-negative" wording (which is structurally unachievable for any probabilistic classifier).

Order matters: scope and PII strip run first because they are deterministic and can short-circuit cheaply. The injection-detect Flash call runs only on scope-passing, PII-stripped questions — so the payload sent to Gemini for injection screening never contains the unredacted question.

The output security gate must run after every execution and must run on every request that reaches the MCP executor. It performs four checks: PHI scrubbing (remove protected health information not within the user's access level), row capping (100 rows for ad-hoc, 1,000 for exports), scope confirmation (every row belongs to this user's caseload), and Bounds Registry sanity checking (values within expected clinical ranges).

The translator must be prompted to select only the fields required to answer the care manager's stated intent — no broad SELECT * or equivalent MQL projections that return the full document. The output security gate enforces a per-intent field allowlist: each intent type (count, filter, trend, ranking, lookup) has an approved set of returnable fields. Any field not on the allowlist for the detected intent type is stripped from the response before delivery to the care manager, even if the query returned it from the database.

### 6.4 Accuracy-First Fallback

When the source router returns routing_confidence below 0.7, or the translator returns a CONFIDENT_VERDICT of UNCERTAIN, or the repair loop exhausts its single retry, the system must automatically re-run the translation using Gemini 1.5 Pro via the same ChatGoogleGenerativeAI interface. The escalated query must still pass through the accuracy check, validator, and output gate — escalation bypasses no safety control. The escalation rate must be tracked as a named SLO (`tier_2_escalation_rate`) and exposed on the engineering dashboard.

### 6.5 Repair Loop

If the accuracy check or query validator flags a repairable issue (hallucinated field, schema misalignment), the system must retry the translation once with a **sanitized** error message injected into the translation prompt. The router decision must not be re-run on a repair — only the translator is retried, preserving the routing classification. The repair count is hard-capped at 1; a second failure routes to audit without further retries.

**Error sanitization (PHI control).** The error message added to the repair prompt must never contain MCP result data or query bind values. It carries only structural information: the offending field name as it appeared in the query, the validator stage that fired, and the expected schema constraint. Any value derived from a row, a user identifier, or a caseload scope is replaced with a placeholder before the error string is concatenated into the prompt. This is enforced by a `sanitize_repair_context()` helper that the translator calls; bypassing it on any code path is a hard architectural violation. The repair cap is enforced in two places — at the conditional edge *and* as a node-level guard at the top of `translator_node` — so a future edge-logic change cannot reintroduce an unbounded retry.

### 6.6 Context Cache Warm-Up

A Kubernetes CronJob must refresh both MQL and SQL context caches every 45 minutes. The same job must be triggered at deploy time and on every schema or few-shots push. The caches carry a 75-minute TTL. A 45-minute refresh interval against a 75-minute TTL guarantees a minimum 30-minute overlap between the new cache and the expiring one, eliminating the jitter gap that would occur with a 1-hour interval on a 1-hour TTL. A missed warm-up cycle causes the next translation to pay the full uncached token rate, not a failure — degraded performance is acceptable, not a system failure.

**Active cache name registry (cross-pod coordination).** The CronJob is the **sole creator** of `CachedContent` objects. After each successful refresh, the CronJob writes the active cache name for each source (MQL, SQL) into a Cloud SQL `context_cache_registry` table (one row per source, last-updated timestamp). The translator function reads the active cache name from the registry at call time (with a short in-process TTL, e.g. 60s, to bound lookup cost). Agent pods never call `CachedContent.create()`. This eliminates per-pod orphaned caches caused by pod restart or HPA scale-out — every pod uses the same active cache name, and the CronJob is responsible for deleting superseded caches one TTL after rotation to bound storage cost.

**CronJob failure alerting.** Three consecutive missed CronJob cycles must raise a P2 alert. Translator latency and uncached-token cost are leading indicators and must also alert when they breach the cache-miss baseline by >25% sustained over 15 minutes.

### 6.7 Multi-Agent Interface

The agent must expose three interfaces for future agent composition: `as_subgraph` (LangGraph subgraph, for supervisor-agent graphs), `as_tool` (LangChain tool, for LLM agents with tool-calling), and an MCP endpoint (`POST /mcp/v1/tools/careconnect_query`). These interfaces must be deliverable within the POC scope. They do not need to be connected to other agents during the POC — only the contracts and export functions need to exist and be tested in isolation.

---

## 7. Non-Functional Requirements

### 7.1 Latency

| Path | p95 Target |
|---|---|
| Cache hit — exact match (post-injection detect) | < 700ms |
| Cache hit — semantic match (embedding pre-computed in load_context) | < 1.0s |
| Cache miss (full pipeline, no Pro escalation) | < 3.0s |
| Cache miss with Pro escalation | < 5.0s |
| Injection detection (Flash) alone | < 300ms |
| MCP execution (GKE → DB) | < 500ms |
| Output security gate | < 50ms |
| save_state + audit (synchronous, in-graph) | < 150ms |

Latency is measured end-to-end from authenticated API request to formatted response returned. `save_state` and `audit` run **synchronously** inside the LangGraph compiled pipeline (see §7.3); they are inside the p95 budgets above. Prior drafts described them as "fire-and-forget" — that pattern is incompatible with the LangGraph checkpointer and with the HIPAA durability requirement for the audit record, so the design is now synchronous.

The budgets above account for the existing `careconnect-gke` cluster running in **us-east4** while the Google AI API endpoint lives in us-central1. The Sprint 0 pre-flight (§13) includes a measured Gemini Flash p50/p95 round-trip from us-east4 — if the measured value materially exceeds the implicit 50ms per-call assumption baked into the budgets above, the budgets are re-pinned to the measurement and Section 9 success criteria updated accordingly. No region change is in scope for the POC.

### 7.2 Accuracy

Accuracy is the percentage of questions on the golden test set where the generated query returns the expected result. The deployment gate requires ≥80% accuracy on every release. This gate is enforced by CI/CD and must block any merge that does not meet the threshold.

The Flash fallback to Gemini 1.5 Pro means the system's runtime accuracy floor is higher than the 80% deployment gate — escalated queries get the highest-quality translation available. The POC should produce an empirical baseline for the tier_2_escalation_rate.

When the translator returns a confidence_verdict of UNCERTAIN, the response must display an explicit on-screen warning banner to the care manager. A metadata flag in the JSON response is insufficient — the UI must surface the warning visibly. The banner must identify the answer as unverified and advise the care manager to confirm against a source report before acting on the information.

The 80% accuracy deployment gate threshold is a clinical safety boundary, not only a technical benchmark. Clinical risk acceptance sign-off from the responsible clinical SME is required before Phase 1 onboarding begins.

**Per-class harm taxonomy.** The 80% aggregate gate is supplemented with per-class harm tiering that the clinical SME signs off on individually before Phase 1:

| Harm tier | Example failure modes | Minimum class accuracy gate |
|---|---|---|
| Tier 1 — high-harm | Member identity confusion (wrong member returned); care-gap status inversion (closed reported as open or vice versa); medication or condition field returned outside the user's caseload | ≥ 95% |
| Tier 2 — moderate-harm | Count off by one or more; trend direction wrong; ranking order wrong | ≥ 85% |
| Tier 3 — low-harm | Cosmetic formatting issues; correct answer with extra non-PHI metadata | ≥ 75% |

The aggregate 80% gate is a floor; failing any per-class minimum is a hard block regardless of aggregate score. The eval harness must report per-class accuracy in addition to aggregate.

**Runtime accuracy circuit breaker.** If the weekly evaluation run on production traffic samples drops below 80% aggregate or below any tier minimum, the system auto-degrades the affected intent/source class to refusal mode and emits a P1 alert; the engineering team must restore accuracy before traffic to that class resumes. This protects against a regression that ships past the CI gate due to golden-set/production-distribution mismatch.

**Statistical validity of the gate.** A golden test set of N = 50 supports a 95% CI of roughly ±11pp on an 80% point estimate — wider than is comfortable for a clinical gate. The golden set must be expanded to N ≥ 150 (≥ 30 per intent class × 5 intents, stratified across MongoDB and BigQuery at ≥ 20% BigQuery) before Phase 1 onboarding. Until expansion is complete, the gate is treated as advisory and accuracy outcomes are reviewed by the SME each weekly run.

### 7.3 Availability

The POC does not require a formal SLA. The system must handle individual component failures gracefully:

- **Gemini API timeout.** The translator and router calls use a hard timeout (5s default for Flash, 10s for Pro escalation). On timeout, the request returns a user-visible error with a request ID; it must not hang. Repair-loop retries do not stack with API retries — total wall time for a single request is capped at 15s.
- **Cloud SQL slowness (not full failure).** Cloud SQL is on the hot path for `load_context`, semantic cache lookup, LangGraph checkpoint writes, and `save_state`. Connection pool acquire timeout is set to 1s; query timeout is 2s. On timeout, the agent **fails fast** to a user-visible error rather than queuing; semantic cache lookup timeout falls through to cache-miss path (i.e., translate) rather than blocking. Cloud SQL HA is recommended at production rollout; POC accepts non-HA with this documented degraded-mode behavior.
- **Cloud SQL unavailability.** In-flight LangGraph checkpoints that have already been written are durable; the checkpointer's transactional semantics handle this. Checkpoints not yet written are lost — this is acceptable because the user will see an error and can retry. No silent data loss.
- **Audit durability.** Audit writes run **synchronously inside the LangGraph audit node** (BigQuery insert). On BigQuery failure, the audit record is written to a Cloud SQL `audit_dead_letter` table by the same node — also synchronous, also inside the request. The dead-letter table is drained by the existing CronJob infrastructure (a new CronJob worker, no new infra service) on a 5-minute cadence. On pod SIGTERM, the FastAPI `lifespan` shutdown handler awaits in-flight LangGraph executions to completion with a 10-second drain budget before the process exits — so a rolling deployment never silently drops an audit record.

A break-glass procedure must be documented and tested before Phase 1 onboarding. The procedure must address the scenario where the agent is fully unavailable (pod crash loop, Gemini API regional outage, Cloud SQL failure). It must specify: the on-call engineering contact responsible for declaring the agent degraded, the fallback query surface available to care managers (direct BigQuery console access or a pre-built Looker dashboard covering the most common intent types), the verified care-manager training that the fallback surface is usable without engineering support, and the criteria for declaring the agent degraded versus temporarily slow. The break-glass document must be published in the engineering runbook repository and linked from the Phase 1 onboarding checklist — not stored only in code comments or informal team channels.

### 7.4 Security and Compliance

**Credential model.** All GKE workloads authenticate to GCP services (Cloud SQL, BigQuery, Secret Manager, the Google AI Generative API) via Workload Identity and ADC — no long-lived GCP credentials in any container. The Google AI Generative API key is held in Secret Manager and read at pod startup. MongoDB Atlas does **not** integrate natively with GCP Workload Identity; Atlas credentials are stored in Secret Manager, pulled at pod startup, and held in process memory only (never written to disk). Atlas credentials are rotated at most every 90 days; the rotation runbook is published with the deploy runbook before Phase 1. X.509 certificate authentication to Atlas is preferred and will be evaluated for adoption in Phase 2.

**HIPAA BAA note.** The POC uses the Google AI Generative API (`google.generativeai`). The BAA status of this surface for CVS Health is being handled separately by Legal/Compliance and is explicitly **out of scope for this POC** — Phase 1 cannot onboard real care managers until BAA status is confirmed (path is either a Google AI BAA confirmation or a future move to Vertex AI; either is a follow-on workstream). All other PHI controls described below are unaffected by the BAA decision and proceed in parallel.

**PHI controls.**
- PHI must never appear in Cloud Trace spans, Grafana dashboards, Cloud Monitoring metric labels, or LangGraph checkpoint rows. The LangGraph PostgresSaver is configured with a custom `serde` that strips `query_results` and `formatted_response` from `AgentState` before the checkpoint write; only the structural fields (status enums, routing decision, validation flags) are persisted. The same applies to Cloud Trace span attributes.
- The BigQuery audit log is the PHI-safe compliance trail and must retain records for **six years** (45 CFR 164.530(j) baseline). The audit log records question hash, user_id, source, outcome, trace_id — never raw question text or result data.
- A pre-deploy CI check (PHI scan) validates that no recently-changed code path writes `state.query_results` or `state.formatted_response` to Trace/Monitoring/Cloud SQL; the check fails the merge if a write is detected.

**No PHI in any Gemini API call.** PHI-bearing MCP execution results must never be included in any Gemini API call payload. This prohibition covers translation prompts, repair loop context, Pro escalation inputs, and any future LLM call added to the pipeline. MCP results flow directly from the MCP executor to the output security gate — no result data is passed back through the LLM. The translator receives only the system prompt, schema, few-shot examples, and the care manager's PII-stripped natural language question. This is a hard architectural constraint, not a convention.

**Semantic cache key PII strip.** Because semantic cache lookup is cross-user-template-safe but the *question text* is what gets embedded, the embedding is computed on the **PII-stripped, normalized** question — not the raw question. The exact-hash key is also computed on the PII-stripped, normalized question. This prevents a question containing a member name from becoming a discoverable cache key for any other user.

**Rate limiting.** The FastAPI endpoint enforces per-user (30 req/min) and global (500 req/min) rate limits backed by a Cloud SQL `rate_limit_counter` table (single row per user, single row global, with a one-minute rolling window) rather than per-pod in-memory counters. This makes the limit hold under HPA scale-out. Requests exceeding either limit return HTTP 429 with a `Retry-After` header. Limits are environment variables, not hardcoded.

**Container hardening.** Both the agent and the MCP server containers run as non-root (`runAsNonRoot: true`) with a read-only root filesystem (`readOnlyRootFilesystem: true`) and all Linux capabilities dropped (`capabilities.drop: ["ALL"]`). Writable scratch space uses `emptyDir` mounts. The CI/CD security gate validates these settings on every manifest; relaxing any of them blocks the merge.

---

## 8. Infrastructure Requirements

### 8.1 GKE Cluster

The existing `careconnect-gke` cluster in `hcb-dev-careconnect-etl` (region: **us-east4**) hosts three workloads: the LangGraph Agent Deployment, the MCP Server Deployment, and the cache warm-up CronJob. All three run under Workload Identity. HPA on QPS is required for both the agent and MCP server Deployments.

The Google AI Generative API endpoint is regional and lives in us-central1. Cross-region round-trip from us-east4 is on the order of 30–60ms per call (vs the ≤10ms an in-region call would achieve). This is accepted as a POC constraint: latency budgets in §7.1 are sized for the measured round-trip from us-east4, not the in-region minimum. A Sprint 0 measurement task (§13) re-pins the budgets if actual round-trip exceeds the planning assumption.

### 8.2 Cloud SQL

The existing Cloud SQL PostgreSQL instance (`cargpgsd1`, pgvector already enabled — used by the ADR agent) is extended with the following schema changes; the LangGraph `checkpoints` table is created on first run by the PostgresSaver. No new Cloud SQL instance is needed.

```sql
-- 1. translation_cache: add embedding and version columns
ALTER TABLE translation_cache
    ADD COLUMN question_embedding VECTOR(768),
    ADD COLUMN bounds_version     TEXT NOT NULL DEFAULT 'v0';
-- Backfill complete (cache is truncatable; no historic rows to preserve).
ALTER TABLE translation_cache ALTER COLUMN bounds_version DROP DEFAULT;

-- 2. HNSW index for vector similarity (explicit parameters; defaults are not optimal at our scale)
CREATE INDEX idx_translation_cache_embedding
    ON translation_cache
    USING hnsw (question_embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- 3. Rate-limit shared counter (per-user and global, one-minute rolling window)
CREATE TABLE rate_limit_counter (
    scope     TEXT     PRIMARY KEY,         -- 'user:<id>' or 'global'
    window_id BIGINT   NOT NULL,            -- floor(now_epoch_seconds / 60)
    count     INT      NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 4. Context cache active-name registry (written by the warm-up CronJob)
CREATE TABLE context_cache_registry (
    source         TEXT PRIMARY KEY,        -- 'mql' or 'sql'
    cache_name     TEXT NOT NULL,           -- google.generativeai CachedContent.name
    refreshed_at   TIMESTAMPTZ NOT NULL,
    expires_at     TIMESTAMPTZ NOT NULL
);

-- 5. Audit dead-letter table (drained by a CronJob worker on 5-minute cadence)
CREATE TABLE audit_dead_letter (
    id           BIGSERIAL PRIMARY KEY,
    payload      JSONB NOT NULL,
    enqueued_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_attempt TIMESTAMPTZ,
    attempts     INT NOT NULL DEFAULT 0,
    last_error   TEXT
);
```

The schema migration runs as a pre-deploy Kubernetes Job (`spec.parallelism: 1`) that must complete before the rolling update begins. The same Job pattern handles the existing cache-row DELETE on `schema_version` or `bounds_version` bumps.

POC accepts non-HA Cloud SQL with the degraded-mode behavior documented in §7.3 (fail-fast on slowness, semantic cache fallthrough on timeout). HA tier is recommended at production rollout.

### 8.3 Service Accounts

Three service accounts are required. `query-agent-gke` covers the LangGraph agent, the cache warm-up CronJob, and the audit dead-letter drain worker, and needs Cloud SQL Client, Secret Manager Secret Accessor (for the Google AI API key and the MongoDB Atlas credentials), and BigQuery Data Editor (for audit log writes) roles. `mcp-server-gke` covers the MCP Server and needs BigQuery Data Viewer, BigQuery Job User, Cloud SQL Client, and Secret Manager Secret Accessor (for the MongoDB Atlas credentials) roles. `cicd-deployer` covers CI/CD pipelines and needs Artifact Registry Writer, GKE Developer, and Cloud SQL Client roles for schema migrations at deploy time.

**Note.** No service account is granted the `Vertex AI User` role. The POC uses the Google AI Generative API key (Secret Manager) directly. Switching to Vertex AI is a follow-on workstream owned by Legal/Compliance — at that time the role grant moves to `Vertex AI User` and the SDK changes from `google.generativeai` to the Vertex AI SDK.

BigQuery audit log access must be restricted by role. Compliance and security team members receive read access via a named reviewer IAM binding. Engineering members may query the audit log through a scoped analyst role — not through their general developer grants. No developer account has edit or delete rights on the audit log dataset. Audit log IAM bindings are reviewed quarterly.

Grafana dashboard access must be segregated by audience and enforced via Grafana team permissions, not IAM. Three access tiers are required: the engineering dashboard (per-node latency, error rates, repair loop counts, escalation rates) is accessible to the engineering team; the business dashboard (cost breakdown, usage trends, cache hit rates, question volume) is accessible to product and finance stakeholders; the compliance dashboard (audit log query summaries, PHI exposure metric, SOC alert counts) is restricted to the compliance and security teams only. No care manager data appears on any dashboard visible outside the compliance tier.

### 8.4 Removed Infrastructure

Cloud Run, the VPC serverless connector, Cloud Scheduler, and any Google AI Studio API keys are not required. The VPC connector removal simplifies networking; both workloads communicate over cluster-internal K8s Service DNS.

---

## 9. Evaluation Criteria — POC Success Definition

The POC is considered successful when all of the following are true at the end of Phase 2 (200-user cohort):

1. **Accuracy:** ≥80% aggregate on the expanded golden test set (N ≥ 150) across three consecutive weekly evaluation runs, AND every per-class harm-tier minimum from §7.2 holds.
2. **Cost (LLM only):** Monthly Gemini Flash + Pro escalation + embedding + BigQuery scanning spend is within 10% of the $36/month projection at 500 conversations/day. Full-system cost (LLM + GKE incremental + Cloud SQL incremental + supporting) is within 10% of $250–$338/month per `COST-ESTIMATION-V3.md` Scenario A.
3. **Latency:** p95 cache miss latency is below 3.0s and p95 exact-match cache hit latency is below 700ms under the Phase 2 load (revised in §7.1 to reflect the us-east4 → us-central1 Gemini round-trip).
4. **Cache hit rate:** Semantic cache hit rate is ≥45% over a 7-day measurement window (this number is the single target — diagrams updated to match).
5. **Security:** Zero PHI exposure events in Cloud Trace, Cloud Monitoring, or Cloud SQL checkpoint rows during the POC window. Zero injection bypass events ≥ severity-MEDIUM in the weekly red-team run; false-negative rate on the golden injection test set ≤ 1%.
6. **Escalation rate:** Gemini 1.5 Pro escalation rate is between 3% and 12% of cache misses (outside this range indicates Flash regression or a tuning problem).
7. **Stability:** No unhandled exception in GKE workloads that escapes the FastAPI error boundary; no audit record loss across the POC window (measured by `audit_dead_letter` table drain success rate ≥ 99.9%); no SIGTERM-induced audit drops (verified by injecting rolling restarts during Phase 2 load test).

---

## 10. Risks and Mitigations

**Flash accuracy regression on complex aggregation queries.** Gemini Flash may underperform on multi-stage MongoDB aggregation pipelines compared to Claude Sonnet. Mitigation: a Sprint 0 pre-flight benchmark (§13) runs Flash against the existing golden set stratified by intent class and produces baseline per-class accuracy *before* infrastructure spend in Sprint 2; the eval gate is a hard block; the Gemini 1.5 Pro fallback catches runtime regressions at the request level; the 50+ few-shot examples and context caching maximise Flash accuracy before escalation. If the Sprint 0 benchmark shows any intent class below 70%, that class is hard-routed to Pro from Sprint 1 and the cost model is re-baselined before Sprint 2 starts.

**pgvector semantic threshold miscalibration.** A threshold of 0.95 may be too strict (low cache hit rate) or too loose (incorrect query served for a different question). Mitigation: threshold is a config value (`SEMANTIC_CACHE_SIMILARITY_THRESHOLD`), not hardcoded. Calibration runs against the golden test set before Phase 1 goes live.

**Cold-start latency on cache miss after CronJob failure.** If the K8s CronJob misses a cycle, the next translation pays the full uncached token rate and a slightly higher latency. Mitigation: the CronJob runs at deploy time as well as hourly, so a single missed hourly cycle has minimal impact; the translator is designed to handle a warm-cache miss gracefully.

**GKE Workload Identity misconfiguration.** If the binding between the Kubernetes service account and the GCP service account is incorrect, Gemini API calls and Cloud SQL connections will fail at pod start. Mitigation: infra team validates Workload Identity bindings in a dedicated staging namespace before Phase 1 load.

**MongoDB Atlas private endpoint latency.** If Atlas private endpoint routing adds unexpected latency, MCP execution times may exceed the 500ms p95 budget. Mitigation: MCP latency is tracked as a distinct span in Cloud Trace; if Atlas latency is the bottleneck, connection pool tuning is the first lever.

**Cross-region Gemini round-trip from us-east4.** The existing cluster is in us-east4; the Google AI Generative API endpoint is in us-central1. Three serial Flash calls on a cache miss can add 90–180ms of cross-region wait time vs an in-region cluster. Mitigation: latency budgets in §7.1 are sized for the measured round-trip from us-east4. Sprint 0 measures actual p50/p95 and re-pins the budget; if the measured value blows the budget, the team considers (a) collapsing the injection-detect Flash call into a deterministic path (the planned future change), (b) parallelising router and translator where intent allows, or (c) caching the most expensive few-shots locally. A region move is out of scope for the POC.

**Google AI Generative API BAA status.** The POC API surface (`google.generativeai`) does not currently have a confirmed CVS Health HIPAA BAA. Mitigation: Phase 1 onboarding (50 real care managers) is gated on Legal/Compliance confirmation; pre-Phase-1 dev work uses synthetic care manager data only. If BAA path requires a move to Vertex AI, the SDK swap is scoped as a separate workstream and not bundled into the POC.

**Per-class harm-tier gate failure.** Aggregate 80% can hide a Tier-1 (member identity / care-gap status) regression that the clinical SME would not accept. Mitigation: §7.2 per-class minimums are hard blocks regardless of aggregate; the eval harness reports per-class accuracy on every CI run.

**Vendor outage (Google AI API regional).** Gemini Flash regional outage stalls every cache-miss request. Mitigation: hard timeout returns a user-visible error; the break-glass procedure (§7.3) directs care managers to the fallback Looker dashboard for the most common intent types; on-call playbook references the dashboard explicitly.

---

## 11. User Stories

---

### US-01 — Natural Language Query

**As a care manager, I want to ask a natural language question about my caseload and receive an accurate, scoped answer in under 2.5 seconds, so that I can get information without writing database queries.**

**Technical Acceptance Criteria:**

- The system accepts a POST request with `{ question, user_id, session_id, jwt }` and returns a structured response within 3.0s p95 on cache miss (no Pro escalation) and 5.0s p95 with Pro escalation.
- The source router correctly classifies MongoDB vs BigQuery for ≥85% of golden test set questions.
- The translator produces a syntactically valid MQL or SQL query for ≥90% of routed questions on the golden test set.
- The full pipeline (route + translate + validate + execute + output gate) produces a correct answer for ≥80% of golden test set questions across three consecutive weekly eval runs.
- Query results are scoped to `state.user_id` and `state.caseload_scope` — no rows from outside the user's caseload are returned.

---

### US-02 — Semantic Cache Hit

**As a care manager, I want paraphrased or previously asked questions to return instantly without triggering a new LLM call, so that common queries are fast and cheap.**

**Technical Acceptance Criteria:**

- An exact-match cache hit (hash of PII-stripped normalized question + schema_version + bounds_version) returns in < 700ms p95. The exact-match hash check is a single indexed lookup; no LLM translation call occurs. The injection-detect Flash call still runs (~300ms) and the question embedding is still pre-computed in `load_context` (~150ms) since the cache result is not known until after both steps complete. (If the injection-detector swap to a local classifier ships, exact-match hit p95 drops to <400ms; this is a planned post-POC improvement.)
- A semantic cache hit (cosine similarity ≥ 0.95) returns in < 1.0s p95. The question embedding is pre-computed in the `load_context` node and stored in `AgentState.question_embedding`. The `semantic_cache_node` performs only the vector similarity lookup. No embedding API call occurs during the cache lookup step.
- The cache hit rate across a 7-day production window meets ≥45%.
- Cache entries store a query template with `:user_id` and `:caseload_scope` as bind parameters. The MCP executor binds them at execution time from `state.user_id` and `state.caseload_scope`. A cached query from user A must not return user B's data when served to user B.
- On schema_version or bounds_version bump, stale cache rows are deleted by a pre-deploy Kubernetes Job (spec.parallelism: 1) that must complete before the rolling update begins. Running the DELETE as a pod startup hook would allow old pods to serve stale queries during the rolling update window — the pre-deploy Job eliminates this race. No stale query is served after a schema promotion.

---

### US-03 — Confidence Indicator

**As a care manager, I want to see a confidence level on every response so that I know when to verify the answer against a source report.**

**Technical Acceptance Criteria:**

- Every response includes `confidence_verdict`: CONFIDENT or UNCERTAIN.
- An UNCERTAIN verdict is communicated to the chat UI via two response fields: `confidence_verdict: "UNCERTAIN"` and `show_uncertainty_banner: true`. The POC delivers the API contract; rendering the on-screen banner is a coordinated change to the in-house chat client and is tracked as a parallel UI workstream (not gated by the POC's acceptance because §3.3 scopes UI changes out of the POC itself). The banner copy is defined in `docs/uncertainty_banner_copy.md` so that the UI change is a one-line render. The Phase 1 onboarding checklist requires the UI banner to be live before any care manager sees an UNCERTAIN-verdict response — Phase 1 cannot begin until both the API contract and the UI render are in production.
- When `confidence_verdict == "UNCERTAIN"`, the system automatically escalates to Gemini 1.5 Pro before returning the response. The care manager sees the result of the Pro translation, not the Flash draft.
- `tier_2_escalation_rate` is tracked in Cloud Monitoring and visible on the engineering Grafana dashboard.
- The escalation rate during Phase 1 (50-user cohort) is used to calibrate the routing_confidence threshold before Phase 2 (200-user cohort).

---

### US-04 — Prompt Injection Protection

**As a security engineer, I want prompt injection attempts to be detected and blocked before they reach any LLM call, so that the agent cannot be manipulated into returning unauthorized data or executing instructions.**

**Technical Acceptance Criteria:**

- The input security gate (scope → PII strip → injection detect) runs on every request, including cache hits, before the semantic cache lookup or any translation LLM call.
- Scope classification and PII strip are deterministic (regex/keyword) — no LLM call. The injection-detect step is a Gemini Flash classifier call with a short, hardened prompt; the question payload sent to that classifier is already PII-stripped.
- A detected injection attempt returns HTTP 200 with a polite refusal response to the user and writes a SOC alert to the audit log in BigQuery.
- False-negative rate on the golden injection test set ≤ 1% (verified weekly via red-team run). A future swap to a local in-process classifier (e.g., Meta `prompt-guard-86m`) is tracked separately and would make the step deterministic; the swap is not in POC scope.
- Injection detection completes within 300ms p95.
- The golden injection test set is owned by the security team, versioned in the eval repo, contains ≥ 100 examples spanning direct-instruction-override, indirect/exfiltration, and obfuscated attempts, and is grown on every detected real-world attempt.

---

### US-05 — PHI Protection and HIPAA Compliance

**As a compliance officer, I want all protected health information to be scrubbed from responses before they reach the care manager and never appear in logs or traces, so that we maintain HIPAA compliance.**

**Technical Acceptance Criteria:**

- The output security gate scrubs PHI fields outside the user's access level from every result row before the response is returned.
- PHI field names and values do not appear in Cloud Trace spans, Cloud Monitoring metric labels, or Grafana dashboards at any time during the POC.
- The BigQuery audit log records every query outcome (question hash, user_id, source, outcome, trace_id) without storing raw question text or result data containing PHI.
- The BigQuery audit log retains records for six years (45 CFR 164.530(j) baseline).
- A scope confirmation check validates every result row against `state.caseload_scope` before the response is returned; rows from outside the user's caseload cause the entire result to be blocked and a SOC alert to be raised.
- PHI exposure events: zero during Phase 1 and Phase 2.
- No MCP query result data is included in any Gemini API request payload. The translator receives only schema, few-shot examples, and the care manager's natural language question — never query results. MCP results travel directly from the MCP executor to the output security gate without re-entering the LLM path.
- The translator prompt specifies only the fields required by the detected intent type — no SELECT * or broad MQL projections. The output security gate enforces a per-intent field allowlist; any field not on the allowlist for the detected intent type is stripped before the response is delivered to the care manager.

---

### US-06 — Write Operation Guard

**As a security engineer, I want any generated query that contains a write, delete, or schema-altering operation to be hard-blocked before execution, so that the database can never be modified through the agent.**

**Technical Acceptance Criteria:**

- The query validator's Stage 1 check fires on any query containing INSERT, UPDATE, DELETE, DROP, CREATE, ALTER, or equivalent MQL mutation operators.
- A hard-blocked query returns HTTP 200 with a polite refusal, writes a SOC alert to the audit log, and does not forward the query to the MCP executor.
- The hard-block path bypasses the output security gate (no result to scrub) and routes directly to the audit node.
- Write-op guard fires: verified against the full golden security test set (100% pass rate required before Phase 1 goes live).

---

### US-07 — Cost Within Budget

**As a product manager, I want the monthly LLM and infrastructure cost to stay within the projected $36/month budget at 500 conversations/day, so that we can justify the production rollout.**

**Technical Acceptance Criteria:**

- Monthly actual LLM spend (Gemini Flash + Pro escalations + BigQuery scanning) is within 10% of the $36/month projection at 500 conversations/day during Phase 2.
- Cost per cache miss is below $0.001 (projected $0.000844) measured as actual token spend divided by cache-miss count.
- Cost per cache hit is below $0.0002 (projected $0.00015).
- Cost breakdown (Flash, Pro escalation, embedding, BigQuery) is visible on the business Grafana dashboard updated daily.
- A monthly cost forecast is emitted as a Cloud Monitoring metric and alerts if projected monthly spend exceeds $50.

---

### US-08 — Out-of-Scope Question Handling

**As a care manager, I want a polite refusal when I ask a question outside the scope of CareConnect data, so that I understand the system's boundaries without receiving a confusing or empty response.**

**Technical Acceptance Criteria:**

- The scope classifier in the input security gate blocks questions not about CareConnect member records or care gap analytics and returns a user-friendly refusal message.
- The refusal message explains what the system can answer without exposing internal architecture details.
- Out-of-scope questions are logged to the audit table with `outcome = "refusal"` and are visible in the product dashboard as a quality signal.
- Scope classification accuracy is ≥95% on the golden scope test set.

---

### US-09 — Expensive Query Handling

**As a care manager, I want a helpful message when my query is too expensive to execute, guiding me to narrow my request, so that I am not left with a silent failure or a timeout.**

**Technical Acceptance Criteria:**

- The query validator's Stage 4 cost check fires when the estimated BigQuery bytes scanned or MongoDB collection scan exceeds the configured threshold.
- The `narrow_request` node returns a user-visible message explaining that the query is too broad and suggesting specific ways to narrow it (e.g., add a date range, filter by care gap type).
- The narrow_request path bypasses the output security gate (no PHI to scrub in a refusal) and routes to the audit node.
- The `too_expensive` path is tested as part of the golden validator test set — 100% correct handling required before Phase 1 goes live.

---

### US-10 — Platform: All-GKE Deployment

**As a platform engineer, I want both the LangGraph agent and the MCP server to run on GKE in the same cluster with Workload Identity, so that there are no long-lived credentials, no VPC connector, and no external scheduler dependencies.**

**Technical Acceptance Criteria:**

- Both the LangGraph Agent Deployment and the MCP Server Deployment run in the same GKE cluster with Workload Identity enabled.
- Neither workload mounts a service account key file. Authentication to Cloud SQL, BigQuery, Secret Manager, and Vertex AI (Gemini) is via ADC through Workload Identity.
- The cache warm-up CronJob runs in the same cluster under the `query-agent-gke` service account and completes successfully on the hourly schedule and at deploy time.
- Agent-to-MCP traffic routes over K8s Service DNS (`mcp-server.default.svc.cluster.local`) with mTLS. No traffic leaves the cluster for agent–MCP communication.
- HPA is configured for both Deployments and correctly scales pods up under a QPS load test.
- Cloud Run, VPC serverless connector, and Cloud Scheduler have zero references in the production infrastructure after migration.
- Both the agent and MCP server containers run as non-root users (`runAsNonRoot: true` in pod securityContext). The root filesystem is mounted read-only (`readOnlyRootFilesystem: true`). All Linux capabilities are dropped (`capabilities.drop: ["ALL"]`) with no capabilities added back. These securityContext settings are enforced by the CI/CD security gate — any Kubernetes manifest that removes or relaxes them must fail the security scan and block the merge.

---

### US-11 — Deployment Gate: Accuracy

**As an ML engineer, I want the CI/CD pipeline to automatically block any deployment that does not achieve ≥80% accuracy on the golden test set, so that accuracy regressions never reach production.**

**Technical Acceptance Criteria:**

- The CI/CD pipeline runs the full golden test set (50+ NL question → expected query + result pairs) on every pull request that touches the translator prompt, few-shot examples, schema, or LangGraph node logic.
- A score below 80% hard-blocks the merge. The pipeline reports which specific questions failed.
- The gate runs against the Gemini Flash model configuration of the branch under test — not a stub or mock. The active context cache name used by the gate is the same name registered in `context_cache_registry` for the staging environment, so the eval and prod code paths converge on the same cached context (eliminating the "eval runs uncached / prod runs cached" divergence).
- A gate pass is logged to Cloud Monitoring as a versioned accuracy metric (model version, schema version, score) for trend analysis.
- The accuracy gate has run successfully on the current golden test set with score ≥80% before Phase 1 onboarding begins.

---

### US-12 — Observability: Engineering Dashboard

**As a platform engineer, I want a real-time dashboard showing per-node pipeline latency, cache hit rates, error rates, and escalation rates, so that I can identify bottlenecks and regressions without reading raw logs.**

**Technical Acceptance Criteria:**

- Cloud Trace has a span per LangGraph node for every request. Node latency percentiles (p50, p95, p99) are queryable by node name.
- Cloud Monitoring has metrics for: cache_hit_rate (exact and semantic separately), tier_2_escalation_rate, gate_block_rate (input and output separately), repair_loop_count, and pipeline_error_rate.
- The engineering Grafana dashboard is live before Phase 1 onboarding and displays all metrics above with a 1-minute refresh.
- Alerts are configured for: SLO breach (p95 cache miss > 2.5s), security gate violation rate > 1% in a 5-minute window, and escalation rate sustained above 12% for 30 minutes.
- No PHI appears in any metric label, trace span, or dashboard panel.

---

## 12. Technical User Stories — POC Pipeline

One story per pipeline block, mapped directly to the LangGraph node in `src/careconnect_agent/pipeline/`.

---

### TS-01 — Input Security Gate (`input_security_node`)
**Story:** As a backend engineer, I want a 3-stage input gate (scope check → PII strip → injection classifier) that runs before every pipeline so that unsafe input never reaches an LLM or database.
**AC:**
- Out-of-scope question returns `outcome=refusal` without touching any LLM or DB
- PII stripped from `state.question` before the text leaves this node; raw question never written to Cloud SQL checkpoint
- Flash injection classifier runs on PII-stripped text; detection triggers SOC alert with `request_id`, `user_id`, `detected_pattern` — no raw question in the payload
**Size:** M

---

### TS-02 — Load Context + Embed (`load_context_node`)
**Story:** As a backend engineer, I want to normalize clinical synonyms and pre-compute the `text-embedding-004` vector on the PII-stripped question before the cache lookup so that `semantic_cache_node` is a pure DB operation with no network call.
**AC:**
- Schema Registry synonyms applied first (`AWV → annual_wellness_visit`, `A1C → hemoglobin_a1c`); result stored as `state.normalized_question`
- `Embedder.embed(normalized_question, task_type="SEMANTIC_SIMILARITY")` produces a 768-dim `list[float]` stored as `state.question_embedding`
- Embedding API failure degrades gracefully: `question_embedding=None`, request continues to translation; no 500 returned to user
**Size:** S

---

### TS-03 — Semantic Cache — Exact Hash (`semantic_cache_node`, tier 1)
**Story:** As a backend engineer, I want an exact-hash lookup (`hash(normalized_question + schema_version + bounds_version)`) against `translation_cache` so that identical questions skip all LLM calls.
**AC:**
- Cache hit returns stored query template in <50 ms; zero LLM calls invoked
- Cache key includes **both** `schema_version` and `bounds_version` — a bump to either invalidates prior entries at lookup time
- Hit sets `cache_hit=True, semantic_cache_hit=False`; pipeline jumps to `mcp_executor`
**Size:** S

---

### TS-04 — Semantic Cache — Vector Similarity (`semantic_cache_node`, tier 2)
**Story:** As a backend engineer, I want a pgvector HNSW cosine-similarity fallback so that paraphrased questions reuse cached query templates without a new LLM translation.
**AC:**
- SQL: `ORDER BY question_embedding <=> %s ASC LIMIT 1` — HNSW index engaged; threshold `(1 - distance) >= 0.95` filtered in Python, not in WHERE clause (preserves index path)
- Vector query filters by `schema_version` and `bounds_version` columns so stale rows from a pre-deploy job can never return
- Cache miss sets `cache_hit=False` and routes to Source Router; hit sets `semantic_cache_hit=True`
**Size:** M

---

### TS-05 — Schema Registry (`schema_registry.py`)
**Story:** As a data engineer, I want a YAML-backed Schema Registry mounted as a ConfigMap that exposes collections, field types, and clinical synonyms so that the translator and validator always use the authoritative field list.
**AC:**
- `SchemaRegistry.load()` returns a `SchemaSnapshot` with `collections`, `fields`, and `synonyms` dict on first call; cached in-process thereafter
- `SchemaRegistry.reload()` hot-reloads from disk without pod restart; triggered by ConfigMap update hook
- `SchemaSnapshot.field(collection, name)` returns `None` for unknown fields — validator Stage 3 treats `None` as a repair trigger
**Size:** S

---

### TS-06 — Context Cache — CronJob Warmup (`workers/cache_warmup.py`)
**Story:** As an AI/ML engineer, I want a K8s CronJob to be the sole creator of `CachedContent` objects (one for MQL, one for SQL) so that agent pods never create orphaned caches on scale-out or restart.
**AC:**
- CronJob builds `MQL_STATIC_PREFIX + schema_dump + 50 MQL few-shot pairs` (~15K tokens) and uploads via `google.generativeai.caching.CachedContent.create(ttl=75min)`
- `context_cache_registry` table updated with new `cache_name` per source; previous name scheduled for deletion after one TTL
- Agent `translator_node` reads `cache_name` from registry with a 60-second in-process TTL; falls back to uncached call on registry miss — no 500 on cache miss
**Size:** M

---

### TS-07 — Source Router (`source_router_node`)
**Story:** As an AI/ML engineer, I want Gemini Flash (LLM Call 1) to classify each question as `mongodb` or `bigquery` so that the translator receives the correct schema and few-shot set.
**AC:**
- Returns `routing_source` (Source enum) and `routing_confidence` (float 0–1)
- `routing_confidence < 0.7` routes to `pro_escalation_node`, skipping Flash translation entirely
- Routing decision is final — it never changes on a repair retry (only the translation is retried)
**Size:** S

---

### TS-08 — Translator (`translator_node` / `pro_escalation_node`)
**Story:** As an AI/ML engineer, I want Gemini Flash (LLM Call 2) with CachedContent to return a strict JSON object containing the query, `intent`, `confidence_verdict`, and `reason` so that no extra LLM call is needed for confidence scoring.
**AC:**
- Output parsed as `{"mql"|"sql": ..., "intent": ..., "confidence_verdict": "CONFIDENT"|"UNCERTAIN", "reason": ...}`; parse failure treated as `UNCERTAIN`
- `confidence_verdict == UNCERTAIN` and `tier == flash` routes to `pro_escalation_node`; `pro_escalation_node` runs same interface with `gemini-1.5-pro` cold-path (no context cache)
- After `MAX_REPAIR_ATTEMPTS` exhausted on Pro tier, graph routes to `audit` with `outcome=validation_failed`
**Size:** L

---

### TS-09 — 4-Stage Query Validator (`query_validator_node` + `accuracy_check_node`)
**Story:** As a backend engineer, I want five deterministic validation checks (accuracy + 4 validator stages) so that no unsafe, out-of-scope, or unaffordable query reaches the MCP executor.
**AC:**
- Accuracy check: intent vs operation type mismatch → repair via Translator (router decision unchanged)
- Stage 1: write/delete/alter op → `hard_block` + SOC alert
- Stage 2: `:user_id` or `:caseload_scope` bind parameter absent → `hard_block`
- Stage 3: field not in `SchemaSnapshot` → `repair` — error injected into next translator prompt (no row data in prompt)
- Stage 4: dry-run cost above threshold → `too_expensive` → `narrow_request_node` (user asked to filter)
**Size:** M

---

### TS-10 — MCP Query Executor (`mcp_executor_node`)
**Story:** As a backend engineer, I want the MCP Server (GKE) to execute validated query templates with `:user_id` and `:caseload_scope` bound at runtime so that no real user IDs are ever stored in the cache.
**AC:**
- MCP binds `state.user_id` and `state.caseload_scope` to all `:user_id` / `:caseload_scope` placeholders before execution
- MongoDB: read-only Atlas SA via Secret Manager; BigQuery: read-only SA with dataset-scoped IAM — no credentials in application code
- Executor enforces read-only at the SA permission level independent of validator Stage 1
**Size:** M

---

### TS-11 — Output Security Gate (`output_security_node`)
**Story:** As a backend engineer, I want a 5-stage deterministic output gate (PHI scrub → row cap → scope confirm → sanity bounds → field allowlist) so that no PHI or out-of-scope data reaches the care manager.
**AC:**
- PHI fields not in `field_allowlist` for detected `intent` are stripped; empty allowlist returns `{}` (fail-closed, not fail-open)
- Row cap: 100 rows ad-hoc, 1,000 rows export — excess silently truncated
- Foreign row (not in `state.caseload_scope`) → `outcome=blocked` + SOC alert; no row data in alert
- Bounds Registry hard-limit breach → `outcome=blocked`; soft-limit breach → `outcome=success` with `show_uncertainty_banner=True`
**Size:** L

---

### TS-12 — Audit + Save State (`audit_node` + `save_state_node`)
**Story:** As a platform engineer, I want every pipeline outcome written synchronously to BigQuery `audit_log` and the query template cached to Cloud SQL so that HIPAA 6-year retention is met and future identical questions hit the cache.
**AC:**
- Audit record fields: `trace_id`, `user_id`, `outcome`, `node_name`, `timestamp` — no raw question, no query results
- BigQuery write failure → synchronous fallback to `audit_dead_letter` (Cloud SQL); CronJob drains every 5 min
- `save_state_node` writes `(cache_key, validated_query, question_embedding, routing_source, intent, expires_at=+24h)` — embedding already in `state.question_embedding`; `PhiSafeJsonPlusSerializer` strips `query_results` and `formatted_response` from LangGraph checkpoint
**Size:** S

---

### TS-13 — Eval Harness + CI Gate (`eval/runner.py`)
**Story:** As an AI/ML engineer, I want a golden-set eval harness that runs Gemini Flash against 50+ NL→query pairs in CI and blocks any deployment below 80% aggregate accuracy so that no regression ships to care managers.
**AC:**
- Eval runs as a GitHub Actions step; failure blocks merge to `dev`
- Per-tier thresholds: Tier 1 (identity/gap status) ≥ 95%, Tier 2 (count/trend/ranking) ≥ 85%, Tier 3 (cosmetic) ≥ 75%, aggregate ≥ 80%
- Results written to Grafana quality dashboard with accuracy trend per deploy and `tier_2_escalation_rate` SLO tracked
**Size:** M

---

## 13. Delivery Timeline

The POC is preceded by a one-week Sprint 0 pre-flight, then delivered in six sprints of two weeks each, followed by two three-week validation phases.

### Sprint 0 — Week 0: Pre-flight (gates Sprint 1 spending)

Deliverables: (1) **Gemini Flash baseline benchmark** against the existing 50-question golden set with few-shot examples but without the repair loop, stratified per intent class and per source (MongoDB vs BigQuery), and per-class accuracy reported; (2) **us-east4 → us-central1 Gemini Flash latency measurement** — p50/p95/p99 round-trip from a pod in `careconnect-gke` — and latency budget in §7.1 re-pinned if the measured value exceeds the planning assumption; (3) **Bounds Registry v1 specification** (storage format, fields covered, SME owner, change-control workflow, hot-reload mechanism, warn-vs-block semantics) — published as a linked spec doc; (4) **Per-intent field allowlist** for count/filter/trend/ranking/lookup intents, signed off by clinical SME; (5) **Per-class harm taxonomy and Tier-1/2/3 minimum accuracy gates** signed off by clinical SME; (6) **Golden injection test set v1** (≥100 examples) published in the eval repo.

Exit criteria: Flash hits ≥80% aggregate AND every per-class minimum on the existing golden set. If any class fails, that class is hard-routed to Pro from Sprint 1 and the cost model is re-baselined before Sprint 2 starts. Bounds Registry, allowlist, harm taxonomy, and injection test set are all merged. Latency budget is updated to the measured round-trip. No Sprint 1 infrastructure spending begins until Sprint 0 exits.

### Sprint 1 — Weeks 1–2: Infrastructure + LLM Swap

Deliverables: GKE cluster provisioned with Workload Identity; Cloud SQL extended with pgvector and new columns; Artifact Registry configured; Workload Identity bindings validated for all three service accounts; Claude Sonnet replaced with Gemini Flash in the translator and router; eval gate runs on golden test set and achieves ≥80% (Stage 1 migration complete).

Exit criteria: eval gate passes at ≥80% accuracy with Gemini Flash. Infrastructure team signs off on Workload Identity configuration in staging.

### Sprint 2 — Weeks 3–4: Context Caching + Semantic Cache

Deliverables: MQL and SQL context caches implemented with Google AI API `CachedContent`; K8s CronJob for hourly cache warm-up deployed and tested; pgvector semantic cache implemented with text-embedding-004 embeddings; HNSW index created; cache invalidation DELETE hook integrated into deploy pipeline (Stage 2 and Stage 3 migration complete).

Exit criteria: context cache warm-up CronJob completes successfully in staging. Semantic cache hit rate on replay of 30-day V2 question log reaches ≥45%.

### Sprint 3 — Weeks 5–6: LangGraph Structured Pipeline

Deliverables: ReAct agent loop replaced with the V3 named-node LangGraph pipeline; `narrow_request` node implemented; Gemini 1.5 Pro fallback implemented; repair loop capped at 1 retry; all conditional edges correct as per the V3 design spec; PostgreSQL checkpointer wired (Stage 4 migration complete).

Exit criteria: full golden test set passes the CI/CD accuracy gate at ≥80%. All LangGraph node edges exercise correctly in integration tests, including repair loop, too_expensive, and fallback to Pro paths.

### Sprint 4 — Weeks 7–8: Multi-Agent Interfaces + GKE Deployment

Deliverables: `as_subgraph`, `as_tool`, and MCP endpoint interfaces implemented and unit-tested; both GKE Deployments (agent + MCP server) containerised and deployed to staging cluster; mTLS configured between agent and MCP server; HPA validated under load test in staging; CI/CD pipeline deploys to staging on merge to main (Stage 5 migration complete).

Exit criteria: all three multi-agent interfaces callable in isolation tests. Load test of 50 concurrent users in staging keeps p95 cache miss below 3.0s. Readiness and liveness probes (`/healthz` HTTP GET, `initialDelaySeconds=20`, `periodSeconds=10`) are present on both Deployments. Both Deployments declare `minReadySeconds: 15` to bound the in-flight-request window during rolling updates. Cold-start time is measured and recorded for the Phase 2 HPA capacity plan. No credentials in any container spec. Snyk container scan passes with zero critical or high CVEs in both agent and MCP server images — Snyk is integrated into the CI/CD pipeline and blocks the merge if critical or high vulnerabilities are detected.

### Sprint 5 — Weeks 9–10: QA, Security Review, and POC Readiness

Deliverables: full security review of all input and output gate paths; red-team injection test set executed (100% block rate required); Grafana dashboards live; Cloud Monitoring alerts configured; documentation for infra team (deploy runbook, rollback steps, CronJob monitoring); POC Phase 1 onboarding checklist signed off.

Exit criteria: zero open CRITICAL or HIGH security findings. All Grafana dashboards live and verified. Infra team has completed deploy runbook review. Accuracy gate passes on golden set. PHI scan of Cloud Trace confirms zero PHI in spans. The golden test set contains ≥20% BigQuery-routing questions before Phase 1 begins — the current set is weighted toward MongoDB and must be expanded to give the source router and SQL translator meaningful coverage before live traffic starts.

### Phase 1 — Weeks 11–13: POC with 50 Care Managers

50 care managers onboarded. Live traffic collected. Weekly accuracy eval runs against the golden test set. Escalation rate baseline established. Grafana dashboards monitored daily. Any p95 latency breach or gate violation triggers an immediate engineering review. Cost actuals compared against projection weekly.

Exit criteria for Phase 2 promotion: ≥80% accuracy for three consecutive weekly eval runs AND all per-class harm-tier minimums hold; p95 cache miss < 3.0s sustained over Phase 1; zero PHI exposure events; actual LLM monthly cost within 10% of $36 projection scaled to the Phase 1 cohort (i.e., approximately $7 at 100 conv/day, not $36); escalation rate between 3% and 12%; audit dead-letter table drain success rate ≥ 99.9%.

### Phase 2 — Weeks 14–16: Expanded POC with 200 Care Managers

200-user cohort onboarded. HPA behaviour under higher load validated. Semantic cache hit rate measurement at scale. Full POC success criteria evaluated against the definitions in Section 9.

### Week 17: POC Readout and Go/No-Go

Engineering, product, security, and compliance teams review POC results against success criteria. Go decision triggers production hardening and full rollout planning. No-go triggers a documented gap analysis with specific remediation steps.

---

## 14. Open Questions

1. What is the agreed MongoDB Atlas private endpoint configuration — VPC peering or private endpoint service? This affects MCP server networking setup in Sprint 1.
2. `ROUTING_CONFIDENCE_THRESHOLD = 0.7` and `SEMANTIC_CACHE_SIMILARITY_THRESHOLD = 0.95` are starting values pending empirical calibration. Sprint 2 includes a calibration run against V2 telemetry logs and the expanded golden set; thresholds become locked configuration values after that run.
3. HIPAA BAA path for the Google AI Generative API — owned by Legal/Compliance — must close before Phase 1 onboarding. Pre-Phase-1 development uses synthetic care manager data only. The Vertex AI SDK migration is a contingency workstream, scoped separately if the BAA path requires it.
4. Atlas credential rotation cadence and X.509 certificate authentication availability for the existing cluster — Atlas DBA team.

**Resolved during ARB revision (v1.1):**
- GCP region: us-east4 (existing `careconnect-gke` cluster). No region migration in POC scope. Sprint 0 measures actual cross-region Gemini latency.
- Golden set BigQuery balance: ≥20% BigQuery, N ≥ 150 total — moved from Sprint 5 exit to Sprint 0 deliverable.
- Audit log retention: 6 years (45 CFR 164.530(j)).
- Cross-pod rate limit: Cloud SQL shared counter (new table in existing instance) — replaces the prior per-pod in-memory approach.
- Injection detector implementation: Gemini Flash classifier for POC; deterministic local classifier (e.g., `prompt-guard-86m`) is a post-POC change.

---
