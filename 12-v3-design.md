# V3 Architecture — Optimized, Composable NL-to-Query Agent
## LangGraph + LangChain + Gemini Flash + pgvector Semantic Cache + Google AI Context Caching

> **What changed from earlier V3 draft:** The ReAct agent loop is replaced with a structured LangGraph pipeline — explicit named nodes, no LLM reasoning overhead between steps. Gemini Flash handles all LLM calls (routing and translation). Context caching runs through the Google AI Generative API — no Vertex AI platform required. The translation cache is upgraded from exact-match to semantic similarity using pgvector inside the existing Cloud SQL instance. Route and Translate remain two separate sequential LLM calls.

---

## Design Philosophy

Three principles guide V3:

**1. Structured pipeline, not agent loop.**
The ReAct pattern (reason → call tool → observe → reason again) burns 4–5 LLM calls per request just on orchestration thinking. For a well-understood pipeline like NL-to-query, that reasoning overhead adds cost and latency without adding intelligence. V3 uses an explicit LangGraph node pipeline where each step has a defined role and the LLM is called only when genuinely needed — twice per cache-miss request. The pipeline is composable and multi-agent ready without needing an agent loop to achieve it.

**2. Security gates are deterministic and structurally unreachable by the LLM.**
Input security screening, PHI scrubbing, and scope confirmation are LangGraph nodes that sit outside the LLM call path. The LLM cannot influence, skip, or route around them — they are graph-level constraints, not prompt-level instructions.

**3. The agent is a composable module.**
It exposes three interfaces — LangGraph subgraph, LangChain tool, MCP endpoint — so any future agent (Notification, Escalation, Analytics, Supervisor) can call or orchestrate it without touching its internals.

> **Why we kept LangGraph despite Design B's recommendation.** A prior architecture review (`11-architecture-options.md`) recommended dropping LangGraph in favor of plain Python + LangChain on the grounds it's over-engineered at 500 req/day. That recommendation considered only the single-agent shape. V3 keeps LangGraph specifically because the multi-agent roadmap (Notification, Escalation, Analytics, Supervisor) needs LangGraph's `as_subgraph` composability — the cleanest way to plug a child agent into a parent supervisor's graph without a custom orchestration layer. The cost paid is ~6,000 extra Cloud SQL writes/day from per-node checkpointing, accepted in exchange for that composability and for crash recovery from any node. If the multi-agent roadmap is canceled or deferred indefinitely, this decision should be revisited against Design B.

---

## What Changed — V2 to V3

| Dimension | V2 | V3 |
|---|---|---|
| Orchestration | Hardcoded pipeline — fixed function sequence | Structured LangGraph pipeline — explicit named nodes |
| LLM provider | Claude Sonnet via CVS LMS Gateway | Gemini Flash via Google AI API (same provider you already use) |
| Routing LLM | Claude Sonnet | Gemini Flash — 40× cheaper per token |
| Translation LLM | Claude Sonnet | Gemini Flash + Google AI context caching |
| Context caching | None — full schema paid every call | Google AI API context cache — schema + few-shots cached, 75% token saving |
| Cache matching | Exact hash match only | Semantic similarity via pgvector — paraphrases hit the cache |
| Cache hit rate | ~30% | ~50% with semantic matching |
| Multi-agent interface | None — standalone service only | Subgraph, LangChain tool, MCP endpoint |
| LLM call count | 2 per cache-miss | 2 per cache-miss (route + translate, sequential) |
| Cost per cache-miss | $0.0615 (Claude Sonnet) | **$0.0008 (Flash + context cache)** |
| Cost per cache-hit | $0.0001 | **$0.0001** |
| Annual LLM-only spend (500/day, 50% cache hit, 6% Pro escalation) | ~$7,968 | **~$432** |

Everything not in this table — validators, MCP execution, guardrails, eval framework, observability, deployment pipeline — is unchanged.

---

## Architecture Overview

