# 01 — Hotfixes & Stabilization (agent work order)

Branch: `v35/01-hotfixes`. Four independent tasks; commit each separately. Flags: `CACHE_CONTEXT_GUARD` (default `false`).

---

## Task H1 — Semantic-cache context guard

**Why:** cache key = SHA-256(normalized_question + routing_source + schema_version); context-dependent turns ("show only Medicaid") are being cached and served across conversations. Stop reads **and** writes for such turns.

### H1.1 Create `src/careconnect_agent/context/standalone.py`

```python
"""Deterministic check: does a question stand alone without conversation context?"""
from __future__ import annotations

_PRONOUNS = frozenset({"it", "that", "those", "them", "these", "this", "same"})
_ELLIPTICAL_PREFIXES = ("only ", "just ", "and ", "now ", "what about", "how about",
                        "break ", "split ", "filter ", "exclude ", "also ", "instead",
                        "again", "drill", "show only", "remove ")
_COMPARATIVE = ("more", "fewer", "less", "earlier", "previous", "prior",
                "last one", "first one", "the first", "the last")

def is_standalone(question: str, domain_nouns: frozenset[str]) -> bool:
    t = " ".join(question.lower().split())
    toks = t.split()
    if len(toks) < 6:
        return False
    if any(w in _PRONOUNS for w in toks):
        return False
    if any(t.startswith(p) or f" {p}" in t for p in _ELLIPTICAL_PREFIXES):
        return False
    has_domain = any(n in t for n in domain_nouns)
    if any(c in t for c in _COMPARATIVE) and not has_domain:
        return False
    return has_domain
```

### H1.2 `domain_nouns` builder

In the SchemaRegistry module (discover: `grep -rn "class SchemaRegistry\|SchemaSnapshot" src/`), add a cached method `domain_nouns() -> frozenset[str]` returning lower-cased: every `domain_hints` phrase, every top-level synonym alias **and** canonical term, every collection display term. Build once at `_warm_registries()`.

### H1.3 Wire the guard

- Locate `semantic_cache` node (`grep -rn "def semantic_cache" src/`). At entry: if `settings.CACHE_CONTEXT_GUARD and state.turn_index > 0 and not is_standalone(state.normalized_question, registry.domain_nouns())` → set `state = state.with_(cache_hit=False, semantic_cache_hit=False)` and return without any DB lookup; increment Prometheus counter `cache_context_guard_skips_total{op="read"}`.
- Locate `save_state` node. Same predicate → skip the `translation_cache` upsert (still allow `conversation_turns` write), counter `{op="write"}`.
- Add `CACHE_CONTEXT_GUARD: bool = False` to settings + `deploy/` ConfigMap docs.
- **Note for 02:** once the resolver lands, the predicate input becomes `state.standalone_question`; leave a `# TODO(v35/02)` marker at both sites.

### H1.4 One-off purge script `scripts/purge_context_dependent_cache.py`

Iterate `careconnect_ai.translation_cache`, delete rows whose stored question fails `is_standalone` (fetch question from cache row if stored; if only the hash is stored, do a full `TRUNCATE` — 24 h TTL makes this cheap). Log count.

### H1.5 Tests `tests/context/test_standalone.py`

30 labeled cases minimum — must all pass:
standalone=True: "How many open gaps do we have this period?", "average days from gap request to provider action", "which vendor sent the most gaps this month", "medicaid gaps by state code", "top hcc codes for this provider", "response rate for commercial line of business", + 9 more from RiskGap_AI_Questions backlog.
standalone=False: "break it down by provider", "show only PSYL", "what about medicaid", "only the dismissed ones", "what's the count?", "same for last month", "now split by state", "and the previous period", "filter to vendor A", "the first one by month", + 5 more.
Also: integration test — two interleaved sessions replay the demo failure pair; assert no cross-context cache hit and `skips_total` incremented.

---

