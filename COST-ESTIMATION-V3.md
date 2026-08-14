# Cost Estimation — CareConnect Query Agent V3

| | |
|---|---|
| Prepared | 2026-06-23 |
| Baseline volume | 500 conversations/day (production target) |
| POC Phase 1 | 50 users · ~100 conversations/day |
| POC Phase 2 | 200 users · ~400 conversations/day |
| GCP Region | us-east4 (existing careconnect-gke cluster). Google AI Generative API endpoint is us-central1; cross-region round-trip is 30–60ms per call and is reflected in the PRD §7.1 latency budget. No region migration in POC scope. |

---

## Key Assumptions

Two deployment scenarios are modelled throughout:

**Scenario A — Incremental on existing CVS infrastructure (realistic)**
The V3 POC deploys into the existing `careconnect-gke` cluster in `hcb-dev-careconnect-etl`. Cloud SQL instance `cargpgsd1` already has pgvector enabled (used by ADR agent). MongoDB Atlas is already connected. Secret Manager is already provisioned. Cost = new AI/LLM spend + additional GKE capacity only.

**Scenario B — Greenfield deployment (worst case / new project)**
All infrastructure is provisioned from scratch. Use this figure for a separate GCP project or new environment.

Additional assumptions:
- 50% semantic cache hit rate (design target; early POC may be lower until cache warms)
- Gemini 1.5 Pro escalation rate: 6% of cache misses (from V2 telemetry estimate)
- 30% of queries route to BigQuery; 70% to MongoDB
- Average BigQuery query scans 500MB
- All pricing: Google Cloud us-east4, June 2026 public list rates, no committed-use discounts applied
- GKE: e2-standard-4 nodes (4 vCPU / 16GB RAM) — same family as existing cluster

---

## Part 1 — AI / LLM Costs (the main new spend)

These costs are entirely new — V2 used Claude Sonnet via LMS Gateway; V3 switches to Gemini Flash via the Google AI Generative API (`google.generativeai`). The HIPAA BAA for that surface is a follow-on workstream owned by Legal/Compliance and is explicitly out of scope for this POC; pre-Phase-1 development uses synthetic care manager data only. A future move to Vertex AI is contingent on the BAA outcome and is scoped separately.

### Per-Request Cost Breakdown

| Step | Model | Tokens | Rate | Cost per Request |
|---|---|---|---|---|
| Injection detector | Gemini Flash | 500 in / 50 out | $0.075/1M · $0.30/1M | $0.0000525 |
| Question embedding | text-embedding-004 | 200 tokens | $0.0001/1M | $0.000000020 |
| Source router | Gemini Flash | 2,000 in / 100 out | $0.075/1M · $0.30/1M | $0.000180 |
| Translator — cached context | Gemini Flash cache read | 15,000 cached tokens | $0.01875/1M | $0.000281 |
| Translator — dynamic input | Gemini Flash standard | 2,000 uncached tokens | $0.075/1M | $0.000150 |
| Translator — output | Gemini Flash | 600 out tokens | $0.30/1M | $0.000180 |
| **Total — cache miss** | | | | **$0.000844** |
| **Total — cache hit** | Injection detect + embedding only | | | **$0.000053** |

Note: injection detection runs on every request including cache hits — it sits on the hot path before the cache lookup.

### Pro Escalation (accuracy fallback)

When Flash returns low confidence (routing_confidence < 0.7 or UNCERTAIN verdict), the translator re-runs with Gemini 1.5 Pro. This runs cold-path — no cached context at Pro tier.

| | |
|---|---|
| Cost per escalation | $0.024 (17K tokens × Pro input rate, cold path) |
| Expected escalation rate | 6% of cache misses |
| Cost impact | ~30% of total LLM spend at production volume |

### Monthly AI/LLM Cost by Volume

| Volume | Cache Misses/Month | Cache Hits/Month | Flash LLM | Pro Escalations | BigQuery Scanning | Context Cache Storage | **AI Total/Month** |
|---|---|---|---|---|---|---|---|
| 100 conv/day (Phase 1) | 1,500 | 1,500 | $1.50 | $2.16 | $2.81 | $2 | **~$8** |
| 400 conv/day (Phase 2) | 6,000 | 6,000 | $5.96 | $8.64 | $11.25 | $2 | **~$28** |
| 500 conv/day (production baseline) | 7,500 | 7,500 | $7.46 | $10.80 | $14.06 | $2 | **~$34** |
| 1,000 conv/day (scale) | 15,000 | 15,000 | $14.91 | $21.60 | $28.13 | $2 | **~$67** |

BigQuery scanning formula: queries/day × 30 × $6.25/TB × 0.5TB average scan = $0.09375/query/day

