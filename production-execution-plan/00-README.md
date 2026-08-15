# CareConnect V3.5 — Agent-Executable Production Plan

> **EXECUTION MODE — read this first.** These files are work orders for an AI coding agent (Claude Code) operating inside the **CareConnect agent source repository** (the Python service: `careconnect-agent` entrypoint, `src/careconnect_agent/…`), NOT this docs repo. Execute files in numeric order, one file per branch/PR. Do not start a file until the previous file's **Done-when** checklist is green.

## Operating protocol for the agent

1. **Discovery before edits.** Module paths below come from the V3 architecture reference. If a named path doesn't exist, locate the real one by symbol search and adapt the path — never the spec:
   ```bash
   grep -rn "def load_context" src/ | head; grep -rn "semantic_cache" src/ | head
   grep -rn "build_state_graph\|StateGraph" src/ | head
   grep -rn "class AgentState" src/;  grep -rn "PhiSafeJsonPlusSerializer" src/
   grep -rn "build_audit_payload" src/;  grep -rn "SESSION_HISTORY_LIMIT" src/ deploy/
   ls deploy/registries/ migrations/ tests/
   ```
2. **Branch/commit convention:** branch `v35/<file-number>-<slug>` (e.g. `v35/02-context-resolver`); conventional commits; every PR includes the file's Done-when checklist in the description with boxes ticked.
3. **Feature flags:** every behavioral change ships OFF by default behind the env flag named in its file. Rollback must always be "flip the flag", never "revert the deploy".
4. **Tests:** run the full existing suite plus the new tests in each file before opening the PR: `pytest -q`. If the repo has a PHI CI scan, it must pass; file 09 extends it.
5. **When reality differs from spec** (a function signature, a table column), keep the *behavioral contract* in the file and adapt mechanics; note the deviation at the bottom of the PR description under `## Deviations`.
6. **Never do:** add new pip dependencies without flagging in the PR · touch `deploy/agent.networkpolicy.yaml` or securityContext blocks · log or persist any field listed in the PHI invariants below.

## Global invariants (violating any of these fails the PR)

- **I1 — No MCP result data in any Gemini call.** `query_results`, `formatted_response`, `summary`, `answer_digest` must never appear in any prompt builder. (PRD §7.4 hard rule; CI check in 09.)
- **I2 — PhiSafe serde stays intact.** `PhiSafeJsonPlusSerializer` strip list may only ever grow (02 adds `standalone_question`). `question` stays `repr=False`, never checkpointed.
- **I3 — Security gates unreachable by the LLM.** `input_security`, `query_validator`, `output_security` remain graph nodes outside every LLM path; nothing added may route around them. Pro escalation always re-enters `accuracy_check` → `query_validator`.
- **I4 — Flash `thinking_budget=0`** on every Flash call including new ones (measured +6–15s otherwise).
- **I5 — Read-only stays read-only.** Any allowlist widening (05) ships with a validator guard + abuse test in the same PR.
- **I6 — `AgentState` discipline:** `extra="forbid"`; updates via `state.with_(**patch)`; new fields added to the model, never ad-hoc dict keys.

## System facts the agent may rely on (from the live V3 reference)

18-node LangGraph StateGraph compiled with `AsyncPostgresSaver(serde=PhiSafeJsonPlusSerializer())`; `thread_id = trace_id` (per-request), `session_id` groups `conversation_turns`. Postgres schema `careconnect_ai` (tables: `translation_cache`, `few_shot_examples` [child `embedding` + `parent_embedding`, both VECTOR(768), RRF-60 retrieval], `schema_rag_entries` [VECTOR(3072), seq-scan], `conversation_turns`, `audit_dead_letter`, `context_cache_registry`, LangGraph checkpoint tables). Registries baked at `/etc/registries/` from `deploy/registries/{schema.yaml,bounds.yaml,field_allowlist.yaml}`, loaded by `_warm_registries()` at startup, never hot-reloaded. Models: `gemini-2.5-flash` (T=0.0 translate / 0.2 summarize), `gemini-2.5-pro` (thinking 8192), `gemini-embedding-001` (3072-d), `text-embedding-005` (768-d few-shots) via `infra/gemini.py` singleton, Vertex ADC. CLIs: `careconnect-agent`, `careconnect-warmup`, `careconnect-drain-audit`, `careconnect-seed-schema`, `seed_few_shots`. Env of note: `SESSION_HISTORY_LIMIT=5`, `SEMANTIC_CACHE_SIMILARITY_THRESHOLD=0.97`, `MAX_REPAIR_ATTEMPTS=1`, `ROUTING_CONFIDENCE_THRESHOLD=0.7`. Existing migrations: `000_dba_bundle.sql`, `010_conversation_turns.sql` → new ones start at `020_`.

## Production gates (what "works as expected" means; enforced in 07)

| Gate | Threshold |
|---|---|
| Behavioral contract — every turn ends as: correct answer · schema-grounded options · targeted "what's missing" · reasoned refusal. Never a raw error/dead-end on the answerable set | **100%** |
| Tier-1 harm classes (identity confusion, status inversion, out-of-caseload) | ≥ 95% |
| Single-turn aggregate (golden N ≥ 150) / Multi-turn resolution (≥ 30 scripts) | ≥ 85% / ≥ 85% |
| Ambiguity set: clarify-not-guess | 100% |
| Injection FN (incl. new generative surface) | ≤ 1% |
| p95: hit <1.0s · miss <3.0s · miss+resolver <3.6s · +Pro <5.5s | per path |

*(100% translation accuracy is not achievable by any NL→query system; 100% correct **behavior** is, and that is what the contract gate enforces.)*

## File order

`01` hotfixes (cache guard, debug_info, audit gaps) → `02` context resolver (follow-up engine) → `03` generative clarification + reasoned refusals → `04` retrieval upgrade + registry v1.3.0 → `05` cross-LOB routing + hard-aggregation coverage → `06` sessions + memory/PHI remediation → `07` eval sets + CI gates → `08` HNSW/scale/load-test → `09` security hardening + compliance closure → `10` flags, staged rollout, dashboards. Suggested owners: 01/02/06/08/10 Vishnu · 03/05/09 Sai · 04/07 Phani (+ clinical SME sign-offs) · Stage gates with Mubeen.
