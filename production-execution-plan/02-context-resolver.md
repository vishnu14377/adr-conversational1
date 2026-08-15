# 02 — Context Resolver / Follow-up Engine (agent work order)

Branch: `v35/02-context-resolver`. Flag: `RESOLVER_MODE=off|shadow|on` (default `off`). Depends on 01 (shares `is_standalone`, `route_trace`, debug plumbing). **Invariants I1/I2 apply hard here** — the resolver prompt may contain questions, query *templates*, and result *shapes*; never `formatted_response`/`summary`/rows.

---

## Step 1 — Migration `migrations/020_resolver.sql`

```sql
ALTER TABLE careconnect_ai.conversation_turns
  ADD COLUMN IF NOT EXISTS result_shape JSONB,
  ADD COLUMN IF NOT EXISTS resolver_mode TEXT,
  ADD COLUMN IF NOT EXISTS resolved_from_turns SMALLINT[],
  ADD COLUMN IF NOT EXISTS standalone_question_hash TEXT;
-- generated_query is already stored per-turn; verify column exists (010 migration). If absent, add generated_query JSONB.
```

Settings: `SESSION_HISTORY_LIMIT` default **5 → 10**; new `RESOLVER_MODE: str = "off"`, `RESOLVER_TIMEOUT_SECONDS: float = 6.0`.

## Step 2 — AgentState additions (respect I6)

```python
resolver_mode: Literal["new", "followup", "unresolvable"] | None = None
standalone_question: str | None = Field(default=None, repr=False)   # question-class PHI
resolved_from_turns: list[int] = []
resolver_confidence: float | None = None
resolver_fallback: bool = False
```

Add `standalone_question` to the `PhiSafeJsonPlusSerializer` strip list (grep the strip set; extend it — I2).

## Step 3 — History bundle in `load_context`

Where prior turns are loaded, build `state.history_bundle: list[dict]` (serde-stripped as well) with EXACTLY:

```python
{"turn": t.turn_index, "question": t.normalized_question, "intent": t.intent,
 "source": t.routing_source, "query_template": t.generated_query,
 "result_shape": t.result_shape}   # {"row_count": int, "columns": [..], "truncated": bool} | None
```