**Context cache storage (revised).** The prior $0.40/month figure used the cached-input *read* rate ($0.01875/1M tokens), not the storage rate. Google AI charges $4.50/1M tokens/hour for `CachedContent` storage. Two caches × 15K tokens held continuously would cost ~$97/month. We do NOT hold them continuously: the CronJob refreshes every 45 min and deletes the previous-active cache one TTL after rotation, so effective storage is ~1.5× single-cache size for ~30 min of each 45-min window. Reconciled estimate **$1–$4/month**. The CronJob's delete-after-rotation behaviour is a hard requirement — without it orphan caches accumulate and the line item is materially higher.

**Stress scenarios — cost sensitivity to assumptions:**

| Stress | What changes | $/month at 500 conv/day |
|---|---|---|
| Cache hit rate drops to 30% (V2 baseline, cold start) | More misses, fewer hits | ~$42 (~+25%) |
| Pro escalation rate sustained at 12% (top of acceptance band) | Pro line item ~doubles | ~$45 (~+32%) |
| Hard-route all MQL aggregations to Pro (worst-case Sprint 0 outcome) | ~30% of misses go Pro cold-path | ~$70 (~+105%) |
| Context cache CronJob fails to delete superseded caches | Storage accumulates per pod restart | +$5–$20/month per missed cleanup cycle |

---

## Part 2 — Infrastructure Costs

### 2.1 GKE Compute

V3 adds three new workloads to the existing cluster:
- LangGraph Agent Deployment (2 pods min, HPA up to 6)
- MCP Server Deployment (2 pods min, HPA up to 6)
- Cache warm-up CronJob (runs ~2 min per hour, negligible)

Pod resource profile: 0.5 vCPU + 512MB RAM each (agent), 0.5 vCPU + 1GB RAM each (MCP server)

**Incremental GKE cost (Scenario A — existing cluster):**

| Phase | Additional Pods | Node Expansion Needed | Monthly Cost |
|---|---|---|---|
| Phase 1 (50 users) | 4 pods (2+2) | Likely 0 if existing nodes have headroom | $0 – $98 |
| Phase 2 (200 users, HPA scaling) | 6–8 pods peak | 1 × e2-standard-4 | ~$98 |
| Production (500 conv/day) | 8–12 pods under load | 2 × e2-standard-4 | ~$196 |
| Scale (1,000+ conv/day) | Up to 16 pods | 3 × e2-standard-4 | ~$293 |

e2-standard-4 (4 vCPU / 16GB RAM) in us-east4: **$0.134/hour = $97.82/month**

**Greenfield GKE cost (Scenario B — new cluster):**

| Component | Monthly Cost |
|---|---|
| Cluster management fee (first cluster free per project) | $0 |
| 2 × e2-standard-4 nodes (Phase 1 minimum) | $196 |
| 3 × e2-standard-4 nodes (production) | $293 |

If using GKE Autopilot (pay-per-pod, better for low utilisation POC):

| Phase | Avg Active Compute | Monthly Autopilot Cost |
|---|---|---|
| Phase 1 | 1.5 vCPU + 2GB | ~$62 |
| Phase 2 | 2.5 vCPU + 4GB | ~$95 |
| Production | 4 vCPU + 6GB | ~$138 |

Autopilot formula: (vCPU × $0.0445/hr) + (GB × $0.00492/hr) × 730 hours/month. Cheaper than Standard at low utilisation; consider Autopilot for POC.

### 2.2 Cloud SQL (pgvector + LangGraph Checkpointer)

**Incremental cost (Scenario A — existing `cargpgsd1` instance):**

pgvector is already enabled on this instance (used by ADR agent). V3 adds:
- `translation_cache` table with VECTOR(768) column + HNSW index
- LangGraph checkpoint table (created automatically by PostgresSaver)
- Additional storage: ~5GB for cache + checkpoints at POC volume

| Item | Monthly Cost |
|---|---|
| Additional SSD storage (5GB) | $0.85 |
| Incremental query load (well within existing capacity) | $0 |
| **Incremental total** | **~$3–5** |

**Greenfield cost (Scenario B — new Cloud SQL instance):**

| Instance Type | Use Case | Monthly Cost |
|---|---|---|
| db-n1-standard-1 (1 vCPU, 3.75GB) + 20GB SSD — POC without HA | $76 |
| db-n1-standard-2 (2 vCPU, 7.5GB) + 40GB SSD + HA — Production | $295 |

Note: HA (high availability) doubles the instance cost but is required for production.

### 2.3 MongoDB Atlas

The existing Atlas cluster is already provisioned and connected. The V3 MCP server connects to the same cluster.

**Incremental cost (Scenario A):**

| Condition | Monthly Cost |
|---|---|
| Existing M10 has capacity for MCP keep-alive pool | $0 |
| Atlas M10 → M20 upgrade needed (more connections) | +$88 delta |

Atlas pricing reference:

