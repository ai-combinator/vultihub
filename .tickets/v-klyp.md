---
id: v-klyp
status: closed
deps: []
links: [va-qbzv, v-duhk, v-fjph, v-cgfd, v-xvcz]
created: 2026-05-06T00:08:09Z
type: bug
priority: 2
assignee: Jibles
---
# agent-backend tool_results: scope uniqueness by message + enforce status enum

## Problem

PR vultisig/agent-backend#235 introduces `POST /agent/messages/:msgId/tool-results/:toolCallId/execution` and the `tool_call_results` table. Codex deep review (gomes, 2026-05-01) flagged two HIGH (preferably-blocking) findings that are the actual un-draft gates on the PR:

1. **Cross-tenant overwrite.** Uniqueness/PK is global on `tool_call_id` only. An attacker submitting a victim's known `toolCallId` against their own valid `msgId` can mutate the victim's row while it's still in `submitted` state.
2. **Status enum unenforced.** Handler accepts any non-empty `status` string, DB has no CHECK, upgrade logic only advances rows currently in `submitted` state. Tests already use off-spec values (`broadcasted`, `error`, `x`); persisting any of those becomes a terminal state and freezes that row against future writes.

"Solved" means: schema scopes uniqueness by `(message_id, tool_call_id)`, the API rejects status values outside the allowlist, the database enforces the same enum at the column level, and existing test fixtures are aligned with the wire contract.

## Background

### Findings

- New table at `internal/storage/postgres/migrations/20260430000001_create_tool_call_results.sql` defines `tool_call_id` as the unique key (codex citation: line 9 + 12). The upsert in `internal/storage/postgres/sqlc/messages.sql:74` and `messages.sql:81` keys conflict on `tool_call_id` only and only progresses rows currently in `submitted` state.
- Owner check in `internal/api/tool_results.go:81` validates `msgId` belongs to the bearer token's account, not that the `(msgId, toolCallId)` pair owns the targeted row.
- Codex deep-review comment posted on PR #235 (issue comment, 2026-05-01).
- CodeRabbit reinforces both themes across multiple inline threads:
  - **Status enum drift** in tests, schema, migration: open threads on `internal/api/tool_results.go:106`, `internal/storage/postgres/migrations/20260430000001_create_tool_call_results.sql:null`, `internal/storage/postgres/schema/schema.sql:null`, `internal/api/tool_results_test.go:90`, `internal/types/conversation_test.go:170`, `internal/service/agent/prompt_test.go:null`.
  - **Cross-tenant scope** on `internal/storage/postgres/schema/schema.sql:405` (open, marked critical, heavy lift).
  - **Same-toolCallId-different-message regression test missing** on `internal/storage/postgres/tool_call_results_upgrade_test.go:261`.
- Off-spec status values currently in test fixtures: `internal/api/tool_results_test.go:86` uses `broadcasted`; `internal/types/conversation_test.go:153` uses `error`; `internal/service/agent/prompt_test.go` uses `error`.
- Wire contract per the documented status set: terminal `success | failed | cancelled` plus intermediate `submitted` for the upsert state machine.

### External Context

- Companion app PR vultisig/vultiagent-app#339 POSTs to this endpoint with `{status, txHash, explorerUrl, error}`. Currently uses `success` / `failed` / `cancelled` on the happy path; mid-flight is `submitted`.
- Cross-tenant fix has an implicit dependency: the upsert's update clause needs `WHERE tool_call_results.message_id = EXCLUDED.message_id` once uniqueness is scoped — defense-in-depth so a future schema change losing the composite uniqueness still can't mutate cross-message.

## Current Thinking

### Schema scope