```
                       ┌──────────────────────────────────────────────────────────┐
                       │           CareConnect Query Agent  ·  V3                 │
                       │           LangGraph Subgraph  ·  Structured Pipeline     │
                       │                                                           │
  User Question ──────►│  ┌───────────────────────────────────────────────────┐   │
                       │  │ INPUT SECURITY GATE          (deterministic node) │   │
                       │  │  Scope check · Injection detect · PII strip       │   │
                       │  └───────────────────────────────────────────────────┘   │
                       │                      │ safe=true                         │
                       │                      ▼                                   │
                       │  ┌───────────────────────────────────────────────────┐   │
                       │  │ LOAD CONTEXT                 (Cloud SQL)          │   │
                       │  │  Session history · Caseload scope                 │   │
                       │  └───────────────────────────────────────────────────┘   │
                       │                      │                                   │
                       │                      ▼                                   │
                       │  ┌───────────────────────────────────────────────────┐   │
                       │  │ SEMANTIC CACHE CHECK     (pgvector · Cloud SQL)   │   │
                       │  │  Exact hash  →  if miss  →  vector similarity     │   │
                       │  │  Embed question · cosine search · threshold 0.95  │   │
                       │  └───────────────────────────────────────────────────┘   │
                       │           │ cache hit                  │ cache miss       │
                       │           │ (skip to executor)         ▼                 │
                       │           │         ┌─────────────────────────────────┐  │
                       │           │         │ SOURCE ROUTER   (Gemini Flash)  │  │
                       │           │         │ LLM Call 1 · MongoDB or         │  │
                       │           │         │ BigQuery classification          │  │
                       │           │         └─────────────────────────────────┘  │
                       │           │                       │                       │
                       │           │                       ▼                       │
                       │           │         ┌─────────────────────────────────┐  │
                       │           │         │ TRANSLATOR      (Gemini Flash)  │  │
                       │           │         │ LLM Call 2 · NL → MQL or SQL   │  │
                       │           │         │ Context cache · confidence in   │  │
                       │           │         │ output · no extra call          │  │
                       │           │         └─────────────────────────────────┘  │
                       │           │                       │                       │
                       │           │                       ▼                       │
                       │           │         ┌─────────────────────────────────┐  │
                       │           │         │ ACCURACY CHECK  (deterministic) │  │
                       │           │         │ Intent ↔ op · schema alignment  │  │
                       │           │         └─────────────────────────────────┘  │
                       │           │                       │                       │
                       │           │                       ▼                       │
                       │           │         ┌─────────────────────────────────┐  │
                       │           │         │ QUERY VALIDATOR (deterministic) │  │
                       │           │         │ 4-stage · write guard · scope   │  │
                       │           │         │ filter · fields · cost estimate │  │
                       │           │         └─────────────────────────────────┘  │
                       │           │                       │                       │
                       │           └───────────────────────┘                       │
                       │                                   │                       │
                       │                      ▼                                    │
                       │  ┌───────────────────────────────────────────────────┐   │
                       │  │ MCP EXECUTOR          (MCP Server · read-only)    │   │
                       │  │  MongoDB  ·  BigQuery  ·  Scope-filtered          │   │
                       │  └───────────────────────────────────────────────────┘   │
                       │                      │                                   │
                       │                      ▼                                   │
                       │  ┌───────────────────────────────────────────────────┐   │
                       │  │ OUTPUT SECURITY GATE         (deterministic node) │   │
                       │  │  PHI scrub · Row cap · Scope confirm · Bounds     │   │
                       │  └───────────────────────────────────────────────────┘   │
                       │                      │                                   │
                       │                      ▼                                   │
                       │  ┌───────────────────────────────────────────────────┐   │
                       │  │ SAVE STATE + AUDIT    (Cloud SQL · BigQuery)      │   │
                       │  │  Cache entry with embedding · Session · Audit log │   │
                       │  └───────────────────────────────────────────────────┘   │
                       └──────────────────────────────────────────────────────────┘
```

---

## LangGraph Structure — Structured Pipeline

V3 uses named nodes with explicit conditional edges. No ReAct loop. No LLM reasoning between steps. Each node does one job.