| Tier | vCPU | RAM | Storage | Monthly |
|---|---|---|---|---|
| M10 | 2 vCPU | 2GB | 10GB | $58 |
| M20 | 2 vCPU | 4GB | 40GB | $146 |
| M30 | 2 vCPU | 8GB | 80GB | $394 |

For POC (50–200 users): M10 is sufficient.
For production (500 conv/day with keep-alive pool): M20 recommended.

### 2.4 Supporting Services

| Service | Notes | Monthly Cost |
|---|---|---|
| Artifact Registry | 2 container images × ~500MB, same-region pulls free | ~$1 |
| Secret Manager | ~10 secrets, ~15K access ops/month | ~$1 |
| Cloud Monitoring | Custom metrics (within free 150 metric tier for POC), alerting policies | ~$5 |
| Cloud Logging | First 50GB/month free; V3 log volume well under that | $0–$5 |
| Cloud Trace | First 2.5M spans/month free; V3 at ~150K spans/month (10 per conv × 15K convs) | $0 |
| BigQuery Audit Log Storage | 365 days × 500 records × ~1KB = 183MB/year | < $1 |
| Network egress | GKE→Google AI API (intra-Google); MongoDB Atlas outbound ~50MB/month | ~$3 |
| **Supporting total** | | **~$11–16** |

---

## Part 3 — Full Cost Summary

### Scenario A — Incremental on Existing CVS Infrastructure

This is the realistic cost for the POC deploying into `hcb-dev-careconnect-etl`.

| Component | Phase 1 (100/day) | Phase 2 (400/day) | Production (500/day) | At Scale (1,000/day) |
|---|---|---|---|---|
| Gemini Flash + embedding | $1.50 | $5.96 | $7.46 | $14.91 |
| Gemini 1.5 Pro escalation | $2.16 | $8.64 | $10.80 | $21.60 |
| BigQuery analytics scanning | $2.81 | $11.25 | $14.06 | $28.13 |
| Context cache storage | $0.40 | $0.40 | $0.40 | $0.40 |
| GKE incremental (additional nodes) | $0–$98 | $98 | $196 | $293 |
| Cloud SQL incremental | $3 | $3 | $5 | $5 |
| MongoDB Atlas incremental | $0 | $0 | $0–$88 | $88 |
| Supporting services | $11 | $13 | $16 | $20 |
| **Monthly Total** | **$21–$119** | **$140** | **$250–$338** | **$471** |
| **Annual Total** | **$252–$1,428** | **$1,680** | **$3,000–$4,056** | **$5,652** |

Low end of Phase 1 range assumes V3 pods fit in existing node capacity with no new nodes. High end adds one new e2-standard-4 node.

### Scenario B — Greenfield Deployment (new GCP project)

| Component | Phase 1 (100/day) | Phase 2 (400/day) | Production (500/day) |
|---|---|---|---|
| AI/LLM (same as above) | $7 | $26 | $33 |
| GKE Standard (2 nodes min) | $196 | $196 | $293 |
| Cloud SQL (new instance, no HA) | $76 | $76 | $295 |
| MongoDB Atlas M10/M20 | $58 | $58 | $146 |
| Supporting services | $11 | $13 | $16 |
| **Monthly Total** | **~$348** | **~$369** | **~$783** |
| **Annual Total** | **~$4,176** | **~$4,428** | **~$9,396** |

In greenfield, Cloud SQL and GKE are the dominant costs — not the AI/LLM. This is the key insight: the AI/LLM component ($33–65/month) is inexpensive; the surrounding compute is the floor.

---

## Part 4 — V1 / V2 / V3 Comparison (AI/LLM costs only)

This is an apples-to-apples comparison. Infrastructure costs were similar across all versions.

| Version | LLM Provider | Annual LLM Cost | Cost per Cache Miss | Key Cost Driver |
|---|---|---|---|---|
| V1 | Gemini Pro, 4 LLM calls, Redis exact cache | ~$18,000 | $0.1318 | 4 LLM calls, wasted translator call |
| V2 | Claude Sonnet via LMS Gateway, 2 calls | ~$7,968 | $0.0615 | Expensive model, no context caching |
| **V3** | **Gemini Flash + context cache + semantic cache** | **~$432** | **$0.000844** | Flash + caching + 45–50% hit rate |
| **LLM-only saving vs V2** | | **~$7,536/year** | **98.6% cheaper per miss** | |
| **LLM-only saving vs V1** | | **~$17,568/year** | **99.4% cheaper per miss** | |

**Reconciled headline.** This document and the PRD now use **$432/year LLM-only** (= $36/month × 12) as the single V3 headline number for the cost-reduction claim. The prior "$288" figure (in the design doc comparison table) was unexplained — removed. The prior "$396" figure here was an arithmetic artifact of an earlier $33/month line — superseded by the table above.

