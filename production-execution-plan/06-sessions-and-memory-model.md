# 06 — Sessions & Memory / PHI Remediation (agent work order)

Branch: `v35/06-sessions`. Flag: `SESSION_BOUNDARIES=off|on` (default off). **Prerequisite for `RESOLVER_MODE=on` in prod** — a 10-turn window without boundaries resolves follow-ups against stale topics. Implements the signed H5 decision (01/DECISIONS.md must be signed before merge).

---

## Step 1 — Migration `migrations/022_sessions.sql`

```sql
CREATE TABLE IF NOT EXISTS careconnect_ai.sessions (
  session_id     TEXT PRIMARY KEY,
  user_id        TEXT NOT NULL,
  title          TEXT,
  status         TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','archived')),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_active_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS sessions_user_recent ON careconnect_ai.sessions (user_id, last_active_at DESC);

ALTER TABLE careconnect_ai.conversation_turns
  ADD COLUMN IF NOT EXISTS answer_digest TEXT;   -- ≤200 chars, PHI-scrubbed
-- formatted_response: STOP WRITING (code change below). Remediation of legacy values in Step 4.
```

Settings: `SESSION_IDLE_HOURS=12`, `CONVERSATION_TURNS_TTL_DAYS=90`.

## Step 2 — Endpoints (FastAPI, JWT-scoped — user_id ALWAYS from token claims, never request body)

`POST /sessions` → create, return `{session_id}` · `GET /sessions?limit=20` → `[{session_id,title,last_active_at,status}]` for the caller only · `POST /sessions/{id}/archive` (404 if not owner). `/query`: unknown `session_id` auto-creates (back-compat); `save_state`/`audit` touch `last_active_at`. Response adds `session_idle: bool` (`now-last_active_at > SESSION_IDLE_HOURS` at request start) so the portal can prompt "continue or start new?". Portal contract note in the PR: sidebar list, New Chat = POST /sessions, resume = reuse id.

## Step 3 — Auto-title

After the first successful turn of a session: Flash call, T=0.2, max 24 tokens, input = **normalized (PII-stripped) question only**, prompt: `Give a 3-6 word title for a chat that starts with this analytics question. Title only.` Scrub + cap 60 chars; failure ⇒ first 6 words of the normalized question. Fire-and-forget task (must not touch request latency).

## Step 4 — Stop persisting answer text (H5)

- In the turn-upsert (save_state/audit): write `formatted_response = NULL`; write `result_shape` (from 02 Step 3) and `answer_digest = output_gate_scrub(summary or first-row preview)[:200]` — reuse the EXISTING output-security scrubber function, do not write a new one.
- Meta-answer rule: resolver mode=followup whose rewrite is a pure restatement of a prior ANSWER ("what was that count again?") may be answered from `answer_digest` **only when the referenced turn's `result_shape.truncated == false` and `row_count == 1`**; every other follow-up re-queries. Implement as a short-circuit in `context_resolver` → response path; anything else falls through to the normal pipeline.
- Remediation script `scripts/remediate_formatted_response.py`: `UPDATE careconnect_ai.conversation_turns SET formatted_response = NULL WHERE formatted_response IS NOT NULL` in batches of 1000 with count logging. Run in dev → staging → prod after the code change deploys.

## Step 5 — Retention + isolation

- Worker CLI `careconnect-cleanup-turns` + CronJob (nightly 03:30, Forbid): delete turns older than `CONVERSATION_TURNS_TTL_DAYS`; archive sessions with zero remaining turns. Metric `turns_deleted_total`.
- `load_context` history query: assert it filters `WHERE session_id = %s` with the caller's session AND joins/validates session ownership (`sessions.user_id = jwt.user_id`) — cross-user session takeover must 403 before the graph runs.

## Step 6 — Tests

`tests/api/test_sessions.py`: CRUD; ownership (user B cannot list/archive/query user A's session → 403/404); auto-create; idle flag. `tests/integration/test_session_isolation.py`: two users, two sessions each, interleaved turns — resolver history bundles never contain another session's turns (assert on bundle content with stubbed model). `tests/workers/test_cleanup_turns.py`: TTL boundary + idempotency. Digest test: seeded PHI string in summary → scrubbed in `answer_digest`; formatted_response written as NULL.

## Done when

- [ ] DECISIONS.md H5 entry signed (name+date) — hard gate.
- [ ] Migrations + endpoints live in dev; ownership tests green; isolation test green.
- [ ] No new `formatted_response` values (query proves 0 rows newer than deploy time); remediation run logged per env.
- [ ] TTL CronJobs applied and pruning; portal contract shared with the frontend team.