```python
graph = StateGraph(AgentState)

# Security — scope and PII strip are deterministic; injection-detect is a Flash classifier call
# (see "Security Model" section for the honest characterisation).
graph.add_node("input_security",     input_security_node)

# Data loading + question embedding pre-compute (PII-stripped, normalized question)
graph.add_node("load_context",       load_context_node)       # Cloud SQL session + embed()

# Semantic cache — pgvector similarity search (lookup only; no embedding call)
graph.add_node("semantic_cache",     semantic_cache_node)     # exact + vector match

# LLM pipeline
graph.add_node("source_router",      source_router_node)      # Gemini Flash — LLM call 1
graph.add_node("translator",         translator_node)         # Gemini Flash + cache — LLM call 2
graph.add_node("pro_escalation",     pro_escalation_node)     # Gemini 1.5 Pro — same translator
                                                              # interface, model swapped per AgentState.tier

# Deterministic validation
graph.add_node("accuracy_check",     accuracy_check_node)     # intent ↔ op alignment
graph.add_node("query_validator",    query_validator_node)    # 4-stage safety validation

# Execution
graph.add_node("mcp_executor",       mcp_executor_node)       # MCP Server (GKE)

# User-facing terminal nodes
graph.add_node("narrow_request",     narrow_request_node)     # too_expensive response

# Deterministic output gate — always runs after execution
graph.add_node("output_security",    output_security_node)    # PHI · row cap · scope · bounds · field allowlist

# Persistence and audit — synchronous, inside the request (see Latency Budget)
graph.add_node("save_state",         save_state_node)         # Cloud SQL cache + session
graph.add_node("audit",              audit_node)              # BigQuery (with audit_dead_letter fallback)

# ── Edges ─────────────────────────────────────────────────────────────────────
graph.add_edge(START, "input_security")

graph.add_conditional_edges("input_security", lambda s: (
    "load_context" if s.safe else "audit"                     # blocked → audit (no PHI to scrub)
))

graph.add_edge("load_context", "semantic_cache")

graph.add_conditional_edges("semantic_cache", lambda s: (
    "mcp_executor" if s.cache_hit else "source_router"        # hit → skip translation
))

# Source router → Pro escalation if low confidence, else continue with Flash translation.
graph.add_conditional_edges("source_router", lambda s: (
    "pro_escalation" if s.routing_confidence < ROUTING_CONFIDENCE_THRESHOLD
                     else "translator"
))

# Translator → if Flash returned UNCERTAIN verdict, escalate to Pro; otherwise continue.
graph.add_conditional_edges("translator", lambda s: (
    "pro_escalation" if s.confidence_verdict == "UNCERTAIN" and s.tier == "flash"
                     else "accuracy_check"
))

# Pro escalation re-emits a Translator-shaped state with tier='pro' and ALWAYS routes through
# accuracy_check + validator. Escalation bypasses no safety control.
graph.add_edge("pro_escalation",   "accuracy_check")

graph.add_conditional_edges("accuracy_check", lambda s: (
    "query_validator"               if s.accuracy_check == "pass"          else
    "translator"                    if s.repair_count < MAX_REPAIR_ATTEMPTS and s.tier == "flash" else
    "pro_escalation"                if s.repair_count < MAX_REPAIR_ATTEMPTS and s.tier == "pro"   else
    "audit"                                                                       # repair budget exhausted
))

graph.add_conditional_edges("query_validator", lambda s: (
    "mcp_executor"   if s.validation_status == "pass"          else
    "translator"     if s.validation_status == "repair"
                        and s.repair_count < MAX_REPAIR_ATTEMPTS
                        and s.tier == "flash"                  else
    "pro_escalation" if s.validation_status == "repair"
                        and s.repair_count < MAX_REPAIR_ATTEMPTS
                        and s.tier == "pro"                    else
    "narrow_request" if s.validation_status == "too_expensive" else
    "audit"                                                                       # hard_block or repair budget exhausted
))

graph.add_edge("mcp_executor",     "output_security")
graph.add_edge("narrow_request",   "audit")                    # narrow path bypasses output_security

graph.add_conditional_edges("output_security", lambda s: (
    "save_state" if s.outcome != "blocked" else "audit"
))

graph.add_edge("save_state", "audit")
graph.add_edge("audit",      END)

# PostgreSQL checkpointer — saves state after every node.
# Custom serde strips state.query_results and state.formatted_response before write
# (PHI never enters the checkpoint table; see Security Model).
graph.compile(
    checkpointer=PostgresSaver.from_conn_string(
        CLOUD_SQL_DSN,
        serde=PhiSafeJsonPlusSerializer(strip_fields={"query_results", "formatted_response"}),
    ),
)
```

---

## Semantic Cache — pgvector in Cloud SQL

The most impactful single change in V3. Instead of only matching identical questions, the cache now catches paraphrases and semantically equivalent questions.

**How it works:**