## Task H2 — `debug_info` on QueryResponse (Issue #3)

- `QueryRequest`: add `debug: bool = False`. Gate: honored only when the verified JWT `roles` claim contains `cc-eng`; otherwise ignore silently and write a structlog warn + audit outcome unchanged.
- Capture `route_trace: list[str]` — node names in execution order. Preferred: append `node_name` inside each node via a tiny decorator `@traced("node_name")` writing to `state.route_trace`; add `route_trace: list[str] = []` to AgentState (structural, serde-safe).
- `QueryResponse.debug_info` (only when gated on):

```python
class DebugInfo(BaseModel):
    generated_query: dict | None      # state.selected_query — templates only (:user_id/:caseload_scope placeholders)
    routing_source: str | None
    tier: str | None
    route_trace: list[str]
    confidence_verdict: str | None
    schema_enrichment_used: list[str] # element_keys of retrieved chunks (store them in state during schema_enrichment)
    few_shots_used: list[int]         # few_shot_examples.id list (store during prompt build)
    resolver: dict | None             # filled by 02: {mode, standalone_question, resolved_from_turns}
```

- Store `schema_enrichment_used` / `few_shots_used` into AgentState at their producing nodes (ids/keys only — never chunk text in the response).
- Tests: role-gated 200-with-debug vs silent-ignore; assert no bound values in `generated_query` (regex: no caseload literal from the test JWT appears).

---

## Task H3 — Audit gaps + checkpoint TTL (pre-Phase-1 blockers from V3-Technical-Design)

- In `build_audit_payload` (grep in `audit` node / `bigquery_audit.py`): populate `question_hash = sha256(normalized_question.encode()).hexdigest()` (normalized = already PII-stripped) and `pipeline_node = state.route_trace[-1] if state.route_trace else outcome-node-name`.
- New worker CLI `careconnect-cleanup-checkpoints` (mirror `careconnect-drain-audit` structure): `DELETE FROM checkpoints WHERE thread_id IN (SELECT thread_id FROM checkpoints GROUP BY thread_id HAVING max(<created-ts-column>) < now() - interval '30 days') LIMIT batches of 500` — inspect actual checkpoint table columns first (`\d checkpoints`) and use the library-appropriate timestamp; cascade blobs/writes tables the same way. K8s CronJob `deploy/cronjob.checkpoint-cleanup.yaml` nightly 03:00, policy Forbid, copied from the drain-audit manifest.
- Verify/export metric: dead-letter drain success ratio ≥ 99.9% (add gauge if absent).
- Tests: payload unit test asserts both fields non-null on success and blocked outcomes; cleanup job idempotency test against a seeded checkpoint fixture.

---

## Task H4 — Doc drift banners

Prepend to `PRD-CareConnect-QueryAgent-V3-POC.md`, `12-v3-design.md`, and each `.mmd` in the docs repo: `> ⚠️ HISTORICAL (POC era). Superseded by careconnect-doc.html — live: Gemini 2.5 Pro, gemini-embedding-001 3072-d, cache threshold 0.97, Vertex ADC, export cap 10,000.`

## Task H5 — PHI decision record (implementation in 06)

Add `production-execution-plan/DECISIONS.md` entry: *"`conversation_turns.formatted_response` will stop being persisted; replaced by `result_shape` + PHI-scrubbed ≤200-char `answer_digest`. Compliance sign-off: ______ / date."* Blocks 06 merge until signed.

---

## Done when

- [ ] `pytest tests/context/ -q` green; full suite green.
- [ ] Guard on in dev: metrics visible; interleaved-session replay shows zero cross-context hits.
- [ ] `debug=true` with `cc-eng` returns full block incl. route_trace; without role returns normal response.
- [ ] BQ sample rows show `question_hash` + `pipeline_node` populated; cleanup CronJob applied in dev and pruning.
- [ ] Banners committed; DECISIONS.md entry created.
