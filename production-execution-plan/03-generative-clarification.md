# 03 — Generative Clarification & Reasoned Refusals (agent work order)

Branch: `v35/03-generative-clarification`. Flags: `CLARIFY_GEN_MODE=off|on`, `REFUSAL_REASONS=off|on` (both default off). Parallel-safe with 02; the renderer also serves 02's `clarifying_question`.

Target behavior: *"Which vendor is doing best?"* → "'Best' can mean a few different things in our gap data — (a) highest provider response rate (accepted + dismissed vs total sent) · (b) highest acceptance rate (Assessed-present share) · (c) highest gap volume this period?"

---

## Step 1 — Translator/Pro contract v2

Locate the translator output model(s) (MQL + SQL + pro_escalation share or mirror one). Add optional fields (backward compatible — absent ⇒ null):

```python
assumptions: list[str] = []
clarifying_question: str | None = None
interpretation_options: list[InterpretationOption] | None = None

class InterpretationOption(BaseModel, extra="forbid"):
    label: str        # ≤ 60 chars, business phrasing
    field_path: str   # must exist in SchemaRegistry
    reason: str       # ≤ 120 chars
```

Append to BOTH cached static prompt prefixes (MQL 16-rule block, SQL 12-rule block) — verbatim:

```
AMBIGUITY RULE: If a phrase in the question maps to more than one plausible metric or field in the provided schema (e.g. "best", "top", "performance" with no metric named), DO NOT guess. Set confidence_verdict="UNCERTAIN", leave the query empty, set clarifying_question to one sentence naming the ambiguous phrase, and provide 2-4 interpretation_options. Each option = {"label","field_path","reason"} where field_path is copied EXACTLY from the schema context above. When you DO answer, always fill assumptions[] with the interpretive choices you made (status sets, time period, LOB scope, metric definition).
```

Re-warm CachedContent after prefix change (bump the registry entry).

## Step 2 — Deterministic ambiguity nudge

New `src/careconnect_agent/context/ambiguity.py`:

```python
_VAGUE = ("best", "worst", "top", "biggest", "most improved", "doing well",
          "doing best", "performance", "how are we doing", "strongest", "weakest")
def build_metric_terms(registry) -> frozenset[str]:
    # canonical + synonym terms of numeric/metric-bearing fields (rates, counts, dates, volumes)
    ...
def is_vague_metric(q: str, metric_terms: frozenset[str]) -> bool:
    t = q.lower()
    return any(v in t for v in _VAGUE) and not any(m in t for m in metric_terms)
```

In translator prompt assembly: when `is_vague_metric(standalone_question, …)` append one dynamic line: `NOTE: the metric in this question is unmapped — prefer interpretation_options over guessing.` (Hint only; schema context may still disambiguate.)

## Step 3 — `ask_clarification` becomes a guarded renderer

Rewrite the node:

```python
def render_clarification(state) -> Response:
    origin, question, options = _pick_source(state)   # translator | pro | resolver(02)
    valid = []
    for o in (options or []):
        if not registry.field_exists(o.field_path):           # any collection in routing_source scope
            METRIC_option_dropped.labels(reason="bad_field").inc(); continue
        o = scrub_and_cap(o)                                  # PII regex strip; label≤60, reason≤120; strip md/URLs/backticks
        valid.append(o)
    if options is not None and len(valid) < 2:
        return canned_clarification(state)                    # today's text — never a dead-end, never hallucinated fields
    return ClarificationResponse(kind="clarification",
                                 message=scrub_and_cap_q(question),   # ≤ 300 chars
                                 options=[{"key": k, "label": o.label, "reason": o.reason}
                                          for k, o in zip("abcdefgh", valid)])
```

Flag off ⇒ always `canned_clarification`. Portal contract: options render as tap-chips; the pick returns as the next user turn (`"(b)"` or label text) — 02's few-shots include pick-replies, so resolution is already handled. Metrics: `clarification_rendered_total{origin}`, `clarification_answered_total` (increment when the next turn in the same session follows a rendered clarification), `option_pick_total{key}`.

## Step 4 — Dead-end elimination (edge changes)

