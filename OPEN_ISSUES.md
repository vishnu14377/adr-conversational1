# CareConnect Agent V3 — Open Issues

> Captured from demo feedback session with business stakeholders.
> Owner: Sai. Target internal demo: ~Wednesday.

---

## 🔴 High Priority

### 1. Follow-up conversation handling is unreliable

**Why it matters:** Most repeated feedback during the demo. Business users expect ChatGPT/Copilot-style continuity — each question builds on the last without restating context.

**Expected behavior:**
```
User:  "Give me HCC counts"
Agent: [returns result]
User:  "Break it down by provider"       ← should use prior context
User:  "Show only PSYL"                  ← should filter prior result
User:  "What's the count?"               ← should reference prior answer
```
All of the above should resolve without requiring the user to re-state the full question.

**Action:** Investigate how context from prior answers is passed into the next turn's prompt. Confirm the `conversation_history` block in the dynamic translator suffix is actually carrying enough signal to resolve these follow-ups.

---

### 2. Memory/context retention — debug and document

**Questions to answer:**
- How many prior turns are currently stored and sent to the LLM? (check `SESSION_HISTORY_LIMIT`, currently 5)
- What fields are included per turn in `conversation_history`? (question, intent, outcome, bounded answer snapshot)
- Is the prior answer snapshot large enough for follow-up resolution? (currently capped at 1 200 chars per turn)
- Should answer data (rows/columns) be included in memory for follow-up questions that don't need a new DB trip?
- When can a follow-up be resolved from cached context without hitting MongoDB/BigQuery again?

**Action:** Trace a failing follow-up end-to-end through `load_context_node` → translator dynamic suffix → LLM response. Log exactly what the model receives.

---

### 3. Query preview / debug mode

**Requested by:** Vishnu.

**What needs to be visible:**
- Generated SQL or MongoDB aggregation pipeline (raw, before execution)
- Selected collection or table
- Filters applied
- Route taken by the agent (which nodes fired, which tier was used)

**Why:** Without this, the team cannot validate question translation, collection selection, or query correctness during debugging or stakeholder walkthroughs.

**Action:** Add a `debug` flag to `QueryRequest`. When set, include a `debug_info` block in `QueryResponse` containing `generated_query`, `routing_source`, `tier`, `confidence_verdict`, `schema_enrichment_used` (top chunks), and `few_shots_used` (example keys). Do not expose in production UI — dev/internal only.

---

## 🟡 Medium Priority

### 4. Cross-collection routing for Risk Gap questions

**Issue observed:** Generic Risk Gap questions only hit `RiskgapsProviderActions_MA_ACA` (MAACA). `RiskgapsProviderActions_MCD` (MMCD/Medicaid) data was ignored.

**Expected behavior:** When a question applies across lines of business and no LOB filter is specified, the agent should:
- Recognise that both MAACA and MMCD collections are in scope
- Query both and union the results, or ask the user to clarify LOB
- Not silently drop half the data

**Action:** Review `source_router` discriminator logic and `domain_hints` in `schema.yaml` for both provider-action collections. Add cross-LOB routing logic or update few-shot examples to demonstrate UNION/parallel queries.

---

### 5. Schema metadata enrichment

**Issue observed:** Agent performs well on high-level fields but struggles with deeper nested fields and relationships between collections.

**Gaps to address:**
- Nested field paths (e.g. `searchFields[].hccGroups[].hccId`) need richer descriptions
- Cross-collection relationships (which fields join MAACA ↔ MMCD) need explicit schema hints
- Business terminology synonyms need expansion (e.g. "risk score", "gap closure", "suspect gap")

**Action:** Audit `deploy/registries/schema.yaml` against the full MongoDB document structure. Add `synonyms`, `description`, and `relationships` metadata for nested fields. Re-run `careconnect-seed-schema` after updates to refresh the 325 pgvector RAG chunks.

---

### 6. Expand few-shot examples

**Requested by:** Phani.

**Gaps:**
- More coverage of business-specific questions (LOB breakdowns, vendor comparisons, gap aging)
- Follow-up conversation patterns (Question A → follow-up B referencing A's result)
- Cross-collection query examples (MAACA + MMCD together)

**Action:** Add examples to `few_shot_examples` (both `embedding` and `parent_embedding` columns). Prioritise follow-up patterns since that maps to Issue #1.

---

## 🟢 Nice to Have

### 7. Session/history model — ChatGPT-style sessions

**Current behaviour:** Every question is a separate history item stored against a `session_id`. There is no session boundary concept beyond the `SESSION_HISTORY_LIMIT` window.

**Desired behaviour:**
- Named or bounded chat sessions (like Copilot / ChatGPT threads)
- Context stays within a session; new session starts completely fresh
- Users can see and resume prior sessions

**Note:** Not a Day-1 requirement. Discussed as a medium-term UX improvement.

---

### 8. Rethink the confidence indicator

**Issue:** Current `confidence_verdict` (CONFIDENT / UNCERTAIN) reflects query-generation confidence — not answer accuracy. Business users may interpret it as "how accurate is this answer", which it is not.

**Options discussed:**
- Keep it as-is but rename to `query_confidence` or `translation_confidence`
- Remove it from the UI entirely
- Research answer-confidence alternatives (e.g. bounds check + result shape as a proxy)

**Action:** Decide on labelling before next demo to avoid stakeholder confusion.

---

## What Mubeen Wants Before Next Demo

| Item | Priority |
|---|---|
| Better follow-up conversations (Issues 1 & 2) | Must Have |
| Agent correctly understands Risk Gap questions across collections (Issue 4) | Must Have |
| More natural conversational experience overall | Must Have |
| Query preview / debug screen (Issue 3) | Nice to Have |
| Improved session history model (Issue 7) | Nice to Have |

---

## Immediate Next Steps (Monday)

1. Document all follow-up conversation failures observed during the demo with exact inputs and outputs.
2. Trace what context is currently stored in Postgres `conversation_turns` and what gets sent to the LLM per turn.
3. Add `debug_info` output to `QueryResponse` (generated query + collection + route).
4. Test MAACA + MMCD cross-collection scenarios and log routing decisions.
5. Review `schema.yaml` tool definitions and expand metadata for nested fields.
6. Schedule internal demo with Mubeen (target: Wednesday).
