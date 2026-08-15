# 08 — Scalability & Performance (agent work order)

Branch: `v35/08-scale`. Runs after 02–05 are in staging. Release-blocking load test; index migration before rollout Stage C.

---

## Step 1 — Vector index fix (3072-d > HNSW 2000-d ceiling ⇒ current seq-scans)

Decide A (preferred) or B after checking `SELECT extversion FROM pg_extension WHERE extname='vector';`:

**A. Matryoshka 1536-d** (gemini-embedding-001 supports `output_dimensionality`): migration `023_vec1536.sql` —

```sql
ALTER TABLE careconnect_ai.translation_cache  ADD COLUMN embedding_1536 vector(1536);
ALTER TABLE careconnect_ai.schema_rag_entries ADD COLUMN embedding_1536 vector(1536);
-- backfill schema_rag_entries by re-embedding (careconnect-seed-schema --dims 1536); translation_cache cold-starts (24h TTL) — no backfill.
CREATE INDEX CONCURRENTLY schema_rag_hnsw ON careconnect_ai.schema_rag_entries USING hnsw (embedding_1536 vector_cosine_ops) WITH (m=16, ef_construction=64);
CREATE INDEX CONCURRENTLY trans_cache_hnsw ON careconnect_ai.translation_cache USING hnsw (embedding_1536 vector_cosine_ops) WITH (m=16, ef_construction=64);
-- after cutover + verification: drop old 3072 columns.
```

Embed calls: pass `output_dimensionality=1536`, then **L2-normalize** client-side (required for truncated MRL vectors before cosine). **B. halfvec(3072)+HNSW** (pgvector ≥0.7): cast columns, index halfvec_cosine_ops, no re-embed.

**Threshold re-calibration:** build `eval/data/paraphrase_pairs.jsonl` (40 positive paraphrase pairs from golden + 40 near-miss negatives incl. the historical 0.95 false-hit examples). Sweep threshold 0.93–0.99 at the new dims; pick max hit-rate with **zero false hits**; update `SEMANTIC_CACHE_SIMILARITY_THRESHOLD` + note in PR. Gate: no false hit on the negative set.

## Step 2 — Hot-path & capacity

- CachedContent: confirm `resolver` key warmed (02); verify prefetch still overlaps input_security with the conditional resolver (trace check).
- **I4 audit:** grep every Gemini call site for `thinking_budget` — Flash sites must be 0 (incl. resolver, auto-title).
- HPA: range 2→6; add QPS-based scaling (custom metric via the existing Prometheus adapter or `container/accelerator` fallback: CPU target 60%). Cloud SQL: verify `max_connections` ≥ (6 pods × (pool 10 + checkpointer pool)) × 1.5 headroom; keep `pool_max_lifetime=1800`.
- Rate limits: `RATE_LIMIT_PER_USER_PER_MIN` 30 → 45 (clarify+pick+answer = 3 requests); global stays 500. Env only.
- MCP: `maxTimeMS=30000` (05) verified on every Mongo call; `BQ_MAX_BYTES_BILLED` on every BQ call.

## Step 3 — Load test `eval/load/k6_v35.js` (k6)

200 VUs, 30 min steady + 10 min spike to 300. Mix: 45% cache-hit repeats · 35% miss standalone · 12% miss follow-up (scripted 3-turn sessions) · 5% clarification rounds · 3% Pro-escalated (seed with known-UNCERTAIN questions). Thresholds in-script:

```js
thresholds: {
  'http_req_duration{path:hit}':          ['p(95)<1000'],
  'http_req_duration{path:miss}':         ['p(95)<3000'],
  'http_req_duration{path:miss_resolver}':['p(95)<3600'],
  'http_req_duration{path:pro}':          ['p(95)<5500'],
  'http_req_failed':                      ['rate<0.005'],
}
```

During spike: trigger a rolling restart; assert zero audit loss (dead-letter drain ratio) — the PRD SIGTERM-drain test, automated. Nightly-able job in the eval namespace.

## Step 4 — Cost refresh

Measure in staging over the load run; update `COST-ESTIMATION-V3.md` with a V3.5 delta table: +resolver Flash (~30–40% of turns × ~3K in/0.15K out) · +BQ union scans (bounded by bytes-billed) · − semantic-hit-rate gain from canonicalized standalone questions (report measured pp change). Gate: within 10% of projection (PRD §9.2 pattern).

## Done when

- [ ] HNSW live on both tables; `EXPLAIN` shows index scans; threshold re-calibrated with zero false hits; cache hit-rate ≥ pre-migration baseline after 48 h.
- [ ] Load test green 3 consecutive nightly runs incl. restart-during-spike audit check.
- [ ] I4 grep audit clean; HPA scales in test; pools within 60% instance utilization at peak.
- [ ] Cost doc updated with measured numbers.
