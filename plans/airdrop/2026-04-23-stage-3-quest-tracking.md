# Vultiverse Airdrop — Stage 3 (Quest Tracking) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A single new internal endpoint (`POST /internal/quests/event`) plus a hook in the existing `SignTxProposal` handler that fires quest events whenever the mobile app reports a tx as signed-and-broadcast. Five hard-coded validators (three real, two stubs). Idempotent on `tool_call_id`. Activation-gated by `QUEST_ACTIVATION_TIMESTAMP`.

**Architecture:** New `/internal/quests/event` route on the existing `Server`, protected by `InternalTokenMiddleware` from shared infra. Validators are pure functions in `internal/service/airdrop/quests/validators/` — one file per quest type, each reading the normalised `quest_metadata` map (already populated by the agent-backend at proposal-creation time) plus relevant config, returning `(qualifies bool, reason string)`. A small lookup table in `internal/service/airdrop/quests/dispatch.go` maps `tool_name` → `quest_type` (with swap-vs-bridge resolved by comparing `source_chain_id` and `dest_chain_id` from `quest_metadata`). Repo layer wraps two sqlc files (`airdrop_quest_events.sql`, `airdrop_user_quests.sql`). The hook is a small additive change at `internal/api/tx_proposals.go` (after `MarkSigned` succeeds) that reads `tool_name` + `quest_metadata` off the signed proposal and fires an in-process HTTP POST to `http://127.0.0.1:<SERVER_PORT>/internal/quests/event` with the `X-Internal-Token` header — errors logged, never propagated to the user.

**Cross-team prerequisite (HARD BLOCKER):** This plan depends on two new columns (`tool_name`, `quest_metadata`) on `agent_tx_proposals`, populated by the agent-backend's scheduler.go at proposal-creation time, with values supplied by the MCP server's `build_*` tools. See **Task 0** below. None of the airdrop-side code paths in this plan can be E2E-tested until that prerequisite ships.

**Tech Stack:**
- Go 1.25, Echo v4
- pgx/v5 + sqlc (existing pattern — see `internal/storage/postgres/sqlc/conversations.sql`)
- `InternalTokenMiddleware` from shared-infra (`internal/api/internal_token_middleware.go`)
- `cfg.Airdrop.*` substruct from shared-infra (`internal/config/config.go:41`)
- `net/http` stdlib for the in-process POST (no new deps)

**Spec source of truth:** `docs/airdrop-specs/stage-3-quest-tracking.md`

**Working directory for all tasks:** `/Users/apotheosis/git/vultisig/agent-backend/`

**Branch:** `feat/airdrop-stage-3-quest-tracking`. Branch off `main` after the shared-infra PR has merged. Stage 1's `eligibility.go` does not block this stage — Stage 3 only writes to `agent_quest_events` + `agent_user_quests`; Stage 1 is the consumer.

**Hard deadline:** Live in staging by Day 13 (the quest-earning window opens). This plan is roughly 1.5 days of focused work.

---

## Pre-flight (do once before Task 1)

Confirm the shared-infra PR has merged AND the cross-team prerequisite (Task 0) has landed:

```bash
cd /Users/apotheosis/git/vultisig/agent-backend
git checkout main && git pull
psql "$DATABASE_DSN" -c "\d agent_quest_events"
psql "$DATABASE_DSN" -c "\d agent_user_quests"
psql "$DATABASE_DSN" -c "\d agent_tx_proposals" | grep -E "tool_name|quest_metadata"
grep -n "InternalTokenMiddleware" internal/api/internal_token_middleware.go
grep -n "QuestActivationTimestamp" internal/config/config.go
```

All five checks should succeed. The third one (column existence on `agent_tx_proposals`) is the cross-team prerequisite gate — if it fails, **stop**: the agent-backend migration from Task 0 hasn't merged yet and Stage 3 cannot proceed.

```bash
git checkout -b feat/airdrop-stage-3-quest-tracking
```

---

## Task 0: Cross-team prerequisite — `tool_name` + `quest_metadata` on `agent_tx_proposals`

**Why:** The airdrop's quest dispatch needs a reliable, structured signal of "what tool built this tx and how much was it worth in USD." That data exists at the MCP layer (the build_* tool that built the tx). Reverse-engineering it from raw `action_params`/`quote_data` JSONB at quest-event time is brittle and couples the airdrop module to every tool's bespoke shape. The fix is to have the MCP tools emit a normalised `quest_metadata` block in their result, have the agent-backend persist it on the proposal, and have the airdrop module read it at quest-event time.

**This task is OWNED BY OTHER TEAMS.** This plan documents the contract, the file paths, and the acceptance criteria; the work itself is done by the MCP team and the agent-backend team. Stage 3 cannot ship until both PRs land.

### Schema contract

Two new columns on `agent_tx_proposals`:

```sql
ALTER TABLE agent_tx_proposals ADD COLUMN tool_name TEXT;        -- e.g. "build_swap_tx"
ALTER TABLE agent_tx_proposals ADD COLUMN quest_metadata JSONB;  -- normalised, see below
```

`quest_metadata` is a flat JSON object. All fields are optional — the airdrop validators only read what they need. Field semantics:

```json
{
  "amount_usd":         25.50,
  "token_in_symbol":    "USDC",
  "token_out_symbol":   "ETH",
  "source_chain_id":    1,
  "dest_chain_id":      1,
  "target_contract":    "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
  "router_address":     "0x6131b5fae19ea4f9d964eac0408e4408b66337b5"
}
```

- `amount_usd` is a JSON number (decimal), not an integer in cents. Computed by the build tool using its existing quote pipeline. Omit if not available — validators treat absence as "below threshold."
- Chain IDs are **integers** (1, 42161, 8453, …), not strings. Deliberate divergence from `tx_proposals.chain` (strings). Universal EVM convention; lets validators do `source_chain_id != dest_chain_id` cleanly.
- Addresses lowercase hex with `0x` prefix. The DeFi-action validator does a lowercase compare against the configured allowlist.

### Sub-task A — MCP team

In `vultisig/mcp`, update each `build_*` tool to emit `quest_metadata` in its result. Sizing is **bigger than the original spec implied** because `build_swap_tx` needs a CoinGecko price lookup to compute `amount_usd` from the user's input amount.

Concretely for the three tools the airdrop directly uses:

| Tool | File | Required fields | Effort |
|---|---|---|---|
| `build_evm_tx` | `internal/tools/build_evm_tx.go:142-153` (the result map) | `target_contract` (= the existing `to` field), `source_chain_id` (= the existing `chain_id` parsed to integer) | ~5 lines |
| `build_swap_tx` | `internal/tools/build_swap_tx.go:39-60` (the `swapResult` struct) | `amount_usd`, `token_in_symbol`, `token_out_symbol`, `source_chain_id`, `dest_chain_id`, `router_address` | ~25 lines |
| Any future bridge tool | n/a yet | `amount_usd`, `source_chain_id`, `dest_chain_id` | n/a |

For `build_swap_tx`, three things to add:
1. **`AmountUsd float64`** — call the existing `coingecko.Client.GetSimplePrice(ctx, ...)` (already wired into MCP) to fetch the USD price of the input token; multiply by `params.Amount / 10^FromDecimals`.
2. **`RouterAddress string`** — `bundle.Quote.Router` exists internally (post `svc.GetSwapTxBundle()`) but isn't currently exposed in the result struct. Just add the field and pass through.
3. **`SourceChainID, DestChainID int64`** — convert the existing `FromChain`/`ToChain` strings to integer chain IDs. The MCP repo has chain-ID lookup utilities (`evmclient.ChainIDByName` or similar — check `internal/evm/`). Emit as JSON numbers (not strings).

For `build_evm_tx`, just two map keys to add to the existing result map. `chain_id` already exists as a string — emit a parallel `source_chain_id` as integer for the airdrop, leave the existing string for backward compat.

Other build tools (Solana, Sui, Ton, Trx, etc.) can omit `quest_metadata` for now — none of them map to airdrop quests.

**Chain ID encoding gotcha:** the airdrop validators expect chain IDs as JSON numbers (e.g. `1`, `42161`), not strings (`"1"`, `"42161"`). Make sure the new fields marshal as numbers — easy to get wrong if the engineer reuses MCP's existing chain_id-as-string pattern.

**Acceptance:** PR open in `vultisig/mcp` with `quest_metadata` populated by `build_swap_tx` and `build_evm_tx`, plus a fixture or unit test asserting the shape (including chain IDs being JSON numbers).

### Sub-task B — agent-backend team

In `vultisig/agent-backend`:

1. **Migration.** Add a new migration file `internal/storage/postgres/migrations/<TIMESTAMP>_add_tool_name_quest_metadata_to_tx_proposals.sql`:

```sql
-- +goose Up
-- +goose StatementBegin
ALTER TABLE agent_tx_proposals ADD COLUMN tool_name TEXT;
ALTER TABLE agent_tx_proposals ADD COLUMN quest_metadata JSONB;
-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin
ALTER TABLE agent_tx_proposals DROP COLUMN tool_name;
ALTER TABLE agent_tx_proposals DROP COLUMN quest_metadata;
-- +goose StatementEnd
```

2. **Type update.** Add `ToolName *string` and `QuestMetadata json.RawMessage` to the `TxProposal` struct at `internal/types/tx_proposal.go`.

3. **Repo update.** Update `internal/storage/postgres/tx_proposal.go`'s `Create` method to write the new columns, and `GetByID` / `MarkSigned` to read them. (sqlc regeneration after updating the corresponding `.sql` file in the sqlc directory.)

4. **Scheduler change.** In `internal/service/scheduler/scheduler.go`, around line 320 (call site of `txProposalRepo.Create`; line numbers may have drifted — `grep -n "txProposalRepo.Create" internal/service/scheduler/scheduler.go` to find current location). The surrounding loop reads `result.ToolLogs` (defined at `internal/service/agent/headless.go:21-34` as `[]ToolCallLog{Name, Input, Result}`). Before building the `TxProposal` struct:
   - Capture the active `tool.Name` from the `result.ToolLogs` entry — this becomes `proposal.ToolName`.
   - Extract `quest_metadata` from the tool's result JSON if present — this becomes `proposal.QuestMetadata`.
   - Pass both into `txProposalRepo.Create(...)`.

   Implementation sketch (~10 lines insert):

```go
// (around scheduler.go:306, before building the TxProposal struct)
var toolName string
var questMetadata json.RawMessage
if log := activeBuildToolLog(result.ToolLogs); log != nil {
    toolName = log.Name
    if raw, ok := log.Result.(map[string]any)["quest_metadata"]; ok {
        if b, err := json.Marshal(raw); err == nil {
            questMetadata = b
        }
    }
}

proposal := &types.TxProposal{
    // ... existing fields ...
    ToolName:      &toolName,
    QuestMetadata: questMetadata,
}
```

(Adjust `activeBuildToolLog` to whatever helper or inline loop the existing scheduler uses to pick the relevant tool from the result logs — see line 277–287 today.)

**Acceptance:** PR open in `vultisig/agent-backend` with the migration applied, the type updated, the scheduler populating both fields. Unit test in scheduler covering: (a) tool emits `quest_metadata` → both columns populated, (b) tool omits → both columns NULL, (c) older proposals (without the new columns) still work.

### Verification

When both sub-task PRs are merged to `main`, this command should succeed against your local DB:

```bash
psql "$DATABASE_DSN" -c "\d agent_tx_proposals" | grep -E "tool_name|quest_metadata"
```

Expected: two lines (tool_name as TEXT, quest_metadata as JSONB).

To confirm end-to-end, trigger a real swap through the agent (in dev) and inspect:

```bash
psql "$DATABASE_DSN" -c "SELECT id, action_type, tool_name, quest_metadata FROM agent_tx_proposals ORDER BY created_at DESC LIMIT 1;"
```

Expected: the most recent row has `tool_name = 'build_swap_tx'` and a populated `quest_metadata` blob with at least `amount_usd` and chain IDs.

### Once Task 0 acceptance is met

Proceed to Task 1. Until it's met, do not write any Stage 3 code — every E2E test will be untestable.

---

## Task 1: sqlc queries for quest events + user quests

> **Schema update (2026-04-23):** apply this column rename across all SQL + sqlc-generated Go in this task:
>
> | Old column | New column |
> |---|---|
> | `status airdrop_quest_event_status` (enum) | `counted BOOLEAN NOT NULL` |
> | `payload JSONB NOT NULL` | `tx_proposal_id UUID` (nullable, soft reference to agent_tx_proposals — no FK) |
>
> The `airdrop_quest_event_status` enum was dropped entirely; both shared-infra plan and stage-3 spec now reflect this. Status values "counted" / "rejected" become `counted = true / false`. Insert SQL parameters become `(tool_call_id, public_key, quest_id, tx_hash, counted, reject_reason, tx_proposal_id)`. The `payload` JSONB column was redundant with `agent_tx_proposals.quest_metadata`; audit consumers should join by `tx_proposal_id` if they need the metadata.

