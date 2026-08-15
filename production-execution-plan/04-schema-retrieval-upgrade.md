# 04 — Schema Retrieval & Registry v1.3.0 (agent work order)

Branch: `v35/04-retrieval`. No user-facing flag (retrieval is internal); gated by the linking-recall test. Two parts: **A** mechanics, **B** registry content. Parallel-safe with 02/03.

---

## A. Hybrid retrieval + field cards

### A1 Migration `migrations/021_schema_rag_lexical.sql`

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
ALTER TABLE careconnect_ai.schema_rag_entries
  ADD COLUMN IF NOT EXISTS tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('english', text)) STORED;
CREATE INDEX IF NOT EXISTS schema_rag_tsv_gin ON careconnect_ai.schema_rag_entries USING gin (tsv);
CREATE INDEX IF NOT EXISTS schema_rag_key_trgm ON careconnect_ai.schema_rag_entries USING gin (element_key gin_trgm_ops);
```

### A2 Structured element_key

In `careconnect-seed-schema`, change key format to `"{collection}|{field_path}|{kind}"` (kind ∈ collection|field|enum|synonym|hcc|relationship). Regrouping becomes `key.split("|")`.

### A3 Retrieval function (replace dense-only top-8 in `schema_enrichment`)

```python
async def retrieve_schema_context(q_text, q_emb, source, budget=settings.SCHEMA_CONTEXT_TOKEN_BUDGET):  # default 1800
    dense = await db.fetch("""SELECT element_key, text, embedding <=> %s AS d
                              FROM careconnect_ai.schema_rag_entries
                              WHERE schema_version=%s ORDER BY d ASC LIMIT 12""", (q_emb, ver))
    lex = await db.fetch("""SELECT element_key, text,
                                   ts_rank(tsv, websearch_to_tsquery('english', %s)) AS r
                            FROM careconnect_ai.schema_rag_entries
                            WHERE schema_version=%s AND tsv @@ websearch_to_tsquery('english', %s)
                            ORDER BY r DESC LIMIT 12""", (q_text, ver, q_text))
    short_toks = [t for t in q_text.split() if len(t) <= 4 and any(c.isdigit() for c in t) or t.isupper()]
    trgm = await db.fetch("""SELECT element_key, text, similarity(element_key, %s) AS s
                             FROM careconnect_ai.schema_rag_entries
                             WHERE schema_version=%s AND element_key %% %s
                             ORDER BY s DESC LIMIT 6""", (tok, ver, tok)) if short_toks else []
    merged = rrf_merge([dense, lex, trgm], k=60)              # reuse the few-shot RRF helper
    cards = group_into_field_cards(merged)                     # by "{collection}|{field_path}" parent; field+enums+synonyms together
    cards = [collection_card(source)] + cards                  # routed collection card always first
    state.schema_enrichment_used = [c.key for c in cards]      # for debug_info (01/H2)
    return truncate_to_tokens(cards, budget)
```

Env: `SCHEMA_CONTEXT_TOKEN_BUDGET=1800`.

### A4 Linking-recall eval (the missing diagnostic)

`eval/linking_recall.py` + `eval/data/linking_set.jsonl` (60 cases: `{"question": ..., "required_paths": ["RiskgapsProviderActions_MCD|providerAction", ...]}` — author from the RiskGap backlog incl. ≥15 MCD, ≥10 enum/short-code cases like "HCC 1", "PSYL", state codes). Metric = % required paths present in assembled context. **CI gate ≥ 95%.** Run before/after and paste the delta in the PR.

## B. Registry v1.3.0 (`deploy/registries/schema.yaml` + `field_allowlist.yaml`)

1. **MCD deep enrichment** — for `RiskgapsProviderActions_MCD`, add/extend these fields with description + synonyms **taken from `riskgap_provider_mcd_keys.xlsx`** + enum values where known: `hcc`, `metadata.hcc`, `providerAction` (enums: `Assessed-present`, `Assessed Not Present`; note MCD path is top-level vs MA/ACA `providerActionDetails.providerAction`), `providerActionUpdateTimeStamp`, `metadata.providerActionDateTimeStamp`, `metadata.providerActionReceiveDateTimeStamp`, `metadata.consumerSystem`, `metadata.operationalStatus` (Success|Failed|Pending), `metadata.aetnaTransactionId`, `metadata.memberId` (mark `aggregate_only: true`), `searchParameters[].key/value`, encounter `status/class.code/gapRequestId/patientEncounter.value`, practitioner NPI value, `errorLog.errorStage/errorMessage`.
2. **Block the FHIR identity subtree** in `field_allowlist.yaml` (deny — translator may never emit; output gate strips): `resource.parameter[Patient].resource.name*`, `...birthDate`, `...gender`, `...identifier[MB].value`, payer identifiers, `resource.parameter[Practitioner].resource.name*`.
3. **New registry key `do_not_confuse_with`** rendered into field cards by the seeder:

```yaml
do_not_confuse_with:
  - fields: [metadata.providerActionDateTimeStamp, metadata.providerActionReceiveDateTimeStamp, providerActionUpdateTimeStamp]
    rule: "action=when the provider acted (use for response timing); receive=system ingestion; update=record modification. Default to action for business timing questions."
  - fields: [metadata.hcc, hcc]
    rule: "Same value; prefer top-level hcc."
  - fields: [metadata.appointmentDate, audit.createdDateTime]
    rule: "createdDateTime is absent on open gaps — never use for gap age; use appointmentDate."
```

4. **New key `relationships`** (cross-collection/backend join semantics — Mongo has no FKs to learn from):

```yaml
relationships:
  - concept: hcc_code
    paths: {RiskgapsProviderActions_MA_ACA: hcc, RiskgapsProviderActions_MCD: hcc,
            RiskgapResponseTracker_MA_ACA: "searchFields[].hccGroups[].hccId",
            RiskgapResponseTracker_MCD: "searchFields[].hccGroups[].hccId", bigquery: hcc_code}
  - concept: provider_action
    paths: {RiskgapsProviderActions_MA_ACA: providerActionDetails.providerAction,
            RiskgapsProviderActions_MCD: providerAction, bigquery: provider_action}
    values: {accepted: "Assessed-present", dismissed: ["Assessed Not Present", "Assessed-absent"]}
  - concept: line_of_business
    rule: "MA_ACA collections: program.lob in ['C','M']. MCD: program.lob='D'; geographic grouping via program.stateCode."
```

5. **Synonym expansion** (top-level): risk score, gap closure, suspect gap, gap aging, response rate, no response, engagement, acceptance rate, dismissal rate, vendor performance.
6. Seeder: emit 1 chunk per relationship + per do_not_confuse rule; bump `schema_version: "1.3.0"`; run `careconnect-seed-schema`; verify counts (~325 → ~430) and that the trigram channel resolves "HCC 1" (add to linking set).

## Done when

- [ ] Migration applied; seeder run; chunk counts logged in PR.
- [ ] Linking recall ≥ 95% on the 60-case set (report before/after; before is expected ~70–85%).
- [ ] Blocked-field probes ("list patient birth dates with open gaps") fail at validator with shell `phi_restricted` — add 3 such cases to tests.
- [ ] Existing single-turn golden subset shows no regression (run the current golden set; attach diff).
- [ ] `schema_enrichment_used` visible in debug_info.
