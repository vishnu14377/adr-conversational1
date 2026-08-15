# 10 — Rollout & Operations (agent work order)

Branch: `v35/10-rollout` (mostly manifests, dashboards, docs). Nothing here writes feature code; it promotes what 01–09 built, with rollback = flag flip.

---

## Step 1 — Flag inventory (ConfigMap `careconnect-agent-config`; document defaults per env)

| Flag | dev | staging | prod start | Owner file |
|---|---|---|---|---|
| CACHE_CONTEXT_GUARD | on | on | on | 01 |
| RESOLVER_MODE | on | shadow→on | off→shadow→on | 02 |
| CLARIFY_GEN_MODE / REFUSAL_REASONS | on | on | off→on | 03 |
| CROSS_LOB_STRATEGY | bq_union | bq_union | legacy→bq_union | 05 |
| SESSION_BOUNDARIES | on | on | off→on (with resolver on) | 06 |
| PRO_REPAIR_ATTEMPTS | 2 | 2 | 1→2 | 05 |

## Step 2 — Staged promotion (each stage = a dated checklist PR against this file)

**Stage A (wk 1) — dev+staging:** 01 merged; guard metrics sane; debug_info in use by the team. Gate: 01 Done-when.
**Stage B (wks 2–3) — prod shadow:** `RESOLVER_MODE=shadow` in prod; 04 retrieval live in staging. Gate: ≥1 wk shadow on real traffic; grade ≥100-sample shadow log vs human judgment ≥85% mode+rewrite correctness; `resolver_fallback_rate` <5%; linking recall ≥95% in staging.
**Stage C (wks 4–5) — internal cohort:** resolver **on** + clarify-gen + bq_union + sessions for the eng+Mubeen cohort (JWT-claim or header-based cohort routing if available; else time-boxed full-prod with heightened alerting). Gate: all 07 CI gates green; nightly multi-turn green 2 consecutive wks; 08 load test green; 09 §1–3 done.
**Stage D (wk 6) — all care managers:** everything on. Gate: behavioral-contract 100% on the weekly prod sample; Tier-1 ≥95%; **signed:** Compliance (H5/06), Legal (BAA, 09), clinical SME (harm tiers, 07), Mubeen (UX walkthrough of the behavioral-contract set); rollback rehearsed in staging (<5 min flags-off to V3, timed).

Rollback playbook per stage: flag flips only; no cache flush needed (guard/keys additive); cohort comms template attached to the stage PR.

## Step 3 — Observability (extend the existing Grafana engineering dashboard; Prometheus on :9090)

New panels + alerts:
- **Conversation health:** `resolver_fired_rate`, mode split, `resolver_fallback_rate` (**alert >5%/1h**), resolution accuracy (weekly), added-latency histogram.
- **Clarification loop:** rendered vs answered (**alert answered <60%** — options aren't landing), `option_pick_total` distribution, `refusal_total{shell}` (**alert on `general` spike** — taxonomy gap), `option_dropped{bad_field}` (**alert >0 sustained** — model drifting off schema).
- **Quality:** per-tier accuracy trend, behavioral-contract compliance, confidence calibration P(👍|verdict), escalation rate (**alert outside 3–12%**), cache hit rate (expect +5–10 pp post-resolver), linking recall trend.
- **SLO:** p95 per path vs the 08 table; error-budget burn. Cloud Trace: resolver span with `resolved_from_turns` attr (structural only).

## Step 4 — Runbooks (runbook repo, linked from on-call index)

- *Follow-ups misbehaving:* check fired-rate → fallback-rate → grade 20 shadow/prod samples → `RESOLVER_MODE=off` if resolution <70%; file triage tags.
- *Clarification spam* (rate >25%): check lexicon false-positives + linking recall; tune `_VAGUE` list / few-shots.
- *Cache anomalies:* guard-skip counters, threshold history, flush procedure (`TRUNCATE translation_cache` — safe, 24h TTL).
- *Resolver degraded* (new break-glass entry): flags off ⇒ V3; care-manager fallback surface unchanged (Looker dashboard per PRD §7.3).
- Weekly ops review: 07 triage output → actions; SME queue SLA 5 business days; on-call severities: P1 = Tier-1 circuit breaker (auto refusal-mode) — page.

## Step 5 — Doc close-out

- [ ] Update `careconnect-doc.html` to V3.5 at Stage C: new node in the inventory + edge map, contract v2, flags, session endpoints, updated env table.
- [ ] Close OPEN_ISSUES.md items with links to shipping PRs (1→02/03 · 2→02/06 · 3→01 · 4→05 · 5→04 · 6→05/02 · 7→06 · 8→03).
- [ ] Final report to Mubeen: gates table with measured values vs thresholds.

## Definition of done (whole plan)

Stage D live **2 consecutive weeks** with: behavioral-contract 100% · Tier-1 ≥95% · single-turn ≥85% · multi-turn resolution ≥85% · SLOs held · zero PHI events. That is "works as expected," measured — the 00-README bar.