**Why:** Idempotent insert keyed on `tool_call_id`, plus the user-quest mark-complete with `xmax = 0 AS was_inserted` for first-completion detection (per spec response field `quest_completed`), plus a count for the response's `quests_completed` field.

**Files:**
- Create: `internal/storage/postgres/sqlc/airdrop_quest_events.sql`
- Create: `internal/storage/postgres/sqlc/airdrop_user_quests.sql`
- Generated: `internal/storage/postgres/queries/airdrop_quest_events.sql.go` (do not hand-edit)
- Generated: `internal/storage/postgres/queries/airdrop_user_quests.sql.go` (do not hand-edit)

- [ ] **Step 1: Write the quest-events queries**

Create `internal/storage/postgres/sqlc/airdrop_quest_events.sql`:

```sql
-- name: InsertQuestEvent :one
-- Idempotent insert keyed on tool_call_id (the events PK). On conflict, returns
-- the existing row so the handler can decide what to do (likely nothing — same
-- tool_call_id means the same broadcast was reported twice, which is a no-op).
WITH ins AS (
    INSERT INTO agent_quest_events (tool_call_id, public_key, quest_id, tx_hash, status, reject_reason, payload)
    VALUES ($1, $2, $3, $4, $5, $6, $7)
    ON CONFLICT (tool_call_id) DO NOTHING
    RETURNING *, true AS was_inserted
)
SELECT tool_call_id, public_key, quest_id, tx_hash, status, reject_reason, payload, created_at, was_inserted
FROM ins
UNION ALL
SELECT tool_call_id, public_key, quest_id, tx_hash, status, reject_reason, payload, created_at, false AS was_inserted
FROM agent_quest_events
WHERE tool_call_id = $1
  AND NOT EXISTS (SELECT 1 FROM ins)
LIMIT 1;
```

- [ ] **Step 2: Write the user-quests queries**

Create `internal/storage/postgres/sqlc/airdrop_user_quests.sql`:

```sql
-- name: MarkUserQuestComplete :one
-- Marks a quest complete for a user. Returns was_inserted=true on the first
-- qualifying event for that quest (drives the spec's `quest_completed: true`
-- response field). xmax = 0 is the canonical Postgres trick for "this row
-- was just inserted, not pre-existing" inside an INSERT ... ON CONFLICT.
INSERT INTO agent_user_quests (public_key, quest_id)
VALUES ($1, $2)
ON CONFLICT (public_key, quest_id) DO UPDATE SET completed_at = agent_user_quests.completed_at
RETURNING (xmax = 0) AS was_inserted;

-- name: CountUserQuests :one
-- Running count of completed quests for the user. Drives the `quests_completed`
-- response field. Fast PK lookup (covered by the (public_key, quest_id) PK).
SELECT COUNT(*)::bigint AS count
FROM agent_user_quests
WHERE public_key = $1;
```

`ON CONFLICT DO UPDATE SET completed_at = agent_user_quests.completed_at` is a no-op write that lets `xmax = 0` work as the "fresh insert" detector. `DO NOTHING` would skip the RETURNING clause entirely, leaving us with no rows on the retry path — we'd need a second SELECT to disambiguate. The no-op UPDATE is cheaper.

- [ ] **Step 3: Run sqlc generation**

```bash
grep -n sqlc Makefile
```

Use the existing `make sqlc` target if one exists, otherwise:

```bash
sqlc generate
```

Expected: two new files appear in `internal/storage/postgres/queries/`:
- `airdrop_quest_events.sql.go`
- `airdrop_user_quests.sql.go`

- [ ] **Step 4: Verify generated code compiles**

```bash
go build ./internal/storage/postgres/...
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/storage/postgres/sqlc/airdrop_quest_events.sql \
         internal/storage/postgres/sqlc/airdrop_user_quests.sql \
         internal/storage/postgres/queries/airdrop_quest_events.sql.go \
         internal/storage/postgres/queries/airdrop_user_quests.sql.go
git commit -m "feat(airdrop): add sqlc queries for quest events + user quests"
```

---

## Task 2: `QuestEventRepository` wrapping the queries

**Why:** Map sqlc-generated types to plain Go domain types, same pattern as `internal/storage/postgres/conversation.go`. Two methods (`InsertEvent`, `MarkComplete`) plus a count helper.

**Files:**
- Create: `internal/storage/postgres/airdrop_quest_event.go`
- Create: `internal/storage/postgres/airdrop_quest_event_test.go`
- Modify: `internal/types/airdrop.go` (add quest types — file already exists from Stage 1)

- [ ] **Step 1: Add domain types**