Runtime assertion (raise `ValueError`, don't silently strip): bundle dicts contain no keys named `formatted_response`, `summary`, `rows`, `answer_digest`. Populate `result_shape` at write time: in `save_state`/`audit` turn-upsert, derive from `state.result_shape` (already in AgentState) — columns = keys of first result row **names only**.

## Step 4 — Prompt module `src/careconnect_agent/prompts/resolver.py`

```python
RESOLVER_STATIC_RULES = """You are the conversation-context resolver for CareConnect, a read-only healthcare analytics assistant for care managers.
Your ONLY job: decide whether the user's new message depends on earlier turns in this session, and if it does, rewrite it as ONE fully self-contained question.

Output STRICT JSON only — no markdown, no fences — exactly:
{"mode":"new"|"followup"|"unresolvable","standalone_question":"...","resolved_from_turns":[<int>...],"clarifying_question":null|"...","confidence":<0.0-1.0>}

Rules:
1. mode="new": the message is understandable with no earlier turns (even if terse). Greetings, capability questions, and out-of-domain chatter are also "new".
2. mode="followup": the message references earlier turns via pronouns (it/that/those), ellipsis ("only Medicaid", "break it down"), ordinals ("the first one"), bare metrics ("what's the count?"), or comparatives ("vs the previous period").
3. For followups, standalone_question MUST copy every filter, grouping, time period, metric, and entity needed from the referenced turn(s), phrased in the same business vocabulary. It must contain no pronouns pointing at prior turns and must be answerable in isolation.
4. NEVER invent a filter, value, or entity that appears in neither the new message nor the referenced turns.
5. References may target ANY turn in the window, not only the most recent. A bare metric question refers to the most recent data-bearing turn (result_shape present).
6. mode="unresolvable": the referent cannot be identified, or two turns are equally plausible. Then clarifying_question = ONE short question naming the candidate referents (e.g., "the vendor ranking from earlier, or the state breakdown?"), and standalone_question = the original message verbatim.
7. resolved_from_turns lists every turn number you used; [] for mode="new".
8. confidence: your probability the mode AND rewrite are correct."""

def render_history(bundle: list[dict]) -> str:
    lines = []
    for t in bundle:
        rs = t.get("result_shape") or {}
        lines.append(
            f"[turn {t['turn']}] Q: {t['question']}\n"
            f"  intent={t.get('intent')} source={t.get('source')} "
            f"result: rows={rs.get('row_count')} cols={rs.get('columns')} truncated={rs.get('truncated')}\n"
            f"  query_template: {t.get('query_template')}"
        )
    return "\n".join(lines)

def build_resolver_prompt(question: str, bundle: list[dict], few_shots: str) -> str:
    return (f"{few_shots}\n\nSESSION HISTORY (oldest first):\n{render_history(bundle)}\n\n"
            f"NEW MESSAGE:\n{question}\n\nJSON:")
```

Static rules ride CachedContent: add registry key `resolver` to `context_cache_registry`; extend the `careconnect-warmup` CronJob worker to warm it (copy the mql/sql pattern; TTL 75 min).

## Step 5 — Node `context_resolver` (new module beside the other nodes)

Behavior (Pydantic result model `extra="forbid"` mirroring the contract):

```python
async def context_resolver_node(state: AgentState) -> AgentState:
    if settings.RESOLVER_MODE == "off" or state.turn_index == 0:
        return state.with_(resolver_mode="new", standalone_question=state.normalized_question)
    if is_standalone(state.normalized_question, registry.domain_nouns()):
        return state.with_(resolver_mode="new", standalone_question=state.normalized_question)
    METRIC_resolver_fired.inc()
    few = await few_shot_store.retrieve(source_key="resolver", query=state.normalized_question, k=4)  # RRF like translator
    prompt = build_resolver_prompt(state.normalized_question, state.history_bundle, few)
    try:
        raw = await gemini.flash(prompt, temperature=0.0, max_output_tokens=400,
                                 thinking_budget=0, timeout=settings.RESOLVER_TIMEOUT_SECONDS,
                                 cached_context_key="resolver")     # match existing GeminiClient API
        out = ResolverResult.model_validate_json(strip_fences(raw)) # one repair retry inside the same timeout budget
    except Exception:
        METRIC_resolver_fallback.inc()
        return state.with_(resolver_mode="new", standalone_question=state.normalized_question,
                           resolver_fallback=True)
    if settings.RESOLVER_MODE == "shadow":
        log_shadow(out)                                             # structlog + turn columns; downstream uses raw question
        return state.with_(resolver_mode="new", standalone_question=state.normalized_question)
    patch = dict(resolver_mode=out.mode, resolved_from_turns=out.resolved_from_turns,
                 resolver_confidence=out.confidence)
    if out.mode == "followup":
        emb = await gemini.embed(out.standalone_question)           # 100 ms re-embed; replaces prefetch embedding
        return state.with_(**patch, standalone_question=out.standalone_question, question_embedding=emb)
    if out.mode == "unresolvable":
        return state.with_(**patch, standalone_question=state.normalized_question,
                           clarifying_question=out.clarifying_question)   # field consumed by 03 renderer
    return state.with_(**patch, standalone_question=state.normalized_question)
```

## Step 6 — Graph wiring (in `build_state_graph`)

```
load_context → context_resolver
context_resolver: mode=="unresolvable" → ask_clarification → audit → END
                  else                 → semantic_cache
```

Downstream substitutions (grep each consumer of `normalized_question` past load_context and switch to `standalone_question`): `semantic_cache` key+similarity · `source_router` input · `schema_enrichment` retrieval text · translator dynamic suffix — **and** shrink the translator's history suffix to only the turns in `resolved_from_turns` (empty for mode=new) · `save_state` cache row stores `standalone_question` (+ its embedding) and sets `standalone_question_hash` on the turn row · `summarize`/response: when mode=followup prepend `"Building on your question from turn {n}: "` (smallest of resolved_from_turns) · `audit` payload: add `resolver_mode`, `resolver_fallback` (structural only) · `debug_info.resolver` (H2) filled. H1 guard sites: switch predicate input to `standalone_question` and clear the TODO markers — the guard remains as belt-and-braces for `resolver_fallback=True` turns (skip cache read+write on fallback).

## Step 7 — Seed resolver few-shots

`data/few_shots/resolver.jsonl`, loaded via `seed_few_shots` with `source_key='resolver'` (child embedding on the NEW MESSAGE text; parent on history-digest + message). ≥ 20 examples covering: pronoun follow-up · elliptical filter ("show only PSYL" → rewrite with vendor filter) · long-range ordinal ("break that first count down by state" resolving turn 1 of 7) · bare metric after data turn · topic switch → new · unresolvable (two plausible referents → clarifying_question naming both) · reply-to-clarification pick ("(b)" / "the response rate one" → rewrite of the clarified question) · comparative ("vs prior period") · "same for medicaid". Write full JSON answers matching the contract exactly.

## Step 8 — Tests

`tests/nodes/test_context_resolver.py`: contract validation (fences stripped, extra fields rejected) · off/shadow/on behavior · fallback on timeout sets `resolver_fallback` and proceeds · bundle PHI assertion raises on seeded `formatted_response` key.
`tests/integration/test_multiturn.py`: scripted 10-turn session via the compiled graph with a stubbed Gemini: T1→T7 long-range passes; unresolvable yields clarification path; interleaved two-session isolation (needs 06 for full guarantee — until then assert history query filters by session_id); latency test: fired-turn overhead < 700 ms p95 with the stub replaced by a 400 ms sleeper.

## Done when

- [ ] Migration applied; suite green; new tests green.
- [ ] `RESOLVER_MODE=shadow` in dev+staging: `resolver_fired_total`, mode split, `resolver_fallback_total` (<5%), shadow log rows visible in `conversation_turns`.
- [ ] Manual replay of the demo failures ("Give me HCC counts" → "Break it down by provider" → "Show only PSYL" → "What's the count?") in `on` mode in dev: all four resolve; debug_info shows mode + rewrite + resolved_from_turns.
- [ ] Grafana panel added (fired rate, modes, fallback, added-latency histogram).
- [ ] PR notes any Deviations; 00-README invariants checklist pasted and ticked.