**Full-system cost** at production volume (500 conv/day) on existing CVS infrastructure is ~$250–338/month (Scenario A in Part 3). V2 at the same volume cost ~$694/month (LLM $664 + similar infra ~$30 incremental). V3 is still meaningfully cheaper end-to-end, but the 94% figure refers to LLM line items only — total system cost reduction is in the 50–60% range. The PRD executive summary has been updated to make this distinction explicit.

---

## Part 5 — POC-Specific Cost Projection

Expected total spend across the full 17-week POC (Scenario A, incremental):

| Phase | Duration | Conv/Day | Monthly Rate | Phase Cost |
|---|---|---|---|---|
| Sprints 1–5 (build, no user traffic) | 10 weeks | 0 | ~$15 (infra only) | ~$38 |
| Phase 1 POC (50 users) | 3 weeks | ~100 | ~$70 | ~$53 |
| Phase 2 POC (200 users) | 3 weeks | ~400 | ~$140 | ~$105 |
| Readout week | 1 week | ~100 | ~$70 | ~$18 |
| **Total POC Spend** | **17 weeks** | | | **~$214** |

The entire POC costs approximately **$200–250** on existing CVS infrastructure. This is lower than one day's V2 LLM spend at production volume ($7,968 / 365 = $21.83/day).

---

## Part 6 — Cost Optimisation Levers

Listed in order of impact:

**1. Replace Flash injection detector with local classifier (Stage 2 planned)**
Flash injection detection costs $0.0000525 per request and runs on every request including cache hits. A local `prompt-guard-86m` style classifier running in-process costs $0 per request and saves ~$0.0000525 × 15,000 requests/month = $0.79/month at production. Small in dollar terms but removes one network call from every hot path, saving ~200ms p95 latency.

**2. GKE Autopilot over Standard for POC**
At low utilisation (100–400 conv/day), Autopilot charges only for running pod compute. Estimated saving: $98–$196/month vs Standard node pricing. Only relevant if deploying a new cluster; if adding to existing Standard cluster, this lever doesn't apply.

**3. Committed Use Discounts (CUD) on GKE nodes**
1-year CUD: ~20% discount on node compute. 3-year CUD: ~37%. At $196/month baseline (production), 1-year CUD saves ~$39/month = ~$471/year. Not applicable during POC; evaluate at production decision.

**4. Increase semantic cache hit rate beyond 50%**
Every 10% increase in cache hit rate saves approximately $0.000791 per additional cached request (miss cost minus hit cost). At production volume, raising hit rate from 50% to 60% saves ~$71/month. Lever: expand few-shot examples to cover more question variants; lower similarity threshold slightly from 0.95 to 0.93 (calibrate against false-positive rate first).

**5. BigQuery slot reservation vs on-demand**
At 150 BigQuery queries/day (production), on-demand scanning at $6.25/TB costs $14.06/month. A BigQuery flat-rate reservation (100 slots) costs $1,700/month — far more expensive at this volume. On-demand pricing is correct for this use case. Only consider reservations if BigQuery usage grows to 5TB+/day of scanning.

**6. Cloud SQL connection pooling (PgBouncer)**
Not a cost lever directly, but reduces Cloud SQL connection overhead. At POC scale the current setup is fine. Relevant if connection count from multiple GKE pods pushes against Cloud SQL max_connections.

---

## Part 7 — Cost Monitoring Setup

Recommended Cloud Monitoring budget alerts:

| Alert | Threshold | Action |
|---|---|---|
| Monthly AI/LLM spend (Vertex AI / Google AI API) | > $50 | Engineering review — check escalation rate |
| Monthly GCP total spend | > $500 | Engineering + finance review |
| Gemini 1.5 Pro escalation rate | > 12% of misses sustained 30 min | Check Flash regression, may need golden set expansion |
| BigQuery bytes scanned | > 1TB/day | Check for unguarded expensive queries passing validator |

Cost breakdown by label requires enabling resource-level billing labels on GKE pods and Cloud SQL queries. Recommend tagging all V3 resources with `app=careconnect-query-agent` and `version=v3` at Sprint 1.

---

## Summary

| Scenario | POC Total (17 weeks) | Monthly at Production | Annual at Production |
|---|---|---|---|
| Incremental on existing CVS infra | **~$214** | **~$250–$338** | **~$3,000–$4,056** |
| Greenfield new project | **~$1,200** | **~$783** | **~$9,396** |
| V2 equivalent (LLM-only comparison) | — | **~$664** | **~$7,968** |

The POC itself costs roughly **$200** on existing infrastructure — less than one week of V2's LLM spend. The largest ongoing cost driver is GKE compute (60–70% of total), not the AI/LLM layer. The AI/LLM spend is $33/month at production volume — 94% cheaper than V2's $664/month.