Append to `internal/types/airdrop.go` (Stage 1 created this file with `RegistrationSource` etc.; if Stage 1 hasn't merged yet, create the file with just these types):

```go
// QuestID is the canonical identifier for one of the five Vultiverse quests.
// Mirrors the airdrop_quest_id Postgres enum.
type QuestID string

const (
	QuestSwap       QuestID = "swap"
	QuestBridge     QuestID = "bridge"
	QuestDefiAction QuestID = "defi_action"
	QuestAlert      QuestID = "alert"
	QuestDCA        QuestID = "dca"
)

// AllQuestIDs is the canonical list, in display order. Match the Stage 1
// helper if both files end up defining a list — these should be identical.
var AllQuestIDs = []QuestID{QuestSwap, QuestBridge, QuestDefiAction, QuestAlert, QuestDCA}

func (q QuestID) Valid() bool {
	switch q {
	case QuestSwap, QuestBridge, QuestDefiAction, QuestAlert, QuestDCA:
		return true
	}
	return false
}

// QuestEventStatus mirrors the airdrop_quest_event_status Postgres enum.
type QuestEventStatus string

const (
	QuestEventCounted  QuestEventStatus = "counted"
	QuestEventRejected QuestEventStatus = "rejected"
)

// QuestEvent is the domain representation of one row in agent_quest_events.
type QuestEvent struct {
	ToolCallID   string
	PublicKey    string
	QuestID      QuestID
	TxHash       string
	Status       QuestEventStatus
	RejectReason string // empty unless Status == QuestEventRejected
	Payload      []byte // raw JSONB
	CreatedAt    time.Time
	WasInserted  bool   // true if this call inserted; false if existing row returned
}
```

- [ ] **Step 2: Write the repo test**

Create `internal/storage/postgres/airdrop_quest_event_test.go`. Use the same `newTestPool` helper Stage 1 added in `airdrop_registration_test.go` — copy that helper inline if Stage 1 hasn't merged yet:

```go
package postgres_test

import (
	"context"
	"os"
	"testing"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/storage/postgres"
	"github.com/vultisig/agent-backend/internal/types"
)

func newQuestTestPool(t *testing.T) *pgxpool.Pool {
	t.Helper()
	dsn := os.Getenv("TEST_DATABASE_DSN")
	if dsn == "" {
		dsn = os.Getenv("DATABASE_DSN")
	}
	if dsn == "" {
		t.Skip("TEST_DATABASE_DSN or DATABASE_DSN not set")
	}
	pool, err := pgxpool.New(context.Background(), dsn)
	if err != nil {
		t.Fatalf("pgxpool.New: %v", err)
	}
	t.Cleanup(func() { pool.Close() })
	_, err = pool.Exec(context.Background(), `
		TRUNCATE agent_quest_events;
		TRUNCATE agent_user_quests;
	`)
	if err != nil {
		t.Fatalf("truncate: %v", err)
	}
	return pool
}

func TestQuestEventRepo_InsertEventIsIdempotent(t *testing.T) {
	pool := newQuestTestPool(t)
	repo := postgres.NewQuestEventRepository(pool)
	ctx := context.Background()

	first, err := repo.InsertEvent(ctx, postgres.InsertQuestEventParams{
		ToolCallID:   "call-abc",
		PublicKey:    "pk1",
		QuestID:      types.QuestSwap,
		TxHash:       "0xdead",
		Status:       types.QuestEventCounted,
		RejectReason: "",
		Payload:      []byte(`{"amount_usd":42}`),
	})
	if err != nil {
		t.Fatalf("first insert: %v", err)
	}
	if !first.WasInserted {
		t.Errorf("first call should be a fresh insert")
	}

	second, err := repo.InsertEvent(ctx, postgres.InsertQuestEventParams{
		ToolCallID:   "call-abc",
		PublicKey:    "pk1",
		QuestID:      types.QuestSwap,
		TxHash:       "0xdead",
		Status:       types.QuestEventCounted,
		RejectReason: "",
		Payload:      []byte(`{"amount_usd":42}`),
	})
	if err != nil {
		t.Fatalf("second insert: %v", err)
	}
	if second.WasInserted {
		t.Errorf("second call should be a no-op (idempotent retry)")
	}
	if second.ToolCallID != first.ToolCallID {
		t.Errorf("idempotent retry should return same row; got %q vs %q", second.ToolCallID, first.ToolCallID)
	}
}

func TestQuestEventRepo_MarkCompleteFirstTimeOnly(t *testing.T) {
	pool := newQuestTestPool(t)
	repo := postgres.NewQuestEventRepository(pool)
	ctx := context.Background()

	first, err := repo.MarkComplete(ctx, "pk1", types.QuestSwap)
	if err != nil {
		t.Fatalf("MarkComplete first: %v", err)
	}
	if !first {
		t.Errorf("first MarkComplete should report was_inserted=true")
	}

	second, err := repo.MarkComplete(ctx, "pk1", types.QuestSwap)
	if err != nil {
		t.Fatalf("MarkComplete second: %v", err)
	}
	if second {
		t.Errorf("second MarkComplete (already-complete quest) should report was_inserted=false")
	}
}

func TestQuestEventRepo_CountUserQuests(t *testing.T) {
	pool := newQuestTestPool(t)
	repo := postgres.NewQuestEventRepository(pool)
	ctx := context.Background()

	if got, err := repo.CountUserQuests(ctx, "pk1"); err != nil || got != 0 {
		t.Errorf("empty count = %d, err = %v; want 0, nil", got, err)
	}

	_, _ = repo.MarkComplete(ctx, "pk1", types.QuestSwap)
	_, _ = repo.MarkComplete(ctx, "pk1", types.QuestBridge)
	_, _ = repo.MarkComplete(ctx, "pk1", types.QuestSwap) // duplicate, doesn't bump count

	got, err := repo.CountUserQuests(ctx, "pk1")
	if err != nil {
		t.Fatalf("CountUserQuests: %v", err)
	}
	if got != 2 {
		t.Errorf("count = %d, want 2", got)
	}
}
```

- [ ] **Step 3: Run test to verify it fails**

```bash
go test ./internal/storage/postgres/ -run TestQuestEventRepo -v
```

Expected: compile error — `NewQuestEventRepository`, `InsertQuestEventParams` undefined.

- [ ] **Step 4: Implement the repository**

Create `internal/storage/postgres/airdrop_quest_event.go`. Match `conversation.go`'s pattern (struct holding `pool` + `q`, `New...Repository(pool)` constructor):

```go
package postgres

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/storage/postgres/queries"
	"github.com/vultisig/agent-backend/internal/types"
)

// QuestEventRepository wraps the sqlc-generated quest event + user quest queries
// with domain-typed methods.
type QuestEventRepository struct {
	pool *pgxpool.Pool
	q    *queries.Queries
}

func NewQuestEventRepository(pool *pgxpool.Pool) *QuestEventRepository {
	return &QuestEventRepository{
		pool: pool,
		q:    queries.New(pool),
	}
}

// InsertQuestEventParams is the input shape for InsertEvent. Mirrors the spec's
// `agent_quest_events` row 1:1 except for created_at (DB default).
type InsertQuestEventParams struct {
	ToolCallID   string
	PublicKey    string
	QuestID      types.QuestID
	TxHash       string
	Status       types.QuestEventStatus
	RejectReason string // empty for counted events
	Payload      []byte // raw JSONB
}

// InsertEvent inserts an event, idempotent on tool_call_id. On conflict returns
// the existing row with WasInserted = false.
func (r *QuestEventRepository) InsertEvent(ctx context.Context, p InsertQuestEventParams) (*types.QuestEvent, error) {
	var rejectReason *string
	if p.RejectReason != "" {
		rejectReason = &p.RejectReason
	}
	row, err := r.q.InsertQuestEvent(ctx, &queries.InsertQuestEventParams{
		ToolCallID:   p.ToolCallID,
		PublicKey:    p.PublicKey,
		QuestID:      queries.AirdropQuestID(p.QuestID),
		TxHash:       p.TxHash,
		Status:       queries.AirdropQuestEventStatus(p.Status),
		RejectReason: rejectReason,
		Payload:      p.Payload,
	})
	if err != nil {
		return nil, fmt.Errorf("insert quest event: %w", err)
	}
	out := &types.QuestEvent{
		ToolCallID:  row.ToolCallID,
		PublicKey:   row.PublicKey,
		QuestID:     types.QuestID(row.QuestID),
		TxHash:      row.TxHash,
		Status:      types.QuestEventStatus(row.Status),
		Payload:     row.Payload,
		CreatedAt:   pgtimestamptzToTime(row.CreatedAt),
		WasInserted: row.WasInserted,
	}
	if row.RejectReason != nil {
		out.RejectReason = *row.RejectReason
	}
	return out, nil
}

// MarkComplete records that publicKey has completed questID. Returns true on the
// first qualifying call (so the caller can populate the spec's `quest_completed`
// response flag) and false for already-complete quests (subsequent qualifying events).
func (r *QuestEventRepository) MarkComplete(ctx context.Context, publicKey string, questID types.QuestID) (bool, error) {
	wasInserted, err := r.q.MarkUserQuestComplete(ctx, &queries.MarkUserQuestCompleteParams{
		PublicKey: publicKey,
		QuestID:   queries.AirdropQuestID(questID),
	})
	if err != nil {
		return false, fmt.Errorf("mark user quest complete: %w", err)
	}
	return wasInserted, nil
}

// CountUserQuests returns the running count of completed quests for the user.
func (r *QuestEventRepository) CountUserQuests(ctx context.Context, publicKey string) (int64, error) {
	n, err := r.q.CountUserQuests(ctx, publicKey)
	if err != nil {
		return 0, fmt.Errorf("count user quests: %w", err)
	}
	return n, nil
}
```

The exact name of the sqlc-generated enum types (e.g. `queries.AirdropQuestID`, `queries.AirdropQuestEventStatus`) and the `pgtimestamptzToTime` helper depend on what sqlc generated and what `conversation.go` uses today. Open `internal/storage/postgres/queries/airdrop_quest_events.sql.go` to confirm the type names; copy the timestamp helper from `conversation.go`.

- [ ] **Step 5: Run tests to verify they pass**

```bash
DATABASE_DSN="$DATABASE_DSN" go test ./internal/storage/postgres/ -run TestQuestEventRepo -v
```

Expected: PASS for all three subtests.

- [ ] **Step 6: Commit**

```bash
git add internal/storage/postgres/airdrop_quest_event.go \
         internal/storage/postgres/airdrop_quest_event_test.go \
         internal/types/airdrop.go
git commit -m "feat(airdrop): add QuestEventRepository wrapping sqlc queries"
```

---

## Task 3: Per-quest validators (3 real, 2 stubs)

> **Field-name update (2026-04-23):** the validator inputs now match the `quest_metadata` JSONB shape locked in Task 0, not the original spec's free-form `data` field. Apply this rename across all test fixtures and the implementations in this task:
>
> | Old field | New field (from `quest_metadata`) |
> |---|---|
> | `token_in` | `token_in_symbol` |
> | `token_out` | `token_out_symbol` |
> | `chain_id` | `source_chain_id` (and `dest_chain_id` for bridge) |
> | `contract_address` | `target_contract` |
> | `amount_usd`, `router_address` | unchanged |
>
> **Swap validator** additionally must assert `source_chain_id == dest_chain_id` (same-chain swap; otherwise it's a bridge — dispatched there instead). **Bridge validator** asserts `source_chain_id != dest_chain_id`. **DeFi-action validator** does a lowercase compare of `target_contract` against the `DEFI_ACTION_CONTRACT_ALLOWLIST` (config is lowercased at parse time per Stage 3 spec).
>
> All other validator structure (Result type, Pass/Reject helpers, file layout, test patterns) is unchanged. Apply the renames as you copy the test fixtures and the implementations from the steps below.

**Why:** One file per quest type keeps each validator's rules in one place. Pure functions (`quest_metadata` map + relevant config slice → `(qualifies, reason)`) are trivial to unit-test for boundary cases and have no dependency on Postgres or HTTP.

**Files:**
- Create: `internal/service/airdrop/quests/validators/types.go` (shared types)
- Create: `internal/service/airdrop/quests/validators/swap.go` + `swap_test.go`
- Create: `internal/service/airdrop/quests/validators/bridge.go` + `bridge_test.go`
- Create: `internal/service/airdrop/quests/validators/defi_action.go` + `defi_action_test.go`
- Create: `internal/service/airdrop/quests/validators/alert.go` (stub — no test needed; trivial)
- Create: `internal/service/airdrop/quests/validators/dca.go` (stub — no test needed)

- [ ] **Step 1: Define shared types**

Create `internal/service/airdrop/quests/validators/types.go`:

```go
// Package validators holds per-quest validators for the Vultiverse Airdrop.
// Each validator is a pure function: given a JSONB payload (the request body's
// `data` field) and the relevant slice of config, return whether the action
// qualifies for the quest plus a structured rejection reason for logging.
//
// New quests slot in via two changes: a file in this package (the validator)
// and an entry in `quests/dispatch.go` (the dispatch-table mapping).
package validators

// Result is the validator's verdict on a single payload.
type Result struct {
	// Qualifies is true if the payload meets the quest's rules.
	Qualifies bool
	// Reason is a short snake_case identifier for why the payload was rejected.
	// Empty when Qualifies == true. Examples: "amount_below_min",
	// "router_not_allowlisted", "same_chain_bridge", "not_yet_implemented".
	Reason string
}

// Pass is the canonical "qualified" result.
var Pass = Result{Qualifies: true, Reason: ""}

// Reject builds a rejected result with the given reason.
func Reject(reason string) Result {
	return Result{Qualifies: false, Reason: reason}
}
```

- [ ] **Step 2: Write the swap validator test**

Create `internal/service/airdrop/quests/validators/swap_test.go`:

```go
package validators

import "testing"

func TestSwap(t *testing.T) {
	cases := []struct {
		name      string
		payload   []byte
		minUSD    int
		want      bool
		reason    string
	}{
		{
			name:    "exact min usd → qualifies",
			payload: []byte(`{"amount_usd":10,"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xabc"}`),
			minUSD:  10,
			want:    true,
		},
		{
			name:    "above min → qualifies",
			payload: []byte(`{"amount_usd":12.5,"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xabc"}`),
			minUSD:  10,
			want:    true,
		},
		{
			name:    "below min → rejected with amount_below_min",
			payload: []byte(`{"amount_usd":9.99,"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xabc"}`),
			minUSD:  10,
			want:    false, reason: "amount_below_min",
		},
		{
			name:    "missing amount_usd → rejected",
			payload: []byte(`{"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xabc"}`),
			minUSD:  10,
			want:    false, reason: "missing_amount_usd",
		},
		{
			name:    "missing router_address → rejected",
			payload: []byte(`{"amount_usd":15,"token_in":"USDC","token_out":"ETH","chain_id":1}`),
			minUSD:  10,
			want:    false, reason: "missing_router_address",
		},
		{
			name:    "malformed JSON → rejected",
			payload: []byte(`{not-json`),
			minUSD:  10,
			want:    false, reason: "invalid_payload",
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			got := Swap(tc.payload, tc.minUSD)
			if got.Qualifies != tc.want {
				t.Errorf("Qualifies = %v, want %v (reason=%q)", got.Qualifies, tc.want, got.Reason)
			}
			if !tc.want && got.Reason != tc.reason {
				t.Errorf("Reason = %q, want %q", got.Reason, tc.reason)
			}
		})
	}
}
```

- [ ] **Step 3: Run swap test to verify it fails**

```bash
go test ./internal/service/airdrop/quests/validators/ -run TestSwap -v
```

Expected: compile error — `Swap` undefined.

- [ ] **Step 4: Implement swap validator**

Create `internal/service/airdrop/quests/validators/swap.go`:

```go
package validators

import "encoding/json"

// swapPayload is the documented `data` shape for swap quest events.
// Spec: docs/airdrop-specs/stage-3-quest-tracking.md, "Per-quest data payloads".
type swapPayload struct {
	AmountUSD     *float64 `json:"amount_usd"`
	TokenIn       string   `json:"token_in"`
	TokenOut      string   `json:"token_out"`
	ChainID       int      `json:"chain_id"`
	RouterAddress string   `json:"router_address"`
}

// Swap qualifies if the payload's amount_usd >= minUSD AND the router address
// is non-empty. Router-allowlist enforcement is intentionally out of scope for
// the swap quest (see spec): the swap MCP tool's whitelist of integrated
// providers IS the allowlist; if `build_swap_tx` produced the tx then the
// router is by construction one we trust. We capture the address for audit.
func Swap(rawPayload []byte, minUSD int) Result {
	var p swapPayload
	if err := json.Unmarshal(rawPayload, &p); err != nil {
		return Reject("invalid_payload")
	}
	if p.AmountUSD == nil {
		return Reject("missing_amount_usd")
	}
	if p.RouterAddress == "" {
		return Reject("missing_router_address")
	}
	if *p.AmountUSD < float64(minUSD) {
		return Reject("amount_below_min")
	}
	return Pass
}
```

- [ ] **Step 5: Run swap test to verify it passes**

```bash
go test ./internal/service/airdrop/quests/validators/ -run TestSwap -v
```

Expected: PASS.

- [ ] **Step 6: Write the bridge validator test**

Create `internal/service/airdrop/quests/validators/bridge_test.go`:

```go
package validators

import "testing"

func TestBridge(t *testing.T) {
	cases := []struct {
		name    string
		payload []byte
		minUSD  int
		want    bool
		reason  string
	}{
		{
			name:    "valid cross-chain at min → qualifies",
			payload: []byte(`{"amount_usd":10,"source_chain_id":1,"dest_chain_id":42161}`),
			minUSD:  10, want: true,
		},
		{
			name:    "same chain → rejected with same_chain_bridge",
			payload: []byte(`{"amount_usd":50,"source_chain_id":1,"dest_chain_id":1}`),
			minUSD:  10, want: false, reason: "same_chain_bridge",
		},
		{
			name:    "below min → rejected",
			payload: []byte(`{"amount_usd":5,"source_chain_id":1,"dest_chain_id":42161}`),
			minUSD:  10, want: false, reason: "amount_below_min",
		},
		{
			name:    "missing source_chain_id → rejected",
			payload: []byte(`{"amount_usd":15,"dest_chain_id":42161}`),
			minUSD:  10, want: false, reason: "missing_source_chain_id",
		},
		{
			name:    "missing dest_chain_id → rejected",
			payload: []byte(`{"amount_usd":15,"source_chain_id":1}`),
			minUSD:  10, want: false, reason: "missing_dest_chain_id",
		},
		{
			name:    "malformed JSON → rejected",
			payload: []byte(`x`),
			minUSD:  10, want: false, reason: "invalid_payload",
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			got := Bridge(tc.payload, tc.minUSD)
			if got.Qualifies != tc.want {
				t.Errorf("Qualifies = %v, want %v (reason=%q)", got.Qualifies, tc.want, got.Reason)
			}
			if !tc.want && got.Reason != tc.reason {
				t.Errorf("Reason = %q, want %q", got.Reason, tc.reason)
			}
		})
	}
}
```

- [ ] **Step 7: Implement bridge validator**

Create `internal/service/airdrop/quests/validators/bridge.go`:

```go
package validators

import "encoding/json"

type bridgePayload struct {
	AmountUSD     *float64 `json:"amount_usd"`
	SourceChainID *int     `json:"source_chain_id"`
	DestChainID   *int     `json:"dest_chain_id"`
}

// Bridge qualifies if amount_usd >= minUSD AND source_chain_id != dest_chain_id.
func Bridge(rawPayload []byte, minUSD int) Result {
	var p bridgePayload
	if err := json.Unmarshal(rawPayload, &p); err != nil {
		return Reject("invalid_payload")
	}
	if p.AmountUSD == nil {
		return Reject("missing_amount_usd")
	}
	if p.SourceChainID == nil {
		return Reject("missing_source_chain_id")
	}
	if p.DestChainID == nil {
		return Reject("missing_dest_chain_id")
	}
	if *p.SourceChainID == *p.DestChainID {
		return Reject("same_chain_bridge")
	}
	if *p.AmountUSD < float64(minUSD) {
		return Reject("amount_below_min")
	}
	return Pass
}
```

- [ ] **Step 8: Write the defi-action validator test**

Create `internal/service/airdrop/quests/validators/defi_action_test.go`:

```go
package validators

import "testing"

func TestDefiAction(t *testing.T) {
	allow := []string{
		"0x1111111111111111111111111111111111111111",
		"0x2222222222222222222222222222222222222222",
	}

	cases := []struct {
		name      string
		payload   []byte
		allowlist []string
		want      bool
		reason    string
	}{
		{
			name:      "contract on allowlist → qualifies",
			payload:   []byte(`{"contract_address":"0x1111111111111111111111111111111111111111","chain_id":1,"method":"deposit"}`),
			allowlist: allow, want: true,
		},
		{
			name:      "case-insensitive match → qualifies",
			payload:   []byte(`{"contract_address":"0X1111111111111111111111111111111111111111","chain_id":1,"method":"deposit"}`),
			allowlist: allow, want: true,
		},
		{
			name:      "contract not on allowlist → rejected",
			payload:   []byte(`{"contract_address":"0x9999999999999999999999999999999999999999","chain_id":1,"method":"deposit"}`),
			allowlist: allow, want: false, reason: "contract_not_allowlisted",
		},
		{
			name:      "missing contract_address → rejected",
			payload:   []byte(`{"chain_id":1,"method":"deposit"}`),
			allowlist: allow, want: false, reason: "missing_contract_address",
		},
		{
			name:      "empty allowlist → all rejected",
			payload:   []byte(`{"contract_address":"0x1111111111111111111111111111111111111111","chain_id":1,"method":"deposit"}`),
			allowlist: []string{}, want: false, reason: "contract_not_allowlisted",
		},
		{
			name:      "malformed JSON → rejected",
			payload:   []byte(`{`),
			allowlist: allow, want: false, reason: "invalid_payload",
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			got := DefiAction(tc.payload, tc.allowlist)
			if got.Qualifies != tc.want {
				t.Errorf("Qualifies = %v, want %v (reason=%q)", got.Qualifies, tc.want, got.Reason)
			}
			if !tc.want && got.Reason != tc.reason {
				t.Errorf("Reason = %q, want %q", got.Reason, tc.reason)
			}
		})
	}
}
```

- [ ] **Step 9: Implement defi-action validator**

Create `internal/service/airdrop/quests/validators/defi_action.go`:

```go
package validators

import (
	"encoding/json"
	"strings"
)

type defiActionPayload struct {
	ContractAddress string `json:"contract_address"`
	ChainID         int    `json:"chain_id"`
	Method          string `json:"method"`
}

// DefiAction qualifies if contract_address (case-insensitive) is in the allowlist.
// Allowlist comes from cfg.Airdrop.DefiActionContractAllowlist (a comma-separated
// string in env, normalized to []string at boot).
func DefiAction(rawPayload []byte, allowlist []string) Result {
	var p defiActionPayload
	if err := json.Unmarshal(rawPayload, &p); err != nil {
		return Reject("invalid_payload")
	}
	if p.ContractAddress == "" {
		return Reject("missing_contract_address")
	}
	target := strings.ToLower(p.ContractAddress)
	for _, addr := range allowlist {
		if strings.ToLower(strings.TrimSpace(addr)) == target {
			return Pass
		}
	}
	return Reject("contract_not_allowlisted")
}
```

- [ ] **Step 10: Implement alert + dca stubs**

Create `internal/service/airdrop/quests/validators/alert.go`:

```go
package validators

// Alert is a stub validator. The alert quest's rules are TBD with Product
// (per stage-0-preflight decisions). Wiring exists end-to-end so the rules
// are a one-file change here when Product locks them — the dispatch-table
// entry doesn't need to change.
//
// Until then: every alert event is recorded with status='rejected' and
// reason='not_yet_implemented' (audit-trail visible to Product so they can
// see the volume of would-have-counted events).
func Alert(_ []byte) Result {
	return Reject("not_yet_implemented")
}
```

Create `internal/service/airdrop/quests/validators/dca.go`:

```go
package validators

// DCA is a stub validator. See Alert for the same rationale: rules TBD with
// Product, wiring is in place, every event recorded as rejected with
// reason='not_yet_implemented'.
func DCA(_ []byte) Result {
	return Reject("not_yet_implemented")
}
```

- [ ] **Step 11: Run all validator tests**

```bash
go test ./internal/service/airdrop/quests/validators/ -v
```

Expected: PASS for `TestSwap`, `TestBridge`, `TestDefiAction`. Stubs have no tests (function body is one line).

- [ ] **Step 12: Commit**

```bash
git add internal/service/airdrop/quests/validators/
git commit -m "feat(airdrop): add per-quest validators (swap/bridge/defi_action + alert/dca stubs)"
```

---

## Task 4: Quest dispatch table

> **2026-04-23 update:** the `tool_name → quest_type` map is the *primary* dispatch (per the Cross-team contract from Task 0). For `build_swap_tx` specifically, the resolution depends on whether the tx is same-chain (swap) or cross-chain (bridge) — so the dispatcher inspects `quest_metadata.source_chain_id` vs `quest_metadata.dest_chain_id`. Reference shape:
>
> ```go
> // QuestForTool returns the quest_type this tool maps to, or "" if none.
> // Reads quest_metadata for the swap-vs-bridge resolution.
> func QuestForTool(toolName string, meta map[string]any) string {
>     switch toolName {
>     case "build_swap_tx":
>         srcRaw, dstRaw := meta["source_chain_id"], meta["dest_chain_id"]
>         if asInt(srcRaw) == asInt(dstRaw) {
>             return "swap"
>         }
>         return "bridge"
>     case "build_bridge_tx":
>         return "bridge"
>     case "build_evm_tx":
>         return "defi_action"
>     default:
>         return ""
>     }
> }
> ```
>
> Add `QuestForTool` (and a small `asInt` helper) to `dispatch.go` alongside the existing `quest_type → validator` map. The TestDispatch tests should cover: each mapped tool, the swap-vs-bridge resolution by chain ID, and an unmapped tool returning "".

**Why:** One Go file mapping `quest_type → validator function` AND `tool_name → quest_type`. Adding a new quest is a one-file change here; the event handler and the tx-proposal hook both consume this mapping.

**Files:**
- Create: `internal/service/airdrop/quests/dispatch.go`
- Create: `internal/service/airdrop/quests/dispatch_test.go`

- [ ] **Step 1: Write the dispatch test**

Create `internal/service/airdrop/quests/dispatch_test.go`:

```go
package quests

import (
	"testing"

	"github.com/vultisig/agent-backend/internal/types"
)

func TestQuestTypeForTool(t *testing.T) {
	cases := []struct {
		toolName string
		want     types.QuestID
		wantOK   bool
	}{
		{"build_swap_tx", types.QuestSwap, true},
		{"build_bridge_tx", types.QuestBridge, true},
		{"build_evm_tx", types.QuestDefiAction, true},
		{"build_send_tx", "", false},   // sends don't count toward any quest
		{"get_balances", "", false},    // read-only tools never count
		{"", "", false},
	}
	for _, tc := range cases {
		got, ok := QuestTypeForTool(tc.toolName)
		if ok != tc.wantOK {
			t.Errorf("QuestTypeForTool(%q) ok = %v, want %v", tc.toolName, ok, tc.wantOK)
		}
		if got != tc.want {
			t.Errorf("QuestTypeForTool(%q) = %q, want %q", tc.toolName, got, tc.want)
		}
	}
}

func TestValidate_DispatchesByQuestType(t *testing.T) {
	cfg := DispatchConfig{
		SwapQuestMinUSD:             10,
		BridgeQuestMinUSD:           10,
		DefiActionContractAllowlist: []string{"0xabc"},
	}

	swapPayload := []byte(`{"amount_usd":15,"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xrouter"}`)
	res, err := Validate(types.QuestSwap, swapPayload, cfg)
	if err != nil {
		t.Fatalf("validate swap: %v", err)
	}
	if !res.Qualifies {
		t.Errorf("expected swap to qualify, got %+v", res)
	}

	bridgePayload := []byte(`{"amount_usd":15,"source_chain_id":1,"dest_chain_id":42161}`)
	res, err = Validate(types.QuestBridge, bridgePayload, cfg)
	if err != nil || !res.Qualifies {
		t.Errorf("bridge: %+v, err=%v", res, err)
	}

	// Unknown quest type → error
	if _, err := Validate(types.QuestID("invalid"), []byte(`{}`), cfg); err == nil {
		t.Error("expected error for unknown quest type")
	}

	// Stub returns rejected
	res, err = Validate(types.QuestAlert, []byte(`{}`), cfg)
	if err != nil {
		t.Fatalf("alert: %v", err)
	}
	if res.Qualifies || res.Reason != "not_yet_implemented" {
		t.Errorf("alert stub: got %+v, want rejected/not_yet_implemented", res)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/service/airdrop/quests/ -run "TestQuestTypeForTool|TestValidate_DispatchesByQuestType" -v
```

Expected: compile error.

- [ ] **Step 3: Implement the dispatch table**

Create `internal/service/airdrop/quests/dispatch.go`:

```go
package quests

import (
	"fmt"

	"github.com/vultisig/agent-backend/internal/service/airdrop/quests/validators"
	"github.com/vultisig/agent-backend/internal/types"
)

// toolToQuest maps the agent's tool names to the quest type they trigger.
// This is the ONE place to update when a new triggering tool ships.
//
// Mapping rationale:
//   - build_swap_tx       → swap (the canonical swap MCP tool; see
//                           internal/service/agent/prompt.go for the prompt
//                           guidance the LLM follows)
//   - build_bridge_tx     → bridge (cross-chain via the bridge MCP tool)
//   - build_evm_tx        → defi_action (any EVM tx that's not a send;
//                           the validator filters by contract allowlist so
//                           a vanilla ETH transfer to a friend doesn't count)
//
// build_send_tx, build_btc_send, build_solana_tx, build_sui_send, build_ton_send,
// sign_typed_data, etc. are intentionally absent — sends and signatures don't
// qualify for any quest in the launch v1 list.
var toolToQuest = map[string]types.QuestID{
	"build_swap_tx":   types.QuestSwap,
	"build_bridge_tx": types.QuestBridge,
	"build_evm_tx":    types.QuestDefiAction,
}

// QuestTypeForTool returns the quest type a given tool name triggers, or
// (zero, false) if the tool isn't quest-relevant.
func QuestTypeForTool(toolName string) (types.QuestID, bool) {
	q, ok := toolToQuest[toolName]
	return q, ok
}

// DispatchConfig is the slice of cfg.Airdrop the validators need. Passed
// explicitly (rather than imported) so this package stays free of import
// cycles with internal/config.
type DispatchConfig struct {
	SwapQuestMinUSD             int
	BridgeQuestMinUSD           int
	DefiActionContractAllowlist []string
}

// FromAirdropConfig converts the comma-separated allowlist string from the
// config struct into a slice. Call this once at boot and pass the result
// into the handler — splitting per-request would be wasteful.
func FromAirdropConfig(swapMinUSD, bridgeMinUSD int, defiAllowlistCSV string) DispatchConfig {
	var allow []string
	if defiAllowlistCSV != "" {
		// Manual split to keep imports tight; equivalent to strings.Split + trim.
		var current []byte
		for i := 0; i <= len(defiAllowlistCSV); i++ {
			if i == len(defiAllowlistCSV) || defiAllowlistCSV[i] == ',' {
				if len(current) > 0 {
					allow = append(allow, string(current))
					current = current[:0]
				}
				continue
			}
			c := defiAllowlistCSV[i]
			if c == ' ' || c == '\t' {
				continue
			}
			current = append(current, c)
		}
	}
	return DispatchConfig{
		SwapQuestMinUSD:             swapMinUSD,
		BridgeQuestMinUSD:           bridgeMinUSD,
		DefiActionContractAllowlist: allow,
	}
}

// Validate dispatches to the per-quest validator. Returns the validator's
// Result. Returns an error only for unknown quest types — that's a programmer
// bug (the handler validated the enum already), not a per-event failure.
func Validate(questID types.QuestID, payload []byte, cfg DispatchConfig) (validators.Result, error) {
	switch questID {
	case types.QuestSwap:
		return validators.Swap(payload, cfg.SwapQuestMinUSD), nil
	case types.QuestBridge:
		return validators.Bridge(payload, cfg.BridgeQuestMinUSD), nil
	case types.QuestDefiAction:
		return validators.DefiAction(payload, cfg.DefiActionContractAllowlist), nil
	case types.QuestAlert:
		return validators.Alert(payload), nil
	case types.QuestDCA:
		return validators.DCA(payload), nil
	default:
		return validators.Result{}, fmt.Errorf("unknown quest type: %q", questID)
	}
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/quests/ -run "TestQuestTypeForTool|TestValidate_DispatchesByQuestType" -v
```

Expected: PASS for both.

- [ ] **Step 5: Commit**

```bash
git add internal/service/airdrop/quests/dispatch.go \
         internal/service/airdrop/quests/dispatch_test.go
git commit -m "feat(airdrop): add quest dispatch table (tool→type, type→validator)"
```

---

## Task 5: `POST /internal/quests/event` handler

> **2026-04-23 update:** the request body now carries `tool_name` + `quest_metadata` (forwarded from the agent-backend's signed proposal). The handler resolves quest_type via `quests.QuestForTool(tool_name, quest_metadata)`. If unmapped, return 200 with `{"recorded": false}` — not an error. New request shape:
>
> ```json
> {
>   "public_key":     "...",
>   "tool_call_id":   "<proposal_uuid>",
>   "tx_hash":        "0x...",
>   "tool_name":      "build_swap_tx",
>   "quest_metadata": { "amount_usd": 25.5, "token_in_symbol": "USDC", ... }
> }
> ```
>
> The handler then passes `quest_metadata` (as `map[string]any`) to the resolved validator. Update the request struct, the dispatch call, and any test fixtures accordingly. The 400 error key changes from `INVALID_QUEST_TYPE` to `MISSING_FIELDS` (per the updated spec) — quest_type is no longer in the request envelope.

**Why:** The single ingestion endpoint. Validates envelope, gates on activation timestamp, dispatches to the right validator, persists event + (if qualifying) marks the quest complete.

**Files:**
- Create: `internal/api/airdrop_quest_event.go`
- Create: `internal/api/airdrop_quest_event_test.go`
- Modify: `internal/api/server.go` (add `questEventRepo` field + setter)

- [ ] **Step 1: Write the failing test**

Create `internal/api/airdrop_quest_event_test.go`:

```go
package api

import (
	"bytes"
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/vultisig/agent-backend/internal/service/airdrop/quests"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
	"github.com/vultisig/agent-backend/internal/types"
)

// fakeQuestRepo lets us exercise the handler without Postgres.
type fakeQuestRepo struct {
	insertEvent   func(ctx context.Context, p postgres.InsertQuestEventParams) (*types.QuestEvent, error)
	markComplete  func(ctx context.Context, pk string, q types.QuestID) (bool, error)
	countComplete func(ctx context.Context, pk string) (int64, error)
}

func (f *fakeQuestRepo) InsertEvent(ctx context.Context, p postgres.InsertQuestEventParams) (*types.QuestEvent, error) {
	return f.insertEvent(ctx, p)
}
func (f *fakeQuestRepo) MarkComplete(ctx context.Context, pk string, q types.QuestID) (bool, error) {
	return f.markComplete(ctx, pk, q)
}
func (f *fakeQuestRepo) CountUserQuests(ctx context.Context, pk string) (int64, error) {
	return f.countComplete(ctx, pk)
}

func TestQuestEventHandler(t *testing.T) {
	now := time.Date(2026, 5, 1, 12, 0, 0, 0, time.UTC)
	activation := time.Date(2026, 5, 1, 0, 0, 0, 0, time.UTC) // before `now`

	type recordedInsert struct {
		params postgres.InsertQuestEventParams
		called bool
	}
	type recordedMark struct {
		pk     string
		quest  types.QuestID
		called bool
	}

	cases := []struct {
		name           string
		body           string
		now            time.Time
		activation     time.Time
		fakeMarkResult bool
		fakeCount      int64
		wantStatus     int
		wantErrKey     string
		wantInsert     bool
		wantInsertStat types.QuestEventStatus
		wantMark       bool
		wantBodyJSON   map[string]any
	}{
		{
			name: "valid swap → 200, recorded, quest_completed=true",
			body: `{"public_key":"pk1","quest_type":"swap","tool_call_id":"tc1","tx_hash":"0xabc","data":{"amount_usd":15,"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xrouter"}}`,
			now: now, activation: activation,
			fakeMarkResult: true, fakeCount: 1,
			wantStatus: 200, wantInsert: true, wantInsertStat: types.QuestEventCounted, wantMark: true,
			wantBodyJSON: map[string]any{"recorded": true, "quest_completed": true, "quests_completed": float64(1)},
		},
		{
			name: "valid swap, already-complete quest → 200, quest_completed=false",
			body: `{"public_key":"pk1","quest_type":"swap","tool_call_id":"tc2","tx_hash":"0xabc","data":{"amount_usd":15,"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xrouter"}}`,
			now: now, activation: activation,
			fakeMarkResult: false, fakeCount: 2,
			wantStatus: 200, wantInsert: true, wantInsertStat: types.QuestEventCounted, wantMark: true,
			wantBodyJSON: map[string]any{"recorded": true, "quest_completed": false, "quests_completed": float64(2)},
		},
		{
			name: "swap below min → 200, recorded as rejected, no MarkComplete",
			body: `{"public_key":"pk1","quest_type":"swap","tool_call_id":"tc3","tx_hash":"0xabc","data":{"amount_usd":5,"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xrouter"}}`,
			now: now, activation: activation,
			fakeCount: 0,
			wantStatus: 200, wantInsert: true, wantInsertStat: types.QuestEventRejected, wantMark: false,
			wantBodyJSON: map[string]any{"recorded": true, "quest_completed": false, "quests_completed": float64(0)},
		},
		{
			name: "alert (stub) → 200, rejected with not_yet_implemented",
			body: `{"public_key":"pk1","quest_type":"alert","tool_call_id":"tc4","tx_hash":"0xabc","data":{}}`,
			now: now, activation: activation,
			fakeCount: 0,
			wantStatus: 200, wantInsert: true, wantInsertStat: types.QuestEventRejected, wantMark: false,
		},
		{
			name: "before activation → 403 QUEST_NOT_ACTIVE",
			body: `{"public_key":"pk1","quest_type":"swap","tool_call_id":"tc5","tx_hash":"0xabc","data":{"amount_usd":15,"router_address":"0xr"}}`,
			now:  time.Date(2026, 4, 30, 12, 0, 0, 0, time.UTC), // before activation
			activation: activation,
			wantStatus: 403, wantErrKey: "QUEST_NOT_ACTIVE",
		},
		{
			name: "unknown quest_type → 400 INVALID_QUEST_TYPE",
			body: `{"public_key":"pk1","quest_type":"hodl","tool_call_id":"tc6","tx_hash":"0xabc","data":{}}`,
			now: now, activation: activation,
			wantStatus: 400, wantErrKey: "INVALID_QUEST_TYPE",
		},
		{
			name: "missing tool_call_id → 400 INVALID_PAYLOAD",
			body: `{"public_key":"pk1","quest_type":"swap","tx_hash":"0xabc","data":{}}`,
			now: now, activation: activation,
			wantStatus: 400, wantErrKey: "INVALID_PAYLOAD",
		},
		{
			name: "missing tx_hash → 400 INVALID_PAYLOAD",
			body: `{"public_key":"pk1","quest_type":"swap","tool_call_id":"tc","data":{}}`,
			now: now, activation: activation,
			wantStatus: 400, wantErrKey: "INVALID_PAYLOAD",
		},
		{
			name: "missing public_key → 400 INVALID_PAYLOAD",
			body: `{"quest_type":"swap","tool_call_id":"tc","tx_hash":"0xabc","data":{}}`,
			now: now, activation: activation,
			wantStatus: 400, wantErrKey: "INVALID_PAYLOAD",
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			ins := &recordedInsert{}
			mark := &recordedMark{}

			repo := &fakeQuestRepo{
				insertEvent: func(_ context.Context, p postgres.InsertQuestEventParams) (*types.QuestEvent, error) {
					ins.params = p
					ins.called = true
					return &types.QuestEvent{
						ToolCallID: p.ToolCallID, PublicKey: p.PublicKey, QuestID: p.QuestID,
						TxHash: p.TxHash, Status: p.Status, RejectReason: p.RejectReason,
						Payload: p.Payload, WasInserted: true,
					}, nil
				},
				markComplete: func(_ context.Context, pk string, q types.QuestID) (bool, error) {
					mark.pk = pk
					mark.quest = q
					mark.called = true
					return tc.fakeMarkResult, nil
				},
				countComplete: func(_ context.Context, _ string) (int64, error) {
					return tc.fakeCount, nil
				},
			}
			h := &questEventHandler{
				repo:       repo,
				now:        func() time.Time { return tc.now },
				activation: tc.activation,
				dispatch: quests.DispatchConfig{
					SwapQuestMinUSD:             10,
					BridgeQuestMinUSD:           10,
					DefiActionContractAllowlist: []string{"0xabc"},
				},
				logger: testLogger(t),
			}

			e := echo.New()
			req := httptest.NewRequest(http.MethodPost, "/internal/quests/event", bytes.NewBufferString(tc.body))
			req.Header.Set("Content-Type", "application/json")
			rec := httptest.NewRecorder()
			c := e.NewContext(req, rec)

			err := h.Handle(c)
			if err != nil {
				t.Fatalf("handler returned err: %v", err)
			}
			if rec.Code != tc.wantStatus {
				t.Fatalf("status = %d, want %d (body=%s)", rec.Code, tc.wantStatus, rec.Body.String())
			}
			if tc.wantErrKey != "" {
				var body map[string]string
				if err := json.Unmarshal(rec.Body.Bytes(), &body); err != nil {
					t.Fatalf("unmarshal err body: %v", err)
				}
				if body["error"] != tc.wantErrKey {
					t.Errorf("error = %q, want %q", body["error"], tc.wantErrKey)
				}
				return
			}
			if tc.wantInsert != ins.called {
				t.Errorf("insert called = %v, want %v", ins.called, tc.wantInsert)
			}
			if tc.wantInsert && ins.params.Status != tc.wantInsertStat {
				t.Errorf("insert status = %q, want %q (reject_reason=%q)", ins.params.Status, tc.wantInsertStat, ins.params.RejectReason)
			}
			if tc.wantMark != mark.called {
				t.Errorf("mark called = %v, want %v", mark.called, tc.wantMark)
			}
			if tc.wantBodyJSON != nil {
				var body map[string]any
				if err := json.Unmarshal(rec.Body.Bytes(), &body); err != nil {
					t.Fatalf("unmarshal: %v", err)
				}
				for k, v := range tc.wantBodyJSON {
					if body[k] != v {
						t.Errorf("body[%q] = %v (%T), want %v (%T)", k, body[k], body[k], v, v)
					}
				}
			}
		})
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/api/ -run TestQuestEventHandler -v
```

Expected: compile error — `questEventHandler` undefined.

- [ ] **Step 3: Implement the handler**

Create `internal/api/airdrop_quest_event.go`:

```go
package api

import (
	"context"
	"encoding/json"
	"net/http"
	"time"

	"github.com/labstack/echo/v4"
	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/service/airdrop/quests"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
	"github.com/vultisig/agent-backend/internal/types"
)

// questEventRepo is the slice of QuestEventRepository the handler needs.
// Defined as an interface so the test can pass a fake.
type questEventRepo interface {
	InsertEvent(ctx context.Context, p postgres.InsertQuestEventParams) (*types.QuestEvent, error)
	MarkComplete(ctx context.Context, publicKey string, questID types.QuestID) (bool, error)
	CountUserQuests(ctx context.Context, publicKey string) (int64, error)
}

// questEventRequest is the documented request body for POST /internal/quests/event.
type questEventRequest struct {
	PublicKey  string          `json:"public_key"`
	QuestType  string          `json:"quest_type"`
	ToolCallID string          `json:"tool_call_id"`
	TxHash     string          `json:"tx_hash"`
	Data       json.RawMessage `json:"data"`
}

type questEventResponse struct {
	Recorded         bool  `json:"recorded"`
	QuestCompleted   bool  `json:"quest_completed"`
	QuestsCompleted  int64 `json:"quests_completed"`
}

// questEventHandler handles POST /internal/quests/event.
//
// Behavior (per spec):
//  1. Validate envelope (public_key, quest_type, tool_call_id, tx_hash present;
//     quest_type in enum). 400 on failure.
//  2. Reject if now() < activation. 403.
//  3. Dispatch to the per-quest validator.
//  4. On reject: insert quest_events row with status='rejected' + reason. 200.
//  5. On qualify: insert quest_events row with status='counted', mark the quest
//     complete (idempotent), read the running count. 200 with quest_completed
//     populated from the MarkComplete return.
//
// The insert is idempotent on tool_call_id, so duplicate POSTs of the same
// tool result are no-ops at the DB layer; both responses return 200.
type questEventHandler struct {
	repo       questEventRepo
	now        func() time.Time
	activation time.Time
	dispatch   quests.DispatchConfig
	logger     *logrus.Logger
}

func (h *questEventHandler) Handle(c echo.Context) error {
	var req questEventRequest
	if err := c.Bind(&req); err != nil {
		return c.JSON(http.StatusBadRequest, ErrorResponse{Error: "INVALID_PAYLOAD"})
	}
	if req.PublicKey == "" || req.ToolCallID == "" || req.TxHash == "" {
		return c.JSON(http.StatusBadRequest, ErrorResponse{Error: "INVALID_PAYLOAD"})
	}
	questID := types.QuestID(req.QuestType)
	if !questID.Valid() {
		return c.JSON(http.StatusBadRequest, ErrorResponse{Error: "INVALID_QUEST_TYPE"})
	}
	if h.now().Before(h.activation) {
		return c.JSON(http.StatusForbidden, ErrorResponse{Error: "QUEST_NOT_ACTIVE"})
	}

	payload := []byte(req.Data)
	if len(payload) == 0 {
		payload = []byte("{}")
	}

	result, err := quests.Validate(questID, payload, h.dispatch)
	if err != nil {
		// Programmer bug — the enum check above should make this impossible.
		h.logger.WithError(err).WithField("quest_type", req.QuestType).Error("dispatch failed")
		return c.JSON(http.StatusInternalServerError, ErrorResponse{Error: "INTERNAL_ERROR"})
	}

	status := types.QuestEventCounted
	rejectReason := ""
	if !result.Qualifies {
		status = types.QuestEventRejected
		rejectReason = result.Reason
	}

	event, err := h.repo.InsertEvent(c.Request().Context(), postgres.InsertQuestEventParams{
		ToolCallID:   req.ToolCallID,
		PublicKey:    req.PublicKey,
		QuestID:      questID,
		TxHash:       req.TxHash,
		Status:       status,
		RejectReason: rejectReason,
		Payload:      payload,
	})
	if err != nil {
		h.logger.WithError(err).WithFields(logrus.Fields{
			"tool_call_id": req.ToolCallID,
			"quest_type":   req.QuestType,
		}).Error("insert quest event failed")
		return c.JSON(http.StatusInternalServerError, ErrorResponse{Error: "INTERNAL_ERROR"})
	}

	resp := questEventResponse{Recorded: true}

	if !result.Qualifies {
		// Recorded but doesn't progress the user. Read the count for the response.
		count, err := h.repo.CountUserQuests(c.Request().Context(), req.PublicKey)
		if err != nil {
			h.logger.WithError(err).Warn("count user quests failed (rejected path)")
		}
		resp.QuestsCompleted = count
		h.logQuestEvent(req, "rejected", rejectReason)
		return c.JSON(http.StatusOK, resp)
	}

	// Qualifying path. Mark complete (idempotent) + count.
	wasInserted, err := h.repo.MarkComplete(c.Request().Context(), req.PublicKey, questID)
	if err != nil {
		h.logger.WithError(err).WithFields(logrus.Fields{
			"public_key": shortPublicKey(req.PublicKey),
			"quest_type": req.QuestType,
		}).Error("mark quest complete failed")
		// We've already inserted the event row; the count won't move. Return
		// 200 so the caller doesn't retry — the event is on disk.
		return c.JSON(http.StatusOK, resp)
	}
	resp.QuestCompleted = wasInserted && event.WasInserted

	count, err := h.repo.CountUserQuests(c.Request().Context(), req.PublicKey)
	if err != nil {
		h.logger.WithError(err).Warn("count user quests failed (qualifying path)")
	}
	resp.QuestsCompleted = count

	outcome := "counted"
	if !event.WasInserted {
		outcome = "idempotent_retry"
	}
	h.logQuestEvent(req, outcome, "")
	return c.JSON(http.StatusOK, resp)
}

func (h *questEventHandler) logQuestEvent(req questEventRequest, outcome, reason string) {
	fields := logrus.Fields{
		"action":       "quest_event",
		"public_key":   shortPublicKey(req.PublicKey),
		"quest_type":   req.QuestType,
		"tool_call_id": req.ToolCallID,
		"tx_hash":      req.TxHash,
		"result":       outcome,
	}
	if reason != "" {
		fields["reject_reason"] = reason
	}
	h.logger.WithFields(fields).Info("quest event processed")
}

// shortPublicKey returns the first 8 characters of the key for logging
// (privacy convention from shared-concerns.md).
func shortPublicKey(pk string) string {
	if len(pk) <= 8 {
		return pk
	}
	return pk[:8]
}
```

- [ ] **Step 4: Add the handler-wiring fields to `Server`**

In `internal/api/server.go`, append to the `Server` struct:

```go
	questEventRepo *postgres.QuestEventRepository
	questHandler   *questEventHandler
```

Add a setter at the bottom of the file (matches the existing setter pattern around `SetInternalToken` at line 124):

```go
// SetQuestEventRepo sets the quest event repository for the
// /internal/quests/event handler.
func (s *Server) SetQuestEventRepo(r *postgres.QuestEventRepository) {
	s.questEventRepo = r
}

// InitQuestEventHandler constructs the quest event handler. Call after
// SetQuestEventRepo + SetInternalToken at boot. Activation is the wall-clock
// time after which quest events are accepted; before that, the handler returns
// 403 QUEST_NOT_ACTIVE.
func (s *Server) InitQuestEventHandler(activation time.Time, dispatch quests.DispatchConfig) {
	s.questHandler = &questEventHandler{
		repo:       s.questEventRepo,
		now:        time.Now,
		activation: activation,
		dispatch:   dispatch,
		logger:     s.logger,
	}
}

// QuestEvent is the HTTP entry-point exposed under
// agentGroup.POST("/internal/quests/event", ..., s.InternalTokenMiddleware).
func (s *Server) QuestEvent(c echo.Context) error {
	return s.questHandler.Handle(c)
}
```

The first import block in `server.go` will need `"github.com/vultisig/agent-backend/internal/service/airdrop/quests"` and `"github.com/labstack/echo/v4"` if not already there.

- [ ] **Step 5: Run test to verify it passes**

```bash
go test ./internal/api/ -run TestQuestEventHandler -v
```

Expected: PASS for all subtests.

- [ ] **Step 6: Commit**

```bash
git add internal/api/airdrop_quest_event.go \
         internal/api/airdrop_quest_event_test.go \
         internal/api/server.go
git commit -m "feat(airdrop): add POST /internal/quests/event handler"
```

---

## Task 6: Wire the route into `cmd/server/main.go`

**Why:** The handler is built but not reachable. Add the repo construction, the handler init, and the route registration under the internal-token middleware.

**Files:**
- Modify: `cmd/server/main.go`

- [ ] **Step 1: Construct the repo + init the handler**

In `cmd/server/main.go`, add after the `txProposalRepo` line (`cmd/server/main.go:210`):

```go
	questEventRepo := postgres.NewQuestEventRepository(db.Pool())
```

After the `server.SetInternalToken(...)` call that the shared-infra plan added (likely around the other setters near line 234), wire the quest repo + handler init. If shared-infra didn't add a `SetInternalToken` call yet, add it now too — both are required for the route to function:

```go
	server.SetInternalToken(cfg.Airdrop.InternalAPIKey)
	server.SetQuestEventRepo(questEventRepo)
	server.InitQuestEventHandler(
		cfg.Airdrop.QuestActivationTimestamp,
		quests.FromAirdropConfig(
			cfg.Airdrop.SwapQuestMinUSD,
			cfg.Airdrop.BridgeQuestMinUSD,
			cfg.Airdrop.DefiActionContractAllowlist,
		),
	)
```

Add the import to the import block at the top of `cmd/server/main.go`:

```go
	"github.com/vultisig/agent-backend/internal/service/airdrop/quests"
```

- [ ] **Step 2: Register the route under the internal-token middleware**

After the existing route registrations (around `cmd/server/main.go:340`), add:

```go
	// Internal endpoints — protected by X-Internal-Token shared secret.
	// Called from elsewhere in this same binary (Stage 3 tx-proposal hook,
	// Stage 4 relayer endpoints) and by ops curl.
	internalGroup := e.Group("/internal", server.InternalTokenMiddleware)
	internalGroup.POST("/quests/event", server.QuestEvent)
```

- [ ] **Step 3: Build and smoke-test**

```bash
make build

# Set required env including the new airdrop vars (use defaults for the rest).
INTERNAL_API_KEY=local-secret \
QUEST_ACTIVATION_TIMESTAMP=2025-01-01T00:00:00Z \
TRANSITION_WINDOW_START=2025-01-01T00:00:00Z \
TRANSITION_WINDOW_END=2026-12-31T00:00:00Z \
KMS_RELAYER_KEY_ARN=arn \
EVM_RPC_URL=https://ethereum-rpc.publicnode.com \
AIRDROP_CLAIM_CONTRACT_ADDRESS=0x0000000000000000000000000000000000000000 \
CLAIM_WINDOW_OPEN_AT=2026-12-31T00:00:00Z \
OPS_USERNAME=ops OPS_PASSWORD=ops \
DATABASE_DSN="$DATABASE_DSN" \
./bin/server &
SERVER_PID=$!
sleep 2

# Reject without the header
curl -i -s -X POST http://localhost:8080/internal/quests/event \
  -H "Content-Type: application/json" \
  -d '{"public_key":"pk1","quest_type":"swap","tool_call_id":"tc1","tx_hash":"0xabc","data":{"amount_usd":15,"router_address":"0xrouter"}}' \
  | head -1
# expect: HTTP/1.1 401 Unauthorized

# Accept with the header
curl -i -s -X POST http://localhost:8080/internal/quests/event \
  -H "X-Internal-Token: local-secret" \
  -H "Content-Type: application/json" \
  -d '{"public_key":"pk1","quest_type":"swap","tool_call_id":"tc1","tx_hash":"0xabc","data":{"amount_usd":15,"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xrouter"}}'
# expect: HTTP/1.1 200 OK
# body: {"recorded":true,"quest_completed":true,"quests_completed":1}

# Repeat the same call — idempotent
curl -i -s -X POST http://localhost:8080/internal/quests/event \
  -H "X-Internal-Token: local-secret" \
  -H "Content-Type: application/json" \
  -d '{"public_key":"pk1","quest_type":"swap","tool_call_id":"tc1","tx_hash":"0xabc","data":{"amount_usd":15,"token_in":"USDC","token_out":"ETH","chain_id":1,"router_address":"0xrouter"}}'
# expect: HTTP/1.1 200 OK
# body: {"recorded":true,"quest_completed":false,"quests_completed":1}  -- WasInserted=false on retry

kill $SERVER_PID
```

If the binary fails to build with "undefined: SERVER_PORT" or similar, check that the `/internal/quests/event` route registration matches what `Server.QuestEvent` expects (same name).

- [ ] **Step 4: Commit**

```bash
git add cmd/server/main.go
git commit -m "feat(airdrop): wire /internal/quests/event into cmd/server"
```

---

## Task 7: Hook the `SignTxProposal` handler

**Why:** The mobile app reports a tx as signed-and-broadcast by calling `POST /agent/tx-proposals/:id/sign` with `{tx_hash}` (see `internal/api/tx_proposals.go:95-117`). After the existing `MarkSigned` succeeds, we need to fire `POST /internal/quests/event` for any quest-relevant action. This is the spec's "tool-result hook."

> **2026-04-23 update:** the hook reads `tool_name` and `quest_metadata` directly from the now-marked-signed `agent_tx_proposals` row (populated at proposal-creation time per Task 0's cross-team prerequisite). It does NOT derive a quest from `action_type` heuristics. After `MarkSigned` succeeds:
>
> 1. Re-fetch the proposal (or extend `MarkSigned` to return it) so you have `tool_name` and `quest_metadata` in scope.
> 2. If `tool_name` is empty (older proposals from before the migration, or tools that don't tag themselves), skip the hook — no quest event fires. Log at debug.
> 3. Build the request body: `{ public_key, tool_call_id: proposal.ID, tx_hash: req.TxHash, tool_name: *proposal.ToolName, quest_metadata: proposal.QuestMetadata }`.
> 4. POST in-process via the dispatcher (Step 1 below) with `X-Internal-Token`.
> 5. Errors logged but do NOT propagate to the user's `/sign` response. Best-effort.
>
> Goroutine the POST (per the existing dispatcher pattern) so it doesn't add latency to the user's sign flow.

**Files:**
- Modify: `internal/api/tx_proposals.go` (add the hook after `MarkSigned`)
- Create: `internal/api/airdrop_quest_dispatcher.go` (in-process HTTP poster — small, isolated)
- Create: `internal/api/airdrop_quest_dispatcher_test.go`

- [ ] **Step 1: Write the dispatcher test**

Create `internal/api/airdrop_quest_dispatcher_test.go`:

```go
package api

import (
	"context"
	"encoding/json"
	"io"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"
)

func TestPostQuestEvent_SuccessAndFailureLoggedNotPropagated(t *testing.T) {
	// Spin up a fake /internal/quests/event server and confirm the dispatcher
	// posts the right shape with the right header.
	var got struct {
		gotHeader string
		gotBody   []byte
	}
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		got.gotHeader = r.Header.Get("X-Internal-Token")
		got.gotBody, _ = io.ReadAll(r.Body)
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte(`{"recorded":true,"quest_completed":true,"quests_completed":1}`))
	}))
	defer srv.Close()

	d := &questEventDispatcher{
		baseURL:  srv.URL,
		token:    "test-token",
		client:   &http.Client{Timeout: 2 * time.Second},
		logger:   testLogger(t),
	}
	d.PostEvent(context.Background(), questEventRequest{
		PublicKey: "pk1", QuestType: "swap", ToolCallID: "tc-1",
		TxHash: "0xabc", Data: json.RawMessage(`{"amount_usd":15}`),
	})

	if got.gotHeader != "test-token" {
		t.Errorf("header = %q, want test-token", got.gotHeader)
	}
	var parsed map[string]any
	if err := json.Unmarshal(got.gotBody, &parsed); err != nil {
		t.Fatalf("body unmarshal: %v", err)
	}
	if parsed["public_key"] != "pk1" || parsed["quest_type"] != "swap" || parsed["tool_call_id"] != "tc-1" {
		t.Errorf("forwarded body wrong: %+v", parsed)
	}
}

func TestPostQuestEvent_ErrorIsSwallowed(t *testing.T) {
	// Server returns 500 — the dispatcher must NOT panic or return an error;
	// quest tracking is best-effort and never propagates to the user-facing
	// response of the hooked handler.
	srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		w.WriteHeader(http.StatusInternalServerError)
	}))
	defer srv.Close()

	d := &questEventDispatcher{
		baseURL: srv.URL, token: "x",
		client: &http.Client{Timeout: 2 * time.Second},
		logger: testLogger(t),
	}
	// Should not panic, nothing to assert except "doesn't crash."
	d.PostEvent(context.Background(), questEventRequest{
		PublicKey: "pk1", QuestType: "swap", ToolCallID: "tc-1", TxHash: "0xabc",
	})
}

func TestPostQuestEvent_NoopIfNotQuestRelevant(t *testing.T) {
	// Caller decides whether to invoke the dispatcher; the hook in
	// SignTxProposal calls QuestTypeForTool and skips the dispatch when the
	// action type doesn't map to a quest. Verified end-to-end in the smoke
	// test of Task 8 — this unit test confirms the helper itself.
	if _, ok := questFromActionType("send"); ok {
		t.Errorf("send should not be quest-relevant")
	}
	if got, ok := questFromActionType("swap"); !ok || got != "swap" {
		t.Errorf("swap should map to swap quest, got %q ok=%v", got, ok)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/api/ -run "TestPostQuestEvent" -v
```

Expected: compile error — `questEventDispatcher`, `questFromActionType` undefined.

- [ ] **Step 3: Implement the dispatcher**

Create `internal/api/airdrop_quest_dispatcher.go`:

```go
package api

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"time"

	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/types"
)

// questEventDispatcher fires POST /internal/quests/event in-process from the
// SignTxProposal hook. The "in-process but over HTTP" pattern is per spec:
// keeps the boundary identical to the one ops uses with curl, so future
// out-of-binary callers don't need a second code path. Dispatcher errors are
// logged but never propagated to the user — quest tracking is best-effort.
type questEventDispatcher struct {
	baseURL string       // e.g. "http://127.0.0.1:8080"
	token   string       // shared secret for X-Internal-Token
	client  *http.Client
	logger  *logrus.Logger
}

func newQuestEventDispatcher(serverPort, token string, logger *logrus.Logger) *questEventDispatcher {
	return &questEventDispatcher{
		baseURL: fmt.Sprintf("http://127.0.0.1:%s", serverPort),
		token:   token,
		client:  &http.Client{Timeout: 5 * time.Second},
		logger:  logger,
	}
}

// PostEvent fires the request synchronously. Caller is responsible for running
// it in a goroutine if it shouldn't block the HTTP response. The hook in
// SignTxProposal does run it in a goroutine — see that callsite.
func (d *questEventDispatcher) PostEvent(ctx context.Context, req questEventRequest) {
	body, err := json.Marshal(req)
	if err != nil {
		d.logger.WithError(err).Warn("quest dispatch: marshal failed")
		return
	}
	httpReq, err := http.NewRequestWithContext(ctx, http.MethodPost,
		d.baseURL+"/internal/quests/event", bytes.NewReader(body))
	if err != nil {
		d.logger.WithError(err).Warn("quest dispatch: build request failed")
		return
	}
	httpReq.Header.Set("Content-Type", "application/json")
	httpReq.Header.Set("X-Internal-Token", d.token)

	resp, err := d.client.Do(httpReq)
	if err != nil {
		d.logger.WithError(err).WithFields(logrus.Fields{
			"tool_call_id": req.ToolCallID,
			"quest_type":   req.QuestType,
		}).Warn("quest dispatch: request failed (event NOT recorded)")
		return
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		d.logger.WithFields(logrus.Fields{
			"status":       resp.StatusCode,
			"tool_call_id": req.ToolCallID,
			"quest_type":   req.QuestType,
		}).Warn("quest dispatch: non-200 (event MAY not have been recorded)")
	}
}

// questFromActionType maps an agent_tx_proposals.action_type value to the
// quest type it triggers, if any. Mirrors quests.QuestTypeForTool but keyed
// on the tx-proposal action_type instead of the raw tool name — those two
// vocabularies happen to match for the launch v1 quests, but having the
// indirection here keeps the door open for a future divergence.
//
// Mapping rationale (see also internal/service/airdrop/quests/dispatch.go):
//   - "swap" / "build_swap_tx"  → swap quest
//   - "bridge" / "build_bridge_tx" → bridge quest
//   - "call_contract" / "build_evm_tx" → defi_action quest (validator
//     filters by contract allowlist so a vanilla send doesn't qualify)
func questFromActionType(actionType string) (types.QuestID, bool) {
	switch actionType {
	case "swap", "build_swap_tx":
		return types.QuestSwap, true
	case "bridge", "build_bridge_tx":
		return types.QuestBridge, true
	case "call_contract", "build_evm_tx":
		return types.QuestDefiAction, true
	}
	return "", false
}
```

- [ ] **Step 4: Add the hook to `SignTxProposal`**

Modify `internal/api/tx_proposals.go`. The current implementation ends at line 117 returning the success JSON; add the hook between `MarkSigned` succeeding (line 113) and the response:

```go
// SignTxProposal marks a tx proposal as signed.
func (s *Server) SignTxProposal(c echo.Context) error {
	publicKey := GetPublicKey(c)
	id, err := uuid.Parse(c.Param("id"))
	if err != nil {
		return c.JSON(http.StatusBadRequest, ErrorResponse{Error: "invalid proposal id"})
	}

	var req struct {
		TxHash string `json:"tx_hash"`
	}
	if err := c.Bind(&req); err != nil || req.TxHash == "" {
		return c.JSON(http.StatusBadRequest, ErrorResponse{Error: "tx_hash is required"})
	}
	if !txHashPattern.MatchString(req.TxHash) {
		return c.JSON(http.StatusBadRequest, ErrorResponse{Error: "invalid tx_hash format"})
	}

	if err := s.txProposalRepo.MarkSigned(c.Request().Context(), id, publicKey, req.TxHash); err != nil {
		return c.JSON(http.StatusNotFound, ErrorResponse{Error: "proposal not found or already resolved"})
	}

	// Airdrop Stage 3 hook — fire a quest event if the tx is quest-relevant.
	// Best-effort: errors are logged inside the dispatcher and never
	// propagated to the user. Run async so we don't add latency to the sign
	// response.
	s.dispatchQuestEventForSignedProposal(c.Request().Context(), id, publicKey, req.TxHash)

	return c.JSON(http.StatusOK, map[string]string{"status": "signed"})
}
```

Then add the dispatch helper at the bottom of `tx_proposals.go` (or in a new file `internal/api/tx_proposals_quest_hook.go` if you prefer clean separation — both are fine; pick the one that matches the rest of the package's style):

```go
// dispatchQuestEventForSignedProposal looks up the just-signed proposal,
// translates its (action_type, action_params) into a quest event, and POSTs
// to /internal/quests/event in-process. No-op if the proposal isn't
// quest-relevant or if the airdrop dispatcher hasn't been wired (i.e. running
// outside the airdrop campaign window or in a test that didn't init it).
func (s *Server) dispatchQuestEventForSignedProposal(ctx context.Context, proposalID uuid.UUID, publicKey, txHash string) {
	if s.questDispatcher == nil {
		// Airdrop disabled / not wired. Silently no-op.
		return
	}

	// Re-read the just-signed proposal to get action_type + action_params.
	// We could plumb these through MarkSigned's return, but a re-read keeps
	// the existing repo signature unchanged and the cost is sub-ms (PK lookup).
	prop, err := s.txProposalRepo.GetByID(ctx, proposalID, publicKey)
	if err != nil {
		s.logger.WithError(err).WithField("proposal_id", proposalID.String()).
			Warn("quest hook: cannot re-read just-signed proposal")
		return
	}

	questID, ok := questFromActionType(prop.ActionType)
	if !ok {
		// Not a quest-relevant tool. Nothing to do.
		return
	}

	// Build the per-quest payload from action_params + quote_data. The shape
	// matches the spec's documented `data` schemas (see Task 5 handler).
	payload := buildQuestPayload(questID, prop)

	go s.questDispatcher.PostEvent(context.Background(), questEventRequest{
		PublicKey:  publicKey,
		QuestType:  string(questID),
		ToolCallID: proposalID.String(),
		TxHash:     txHash,
		Data:       payload,
	})
}

// buildQuestPayload extracts the quest-specific fields from the tx proposal's
// action_params + quote_data. Returns an empty `{}` payload if the expected
// fields aren't present — the validator will then reject with a "missing_*"
// reason, which is the right outcome (we've recorded the broadcast for audit
// but it doesn't count toward a quest).
func buildQuestPayload(questID types.QuestID, prop *types.TxProposal) json.RawMessage {
	merged := map[string]any{}
	// action_params is the source of truth for tool inputs.
	if len(prop.ActionParams) > 0 {
		var ap map[string]any
		if json.Unmarshal(prop.ActionParams, &ap) == nil {
			for k, v := range ap {
				merged[k] = v
			}
		}
	}
	// quote_data carries provider-side enrichment (e.g. amount_usd from the
	// swap router quote). Quote fields take precedence over action_params
	// since they reflect the actual quoted execution.
	if len(prop.QuoteData) > 0 {
		var qd map[string]any
		if json.Unmarshal(prop.QuoteData, &qd) == nil {
			for k, v := range qd {
				merged[k] = v
			}
		}
	}

	out := map[string]any{}
	switch questID {
	case types.QuestSwap:
		copyKeys(out, merged, "amount_usd", "token_in", "token_out", "chain_id", "router_address")
	case types.QuestBridge:
		copyKeys(out, merged, "amount_usd", "source_chain_id", "dest_chain_id")
	case types.QuestDefiAction:
		copyKeys(out, merged, "contract_address", "chain_id", "method")
		// agent_tx_proposals.chain is "ethereum"/"arbitrum"/etc.; the validator
		// only cares about contract_address + a chain hint, so plumb chain too.
		if _, ok := out["chain_id"]; !ok && prop.Chain != "" {
			out["chain"] = prop.Chain
		}
	}
	b, _ := json.Marshal(out)
	return b
}

func copyKeys(dst, src map[string]any, keys ...string) {
	for _, k := range keys {
		if v, ok := src[k]; ok {
			dst[k] = v
		}
	}
}
```

The new imports `tx_proposals.go` needs: `"context"`, `"encoding/json"`, plus `"github.com/vultisig/agent-backend/internal/types"`.

- [ ] **Step 5: Add the dispatcher field + setter to `Server`**

In `internal/api/server.go`, append to the `Server` struct:

```go
	questDispatcher *questEventDispatcher
```

And add a setter:

```go
// InitQuestDispatcher wires the in-process HTTP client used by SignTxProposal
// to fire quest events. Pass the server's listening port (cfg.Server.Port) so
// the dispatcher can target localhost:<port>/internal/quests/event.
func (s *Server) InitQuestDispatcher(serverPort string) {
	s.questDispatcher = newQuestEventDispatcher(serverPort, s.internalToken, s.logger)
}
```

- [ ] **Step 6: Wire the dispatcher in `cmd/server/main.go`**

After the existing quest handler init from Task 6 (Step 1), add:

```go
	server.InitQuestDispatcher(cfg.Server.Port)
```

- [ ] **Step 7: Run dispatcher tests**

```bash
go test ./internal/api/ -run "TestPostQuestEvent|TestQuestEventHandler" -v
```

Expected: PASS for all subtests. The whole `internal/api` package should still build and all existing tests pass:

```bash
go test ./internal/api/...
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add internal/api/airdrop_quest_dispatcher.go \
         internal/api/airdrop_quest_dispatcher_test.go \
         internal/api/tx_proposals.go \
         internal/api/server.go \
         cmd/server/main.go
git commit -m "feat(airdrop): hook SignTxProposal to fire quest events"
```

---

## Task 8: End-to-end test + push PR

- [ ] **Step 1: Run the full unit test suite**

```bash
go test ./...
```

Expected: PASS. Investigate any failures before continuing.

- [ ] **Step 2: Run integration tests against a real DB**

```bash
DATABASE_DSN="$DATABASE_DSN" go test ./internal/storage/postgres/ ./internal/api/ ./internal/service/airdrop/...
```

Expected: PASS. This catches the sqlc-generated repo paths against a real `agent_quest_events` + `agent_user_quests`.

- [ ] **Step 3: End-to-end smoke against a running server**

The full Stage 3 user-visible flow: a tx proposal gets signed → the hook fires → the event endpoint records it → the user's quest count moves.

Boot the server with the same env block as Task 6 Step 3, then:

```bash
# Insert a synthetic tx proposal with action_type=swap directly in the DB
# (skipping the agent flow that would normally create it).
PROPOSAL_ID=$(uuidgen)
SCHEDULED_TASK_ID=$(uuidgen)
PUBLIC_KEY="pk-e2e-test"
psql "$DATABASE_DSN" -c "
INSERT INTO agent_tx_proposals (id, scheduled_task_id, public_key, action_type, action_params, quote_data, chain, description, status, expires_at)
VALUES ('$PROPOSAL_ID', '$SCHEDULED_TASK_ID', '$PUBLIC_KEY', 'swap',
        '{\"token_in\":\"USDC\",\"token_out\":\"ETH\",\"chain_id\":1,\"router_address\":\"0xrouter\"}',
        '{\"amount_usd\":42}',
        'ethereum', 'Swap 42 USDC for ETH', 'pending', NOW() + INTERVAL '1 hour');
"

# Mint a JWT for the test public key. In dev with DEV_SKIP_AUTH=true any
# JWT-shaped token with a public_key claim is accepted.
TOKEN="..."  # from local /auth/token call or DEV_SKIP_AUTH

# POST sign — this should fire the quest hook in the background.
curl -s -X POST "http://localhost:8080/agent/tx-proposals/$PROPOSAL_ID/sign" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tx_hash":"0xdeadbeef"}'
# expect: {"status":"signed"}

# Give the goroutine a beat to land
sleep 1

# Check the event was recorded
psql "$DATABASE_DSN" -c "
SELECT tool_call_id, public_key, quest_id, status, reject_reason
FROM agent_quest_events
WHERE public_key = '$PUBLIC_KEY';
"
# expect: 1 row, quest_id=swap, status=counted

# Check the quest is marked complete
psql "$DATABASE_DSN" -c "
SELECT quest_id FROM agent_user_quests WHERE public_key = '$PUBLIC_KEY';
"
# expect: 1 row, quest_id=swap

# Repeat the sign call — should be a 404 from MarkSigned (already resolved)
# AND no new quest event row (the hook never fires because MarkSigned errored).
curl -s -X POST "http://localhost:8080/agent/tx-proposals/$PROPOSAL_ID/sign" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tx_hash":"0xdeadbeef"}'
# expect: {"error":"proposal not found or already resolved"}

# Idempotency at the quest endpoint level: POST the same tool_call_id twice
# directly to /internal/quests/event.
curl -s -X POST http://localhost:8080/internal/quests/event \
  -H "X-Internal-Token: local-secret" \
  -H "Content-Type: application/json" \
  -d "{\"public_key\":\"$PUBLIC_KEY\",\"quest_type\":\"bridge\",\"tool_call_id\":\"manual-tc-1\",\"tx_hash\":\"0xfeed\",\"data\":{\"amount_usd\":50,\"source_chain_id\":1,\"dest_chain_id\":42161}}"
# expect: {"recorded":true,"quest_completed":true,"quests_completed":2}

curl -s -X POST http://localhost:8080/internal/quests/event \
  -H "X-Internal-Token: local-secret" \
  -H "Content-Type: application/json" \
  -d "{\"public_key\":\"$PUBLIC_KEY\",\"quest_type\":\"bridge\",\"tool_call_id\":\"manual-tc-1\",\"tx_hash\":\"0xfeed\",\"data\":{\"amount_usd\":50,\"source_chain_id\":1,\"dest_chain_id\":42161}}"
# expect: {"recorded":true,"quest_completed":false,"quests_completed":2}  -- idempotent retry
```

- [ ] **Step 4: Activation gate check**

```bash
# Restart the server with QUEST_ACTIVATION_TIMESTAMP set in the future
QUEST_ACTIVATION_TIMESTAMP=2099-01-01T00:00:00Z [...other env vars...] ./bin/server &

curl -s -X POST http://localhost:8080/internal/quests/event \
  -H "X-Internal-Token: local-secret" \
  -H "Content-Type: application/json" \
  -d '{"public_key":"pk1","quest_type":"swap","tool_call_id":"tc-pre","tx_hash":"0xabc","data":{"amount_usd":15,"router_address":"0xr"}}'
# expect: {"error":"QUEST_NOT_ACTIVE"} (HTTP 403)
```

- [ ] **Step 5: Run lint**

```bash
make lint
```

Expected: no errors in the new files. Fix any flagged.

- [ ] **Step 6: Push branch**

```bash
git push -u origin feat/airdrop-stage-3-quest-tracking
```

- [ ] **Step 7: Open PR**

```bash
gh pr create --title "feat(airdrop): Stage 3 quest tracking — event endpoint + tx-proposal hook" --body "$(cat <<'EOF'
## Summary

Adds the Vultiverse Airdrop Stage 3 quest-tracking surface:
- \`POST /internal/quests/event\` — single ingestion endpoint behind \`X-Internal-Token\`
- Five validators (swap, bridge, defi_action live; alert + dca stubbed pending Product lock)
- Hook in \`SignTxProposal\` that fires quest events for swap / bridge / call_contract action types

Spec source of truth: \`docs/airdrop-specs/stage-3-quest-tracking.md\`.

## Architecture

- New repo \`postgres.QuestEventRepository\` wraps two sqlc files; idempotent insert keyed on \`tool_call_id\`; \`xmax = 0\` trick on user-quest insert detects first-completion for the spec's \`quest_completed\` response field
- Per-quest validators are pure functions in \`internal/service/airdrop/quests/validators/\` — one file per quest, trivially unit-testable for boundary cases
- \`quests.Validate\` dispatches via a \`switch\` on \`quest_type\`; the same package's \`QuestTypeForTool\` map drives the tx-proposal hook
- The hook is additive — five lines after \`MarkSigned\` succeeds in \`SignTxProposal\`. Errors logged, never propagated to the user (quest tracking is best-effort)
- In-process HTTP for the dispatch (per spec's API-endpoint pattern) — uses \`cfg.Server.Port\` to target \`http://127.0.0.1:<port>/internal/quests/event\` with the \`X-Internal-Token\` header

## Spec coverage
- ✅ Event endpoint with the four documented status codes (200, 400 INVALID_QUEST_TYPE, 400 INVALID_PAYLOAD, 403 QUEST_NOT_ACTIVE)
- ✅ Three live validators with boundary-case tests
- ✅ Two stubs returning \`(false, "not_yet_implemented")\`
- ✅ Idempotency at the DB layer (PK on \`tool_call_id\`); both fresh insert and retry return 200
- ✅ Activation timestamp gate
- ✅ Tool-result hook on \`SignTxProposal\`; failures logged not propagated
- ⏭ Eligibility helper not in this PR — owned by Stage 1's plan

## Hook target

The hook lives in `SignTxProposal` at `internal/api/tx_proposals.go:95-117` (the codebase's existing "tool just signed and broadcast a tx" notification). It reads `tool_name` + `quest_metadata` directly from the proposal row (populated at proposal-creation time by the agent-backend per the 2026-04-23 cross-team contract — see Task 0). The proposal UUID serves as `tool_call_id`. No JSONB sniffing, no action_type heuristics.

## Test plan
- [ ] \`go test ./...\` passes (unit + integration with DATABASE_DSN set)
- [ ] Manual end-to-end: sign a synthetic swap proposal → verify \`agent_quest_events\` + \`agent_user_quests\` rows
- [ ] Idempotent POST to \`/internal/quests/event\` returns 200 both times, only one event row
- [ ] Pre-activation request returns 403 QUEST_NOT_ACTIVE
- [ ] Missing \`X-Internal-Token\` returns 401

## Hard deadline
Quest endpoint live in staging by Day 13 (quest-earning window opens). See Stage 0 decisions table.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Capture and report the PR URL.

---

## Self-review notes

This plan was reviewed against `stage-3-quest-tracking.md` after writing. Items confirmed:

- Single internal endpoint (`POST /internal/quests/event`) with the four documented status codes (200, 400 × 2 reasons, 403). Maps to spec §"Endpoints" + "Errors".
- Three live validators (swap, bridge, defi_action), each as a pure function in its own file. Stubs (alert, dca) return `(false, "not_yet_implemented")` — matches spec §"The 5 quests" guidance that wiring exists for stubs.
- `tool_call_id` is the events PK; insert is idempotent. Both fresh insert and retry return 200 with `recorded: true`. Maps to spec §"Behavior" steps 4–6 + §"Schema".
- `xmax = 0` trick on user-quest insert detects first-completion → drives `quest_completed: true|false` response field. Maps to spec §"Response — 200".
- Activation-timestamp gate enforced before validation. 403 QUEST_NOT_ACTIVE. Matches spec §"Behavior" step 2.
- Per-quest data shapes (swap, bridge, defi_action) match spec exactly. The validator's "missing_*" reasons surface as `reject_reason` for audit.
- The tx-proposal hook is additive — five lines after `MarkSigned` succeeds, errors logged not propagated. Goroutine'd so it doesn't add latency. Matches spec §"Wiring the runtime hook" step 4.
- `quest_dispatch.go` (`internal/service/airdrop/quests/dispatch.go`) holds both the `tool_name → quest_type` map (for the hook) and the `quest_type → validator` switch (for the handler). Spec calls out that adding/renaming a triggering tool is a one-line change here. Confirmed.
- All six configured env vars consumed: `INTERNAL_API_KEY` (middleware + dispatcher), `QUEST_ACTIVATION_TIMESTAMP` (handler gate), `QUEST_THRESHOLD` (NOT consumed in this stage — only Stage 1's eligibility helper reads it), `DEFI_ACTION_CONTRACT_ALLOWLIST` (defi_action validator), `SWAP_QUEST_MIN_USD` (swap validator), `BRIDGE_QUEST_MIN_USD` (bridge validator).

Items intentionally NOT in this plan:

- **`/internal/quests/eligibility` endpoint** — the spec explicitly says no such endpoint exists; Stage 4's relayer reads the same DB tables directly via Stage 1's `eligibility.go` helper.
- **Eligibility helper changes** — Stage 1 owns `internal/service/airdrop/quests/eligibility.go`. Stage 3 only writes the rows it consumes.
- **Prometheus metrics** (`airdrop_quest_event_total`, etc.) — listed in spec §"Observability". Adding them is a small follow-up; the structured logs in `logQuestEvent` give us per-event visibility now and the metric increments would just wrap the same code paths. Defer to a follow-up PR if the team wants dashboards before going live.

### Hook target — resolved

The hook lives in `SignTxProposal` at `internal/api/tx_proposals.go:95-117`. It reads `tool_name` + `quest_metadata` from the now-marked-signed proposal row — both columns are populated at proposal-creation time per Task 0's cross-team contract (MCP team + agent-backend team). The proposal UUID serves as `tool_call_id`. The original "POST /conversations/{id}/tool-result" reference in the spec was aspirational — that endpoint doesn't exist; `SignTxProposal` is the codebase's actual broadcast-notification path.

### Major dependency outside this plan

**Task 0 must land in `main` before any Stage 3 code can be E2E-tested.** That's owned by the MCP team (`vultisig/mcp` adds `quest_metadata` to build_* tool results) and the agent-backend team (migration + scheduler.go change to capture and persist). This plan describes the contract; the work itself is cross-team. Stage 0 row 12 tracks the schema sign-off.
