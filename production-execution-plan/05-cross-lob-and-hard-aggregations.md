# 05 — Cross-LOB Routing & Hard Aggregations (agent work order)

Branch: `v35/05-cross-lob-hard-agg`. Flags: `CROSS_LOB_STRATEGY=legacy|bq_union|fanout|clarify` (default `legacy`), `PRO_REPAIR_ATTEMPTS=1` (raise to 2 in dev). Depends on 04 (relationships in registry). **I5:** every allowlist widening ships its abuse test in the same commit.

---

## Step 1 — Router policy

In `source_router`, build `LOB_TERMS = {"mcd": {"medicaid","mcd", <state names from registry>}, "maaca": {"ma","aca","medicare","marketplace","commercial"}}` from registry hints. New rule, evaluated before the discriminator when `CROSS_LOB_STRATEGY != legacy`:

```python
if intent_is_riskgap(state) and not mentions_lob(state.standalone_question, LOB_TERMS):
    if settings.CROSS_LOB_STRATEGY == "bq_union":
        return state.with_(routing_source="bigquery", union_scope="all_lob", routing_confidence=0.95)
    if settings.CROSS_LOB_STRATEGY == "fanout":
        return state.with_(routing_source="mongodb", fanout_collections=[MAACA_PA, MCD_PA])
    if settings.CROSS_LOB_STRATEGY == "clarify":
        return state.with_(routing_alternatives=["MA/ACA", "Medicaid", "Both"])
```

Also fix `domain_hints` for the MCD collections (medicaid/mcd/state vocabulary) so explicit-LOB questions route correctly under `legacy` too. AgentState: add `union_scope: str | None`, `fanout_collections: list[str] = []`.

## Step 2 — SQL translator: union pattern

Append to the SQL static prefix (re-warm CachedContent):

```
CROSS-LOB RULE: when the request context contains union_scope=all_lob, query BOTH conformance tables with identical column lists and UNION ALL, adding a literal lob column:
SELECT 'MA_ACA' AS lob, ... FROM `hcb-dev-careconnect-etl.conformance_layer_dataset.RiskgapProviderActions_MA_ACA` WHERE <scope+filters>
UNION ALL SELECT 'MCD' AS lob, ... FROM `...RiskgapProviderActions_MCD` WHERE <same>
Aggregate AFTER the union (wrap in a subselect) unless the question asks per-LOB, then GROUP BY lob.
DATE RULE: action_date is an ISO-8601 STRING. In WHERE, compare with string literals (sargable). For arithmetic use DATE_DIFF(PARSE_DATE('%Y-%m-%d', action_date), PARSE_DATE('%Y-%m-%d', other), DAY) in SELECT/HAVING only — never wrap the column in functions inside WHERE.
```

Translator prompt assembly passes `union_scope` in the dynamic suffix. `mcp_executor` BQ call: add `maximum_bytes_billed` param (env `BQ_MAX_BYTES_BILLED`, default 2 GB).

## Step 3 — Mongo fan-out fallback (for response-tracker questions without a BQ mirror)

`mcp_executor`: when `fanout_collections` set, POST the (identical-shape) pipeline per collection concurrently; new deterministic `merge_results` step (inside executor, before `output_security`): tag rows `{"lob": ...}`, concat, re-apply row cap; result_source="mongodb:fanout". If per-collection paths differ for the concept (registry `relationships`), translator emits per-collection pipelines keyed by collection — support `selected_query = {"fanout": {coll: pipeline, ...}}` in the validator (validate each) and executor.

## Step 4 — Allowlist audit + widening (the matrix is the deliverable)

Create `production-execution-plan/allowlist-audit.md`: one row per backlog question (all ~50 from RiskGap_AI_Questions.xlsx) → required operators → current validator verdict (run each through `query_validator` with a hand-written correct query) → disposition (`ok` / `allowlist+guard` / `route_bq` / `out_of_scope shell#1`). Expected widenings, each with guard + abuse test:

| Add | Guard | Abuse test |
|---|---|---|
| `$cond` inside `$group` accumulators | only within accumulator expressions | `$cond` in `$match` → rejected |
| `$bucket`, `$switch` | ≤ 8 buckets/branches | 50-bucket bomb → rejected |
| `$facet` | ≤ 2 branches, each branch must begin with `$match` | 5-branch / no-match branch → rejected |
| `$dateFromString`, `$dateDiff`, `$substrBytes` | barred from the leading `$match` (sargability) | date fn in leading `$match` → rejected |
| `$lookup` | `from` ∈ routed source's collection set only | foreign `from` → rejected |

Cost-heuristic fix: "MQL without leading `$match` = too_expensive" must accept (a) mandatory scope `$match` as the leading stage, (b) `$facet` whose every branch leads with `$match`. Executor: inject `maxTimeMS=30000` into every Mongo call.

## Step 5 — Few-shots (Phani) — `data/few_shots/translator_v35.jsonl`, seed via `seed_few_shots`

Write COMPLETE examples (question → full query JSON) for: 3 BQ cross-LOB unions (total gaps all LOB · per-LOB breakdown · LOB with lowest response rate) · 3 aging buckets (BQ CASE on DATE_DIFF; one MQL `$bucket` variant) · 3 date-diff (avg days request→action · providers slowest to respond · resolved-faster trend Q1 vs Q2 via conditional aggregation) · 3 conditional-rate rankings (response rate = COUNTIF(provider_action IN ('Assessed-present','Assessed Not Present'))/COUNT(*) per vendor · dismiss-rate per HCC · acceptance Historic vs Suspected) · 2 anti-joins (providers with zero responses: LEFT JOIN … IS NULL; MQL `$lookup`+`$match` size 0) · 3 post-resolver phrasings (fully-qualified standalone questions as 02 emits them). Use real column names: `hcc_code`, `provider_action`, `action_date`, `vendor`, `line_of_business`, `organization_tin`, `gap_request_date`, `aetna_gap_status` — verify against the BQ conformance schema first (`bq show --schema` via MCP or ask a teammate; adjust names, keep patterns).

## Step 6 — Tests

`tests/validators/test_allowlist_v35.py`: every widened operator accept+abuse pair; cost-heuristic fixes. `tests/routing/test_cross_lob.py`: no-LOB riskgap question → bigquery+union_scope under bq_union; "medicaid gaps by state" → MCD collection; "compare medicaid vs ma acceptance" → BQ GROUP BY lob; legacy flag ⇒ old behavior byte-identical. `tests/executor/test_fanout_merge.py`: tagging, cap re-application, per-collection validation. Golden subset for the six hard classes (feeds 07).

## Done when

- [ ] `allowlist-audit.md` complete — 100% of backlog dispositioned (paste summary counts in PR).
- [ ] All new validator guards + abuse tests green; full suite green; red-team write-op/DDL suites re-run clean.
- [ ] Dev with `bq_union`: "how many gaps do we have" returns combined result, `lob` visible in debug_info + interpretation echo ("MA/ACA + Medicaid combined").
- [ ] The six hard-class few-shot questions execute end-to-end in dev (attach debug_info for each).