Replace global `tool_call_id` uniqueness with composite `(message_id, tool_call_id)`. Either as the primary key (cleanest semantically) or as a unique constraint alongside an existing PK (lower-friction migration if there's a reason to keep the surrogate PK). Land in a NEW migration; do not mutate the existing migration.

The upsert's `ON CONFLICT` clause becomes `ON CONFLICT (message_id, tool_call_id)`. The SET clause adds `WHERE tool_call_results.message_id = EXCLUDED.message_id` as defense-in-depth.

### Status enum

Three layers, all must agree:

1. **API layer.** `internal/api/tool_results.go` validates `status` against `{success, failed, cancelled, submitted}` before persistence. Off-spec values get `400 Bad Request`. Validator reuses a single source of truth (Go constant set in `internal/types/parts.go` or near the persistence write).
2. **DB layer.** Column-level `CHECK (status IN ('success','failed','cancelled','submitted'))` in a new migration.
3. **Migration backfill.** Pre-existing rows with off-spec status need backfill to a canonical value before the CHECK lands. Audit query first; if the result is empty in all environments, the migration just adds the CHECK with no UPDATE step.

The CHECK and the API allowlist must be derived from the same set. Planner picks the binding mechanism (constants reflected against a CHECK assertion test, paired by hand with a regression test, etc.).

### Fixture cleanup

Tests using off-spec status values must be updated to canonical values before the CHECK lands or they will fail:
- `internal/api/tool_results_test.go:86` — `broadcasted` → `submitted`
- `internal/types/conversation_test.go:153` — `error` → `failed`
- `internal/service/agent/prompt_test.go` — `error` → `failed`

### Same-toolCallId-different-message regression test

Add a fixture in `internal/storage/postgres/tool_call_results_upgrade_test.go` near the existing upgrade tests: seed two messages, reuse the same `tool_call_id`, assert the second write cannot mutate the first row. This is the regression test for the cross-tenant fix.

### 403 body shape (small adjacent fix in the same touch)

`internal/api/tool_results.go:143` returns `{"error":"forbidden"}` on owner mismatch. The contract is "shape like missing-message response so a foreign `msgId` is indistinguishable from a non-existent one." Status 403 stays; body shape changes. Pin in `tool_results_test.go:150`.

## Constraints

- **Idempotent retry semantics on `(message_id, tool_call_id)` survive.** A retried POST with the same composite and the same terminal status must still be a no-op; the `submitted → terminal` upgrade-only behavior must survive the schema change.
- **Migration is additive on production-shaped data.** No DROP without backfill. No constraint that fails to apply if any pre-existing row violates it.
- **API + DB + tests use the same status set.** Drift between layers is the bug; planner MUST land all three together.
- **Wire contract values are not changed.** Wire is `{success, failed, cancelled}` (terminal) + `submitted` (intermediate). Don't introduce `error` or `broadcasted` as accepted values just because tests currently use them — fix the tests.

## Assumptions

- Production environments do not currently have rows with off-spec `status` values (schema is recent, app is pre-launch). If any environment does, the backfill step is required and non-trivial.
- The companion app PR (#339) already only writes canonical status strings on the happy path. If it doesn't, an end-to-end check is needed before the DB CHECK lands or the app starts getting 400s.
- Bearer-token auth is the appropriate auth model for this endpoint (per the threat-model decision deferred elsewhere). This ticket does NOT add per-request signing — only the cross-tenant scope + enum fixes.

## Non-goals

- **Replay nonce / per-request signing** (codex MED #235-4) — explicitly deferred to post-launch hardening; residual replay surface is cosmetic-only after this ticket lands.
- **Prompt-injection escaping for `txHash` / `explorerUrl` / `error`** (codex LOW #235-6) — covered by the batched cleanup ticket.
- **Quest fan-out** (codex MED #235-3) — investigation pending; tracked elsewhere.
- **`/messages/since` tool-result hydration** (codex MED #235-5) — covered by the batched cleanup ticket.
- **Anything app-side** — covered by `v-duhk`, `v-fjph`, and the batched cleanup ticket.
- **Schema redesign beyond the two HIGHs.** Don't take this ticket as license to redesign the table; only uniqueness scope + enum need to change.

## Open Questions

1. **PK or UNIQUE for the composite `(message_id, tool_call_id)`?** PK cleanest semantically; UNIQUE lower-friction if there's a reason to keep an existing surrogate PK. Planner picks based on the existing migration shape.
2. **Where does the canonical status set live in code?** Options: a Go constant slice in `internal/types/parts.go`, a sqlc-generated enum from the migration, or a hand-rolled type with a `Validate()` method. Not load-bearing; planner picks. Constraint: one source, three call-sites.
3. **Does any environment currently have off-spec rows?** Run `SELECT status, COUNT(*) FROM tool_call_results GROUP BY status` against staging/prod before the CHECK migration lands. If non-empty, plan a backfill UPDATE first.
4. **`submitted` state machine survives the CHECK?** After the CHECK lands, only canonical values exist — but the upsert's `submitted → terminal` rule means a `submitted` row with the CHECK in place is still upgradable. Confirm with a unit test.

## Repos / branches

- **agent-backend** — work on top of `feat/va-qbzv-tool-call-results` (PR #235), in the main repo dir `agent-backend/`.
- **vultiagent-app, mcp-ts, vultisig-sdk** — no changes expected.

## What this ticket closes (review threads)

- Codex HIGH-1 (cross-tenant overwrite) on PR #235.
- Codex HIGH-2 (status enum unenforced) on PR #235.
- CodeRabbit threads:
  - `internal/storage/postgres/schema/schema.sql:405` (cross-tenant)
  - `internal/api/tool_results.go:106` (status enum at API)
  - `internal/storage/postgres/migrations/20260430000001_create_tool_call_results.sql:null` (status enum at DB)
  - `internal/storage/postgres/schema/schema.sql:null` (status enum at schema)
  - `internal/api/tool_results_test.go:90` (fixture cleanup)
  - `internal/types/conversation_test.go:170` (fixture cleanup)
  - `internal/service/agent/prompt_test.go:null` (fixture cleanup)
  - `internal/storage/postgres/tool_call_results_upgrade_test.go:261` (same-toolCallId regression)
  - `internal/api/tool_results_test.go:150` + `internal/api/tool_results.go:143` (403 body shape)
