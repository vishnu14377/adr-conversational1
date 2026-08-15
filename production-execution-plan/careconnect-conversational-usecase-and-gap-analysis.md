# CareConnect Query Agent — Conversational Use Case & Gap Analysis

**Purpose:** Define the target conversational behavior in the user's own voice, then map every expectation against what V3 does today and name the specific inconsistency to fix. Written against the V3 architecture reference (careconnect-doc.html), PRD, OPEN_ISSUES.md, and the RiskGap question backlog.

---

## Part 1 — The Use Case, In the User's Voice

**Persona:** Asha, a care manager working provider risk-gap responses daily. No SQL, no MQL, no idea what a collection is. She knows vendors, HCC codes, gap statuses, and lines of business. She has used ChatGPT and Copilot, and that is her baseline expectation for how a conversation works.

### What Asha expects, in her words

"I want to ask my question the way I'd ask a colleague. If my question is vague, don't error out and don't give me a canned 'cannot process' message — tell me *what part* was unclear and give me real choices from the data so I can pick one. If I ask a follow-up, understand it's a follow-up. If I asked something seven questions ago and now say 'break that first one down by state,' connect it back. If you genuinely can't figure out what I'm referring to, ask me what's missing — specifically. When you answer, tell me how you read my question so I can trust the number. And when something is truly outside your data, explain *why* it's outside, not just 'out of scope.'"

### Expectation statements (testable)

1. **E1 — Clarify with options, never dead-end.** Any answerable-but-ambiguous question produces an LLM-written clarification that (a) names the ambiguous phrase, (b) offers 2–4 concrete interpretations, each grounded in a real schema field, and (c) invites a one-tap/one-word reply. Pattern:

   > **User:** "Which vendor is doing best?"
   > **Agent:** "'Best' can mean a few different things in our gap data — which would you like? (a) **highest provider response rate** (accepted + dismissed vs. total gaps sent), (b) **highest acceptance rate** (Assessed-present share), or (c) **highest gap volume** this period?"

   This exact pattern is already documented as a business expectation in the PO Questions sheet ("Agent should ask followup question… one engagement, next volume").

2. **E2 — Refusals carry reasons.** Out-of-scope, PHI-restricted, too-expensive, and cannot-route outcomes use the seven approved fallback shells from the PO sheet, with an LLM-filled, question-specific reason slot — never a bare hardcoded string.