Every question gets embedded using `text-embedding-004` (Google's embedding model — extremely cheap, sub-millisecond). The embedding is stored alongside the cached query. On the next lookup, two checks run in order:

1. **Exact match first** — hash of normalized question + schema version. Zero embedding cost on a hit.
2. **Vector similarity** — if exact match misses, run cosine similarity against stored embeddings. If the closest match scores above 0.95, serve that cached query.

"How many open care gaps do I have?" and "What is my total count of open care gaps?" produce different hashes but similarity score of 0.97 — the second question hits the cache on its first appearance.

**Schema change — Cloud SQL:**

```sql
-- pgvector is already enabled on the existing instance (used by the ADR agent); this is a no-op.
CREATE EXTENSION IF NOT EXISTS vector;

-- Extend existing translation_cache table.
-- bounds_version must have a DEFAULT for the ALTER to succeed against any existing rows.
-- The cache is truncatable; the DEFAULT is dropped after the column is established.
ALTER TABLE translation_cache
    ADD COLUMN question_embedding VECTOR(768),
    ADD COLUMN bounds_version     TEXT NOT NULL DEFAULT 'v0';
ALTER TABLE translation_cache    ALTER COLUMN bounds_version DROP DEFAULT;

-- pgvector HNSW index — recall-stable across cache size, no list tuning required.
-- Explicit (m, ef_construction) — defaults are not optimal for our recall target.
-- Outperforms ivfflat below ~10K rows; translation_cache holds ~250 fresh rows under
-- 24h TTL at projected volume. Reconsider ivfflat only if cache grows beyond 10K rows.
CREATE INDEX idx_translation_cache_embedding
    ON translation_cache
    USING hnsw (question_embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

**Cache invalidation on schema or bounds push:**

The `cache_key` includes `schema_version` so schema bumps invalidate naturally on the next lookup. But stale rows still occupy the table — and the pgvector index — until 24h TTL expires, polluting semantic-similarity scores in the meantime. Two-step invalidation runs as part of every schema or bounds registry promotion:

```sql
-- Run as a pre-deploy Kubernetes Job (spec.parallelism: 1) before the rolling update begins.
-- A pod startup hook would allow old pods to serve stale queries during the rolling window —
-- the pre-deploy Job completes the DELETE before any new pod receives traffic.
DELETE FROM translation_cache
WHERE schema_version  <> :new_schema_version
   OR bounds_version  <> :new_bounds_version;
```

`bounds_version` is added to `cache_key` alongside `schema_version` so Bounds Registry changes also invalidate the cache. Both versions are stored on each cached row to make the DELETE selective.

**Embedding pre-computation in `load_context_node`:**

The question embedding is computed in `load_context`, before the cache lookup, and stored in `AgentState.question_embedding`. This moves the embedding API call off the cache-hit critical path — `semantic_cache_node` performs only the vector similarity lookup, with no network call. If the question ends up being an exact cache hit, the embedding is still pre-computed but the result is discarded at negligible cost. On a cache miss, the embedding is already in state and is written to the new cache row by `save_state`.

The embedding is computed on the **PII-stripped, normalized** question so that a question containing a member name does not become a discoverable cache key for another user. The exact-hash key is also computed on the PII-stripped normalized question, for the same reason.

```python
def load_context_node(state: AgentState) -> AgentState:
    # ... load session history and caseload_scope from Cloud SQL ...

    # PII strip happened in input_security; embed the safe form only.
    embedding = embed(state.normalized_question)   # text-embedding-004 — ~$0.000000020

    return state | {
        "session_history": ...,
        "caseload_scope": ...,
        "question_embedding": embedding,
    }
```

**Cache lookup logic in `semantic_cache_node`:**

> The `validated_query` stored in cache is a *template* with `:user_id` and `:caseload_scope` as bind parameters. The MCP executor binds them at execute time from `state.user_id` and `state.caseload_scope`, so the same cached query template safely serves any user. The validator's "scope filter present" check verifies the *placeholder* is in the right structural position, not that a literal user ID is baked in.

```python
def semantic_cache_node(state: AgentState) -> AgentState:
    key = hash(state.normalized_question + SCHEMA_VERSION + BOUNDS_VERSION)

    # 1. Try exact match first — no embedding cost.
    row = db.query(
        "SELECT * FROM translation_cache WHERE cache_key = %s AND expires_at > now()",
        [key],
    )
    if row:
        return state | {"cache_hit": True, "selected_query": row.validated_query, ...}

    # 2. Vector similarity — index-accelerated.
    # Use ORDER BY <embedding> <=> :q on the raw operator (HNSW-indexable),
    # then filter (1 - distance) >= THRESHOLD in Python. Wrapping the operator
    # in arithmetic inside the WHERE clause defeats the HNSW index — that form
    # is intentionally avoided here.
    row = db.query(
        """
        SELECT *,
               question_embedding <=> %s::vector AS distance
        FROM   translation_cache
        WHERE  expires_at > now()
        ORDER BY question_embedding <=> %s::vector ASC
        LIMIT  1
        """,
        [state.question_embedding, state.question_embedding],
    )
    if row and (1 - row.distance) >= SEMANTIC_CACHE_SIMILARITY_THRESHOLD:
        return state | {
            "cache_hit": True,
            "selected_query": row.validated_query,
            "semantic_cache_hit": True,
            ...
        }

    # 3. Cache miss — embedding already stored in state.question_embedding for save_state to persist.
    return state | {"cache_hit": False}
```

**Impact:** Cache hit rate rises from ~30% (exact match) to ~50% (semantic). Since every cache hit costs ~$0.0001 vs $0.0008 for a cache-miss, this is the single largest cost lever after switching to Flash.

---

## LLM Strategy — Gemini Flash for All Calls

**Why Flash for both routing and translation:**

Routing is a binary classification task — Flash has handled this accurately since V2. The real question was whether Flash could handle complex MongoDB aggregation pipeline generation as accurately as a larger model.

With the following in place, Flash accuracy on structured code generation is comparable to Pro:
- 50+ few-shot examples covering all intent types and edge cases
- Full schema injected (entities, fields, types, samples)
- Context caching means all 15K tokens of that static context are always present
- Pre-execution accuracy check catches intent/schema mismatches before execution
- 4-stage validator catches hallucinated fields before execution
- Repair loop gives one retry with the error message injected

The eval gate enforces ≥80% accuracy on the golden test set before any deployment. If Flash drops below that threshold on a given release, the gate blocks it — no manual oversight needed.

**Context caching via Google AI API — CronJob-owned, registry-coordinated:**

The K8s CronJob is the **sole creator** of `CachedContent` objects. It runs every 45 min, calls `CachedContent.create()` for the MQL and SQL contexts, writes the resulting `cache.name` to a `context_cache_registry` row per source, and deletes the previously-active cache one TTL later (so storage cost stays bounded). Agent pods never call `create()` — they read the active name from the registry on each translator invocation (with a 60s in-process TTL on the lookup so it does not become its own latency tax).

This pattern eliminates per-pod orphaned caches caused by pod restart or HPA scale-out, which the prior module-level-variable design produced.

```python
# CronJob payload (refresh-context-cache.py)
import google.generativeai as genai

def refresh(source: str, prompt: str) -> None:
    cache = genai.caching.CachedContent.create(
        model="models/gemini-1.5-flash",
        contents=[
            {"role": "user",  "parts": [{"text": prompt}]},
            {"role": "model", "parts": [{"text": "Ready. Send the question."}]},
        ],
        ttl=datetime.timedelta(minutes=75),
    )
    prev = registry.get(source)
    registry.put(source, cache.name, expires_at=now() + timedelta(minutes=75))
    if prev:
        schedule_deletion(prev, after=timedelta(minutes=75))

refresh("mql", MQL_SYSTEM_PROMPT + MONGODB_SCHEMA + MQL_FEW_SHOTS)
refresh("sql", SQL_SYSTEM_PROMPT + BIGQUERY_SCHEMA + SQL_FEW_SHOTS)
```

```python
# Agent pod side (translator_node)
@cached_with_ttl(seconds=60)
def active_cache_name(source: str) -> str:
    return registry.get_active(source)  # SELECT cache_name FROM context_cache_registry WHERE source = %s

def translator_llm(routed_source: str, tier: str = "flash") -> ChatGoogleGenerativeAI:
    source = "mql" if routed_source == "mongodb" else "sql"
    model = "gemini-1.5-flash" if tier == "flash" else "gemini-1.5-pro"
    kwargs = {"model": model, "generation_config": {"response_mime_type": "application/json"}}
    if tier == "flash":
        # Pro escalation runs cold-path; cache is Flash-only and does not transfer.
        kwargs["cached_content"] = active_cache_name(source)
    return ChatGoogleGenerativeAI(**kwargs)
```

Context cached per source: ~15,000 tokens (system prompt + schema + 50 few-shots)
Dynamic per request: ~2,000 tokens (question + session context)
Saving: 75% on the static portion — cache read rate vs standard input rate
Two caches × 15K tokens × 24h × 30d storage cost is still negligible (~$0.40/month)

**Minimum data use — translator field selection:**

The translator system prompt instructs the model to project only the fields required by the detected intent type. For a `count` intent, the query returns only the count field. For a `filter` intent, only the fields the care manager asked about. No SELECT * or equivalent broad MQL projections. The few-shot examples must demonstrate this behaviour consistently — broad projections in few-shots produce broad projections in output. The output security gate enforces a per-intent field allowlist as a safety net, but the translator is the first line: less data returned = less PHI surface = smaller audit footprint.

---

## AgentState V3

All V2 fields preserved. The `messages` list from the earlier ReAct design is removed — not needed in a structured pipeline. State enums are typed (`StrEnum`) for compile-time safety on the conditional-edge branches; string-encoded enums (the V2 shape) were silently misroutable on typos. State itself is a Pydantic model so the same model double-checks values at runtime ingress.

```python
from enum import StrEnum
from pydantic import BaseModel, Field

class Tier(StrEnum):       flash = "flash"; pro = "pro"
class Verdict(StrEnum):    confident = "CONFIDENT"; uncertain = "UNCERTAIN"
class AccCheck(StrEnum):   passed = "pass"; mismatch = "mismatch"
class ValStatus(StrEnum):  passed = "pass"; repair = "repair"; hard_block = "hard_block"; too_expensive = "too_expensive"
class Outcome(StrEnum):    success = "success"; refusal = "refusal"; validation_failed = "validation_failed"; clarify = "clarify"; blocked = "blocked"
class Source(StrEnum):     mongodb = "mongodb"; bigquery = "bigquery"

# Module-level constants (read-only; tunable via env)
SCHEMA_VERSION                     : str   = "..."
BOUNDS_VERSION                     : str   = "..."
ROUTING_CONFIDENCE_THRESHOLD       : float = 0.7
SEMANTIC_CACHE_SIMILARITY_THRESHOLD: float = 0.95
MAX_REPAIR_ATTEMPTS                : int   = 1

class AgentState(BaseModel):
    # ── Identity and auth ──────────────────────────────────────────────────────
    session_id:           str
    user_id:              str
    caseload_scope:       str
    jwt:                  str

    # ── Question processing ────────────────────────────────────────────────────
    question:             str                 # original; PHI may be present — never persisted to checkpoint
    normalized_question:  str                 # PII-stripped + synonym-normalized
    intent:               str                 # count · filter · trend · ranking · lookup
    safe:                 bool

    # ── Semantic cache ─────────────────────────────────────────────────────────
    question_embedding:   list[float] | None  # text-embedding-004 output
    cache_hit:            bool                = False
    semantic_cache_hit:   bool                = False

    # ── Routing and translation ────────────────────────────────────────────────
    routing_source:       Source | None       = None
    routing_confidence:   float               = 0.0
    selected_query:       dict | None         = None   # query template with :user_id / :caseload_scope binds
    confidence_verdict:   Verdict | None      = None
    confidence_reason:    str | None          = None
    tier:                 Tier                = Tier.flash   # set to Tier.pro on Pro escalation

    # ── Validation ────────────────────────────────────────────────────────────
    accuracy_check:       AccCheck | None     = None
    validation_status:    ValStatus | None    = None
    validation_error:     str | None          = None        # sanitized — never contains row data
    repair_count:         int                 = Field(0, ge=0, le=MAX_REPAIR_ATTEMPTS)

    # ── Execution ─────────────────────────────────────────────────────────────
    # query_results and formatted_response are PHI-bearing. They are stripped from
    # the AgentState payload before LangGraph checkpoint writes via the custom serde.
    query_results:        list                = Field(default_factory=list, exclude=False)
    result_source:        Source | None       = None
    result_shape:         str | None          = None
    bounds_check:         str | None          = None

    # ── Output ────────────────────────────────────────────────────────────────
    formatted_response:   dict | None         = None
    outcome:              Outcome | None      = None
    trace_id:             str | None          = None

    # ── Multi-agent fields ─────────────────────────────────────────────────────
    parent_agent_id:      str | None          = None
    handoff_context:      dict | None         = None
    agent_version:        str                 = "v3"
```

---

## Cost Model — Optimized V3

### Per-Request Breakdown (Cache Miss)

| Step | Model | Tokens | Rate | Cost |
|---|---|---|---|---|
| Injection detector | Gemini Flash | 500 in / 50 out | $0.075/1M · $0.30/1M | $0.0000525 |
| Question embedding | text-embedding-004 | 200 tokens | ~$0.0001/1M | $0.000000020 |
| Source router | Gemini Flash | 2,000 in / 100 out | $0.075/1M · $0.30/1M | $0.000180 |
| Translator — cached context | Gemini Flash cache read | 15,000 cached | $0.01875/1M | $0.000281 |
| Translator — dynamic input | Gemini Flash standard | 2,000 uncached | $0.075/1M | $0.000150 |
| Translator — output | Gemini Flash | 600 out | $0.30/1M | $0.000180 |
| All other nodes (deterministic) | None | — | $0 | $0 |
| **Total per cache miss** | | | | **$0.000844** |

Cache hit: injection detect ($0.0000525) + embedding lookup ($0.00000002) + Cloud SQL query ≈ **$0.00015**

The injection detector runs on every request, before cache lookup, so its cost lands on both cache hits and misses. See *Latency Budget* below for the latency implication and a planned optimization.

### Monthly Cost (500 conversations/day, 50% semantic cache hit rate)

| Item | Calculation | Cost |
|---|---|---|
| LLM — 7,500 cache misses/month | $0.000844 × 7,500 | $6.33 |
| LLM — 7,500 cache hits/month | $0.00015 × 7,500 | $1.13 |
| Gemini 1.5 Pro tier-2 escalation (~6% of misses) | 450 × $0.024 | $10.80 |
| BigQuery data scanning | 150 queries/day × 30 × $6.25/TB × 500MB | $14.06 |
| Cloud SQL (incremental) | Existing instance + pgvector extension | $3.00 |
| Context cache storage (Flash) | 2 × 15K tokens × $0.01875/1M/hr × 24h × 30d | $0.40 |
| **Monthly Total** | | **~$36** |
| **Annual Total** | | **~$432** |

The Gemini 1.5 Pro escalation line item accounts for the *Accuracy-First Fallback* (described below). Expected rate is 5-8% of cache misses; budget assumes 6%. Pro escalation runs cold-path (no warm context cache) since the volume doesn't justify the cache storage rate at Pro tier — full ~17K input tokens per escalated translation.

### Full Version Comparison

| Version | Annual LLM-only Cost | Cost per Cache-Miss | Key Driver |
|---|---|---|---|
| V1 (Gemini Pro, 4 calls, Redis) | ~$18,000 | $0.1318 | 4 LLM calls, wasted translator |
| V2 (Claude Sonnet, 2 calls) | ~$7,968 | $0.0615 | Expensive model, no caching |
| V3 optimized | **~$432** | **$0.000844** | Flash + context cache + semantic cache + Pro fallback |
| LLM-only saving vs V2 | **~$7,536/year** | **94% cheaper** | |

> The 94% saving compares LLM spend only (V2 was LLM-only $7,968/yr). V3 full-system cost on the existing CVS infra is $250–$338/month per `COST-ESTIMATION-V3.md` Scenario A — driven by GKE and Cloud SQL compute that V2 also paid. The headline saving is the LLM line item shrinking; total system cost reduction is real but smaller.

---

## Latency Budget

V3's three serial Flash calls on cache miss (injection detect, route, translate) make latency a first-order design concern. The existing `careconnect-gke` cluster lives in **us-east4**; the Google AI Generative API endpoint is regional in us-central1. Cross-region round-trip is on the order of 30–60ms per call. Budgets below are sized for that real-world round-trip, not the in-region minimum.

| Path | p95 target | Dominant time slice |
|---|---|---|
| Cache hit — exact match (post-injection + embedding pre-compute) | < 700ms | Injection-detect Flash call (~300ms) + embedding (~150ms) |
| Cache hit — semantic match | < 1.0s | Injection-detect + embedding + pgvector lookup |
| Cache miss (no Pro escalation) | < 3.0s | Three Flash calls (injection + route + translate) + MCP execute + validators + output gate + audit |
| Cache miss with Pro escalation | < 5.0s | Adds one Pro call (~1.5s) plus re-validation |
| Injection-detect Flash call alone | < 300ms | Critical path of every request |
| MCP execute (GKE Agent → GKE MCP → DB) | < 500ms | Cluster-internal K8s Service + HTTP keep-alive pool |
| save_state + audit (synchronous, in-graph) | < 150ms | Cloud SQL write + BigQuery insert |
| Output security gate | < 50ms | Deterministic, no network |

**Rate limiting:** The FastAPI endpoint enforces per-user (30 req/min) and global (500 req/min) rate limits backed by the Cloud SQL `rate_limit_counter` table (single row per scope, one-minute rolling window, single `SELECT FOR UPDATE` + UPSERT per request, p99 < 10ms on the existing instance). Counters are coordinated across pods so HPA scale-out does not multiply the effective limit. Requests exceeding either limit return HTTP 429 with a `Retry-After` header. Limits are environment variables, not hardcoded.

**Planned optimization (post-POC):** replace the Flash injection detector with a local classifier (e.g., Meta `prompt-guard-86m` or equivalent) running in-process inside the FastAPI worker. Saves ~300ms on every request, removes one Flash call from the per-request hot path, and tends to *increase* injection-detection accuracy because dedicated classifiers are trained on injection corpora and outperform general-purpose LLMs on this specific task. Until that swap ships, the injection detector is a probabilistic Flash classifier; the PRD's earlier "deterministic" framing has been retracted in v1.1.

**Region:** GKE (agent + MCP server) runs in `us-east4` (existing cluster); the Google AI API endpoint is in `us-central1`. Cross-region adds 30–60ms per Flash call. Sprint 0 measures and re-pins the budget if needed. A region migration is out of POC scope.

**Synchronous persistence (HIPAA-correct):** `save_state` and `audit` are normal LangGraph nodes that run synchronously inside the request — they are NOT fire-and-forget. The latency tax is ~50–150ms, charged inside the cache-miss budget above. On BigQuery insert failure, `audit_node` writes the audit payload to a Cloud SQL `audit_dead_letter` table (also synchronous) so the record is durable before the request returns. A separate CronJob worker drains `audit_dead_letter` on a 5-minute cadence. The FastAPI `lifespan` shutdown handler awaits in-flight LangGraph executions for up to 10 seconds before exit, so rolling deployments do not drop audit records. This is the HIPAA-correct trade — accept ~150ms of latency in exchange for a guaranteed-durable audit trail.

---

## Accuracy-First Fallback to Gemini 1.5 Pro

V3 prioritizes accuracy over cost. The eval gate enforces ≥80% accuracy on the golden test set at release time, but does not block runtime. When Gemini Flash returns low confidence on a specific query, the orchestration escalates that single query to Gemini 1.5 Pro via the same `ChatGoogleGenerativeAI` interface — no provider hop, no extra auth surface.

**Trigger conditions** (any of):
- `routing_confidence < ROUTING_CONFIDENCE_THRESHOLD` (default 0.7) from Source Router → graph routes to `pro_escalation` immediately, skipping Flash translation.
- `confidence_verdict == "UNCERTAIN"` from Translator (and `tier == "flash"`) → graph routes to `pro_escalation` after Flash returns.
- Repair budget exhausted on Flash (`repair_count >= MAX_REPAIR_ATTEMPTS` after a Flash repair attempt) → graph routes to `pro_escalation` to retry the translation at Pro.

**Behavior:**
- `pro_escalation_node` is a real graph node, not a model-switching branch hidden inside `translator_node`. It re-emits a Translator-shaped state with `tier=Tier.pro` and the model handle swapped to `gemini-1.5-pro`. The graph then **always** flows `pro_escalation → accuracy_check → query_validator → mcp_executor → output_security → save_state → audit` — escalation bypasses no safety control.
- Pro runs cold-path: cached context does not transfer between Flash and Pro caches (Google AI cache rate is model-specific), so Pro pays full schema cost (~$0.024 per escalation).
- `AgentState.tier` carries `flash | pro` so observability splits metrics by tier and the conditional-edge logic can branch on which tier exhausted its repair budget.
- Pro repair budget is also `MAX_REPAIR_ATTEMPTS` (1 retry on Pro). After Pro repair budget is exhausted, the graph routes to `audit` with `outcome=Outcome.validation_failed`.

**Tracked as `tier_2_escalation_rate` SLO:**
- Expected rate: 5-8% of cache misses based on V2 telemetry patterns
- Annual cost impact: ~$130/year additional at projected volume
- A rising trend signals Flash regression on a class of queries — actionable signal, doesn't break the release train

**UNCERTAIN verdict — explicit user disclosure required:**

When `confidence_verdict == "UNCERTAIN"`, the response payload must include a `show_uncertainty_banner: true` field. The chat UI must render a visible on-screen warning banner — a metadata field alone is not sufficient. The banner must tell the care manager the answer is unverified and advise them to confirm against a source report before acting. Even after Pro escalation, the UNCERTAIN flag is retained in the response if the Pro translation still returns UNCERTAIN — the escalation improves translation quality but does not suppress the disclosure.

This trades a small ongoing cost for accuracy preservation when Flash hits an edge case it can't handle. Aligns with the principle: accuracy is primary, cost is secondary. Crucially, the fallback stays inside the Gemini family — no Anthropic, no Vertex, no extra provider permissions to procure.

---

## Multi-Agent Interface

Three ways other agents connect to this one. None require changes to V3 internals.

**As a LangGraph subgraph** — tightest integration, shared graph execution:
```python
from careconnect_agent import create_agent

careconnect = create_agent(config).as_subgraph
parent_graph.add_node("careconnect_query", careconnect)
parent_graph.add_edge("supervisor", "careconnect_query")
```

**As a LangChain tool** — any LLM with tool-calling can invoke it:
```python
careconnect_tool = create_agent(config).as_tool
# description: "Query CareConnect member and care gap data using natural language"
supervisor_llm.bind_tools([careconnect_tool, notification_tool, escalation_tool])
```

**As an MCP endpoint** — cross-system, future:
```
POST /mcp/v1/tools/careconnect_query
{ "question": "...", "user_id": "...", "scope": "..." }
```

---

## Security Model

All V2 security gates preserved. Input and output gates are LangGraph nodes outside the LLM path — the model cannot influence them.

**Container hardening (required for both agent and MCP server pods):**

```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
```

No capabilities are added back. The CI/CD security gate validates these settings are present on every manifest — any pod spec that omits or relaxes them blocks the merge. Writable scratch space needed at runtime (e.g., tmp dirs) must use an `emptyDir` volume mount, not a writable root filesystem.

**Snyk container scanning:** Both agent and MCP server images are scanned by Snyk in CI/CD on every build. Zero critical or high CVEs are required before the image is promoted to staging. Snyk is a merge gate — not a post-merge check.

| Gate | Node location | Trigger | Action |
|---|---|---|---|
| Scope classifier | `input_security` | Question not about CareConnect data | Refusal |
| Injection detector | `input_security` | Prompt override attempt | SOC alert + refusal |
| PII strip | `input_security` | Any PII in question | Scrubbed silently |
| Write op guard | `query_validator` | Any non-read operation | Hard block + SOC alert |
| Scope filter missing | `query_validator` | No `:user_id` placeholder in scope filter slot | Hard block + SOC alert |
| PHI in LLM payload | `translator` / `output_security` | MCP results must never enter any Gemini API call | Hard architectural constraint — MCP results go directly to output_security, never back through LLM |
| PHI scrub | `output_security` | PHI field outside user ACL | Field removed from results |
| Row cap | `output_security` | Results exceed limit | Truncated silently |
| Scope confirm | `output_security` | Row outside user caseload | Full result blocked |
| Sanity bounds | `output_security` | Value outside Bounds Registry | Hard block or soft warning |
| Field allowlist | `output_security` | Field not on per-intent allowlist | Field stripped from response |

---

## Migration Path from V2

**Stage 1 — Switch LLM provider (no pipeline change)**
- Replace `ChatAnthropic` (Claude Sonnet via LMS Gateway) with `ChatGoogleGenerativeAI` (Gemini Flash)
- Run eval gate on golden test set — require ≥80% accuracy before promoting
- No orchestration changes, no state schema changes

**Stage 2 — Add context caching**
- Implement Google AI API `CachedContent` for MQL and SQL translator prompts
- Add cache warm-up job (K8s CronJob, refreshes hourly and on schema push)
- Measure token cost reduction — expect ~75% on translator input tokens

**Stage 3 — Add pgvector semantic cache**
- Enable pgvector extension on existing Cloud SQL instance (`CREATE EXTENSION vector`)
- Add `question_embedding VECTOR(768)` column to `translation_cache` table
- Add `text-embedding-004` embedding call in `load_context_node` (not `semantic_cache_node`) — embedding stored in `AgentState.question_embedding` before the cache lookup runs
- Create HNSW index on embedding column (not IVFFlat — HNSW is recall-stable at ~250 row cache size)
- Monitor cache hit rate — expect rise from ~30% to ~50%

**Stage 4 — Implement the structured pipeline**
- This is the target design for Sprint 3 implementation, not a migration from running V2 code. V2 ran a hardcoded function sequence; V3 is a brand-new LangGraph pipeline built from scratch against the V3 spec. The pre-V3 ReAct draft never reached production.

**Stage 5 — Expose multi-agent interfaces**
- Export `as_subgraph` and `as_tool` — enables future agent composition

---