Current edge map sends repair-exhausted / accuracy-mismatch-exhausted to `audit → END` with generic errors. Change: those paths route to `ask_clarification` with a translator-authored `clarifying_question` when one exists, else shell #4 (below). The `narrow_request` node: extend the translator repair prompt for the `too_expensive` verdict to also return 1–2 `interpretation_options`-shaped *narrower rewrites* (label = the narrowed question, field_path = the filter field, reason = why cheaper); render through the same guards. **I3 intact:** these are response-rendering changes only; no validator/gate is bypassed.

## Step 5 — Reasoned refusals (fixed shells + LLM reason slot)

New `src/careconnect_agent/responses/shells.py` — shells verbatim from the approved PO taxonomy; `{reason}`/`{date}` are the only mutable slots:

```python
SHELLS = {
 "out_of_scope":   "I can only answer questions related to Risk Gap data. {reason} For information on that topic, please navigate to the relevant section of the portal.",
 "phi_restricted": "I'm not able to share member-identifiable information such as names, dates of birth, or full member IDs. {reason} I can provide aggregate counts or anonymized summaries — would that help?",
 "data_stale":     "I'm unable to answer that right now. Risk Gap data was last refreshed on {date}. Please try again after the next scheduled refresh or contact your data team.",
 "not_understood": "I didn't quite understand that question. {reason} Could you rephrase it, or try one of the suggested questions to get started?",
 "partial":        "I can partially answer that — {reason} For the complete picture, some data may not be available in the current report period.",
 "clinical":       "I'm not able to provide clinical recommendations. HCC descriptions are shown for informational purposes only and should not be used as a basis for treatment decisions.",
 "general":        "I'm not able to answer that question. I'm designed to help with Risk Gap data only. If you need further assistance, please reach out to your care coordinator.",
}
```

Intent-classifier contract: add `"reason": "<≤140 chars referencing the user's own words, or null>"` to its strict-JSON output (same call — zero added latency; bump max_output_tokens 256→320). Wire: `input_security` refusal → `phi_restricted`/`out_of_scope` shell with classifier reason (scrubbed, capped) · translator no-options failure → `not_understood` · clinical category → `clinical` · MCP stale/unavailable → `data_stale` with real refresh date if the MCP health payload exposes one, else omit sentence. Classifier reason missing/failed → shell with `{reason}` removed (today's behavior). Shell text itself is never model-authored. Track `refusal_total{shell}` — a spike in `general` = taxonomy gap.

## Step 6 — Interpretation echo + confidence relabel

- Response model: add `how_i_read_it: str | None` — deterministically joined from `assumptions` ("Counted open + suspect statuses; current report period; MA/ACA + Medicaid combined."). No extra LLM call. `summarize` receives assumptions in its input for tone consistency.
- Rename API field `confidence_verdict` → `translation_confidence` (keep the old key as a duplicate for one release; portal ticket to migrate). Keep the PRD-mandated UNCERTAIN banner. Surface existing deterministic artifacts on the answer payload: `checks: {validation:"passed", bounds:"pass|warn", scope:"confirmed"}`.

## Step 7 — Tests

`tests/responses/test_clarification_renderer.py`: bad field_path dropped + metric; <2 valid ⇒ canned; caps enforced; markdown/URL stripped; PII string in a label scrubbed. `tests/responses/test_shells.py`: snapshot every shell with and without reason; reason scrub. `tests/integration/test_ambiguity.py`: with a stubbed translator returning options, "which vendor is doing best" yields kind=clarification with ≥2 valid options; repair-exhausted path lands on not_understood shell, never a raw error. Contract-v2 parse tests (old responses without new fields still validate).

## Done when

- [ ] Suite green; CachedContent re-warmed; flags off ⇒ byte-identical V3 behavior (snapshot test).
- [ ] Dev with flags on: ambiguity set spot-check (10 questions) → 10/10 clarify with valid options; zero hallucinated field_paths rendered (`option_dropped` log audit).
- [ ] Grep proves no user-reachable hardcoded dead-end strings remain outside `shells.py` + `canned_clarification`.
- [ ] Metrics + Grafana panel (rendered/answered, picks, refusal-by-shell).