3. **E3 — Follow-up vs. new question is an explicit decision.** Every turn after the first is classified: *follow-up* (resolve against history), *new* (fresh question), or *unresolvable reference* (ask what's missing). The decision uses prior questions, prior generated queries, schema, and few-shot examples — and it is visible in debug mode.

4. **E4 — Ten-turn memory with long-range resolution.** History window is 10 questions. A follow-up at turn 7 referring to turn 1 resolves correctly, and the agent cites which turn it resolved against ("Building on your question about open gaps from earlier…").

5. **E5 — Ask what's missing.** When a reference can't be resolved ("show only PSYL" with no prior vendor context), the agent asks a targeted question: "I couldn't find what PSYL refers back to — are you filtering the vendor ranking from earlier, or is PSYL a vendor or state code for a new question?"

6. **E6 — Hard aggregations work.** Aging buckets (0–30/31–60/61–90/90+), period-over-period comparisons, average days between two dates, dismiss-rate rankings, cross-LOB unions — the classes already sitting in the RiskGap question backlog — translate and execute, escalating to Pro when needed, without silent failure.

7. **E7 — Interpretation echo.** Every data answer states how the question was read and any assumptions made ("Counted open + suspect statuses; current report period; MA/ACA and Medicaid combined").

8. **E8 — Efficiency.** Clarification round-trips and follow-up resolution must not blow the latency budget: rewrite/classification fires only on turns that need it, and resolved standalone questions make the semantic cache *more* effective, not less.

### Example dialogue — the full loop

> **T1 User:** How many open gaps do we have this period?
> **T1 Agent:** 412 open or suspect gaps in the current report period, across both lines of business. *(How I read it: status ∈ {open, open-gap, suspect, suspected}; period = current report period; MA/ACA + Medicaid.)*
> **T2–T6:** vendor ranking, top HCC codes, response-rate questions…
> **T7 User:** Break that first count down by state.
> **T7 Agent:** Building on your open-gaps question from the start of this chat — state-level grouping applies to the Medicaid segment (program.stateCode). CT: 118, NY: 94, NJ: 61… MA/ACA gaps aren't tracked by state; want those shown as one combined row?
> **T8 User:** show only PSYL
> **T8 Agent:** I couldn't find PSYL earlier in our conversation. Is it a vendor I should filter the state breakdown by, or something else? *(If PSYL is in the vendor enum: applies it and says so instead of asking.)*

---

## Part 2 — Current Inconsistencies (V3 vs. the Use Case)

| # | Expectation | What V3 actually does today | Where it lives | The inconsistency |
|---|---|---|---|---|
| 1 | E1 clarify with options | `ask_clarification` is a **deterministic node with canned text**, reached only when source routing ties or Pro returns UNCERTAIN | ask_clarification node; edge map | The system knows *that* it's uncertain but never *what* the ambiguity is. The translator/Pro JSON contract has **no `clarifying_question` or `interpretation_options` fields** — the model's own understanding of the ambiguity is discarded before it reaches the user |
| 2 | E1 no dead-ends | Repair budget exhausted, accuracy-check mismatch, and hard blocks route to `audit → END` with generic errors | Edge map ("budget exhausted → audit → END") | Terminal failure paths **terminate instead of degrading to a clarification**. An answerable-but-hard question exits as an error rather than a conversation |
| 3 | E2 refusals with reasons | Intent classifier returns a category; out-of-scope and unsafe outcomes render fixed strings | input_security → output_security refusal path | Classifier emits no reason; the seven approved fallback shells from the PO sheet aren't wired in as LLM-filled templates |
| 4 | E3 follow-up detection | **No follow-up/new classifier exists.** History is a passive 5-turn suffix on the translator prompt; embedding, cache, router, and schema retrieval all operate on the **raw follow-up text** | load_context → translator dynamic suffix | Coreference resolution is dumped on the translator at the end of a pipeline whose cache and routing already decided on a context-free string. This is the root cause of OPEN_ISSUES #1/#2 |
| 5 | E4 ten-turn window | `SESSION_HISTORY_LIMIT=5`; no turn citation; no long-range resolution mechanism | load_context_node config | Half the required window, and nothing anchors "that first one" to turn 1 — adjacency is implicitly assumed |
| 6 | E4/E8 cache safety | Cache key = SHA-256(normalized_question + routing_source + schema_version); save_state fires on any CONFIDENT success | semantic_cache / save_state | **Context-dependent turns poison the cache**: "show only Medicaid" cached from one conversation can be served at ≥0.97 similarity into a different context. Gets worse at 10 turns |
| 7 | E3/E4 history content | Per-turn memory = question, intent, outcome, 1,200-char **answer-text snapshot** | conversation_history / conversation_turns | Answer text is the weakest signal for query building and the biggest PHI risk (PRD hard rule: MCP results never enter a Gemini call). The strong, PHI-free signals — **generated_query template + result_shape** — are what resolution actually needs |
| 8 | E1 field-level ambiguity | Ambiguity detection exists **only at source level** (MongoDB vs BigQuery discriminator ties) | source_router | "Which vendor is best?" routes cleanly to one source and sails past every ambiguity gate — semantic ambiguity within a source has no detector |
| 9 | E6 tough aggregations | Validator stage-3 cost heuristics + operator allowlist + string-stored dates; repair budget = 1 | query_validator; field semantics rules | The business backlog contains aging buckets, period-over-period, and date-difference questions that need `$bucket`/`$switch`/date parsing (or BQ `DATE_DIFF`) — classes the current allowlist/heuristics likely block or the string-date rules make untranslatable in MQL. Nobody has audited the allowlist against the actual question backlog |
| 10 | E6 few-shot coverage | Few-shots cover single-turn basics; none for follow-up patterns, cross-LOB unions, or the hard aggregation classes | few_shot_examples | OPEN_ISSUES #6, plus a new gap: a **rewriter few-shot store doesn't exist** because the rewriter doesn't exist |
| 11 | E7 interpretation echo | `summarize` produces a 2–3 sentence clinical summary; no assumptions, no "how I read it" | summarize node; translator contract | Translator contract has no `assumptions[]` field; trust artifacts (validation passed, bounds pass, scope confirmed) are computed but never shown |
| 12 | E1–E5 schema understanding | Schema RAG = dense-only top-8 cosine over 325 granular shards (per-field, per-enum, per-synonym chunks); known short-label collisions ("HCC 1" vs "hcc count") | schema_enrichment | Rich synonyms/definitions exist but **retrieval doesn't reliably deliver the right cards**: no lexical/hybrid channel, no reranking, no grouping of shards back into coherent field cards, thin k for multi-entity questions. This is why "even with rich schema it doesn't understand" |
| 13 | E4 session boundaries | Every question appends to one per-session_id stream; no new-chat boundary | conversation_turns; Issue #7 | At a 10-turn window, **session boundaries stop being nice-to-have and become a prerequisite** — without them the resolver will connect "break that down" to a stale topic from yesterday |
| 14 | All | Golden set is single-question only | eval harness; PRD §7.2 | **No multi-turn eval tier exists.** Follow-up quality can regress invisibly — the 80% gate literally cannot see the behavior Mubeen called the top must-have |

---

## Part 3 — Target Shape (V3.5) and Build Order

### New/changed components

**context_resolver node** (new; after load_context, before semantic_cache). Flash, T=0, strict JSON:
```json
{
  "mode": "new" | "followup" | "unresolvable",
  "standalone_question": "...",        // full self-contained rewrite when followup
  "resolved_from_turns": [1, 7],       // turn citations for debug + UX
  "clarifying_question": null | "..."  // when unresolvable: ask what's missing
}
```
Inputs: current question + up to 10 prior turns as {turn_index, normalized_question, routing_source, generated_query *template*, result_shape} — no answer text, keeping the PRD's no-results-to-LLM invariant intact. Fires conditionally: turn_index > 0 AND the question fails a deterministic standalone-ness check (pronouns/ellipsis/"only"/"break it down"/short length) — first questions and clearly-new questions pay zero added latency. Downstream, everything (embedding recompute, cache, router, enrichment, translator) consumes `standalone_question`. Cache key and save_state key on the rewritten question; unresolvable turns skip cache entirely.

**Extended translator + Pro contract:** add `assumptions[]`, `clarifying_question`, `interpretation_options[] {label, field_path, reason}`. UNCERTAIN still escalates; Pro UNCERTAIN now carries its options forward instead of dying into canned text.

**ask_clarification becomes a renderer, not an author:** it renders model-generated options after deterministic guards — every referenced field_path must exist in the SchemaRegistry, PHI scrub, length cap — with the current canned text demoted to fallback when generation fails a guard. Same pattern for refusals: the seven PO-sheet fallback shells become templates with an LLM-filled reason slot.

**History:** SESSION_HISTORY_LIMIT 5 → 10; per-turn content switched to the PHI-free structural bundle above; explicit session boundary (new-chat resets the window) shipped alongside — see inconsistency #13.

**Retrieval:** add a lexical channel (pg_trgm / tsvector) RRF-merged with dense; group shards into field cards before prompt assembly; k=12 with pruning; always include the routed collection's card.

**Aggregation ceiling:** audit the operator allowlist against all ~50 backlog questions; route date-math and cross-LOB unions to BigQuery (conformance layer + `UNION ALL` + `DATE_DIFF` on parsed strings) or add `$bucket`/`$switch`/`$dateFromString` with guards; raise repair budget to 2 on the Pro tier only.

**Eval:** add a multi-turn tier to the golden set — scripted conversations (≥30 scripts, including a turn-1-referenced-at-turn-7 case and an unresolvable-reference case) scored on resolution correctness, not just final query accuracy.

### Build order

1. **Hotfix (pre-Wednesday):** skip semantic-cache read/write for turns failing the standalone-ness check — stops cache poisoning and wrong-context answers in the demo.
2. **context_resolver** node + its few-shot store — fixes Issues #1/#2 at the root.
3. **Contract extension + clarification renderer** — delivers the "best vendor → options" experience and kills dead-ends (E1/E2).
4. **History to 10 + structural turn content + session boundary** (E4, #13).
5. **Retrieval hybrid + field cards** (E12 → proper understanding of rich schema).
6. **Allowlist-vs-backlog audit + BQ routing for time-math** (E6).
7. **Few-shot expansion** — translator (tough patterns, cross-LOB) and resolver (follow-up patterns) together.
8. **Multi-turn eval tier** — so none of the above regresses invisibly.

### Acceptance criteria (demo-ready definition of "works")

Ambiguous questions produce ≥2 schema-grounded options and zero dead-end errors on the answerable set; follow-ups resolve correctly across a 10-turn window with turn citation shown in debug mode; unresolvable references produce a targeted "what's missing" question; context-dependent turns never hit or write the semantic cache under a context-free key; every data answer carries an interpretation echo; the multi-turn eval tier scores ≥85% resolution accuracy before the next stakeholder demo.
