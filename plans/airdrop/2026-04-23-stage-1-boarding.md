# Vultiverse Airdrop — Stage 1 (Boarding) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Three HTTP endpoints (`POST /airdrop/register`, `GET /airdrop/status`, `GET /airdrop/stats`) that record vault entries, drive the in-app pending screen, and expose a marketing counter — all backed by the `agent_airdrop_registrations` table delivered in shared infra.

**Architecture:** Each endpoint is one Echo handler in `internal/api/airdrop_*.go`. Pure business logic lives in `internal/service/airdrop/registration.go` so unit tests can run without HTTP. Postgres access goes through a new `RegistrationRepository` in `internal/storage/postgres/airdrop_registration.go` wrapping sqlc-generated queries. The shared eligibility helper that Stage 1's status endpoint needs (and Stage 4's claim handler will too) ships here as `internal/service/airdrop/quests/eligibility.go`. No Redis — `/airdrop/stats` is a direct DB query.

**Tech Stack:**
- Go 1.25, Echo v4
- pgx/v5 + sqlc (existing)
- Existing `AuthMiddleware` from `internal/api/middleware.go` for JWT-scoped routes
- Existing `GetPublicKey(c)` helper to pull public key from JWT context

**Spec source of truth:** `docs/airdrop-specs/stage-1-boarding.md`

**Working directory for all tasks:** `/Users/apotheosis/git/vultisig/agent-backend/`

**Branch:** `feat/airdrop-stage-1-boarding`. Branch off `feat/airdrop-shared-infra` once that PR merges to `main`. Commit after every task.

**Hard deadline:** All three endpoints live in staging by Day 8 (see Stage 0 decisions table for the wall-clock date). This plan should be executable in 1–2 days of focused work.

---

## Pre-flight (do once before Task 1)

Confirm the shared-infra PR has merged to `main` and the migrations have applied locally:

```bash
cd /Users/apotheosis/git/vultisig/agent-backend
git checkout main && git pull
psql "$DATABASE_DSN" -c "\d agent_airdrop_registrations"
psql "$DATABASE_DSN" -c "\d agent_raffle_winners"
psql "$DATABASE_DSN" -c "\d agent_user_quests"
psql "$DATABASE_DSN" -c "\d agent_claim_submissions"
```

All four `\d` calls should describe the tables. If any returns "Did not find any relation," run `make migrate-up` first.

```bash
git checkout -b feat/airdrop-stage-1-boarding
```

---

## Task 1: sqlc queries for registrations + winners + eligibility reads

**Why:** All five queries Stage 1 needs in one file. Generated code lands in `internal/storage/postgres/queries/airdrop_registrations.sql.go` automatically when sqlc runs.

**Files:**
- Create: `internal/storage/postgres/sqlc/airdrop_registrations.sql`
- Create: `internal/storage/postgres/sqlc/airdrop_eligibility.sql`
- Generated: `internal/storage/postgres/queries/airdrop_registrations.sql.go` (do not hand-edit)
- Generated: `internal/storage/postgres/queries/airdrop_eligibility.sql.go` (do not hand-edit)

- [ ] **Step 1: Write the registration queries**

Create `internal/storage/postgres/sqlc/airdrop_registrations.sql`:

```sql
-- name: InsertAirdropRegistration :one
-- Idempotent insert. If a row already exists for this public_key, returns the existing row;
-- source and recipient_address are immutable after first registration.
WITH ins AS (
    INSERT INTO agent_airdrop_registrations (public_key, source, recipient_address)
    VALUES ($1, $2, $3)
    ON CONFLICT (public_key) DO NOTHING
    RETURNING *
)
SELECT * FROM ins
UNION ALL
SELECT * FROM agent_airdrop_registrations
WHERE public_key = $1
LIMIT 1;

-- name: GetAirdropRegistration :one
SELECT * FROM agent_airdrop_registrations
WHERE public_key = $1;

-- name: CountAirdropRegistrationsBySource :many
SELECT source, COUNT(*)::bigint AS count
FROM agent_airdrop_registrations
GROUP BY source;

-- name: CountAirdropRegistrations :one
SELECT COUNT(*)::bigint FROM agent_airdrop_registrations;
```

- [ ] **Step 2: Write the eligibility-read queries**

Create `internal/storage/postgres/sqlc/airdrop_eligibility.sql`:

```sql
-- name: AirdropRaffleWinByPublicKey :one
-- Returns the winner row if this public_key won the raffle; pgx.ErrNoRows otherwise.
SELECT * FROM agent_raffle_winners
WHERE public_key = $1;

-- name: AirdropRaffleHasAnyWinners :one
-- Cheap "raffle has been drawn" probe. Used by Stage 1's status state machine
-- to distinguish 'awaiting_draw' from 'lost'.
SELECT EXISTS(SELECT 1 FROM agent_raffle_winners LIMIT 1)::bool AS exists;

-- name: AirdropQuestsCompleted :one
-- Returns the count of distinct quests a user has completed.
SELECT COUNT(*)::bigint AS count
FROM agent_user_quests
WHERE public_key = $1;

-- name: AirdropQuestsCompletedMap :many
-- Returns the per-quest completion state for status display. Stage 1's status
-- handler maps this into the {swap, bridge, defi_action, alert, dca} response shape.
SELECT quest_id FROM agent_user_quests
WHERE public_key = $1;

-- name: AirdropLatestClaim :one
-- Returns the user's most recent claim submission (if any) for the status response's
-- already_claimed + claim_tx_hash fields. Sorted by submitted_at DESC so the latest
-- attempt wins (covers the rebroadcast case).
SELECT * FROM agent_claim_submissions
WHERE public_key = $1
ORDER BY submitted_at DESC
LIMIT 1;
```

- [ ] **Step 3: Run sqlc generation**

The repo's sqlc config lives at the repo root or under `internal/storage/postgres/`. Find it:

```bash
ls sqlc.yaml sqlc.yml internal/storage/postgres/sqlc.yaml internal/storage/postgres/sqlc.yml 2>/dev/null
```

Run sqlc (the existing Makefile may have a target — check first):

```bash
grep -n sqlc Makefile
```

If a `make sqlc` target exists, use it. Otherwise:

```bash
sqlc generate
```

Expected: two new files appear in `internal/storage/postgres/queries/`:
- `airdrop_registrations.sql.go`
- `airdrop_eligibility.sql.go`

If sqlc is not installed locally:

```bash
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
```

- [ ] **Step 4: Verify the generated code compiles**

```bash
go build ./internal/storage/postgres/...
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/storage/postgres/sqlc/airdrop_registrations.sql \
         internal/storage/postgres/sqlc/airdrop_eligibility.sql \
         internal/storage/postgres/queries/airdrop_registrations.sql.go \
         internal/storage/postgres/queries/airdrop_eligibility.sql.go
git commit -m "feat(airdrop): add sqlc queries for registrations + eligibility reads"
```

---

## Task 2: `RegistrationRepository`

**Why:** Wraps the sqlc Queries with domain types, mapping pgtype values to plain Go types — same pattern as `internal/storage/postgres/conversation.go`.

**Files:**
- Create: `internal/storage/postgres/airdrop_registration.go`
- Create: `internal/storage/postgres/airdrop_registration_test.go`

- [ ] **Step 1: Define the domain types**

The Stage 1 spec uses three domain types: `AirdropRegistration`, `RegistrationSource`, and a `SourceCount` for the stats endpoint. Add them to `internal/types/airdrop.go` (new file):

```go
package types

import "time"

type RegistrationSource string

const (
	SourceSeed       RegistrationSource = "seed"
	SourceVaultShare RegistrationSource = "vault_share"
)

func (s RegistrationSource) Valid() bool {
	return s == SourceSeed || s == SourceVaultShare
}

type AirdropRegistration struct {
	PublicKey        string
	Source           RegistrationSource
	RecipientAddress string
	RegisteredAt     time.Time
}

type SourceCount struct {
	Source RegistrationSource
	Count  int64
}
```

- [ ] **Step 2: Write the repo test**

Create `internal/storage/postgres/airdrop_registration_test.go`. This is an integration test against a real Postgres — copy the test-DB setup from whichever existing test file is closest (likely `internal/api/auth_logout_test.go` or a `conversation_test.go` if it exists; if neither does, use a plain `pgxpool.New(ctx, dsn)` against `TEST_DATABASE_DSN`):

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

func newTestPool(t *testing.T) *pgxpool.Pool {
	t.Helper()
	dsn := os.Getenv("TEST_DATABASE_DSN")
	if dsn == "" {
		dsn = os.Getenv("DATABASE_DSN")
	}
	if dsn == "" {
		t.Skip("TEST_DATABASE_DSN or DATABASE_DSN not set; skipping integration test")
	}
	pool, err := pgxpool.New(context.Background(), dsn)
	if err != nil {
		t.Fatalf("pgxpool.New: %v", err)
	}
	t.Cleanup(func() { pool.Close() })
	// Reset state for this test
	_, err = pool.Exec(context.Background(), `
		TRUNCATE agent_airdrop_registrations;
	`)
	if err != nil {
		t.Fatalf("truncate: %v", err)
	}
	return pool
}

func TestRegistrationRepo_InsertIsIdempotent(t *testing.T) {
	pool := newTestPool(t)
	repo := postgres.NewRegistrationRepository(pool)
	ctx := context.Background()

	first, err := repo.Insert(ctx, "pk1", types.SourceSeed, "0xaaa")
	if err != nil {
		t.Fatalf("first insert: %v", err)
	}
	if first.PublicKey != "pk1" || first.Source != types.SourceSeed {
		t.Errorf("unexpected first row: %+v", first)
	}

	// Second insert with different source + address should be a no-op
	second, err := repo.Insert(ctx, "pk1", types.SourceVaultShare, "0xbbb")
	if err != nil {
		t.Fatalf("second insert: %v", err)
	}
	if second.Source != types.SourceSeed {
		t.Errorf("source should be immutable; got %v", second.Source)
	}
	if second.RecipientAddress != "0xaaa" {
		t.Errorf("recipient should be immutable; got %s", second.RecipientAddress)
	}
	if !second.RegisteredAt.Equal(first.RegisteredAt) {
		t.Errorf("registered_at should be unchanged on retry")
	}
}

func TestRegistrationRepo_Get(t *testing.T) {
	pool := newTestPool(t)
	repo := postgres.NewRegistrationRepository(pool)
	ctx := context.Background()

	_, err := repo.Get(ctx, "missing")
	if err != postgres.ErrNotFound {
		t.Errorf("expected ErrNotFound, got %v", err)
	}

	_, _ = repo.Insert(ctx, "pk1", types.SourceSeed, "0xaaa")
	got, err := repo.Get(ctx, "pk1")
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	if got.PublicKey != "pk1" {
		t.Errorf("got %q, want pk1", got.PublicKey)
	}
}

func TestRegistrationRepo_CountBySource(t *testing.T) {
	pool := newTestPool(t)
	repo := postgres.NewRegistrationRepository(pool)
	ctx := context.Background()

	_, _ = repo.Insert(ctx, "pk1", types.SourceSeed, "0x1")
	_, _ = repo.Insert(ctx, "pk2", types.SourceSeed, "0x2")
	_, _ = repo.Insert(ctx, "pk3", types.SourceVaultShare, "0x3")

	counts, total, err := repo.CountBySource(ctx)
	if err != nil {
		t.Fatalf("CountBySource: %v", err)
	}
	if total != 3 {
		t.Errorf("total = %d, want 3", total)
	}
	got := map[types.RegistrationSource]int64{}
	for _, c := range counts {
		got[c.Source] = c.Count
	}
	if got[types.SourceSeed] != 2 || got[types.SourceVaultShare] != 1 {
		t.Errorf("counts = %+v, want {seed:2, vault_share:1}", got)
	}
}
```

- [ ] **Step 3: Run test to verify it fails**

```bash
go test ./internal/storage/postgres/ -run TestRegistrationRepo -v
```

Expected: compile error — `NewRegistrationRepository` undefined.

- [ ] **Step 4: Implement the repository**

Create `internal/storage/postgres/airdrop_registration.go`. Match the pattern in `conversation.go` exactly — same struct shape, same `pool` + `q` fields, same `New...Repository(pool)` constructor.

```go
package postgres

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/storage/postgres/queries"
	"github.com/vultisig/agent-backend/internal/types"
)

type RegistrationRepository struct {
	pool *pgxpool.Pool
	q    *queries.Queries
}

func NewRegistrationRepository(pool *pgxpool.Pool) *RegistrationRepository {
	return &RegistrationRepository{
		pool: pool,
		q:    queries.New(pool),
	}
}

// Insert is idempotent — re-inserting the same public_key returns the existing row;
// source and recipient_address are immutable after first registration.
func (r *RegistrationRepository) Insert(
	ctx context.Context,
	publicKey string,
	source types.RegistrationSource,
	recipientAddress string,
) (*types.AirdropRegistration, error) {
	row, err := r.q.InsertAirdropRegistration(ctx, &queries.InsertAirdropRegistrationParams{
		PublicKey:        publicKey,
		Source:           queries.AirdropRegistrationSource(source),
		RecipientAddress: recipientAddress,
	})
	if err != nil {
		return nil, fmt.Errorf("insert airdrop registration: %w", err)
	}
	return registrationFromDB(row), nil
}

func (r *RegistrationRepository) Get(ctx context.Context, publicKey string) (*types.AirdropRegistration, error) {
	row, err := r.q.GetAirdropRegistration(ctx, publicKey)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return nil, ErrNotFound
		}
		return nil, fmt.Errorf("get airdrop registration: %w", err)
	}
	return registrationFromDB(row), nil
}

// CountBySource returns the per-source counts plus the total. Total is computed
// in Go to save a round-trip — the caller almost always wants both.
func (r *RegistrationRepository) CountBySource(ctx context.Context) ([]types.SourceCount, int64, error) {
	rows, err := r.q.CountAirdropRegistrationsBySource(ctx)
	if err != nil {
		return nil, 0, fmt.Errorf("count by source: %w", err)
	}
	out := make([]types.SourceCount, 0, len(rows))
	var total int64
	for _, row := range rows {
		out = append(out, types.SourceCount{
			Source: types.RegistrationSource(row.Source),
			Count:  row.Count,
		})
		total += row.Count
	}
	return out, total, nil
}

func registrationFromDB(row *queries.AgentAirdropRegistration) *types.AirdropRegistration {
	return &types.AirdropRegistration{
		PublicKey:        row.PublicKey,
		Source:           types.RegistrationSource(row.Source),
		RecipientAddress: row.RecipientAddress,
		RegisteredAt:     pgtimestamptzToTime(row.RegisteredAt),
	}
}
```

The exact name of the sqlc-generated enum type (e.g. `queries.AirdropRegistrationSource`) and the timestamp helper (`pgtimestamptzToTime`) depend on what sqlc generated and what the existing repos use. Open `internal/storage/postgres/queries/airdrop_registrations.sql.go` to confirm the type name; copy the timestamp helper from `conversation.go` if it isn't already exported.

`ErrNotFound` is defined elsewhere in the `postgres` package — see `conversation.go` for the existing definition.

- [ ] **Step 5: Run tests to verify they pass**

```bash
go test ./internal/storage/postgres/ -run TestRegistrationRepo -v
```

Expected: PASS for all three subtests, given `DATABASE_DSN` is set.

- [ ] **Step 6: Commit**

```bash
git add internal/storage/postgres/airdrop_registration.go \
         internal/storage/postgres/airdrop_registration_test.go \
         internal/types/airdrop.go
git commit -m "feat(airdrop): add RegistrationRepository wrapping sqlc queries"
```

---

## Task 3: Eligibility helper (`internal/service/airdrop/quests/eligibility.go`)

**Why:** Stage 1's status handler and (later) Stage 4's claim handler both need the same composite check: `won_raffle AND quests_completed >= threshold AND not_already_claimed`. One pure helper, called inline by both — no HTTP boundary, just a DB read.

**Files:**
- Create: `internal/service/airdrop/quests/eligibility.go`
- Create: `internal/service/airdrop/quests/eligibility_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/service/airdrop/quests/eligibility_test.go`:

```go
package quests_test

import (
	"context"
	"os"
	"testing"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/service/airdrop/quests"
)

func newTestPool(t *testing.T) *pgxpool.Pool {
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
		t.Fatalf("pgxpool: %v", err)
	}
	t.Cleanup(func() { pool.Close() })
	_, err = pool.Exec(context.Background(), `
		TRUNCATE agent_raffle_winners;
		TRUNCATE agent_user_quests;
		TRUNCATE agent_claim_submissions;
	`)
	if err != nil {
		t.Fatalf("truncate: %v", err)
	}
	return pool
}

func seedWinner(t *testing.T, pool *pgxpool.Pool, pk string) {
	t.Helper()
	_, err := pool.Exec(context.Background(),
		`INSERT INTO agent_raffle_winners (public_key, recipient, amount) VALUES ($1, $2, 1500)`,
		pk, "0xaaa")
	if err != nil {
		t.Fatalf("seed winner: %v", err)
	}
}

func seedQuestComplete(t *testing.T, pool *pgxpool.Pool, pk, questID string) {
	t.Helper()
	_, err := pool.Exec(context.Background(),
		`INSERT INTO agent_user_quests (public_key, quest_id) VALUES ($1, $2)`, pk, questID)
	if err != nil {
		t.Fatalf("seed quest: %v", err)
	}
}

func seedClaim(t *testing.T, pool *pgxpool.Pool, pk, status string) {
	t.Helper()
	_, err := pool.Exec(context.Background(), `
		INSERT INTO agent_claim_submissions
		(public_key, nonce, recipient, amount, tx_hash, status, max_fee_gwei, max_priority_fee_gwei, submitted_at)
		VALUES ($1, $2, $3, 1500, $4, $5, 50, 2, NOW())`,
		pk, randomNonce(), "0xaaa", "0xtx", status)
	if err != nil {
		t.Fatalf("seed claim: %v", err)
	}
}

var nonceCounter int64

func randomNonce() int64 {
	nonceCounter++
	return nonceCounter
}

func TestIsClaimEligible_AllPaths(t *testing.T) {
	pool := newTestPool(t)
	ctx := context.Background()

	cases := []struct {
		name           string
		setup          func(t *testing.T, pool *pgxpool.Pool, pk string)
		threshold      int
		wantEligible   bool
		wantReason     string
	}{
		{
			name: "won + 3 quests + not claimed → eligible",
			setup: func(t *testing.T, pool *pgxpool.Pool, pk string) {
				seedWinner(t, pool, pk)
				seedQuestComplete(t, pool, pk, "swap")
				seedQuestComplete(t, pool, pk, "bridge")
				seedQuestComplete(t, pool, pk, "defi_action")
			},
			threshold:    3,
			wantEligible: true,
			wantReason:   "",
		},
		{
			name: "won + 2 quests → not eligible (only_2_quests_complete)",
			setup: func(t *testing.T, pool *pgxpool.Pool, pk string) {
				seedWinner(t, pool, pk)
				seedQuestComplete(t, pool, pk, "swap")
				seedQuestComplete(t, pool, pk, "bridge")
			},
			threshold:    3,
			wantEligible: false,
			wantReason:   "only_2_quests_complete",
		},
		{
			name: "lost raffle + 3 quests → not eligible (not_a_winner)",
			setup: func(t *testing.T, pool *pgxpool.Pool, pk string) {
				seedQuestComplete(t, pool, pk, "swap")
				seedQuestComplete(t, pool, pk, "bridge")
				seedQuestComplete(t, pool, pk, "defi_action")
			},
			threshold:    3,
			wantEligible: false,
			wantReason:   "not_a_winner",
		},
		{
			name: "won + 3 quests + claim submitted → not eligible (already_claimed)",
			setup: func(t *testing.T, pool *pgxpool.Pool, pk string) {
				seedWinner(t, pool, pk)
				seedQuestComplete(t, pool, pk, "swap")
				seedQuestComplete(t, pool, pk, "bridge")
				seedQuestComplete(t, pool, pk, "defi_action")
				seedClaim(t, pool, pk, "submitted")
			},
			threshold:    3,
			wantEligible: false,
			wantReason:   "already_claimed",
		},
		{
			name: "won + 3 quests + claim confirmed → not eligible (already_claimed)",
			setup: func(t *testing.T, pool *pgxpool.Pool, pk string) {
				seedWinner(t, pool, pk)
				seedQuestComplete(t, pool, pk, "swap")
				seedQuestComplete(t, pool, pk, "bridge")
				seedQuestComplete(t, pool, pk, "defi_action")
				seedClaim(t, pool, pk, "confirmed")
			},
			threshold:    3,
			wantEligible: false,
			wantReason:   "already_claimed",
		},
	}

	for i, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			// Reset between cases
			_, err := pool.Exec(ctx, `
				TRUNCATE agent_raffle_winners;
				TRUNCATE agent_user_quests;
				TRUNCATE agent_claim_submissions;
			`)
			if err != nil {
				t.Fatalf("truncate: %v", err)
			}
			pk := "pk-" + string(rune('a'+i))
			tc.setup(t, pool, pk)

			eligible, reason, err := quests.IsClaimEligible(ctx, pool, pk, tc.threshold)
			if err != nil {
				t.Fatalf("IsClaimEligible: %v", err)
			}
			if eligible != tc.wantEligible {
				t.Errorf("eligible = %v, want %v", eligible, tc.wantEligible)
			}
			if reason != tc.wantReason {
				t.Errorf("reason = %q, want %q", reason, tc.wantReason)
			}
		})
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/service/airdrop/quests/ -v
```

Expected: compile error — package or function undefined.

- [ ] **Step 3: Implement the helper**

Create `internal/service/airdrop/quests/eligibility.go`:

```go
package quests

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/storage/postgres/queries"
)

// QuestState captures whether a single quest has been completed by the user,
// for status-display purposes.
type QuestState struct {
	QuestID    string // "swap", "bridge", "defi_action", "alert", "dca"
	Completed  bool
}

// AllQuestIDs is the canonical list, in display order. Stage 1's status response
// returns one entry per quest in this order regardless of completion.
var AllQuestIDs = []string{"swap", "bridge", "defi_action", "alert", "dca"}

// EligibilityResult wraps the rich state Stage 1's status handler needs.
// Stage 4's claim handler ignores everything except Eligible + Reason.
type EligibilityResult struct {
	WonRaffle        bool
	QuestsCompleted  int
	QuestStates      []QuestState
	AlreadyClaimed   bool
	ClaimTxHash      string // empty if !AlreadyClaimed
	Eligible         bool
	Reason           string // "" when Eligible == true
}

// IsClaimEligible computes the eligibility composite for the given public_key
// against the live state of agent_raffle_winners, agent_user_quests, and
// agent_claim_submissions. Threshold is the number of completed quests required
// (typically cfg.Airdrop.QuestThreshold = 3).
//
// Returns (eligible, reason, err). On err, the other return values are zero.
// Reason is one of:
//   - ""                       — eligible
//   - "not_a_winner"           — no row in agent_raffle_winners
//   - "only_N_quests_complete" — fewer than threshold quests done
//   - "already_claimed"        — has a row in agent_claim_submissions with status submitted|confirmed
func IsClaimEligible(ctx context.Context, pool *pgxpool.Pool, publicKey string, threshold int) (bool, string, error) {
	q := queries.New(pool)

	// Win check
	_, err := q.AirdropRaffleWinByPublicKey(ctx, publicKey)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return false, "not_a_winner", nil
		}
		return false, "", fmt.Errorf("raffle win check: %w", err)
	}

	// Already-claimed check (against most recent submission)
	_, err = q.AirdropLatestClaim(ctx, publicKey)
	if err == nil {
		// A row exists; status is either submitted, confirmed, or failed. Only
		// submitted | confirmed count as "already claimed."
		row, _ := q.AirdropLatestClaim(ctx, publicKey)
		if row != nil && (string(row.Status) == "submitted" || string(row.Status) == "confirmed") {
			return false, "already_claimed", nil
		}
	} else if !errors.Is(err, pgx.ErrNoRows) {
		return false, "", fmt.Errorf("claim check: %w", err)
	}

	// Quest count check
	completed, err := q.AirdropQuestsCompleted(ctx, publicKey)
	if err != nil {
		return false, "", fmt.Errorf("quest count: %w", err)
	}
	if completed < int64(threshold) {
		return false, fmt.Sprintf("only_%d_quests_complete", completed), nil
	}

	return true, "", nil
}

// FullStatus returns the rich EligibilityResult Stage 1's status endpoint
// renders. Single round-trip across all three tables; cheaper than calling
// IsClaimEligible plus separate queries for the display fields.
func FullStatus(ctx context.Context, pool *pgxpool.Pool, publicKey string, threshold int) (*EligibilityResult, error) {
	q := queries.New(pool)
	out := &EligibilityResult{}

	// Win
	_, err := q.AirdropRaffleWinByPublicKey(ctx, publicKey)
	switch {
	case errors.Is(err, pgx.ErrNoRows):
		out.WonRaffle = false
	case err != nil:
		return nil, fmt.Errorf("raffle win: %w", err)
	default:
		out.WonRaffle = true
	}

	// Per-quest state
	rows, err := q.AirdropQuestsCompletedMap(ctx, publicKey)
	if err != nil {
		return nil, fmt.Errorf("quest map: %w", err)
	}
	doneSet := map[string]bool{}
	for _, r := range rows {
		doneSet[string(r)] = true
	}
	for _, qID := range AllQuestIDs {
		out.QuestStates = append(out.QuestStates, QuestState{
			QuestID:   qID,
			Completed: doneSet[qID],
		})
	}
	out.QuestsCompleted = len(doneSet)

	// Latest claim
	claim, err := q.AirdropLatestClaim(ctx, publicKey)
	if err != nil && !errors.Is(err, pgx.ErrNoRows) {
		return nil, fmt.Errorf("latest claim: %w", err)
	}
	if err == nil && claim != nil {
		status := string(claim.Status)
		if status == "submitted" || status == "confirmed" {
			out.AlreadyClaimed = true
			out.ClaimTxHash = claim.TxHash
		}
	}

	// Composite
	switch {
	case !out.WonRaffle:
		out.Eligible, out.Reason = false, "not_a_winner"
	case out.AlreadyClaimed:
		out.Eligible, out.Reason = false, "already_claimed"
	case out.QuestsCompleted < threshold:
		out.Eligible, out.Reason = false, fmt.Sprintf("only_%d_quests_complete", out.QuestsCompleted)
	default:
		out.Eligible, out.Reason = true, ""
	}
	return out, nil
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/quests/ -v
```

Expected: PASS for all subtests.

- [ ] **Step 5: Commit**

```bash
git add internal/service/airdrop/quests/
git commit -m "feat(airdrop): add eligibility helper for status + claim"
```

---

## Task 4: `POST /airdrop/register` handler

**Why:** First write endpoint. Validates body, enforces window, idempotent insert.

**Files:**
- Create: `internal/api/airdrop_register.go`
- Create: `internal/api/airdrop_register_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/api/airdrop_register_test.go`. Use `httptest` to exercise the handler directly — minimal Echo setup, no full server bootstrap.

```go
package api

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/vultisig/agent-backend/internal/types"
)

// fakeRegistrationRepo lets us exercise the handler without Postgres.
type fakeRegistrationRepo struct {
	insert func(ctx interface{}, pk string, src types.RegistrationSource, addr string) (*types.AirdropRegistration, error)
}

// (Wire as a small interface in airdrop_register.go — see Step 3.)

func TestRegisterHandler(t *testing.T) {
	now := time.Date(2026, 4, 27, 12, 0, 0, 0, time.UTC)
	openAt := time.Date(2026, 4, 25, 0, 0, 0, 0, time.UTC)
	closeAt := time.Date(2026, 4, 30, 0, 0, 0, 0, time.UTC)

	cases := []struct {
		name       string
		body       string
		windowOpen time.Time
		windowEnd  time.Time
		now        time.Time
		jwtPK      string
		wantStatus int
		wantErrKey string // "" if 200
	}{
		{
			name:       "valid request → 200",
			body:       `{"source":"seed","recipient_address":"0x0000000000000000000000000000000000000001"}`,
			windowOpen: openAt, windowEnd: closeAt, now: now,
			jwtPK:      "pk1",
			wantStatus: 200,
		},
		{
			name:       "missing source → 400 INVALID_SOURCE",
			body:       `{"recipient_address":"0x0000000000000000000000000000000000000001"}`,
			windowOpen: openAt, windowEnd: closeAt, now: now,
			jwtPK:      "pk1", wantStatus: 400, wantErrKey: "INVALID_SOURCE",
		},
		{
			name:       "wrong source enum → 400 INVALID_SOURCE",
			body:       `{"source":"qrcode","recipient_address":"0x0000000000000000000000000000000000000001"}`,
			windowOpen: openAt, windowEnd: closeAt, now: now,
			jwtPK:      "pk1", wantStatus: 400, wantErrKey: "INVALID_SOURCE",
		},
		{
			name:       "missing recipient → 400 INVALID_RECIPIENT_ADDRESS",
			body:       `{"source":"seed"}`,
			windowOpen: openAt, windowEnd: closeAt, now: now,
			jwtPK:      "pk1", wantStatus: 400, wantErrKey: "INVALID_RECIPIENT_ADDRESS",
		},
		{
			name:       "malformed recipient → 400",
			body:       `{"source":"seed","recipient_address":"not-an-address"}`,
			windowOpen: openAt, windowEnd: closeAt, now: now,
			jwtPK:      "pk1", wantStatus: 400, wantErrKey: "INVALID_RECIPIENT_ADDRESS",
		},
		{
			name:       "before window opens → 403 WINDOW_NOT_OPEN",
			body:       `{"source":"seed","recipient_address":"0x0000000000000000000000000000000000000001"}`,
			windowOpen: openAt, windowEnd: closeAt,
			now:        time.Date(2026, 4, 24, 12, 0, 0, 0, time.UTC), // before openAt
			jwtPK:      "pk1", wantStatus: 403, wantErrKey: "WINDOW_NOT_OPEN",
		},
		{
			name:       "after window closes → 403 WINDOW_CLOSED",
			body:       `{"source":"seed","recipient_address":"0x0000000000000000000000000000000000000001"}`,
			windowOpen: openAt, windowEnd: closeAt,
			now:        time.Date(2026, 5, 1, 12, 0, 0, 0, time.UTC), // after closeAt
			jwtPK:      "pk1", wantStatus: 403, wantErrKey: "WINDOW_CLOSED",
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			repo := &fakeRegistrationRepo{
				insert: func(_ interface{}, pk string, src types.RegistrationSource, addr string) (*types.AirdropRegistration, error) {
					return &types.AirdropRegistration{
						PublicKey: pk, Source: src, RecipientAddress: addr,
						RegisteredAt: tc.now,
					}, nil
				},
			}
			h := &registerHandler{
				repo:        repo,
				windowOpen:  tc.windowOpen,
				windowEnd:   tc.windowEnd,
				now:         func() time.Time { return tc.now },
			}

			e := echo.New()
			req := httptest.NewRequest(http.MethodPost, "/airdrop/register", bytes.NewBufferString(tc.body))
			req.Header.Set("Content-Type", "application/json")
			rec := httptest.NewRecorder()
			c := e.NewContext(req, rec)
			c.Set("public_key", tc.jwtPK)

			err := h.Handle(c)
			if err != nil {
				t.Fatalf("handler returned err: %v", err)
			}
			if rec.Code != tc.wantStatus {
				t.Errorf("status = %d, want %d (body=%s)", rec.Code, tc.wantStatus, rec.Body.String())
			}
			if tc.wantErrKey != "" {
				var body map[string]string
				json.Unmarshal(rec.Body.Bytes(), &body)
				if body["error"] != tc.wantErrKey {
					t.Errorf("error = %q, want %q", body["error"], tc.wantErrKey)
				}
			}
		})
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/api/ -run TestRegisterHandler -v
```

Expected: compile error.

- [ ] **Step 3: Implement the handler**

Create `internal/api/airdrop_register.go`:

```go
package api

import (
	"context"
	"encoding/json"
	"net/http"
	"regexp"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/vultisig/agent-backend/internal/types"
)

// registrationRepo is the slice of RegistrationRepository this handler needs.
// Defined as an interface so the test can pass a fake.
type registrationRepo interface {
	Insert(ctx context.Context, publicKey string, source types.RegistrationSource, recipientAddress string) (*types.AirdropRegistration, error)
}

type registerHandler struct {
	repo       registrationRepo
	windowOpen time.Time
	windowEnd  time.Time
	now        func() time.Time // injected for tests; production = time.Now
}

type registerRequest struct {
	Source           string `json:"source"`
	RecipientAddress string `json:"recipient_address"`
}

type registerResponse struct {
	RegisteredAt     time.Time `json:"registered_at"`
	Source           string    `json:"source"`
	RecipientAddress string    `json:"recipient_address"`
}

var evmAddressRE = regexp.MustCompile(`^0x[a-fA-F0-9]{40}$`)

// Handle implements POST /airdrop/register. JWT is required (registered via
// AuthMiddleware on the route); public_key comes from the JWT context, not
// the request body.
func (h *registerHandler) Handle(c echo.Context) error {
	publicKey := GetPublicKey(c)
	if publicKey == "" {
		return c.JSON(http.StatusUnauthorized, ErrorResponse{Error: "missing public key"})
	}

	var req registerRequest
	if err := json.NewDecoder(c.Request().Body).Decode(&req); err != nil {
		return c.JSON(http.StatusBadRequest, ErrorResponse{Error: "INVALID_BODY"})
	}

	source := types.RegistrationSource(req.Source)
	if !source.Valid() {
		return c.JSON(http.StatusBadRequest, ErrorResponse{Error: "INVALID_SOURCE"})
	}
	if !evmAddressRE.MatchString(req.RecipientAddress) {
		return c.JSON(http.StatusBadRequest, ErrorResponse{Error: "INVALID_RECIPIENT_ADDRESS"})
	}

	now := h.now()
	if now.Before(h.windowOpen) {
		return c.JSON(http.StatusForbidden, ErrorResponse{Error: "WINDOW_NOT_OPEN"})
	}
	if !now.Before(h.windowEnd) { // closed at exactly windowEnd
		return c.JSON(http.StatusForbidden, ErrorResponse{Error: "WINDOW_CLOSED"})
	}

	row, err := h.repo.Insert(c.Request().Context(), publicKey, source, req.RecipientAddress)
	if err != nil {
		return c.JSON(http.StatusInternalServerError, ErrorResponse{Error: "INSERT_FAILED"})
	}

	return c.JSON(http.StatusOK, registerResponse{
		RegisteredAt:     row.RegisteredAt,
		Source:           string(row.Source),
		RecipientAddress: row.RecipientAddress,
	})
}
```

In the same file or in `airdrop_register_test.go`, satisfy the fake:

```go
func (f *fakeRegistrationRepo) Insert(ctx context.Context, pk string, src types.RegistrationSource, addr string) (*types.AirdropRegistration, error) {
	return f.insert(ctx, pk, src, addr)
}
```

(Move this method to the test file since it's test-only. Adjust the field signature in the fake to use `context.Context` rather than `interface{}` to match the interface.)

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/api/ -run TestRegisterHandler -v
```

Expected: PASS for all subtests.

- [ ] **Step 5: Commit**

```bash
git add internal/api/airdrop_register.go internal/api/airdrop_register_test.go
git commit -m "feat(airdrop): add POST /airdrop/register handler"
```

---

## Task 5: `GET /airdrop/status` handler

**Why:** Drives the in-app pending screen. Single source of truth for `claim_eligible`. Always returns the full shape.

**Files:**
- Create: `internal/api/airdrop_status.go`
- Create: `internal/api/airdrop_status_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/api/airdrop_status_test.go`:

```go
package api

import (
	"context"
	"encoding/json"
	"errors"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/vultisig/agent-backend/internal/service/airdrop/quests"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
	"github.com/vultisig/agent-backend/internal/types"
)

type fakeStatusRepo struct {
	get             func(ctx context.Context, pk string) (*types.AirdropRegistration, error)
	hasAnyWinners   func(ctx context.Context) (bool, error)
	fullStatus      func(ctx context.Context, pk string, threshold int) (*quests.EligibilityResult, error)
}

func (f *fakeStatusRepo) Get(ctx context.Context, pk string) (*types.AirdropRegistration, error) {
	return f.get(ctx, pk)
}
func (f *fakeStatusRepo) HasAnyWinners(ctx context.Context) (bool, error) {
	return f.hasAnyWinners(ctx)
}
func (f *fakeStatusRepo) FullStatus(ctx context.Context, pk string, threshold int) (*quests.EligibilityResult, error) {
	return f.fullStatus(ctx, pk, threshold)
}

func TestStatusHandler_NotRegistered(t *testing.T) {
	repo := &fakeStatusRepo{
		get: func(_ context.Context, _ string) (*types.AirdropRegistration, error) {
			return nil, postgres.ErrNotFound
		},
	}
	h := &statusHandler{repo: repo, threshold: 3, now: func() time.Time {
		return time.Date(2026, 4, 27, 12, 0, 0, 0, time.UTC)
	}}

	e := echo.New()
	req := httptest.NewRequest(http.MethodGet, "/airdrop/status", nil)
	rec := httptest.NewRecorder()
	c := e.NewContext(req, rec)
	c.Set("public_key", "pk1")

	if err := h.Handle(c); err != nil {
		t.Fatalf("handler: %v", err)
	}
	var body map[string]interface{}
	json.Unmarshal(rec.Body.Bytes(), &body)
	if body["registered"] != false {
		t.Errorf("registered = %v, want false", body["registered"])
	}
	if len(body) != 1 {
		t.Errorf("not-registered response should have 1 field, got %d: %+v", len(body), body)
	}
}

func TestStatusHandler_StateMatrix(t *testing.T) {
	regAt := time.Date(2026, 4, 26, 0, 0, 0, 0, time.UTC)
	closeAt := time.Date(2026, 4, 30, 0, 0, 0, 0, time.UTC)

	cases := []struct {
		name           string
		now            time.Time
		hasAnyWinners  bool
		eligibility    *quests.EligibilityResult
		wantState      string
		wantClaimable  bool
		wantClaimed    bool
		wantTxHash     string
	}{
		{
			name: "during window → pending",
			now:  closeAt.Add(-time.Hour), hasAnyWinners: false,
			eligibility: &quests.EligibilityResult{},
			wantState:   "pending",
		},
		{
			name: "after window, no winners loaded → awaiting_draw",
			now:  closeAt.Add(time.Hour), hasAnyWinners: false,
			eligibility: &quests.EligibilityResult{},
			wantState:   "awaiting_draw",
		},
		{
			name: "after window, winners loaded, lost → lost",
			now:  closeAt.Add(time.Hour), hasAnyWinners: true,
			eligibility: &quests.EligibilityResult{WonRaffle: false},
			wantState:   "lost",
		},
		{
			name: "won, 2 quests → won, claim_eligible=false",
			now:  closeAt.Add(time.Hour), hasAnyWinners: true,
			eligibility: &quests.EligibilityResult{
				WonRaffle: true, QuestsCompleted: 2, Eligible: false, Reason: "only_2_quests_complete",
			},
			wantState: "won", wantClaimable: false,
		},
		{
			name: "won, 3 quests, not claimed → won, claim_eligible=true",
			now:  closeAt.Add(time.Hour), hasAnyWinners: true,
			eligibility: &quests.EligibilityResult{
				WonRaffle: true, QuestsCompleted: 3, Eligible: true,
			},
			wantState: "won", wantClaimable: true,
		},
		{
			name: "won, 3 quests, claimed → won, already_claimed=true with tx",
			now:  closeAt.Add(time.Hour), hasAnyWinners: true,
			eligibility: &quests.EligibilityResult{
				WonRaffle: true, QuestsCompleted: 3, AlreadyClaimed: true, ClaimTxHash: "0xdeadbeef",
				Eligible: false, Reason: "already_claimed",
			},
			wantState: "won", wantClaimed: true, wantTxHash: "0xdeadbeef",
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			repo := &fakeStatusRepo{
				get: func(_ context.Context, _ string) (*types.AirdropRegistration, error) {
					return &types.AirdropRegistration{
						PublicKey: "pk1", Source: types.SourceSeed,
						RecipientAddress: "0xaaa", RegisteredAt: regAt,
					}, nil
				},
				hasAnyWinners: func(_ context.Context) (bool, error) { return tc.hasAnyWinners, nil },
				fullStatus: func(_ context.Context, _ string, _ int) (*quests.EligibilityResult, error) {
					return tc.eligibility, nil
				},
			}
			h := &statusHandler{
				repo: repo, threshold: 3, windowEnd: closeAt,
				now: func() time.Time { return tc.now },
			}

			e := echo.New()
			req := httptest.NewRequest(http.MethodGet, "/airdrop/status", nil)
			rec := httptest.NewRecorder()
			c := e.NewContext(req, rec)
			c.Set("public_key", "pk1")

			if err := h.Handle(c); err != nil {
				t.Fatalf("handler: %v", err)
			}
			var body map[string]interface{}
			if err := json.Unmarshal(rec.Body.Bytes(), &body); err != nil {
				t.Fatalf("unmarshal: %v", err)
			}
			if body["raffle_state"] != tc.wantState {
				t.Errorf("raffle_state = %v, want %s", body["raffle_state"], tc.wantState)
			}
			if body["claim_eligible"] != tc.wantClaimable {
				t.Errorf("claim_eligible = %v, want %v", body["claim_eligible"], tc.wantClaimable)
			}
			if body["already_claimed"] != tc.wantClaimed {
				t.Errorf("already_claimed = %v, want %v", body["already_claimed"], tc.wantClaimed)
			}
			if tc.wantTxHash != "" && body["claim_tx_hash"] != tc.wantTxHash {
				t.Errorf("claim_tx_hash = %v, want %s", body["claim_tx_hash"], tc.wantTxHash)
			}
			// All quest keys always present
			questsMap, ok := body["quests"].(map[string]interface{})
			if !ok {
				t.Fatalf("quests should be a map; got %T", body["quests"])
			}
			for _, q := range []string{"swap", "bridge", "defi_action", "alert", "dca"} {
				if _, present := questsMap[q]; !present {
					t.Errorf("quests map missing key %q", q)
				}
			}
		})
	}

	// Defensive: avoid unused-import errors in some builds
	_ = errors.New
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/api/ -run TestStatusHandler -v
```

Expected: compile error.

- [ ] **Step 3: Implement the handler**

Create `internal/api/airdrop_status.go`:

```go
package api

import (
	"context"
	"errors"
	"net/http"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/vultisig/agent-backend/internal/service/airdrop/quests"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
	"github.com/vultisig/agent-backend/internal/types"
)

type statusRepo interface {
	Get(ctx context.Context, publicKey string) (*types.AirdropRegistration, error)
	HasAnyWinners(ctx context.Context) (bool, error)
	FullStatus(ctx context.Context, publicKey string, threshold int) (*quests.EligibilityResult, error)
}

type statusHandler struct {
	repo      statusRepo
	threshold int
	windowEnd time.Time
	now       func() time.Time
}

type statusResponseRegistered struct {
	Registered       bool              `json:"registered"`
	RegisteredAt     time.Time         `json:"registered_at"`
	Source           string            `json:"source"`
	RecipientAddress string            `json:"recipient_address"`
	RaffleState      string            `json:"raffle_state"`
	Quests           map[string]string `json:"quests"`
	QuestsCompleted  int               `json:"quests_completed"`
	ClaimEligible    bool              `json:"claim_eligible"`
	AlreadyClaimed   bool              `json:"already_claimed"`
	ClaimTxHash      *string           `json:"claim_tx_hash"` // null if no claim yet
}

type statusResponseNotRegistered struct {
	Registered bool `json:"registered"`
}

func (h *statusHandler) Handle(c echo.Context) error {
	publicKey := GetPublicKey(c)
	if publicKey == "" {
		return c.JSON(http.StatusUnauthorized, ErrorResponse{Error: "missing public key"})
	}
	ctx := c.Request().Context()

	reg, err := h.repo.Get(ctx, publicKey)
	if err != nil {
		if errors.Is(err, postgres.ErrNotFound) {
			return c.JSON(http.StatusOK, statusResponseNotRegistered{Registered: false})
		}
		return c.JSON(http.StatusInternalServerError, ErrorResponse{Error: "STATUS_LOOKUP_FAILED"})
	}

	hasWinners, err := h.repo.HasAnyWinners(ctx)
	if err != nil {
		return c.JSON(http.StatusInternalServerError, ErrorResponse{Error: "STATUS_LOOKUP_FAILED"})
	}

	full, err := h.repo.FullStatus(ctx, publicKey, h.threshold)
	if err != nil {
		return c.JSON(http.StatusInternalServerError, ErrorResponse{Error: "STATUS_LOOKUP_FAILED"})
	}

	// State machine
	state := computeRaffleState(h.now(), h.windowEnd, hasWinners, full.WonRaffle)

	// Build the always-full quests map
	questsMap := make(map[string]string, len(quests.AllQuestIDs))
	for _, qs := range full.QuestStates {
		if qs.Completed {
			questsMap[qs.QuestID] = "completed"
		} else {
			questsMap[qs.QuestID] = "pending"
		}
	}
	// Defensive: ensure every quest key is present (in case AllQuestIDs grows later
	// and FullStatus hasn't been re-run)
	for _, q := range quests.AllQuestIDs {
		if _, ok := questsMap[q]; !ok {
			questsMap[q] = "pending"
		}
	}

	var txHashPtr *string
	if full.ClaimTxHash != "" {
		v := full.ClaimTxHash
		txHashPtr = &v
	}

	return c.JSON(http.StatusOK, statusResponseRegistered{
		Registered:       true,
		RegisteredAt:     reg.RegisteredAt,
		Source:           string(reg.Source),
		RecipientAddress: reg.RecipientAddress,
		RaffleState:      state,
		Quests:           questsMap,
		QuestsCompleted:  full.QuestsCompleted,
		ClaimEligible:    full.Eligible,
		AlreadyClaimed:   full.AlreadyClaimed,
		ClaimTxHash:      txHashPtr,
	})
}

// computeRaffleState implements the spec's state machine.
//   - pending       — now < windowEnd
//   - awaiting_draw — now >= windowEnd AND no winners loaded yet
//   - won           — winners loaded AND this user is among them
//   - lost          — winners loaded AND this user is not among them
func computeRaffleState(now, windowEnd time.Time, hasAnyWinners, won bool) string {
	if now.Before(windowEnd) {
		return "pending"
	}
	if !hasAnyWinners {
		return "awaiting_draw"
	}
	if won {
		return "won"
	}
	return "lost"
}
```

The repo struct needs a `HasAnyWinners` method and a `FullStatus` method. The `FullStatus` is a thin wrapper around `quests.FullStatus`. Add to `internal/storage/postgres/airdrop_registration.go`:

```go
func (r *RegistrationRepository) HasAnyWinners(ctx context.Context) (bool, error) {
	exists, err := r.q.AirdropRaffleHasAnyWinners(ctx)
	if err != nil {
		return false, fmt.Errorf("has any winners: %w", err)
	}
	return exists, nil
}

func (r *RegistrationRepository) FullStatus(
	ctx context.Context,
	publicKey string,
	threshold int,
) (*quests.EligibilityResult, error) {
	return quests.FullStatus(ctx, r.pool, publicKey, threshold)
}
```

Add the new import:

```go
import "github.com/vultisig/agent-backend/internal/service/airdrop/quests"
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/api/ -run TestStatusHandler -v
```

Expected: PASS for both `TestStatusHandler_NotRegistered` and all subtests of `TestStatusHandler_StateMatrix`.

- [ ] **Step 5: Commit**

```bash
git add internal/api/airdrop_status.go internal/api/airdrop_status_test.go internal/storage/postgres/airdrop_registration.go
git commit -m "feat(airdrop): add GET /airdrop/status handler"
```

---

## Task 6: `GET /airdrop/stats` handler

**Why:** Marketing's live counter. No auth, no Redis, just a Postgres `GROUP BY`.

**Files:**
- Create: `internal/api/airdrop_stats.go`
- Create: `internal/api/airdrop_stats_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/api/airdrop_stats_test.go`:

```go
package api

import (
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/vultisig/agent-backend/internal/types"
)

type fakeStatsRepo struct {
	countBySource func(ctx context.Context) ([]types.SourceCount, int64, error)
}

func (f *fakeStatsRepo) CountBySource(ctx context.Context) ([]types.SourceCount, int64, error) {
	return f.countBySource(ctx)
}

func TestStatsHandler(t *testing.T) {
	openAt := time.Date(2026, 4, 25, 0, 0, 0, 0, time.UTC)
	closeAt := time.Date(2026, 4, 30, 0, 0, 0, 0, time.UTC)

	cases := []struct {
		name        string
		now         time.Time
		wantState   string
		wantTotal   int64
	}{
		{
			name: "before window → upcoming",
			now:  openAt.Add(-time.Hour), wantState: "upcoming", wantTotal: 0,
		},
		{
			name: "during window → open",
			now:  openAt.Add(time.Hour), wantState: "open", wantTotal: 0,
		},
		{
			name: "after window → closed",
			now:  closeAt.Add(time.Hour), wantState: "closed", wantTotal: 0,
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			repo := &fakeStatsRepo{
				countBySource: func(_ context.Context) ([]types.SourceCount, int64, error) {
					return []types.SourceCount{
						{Source: types.SourceSeed, Count: 800},
						{Source: types.SourceVaultShare, Count: 434},
					}, 1234, nil
				},
			}
			h := &statsHandler{
				repo: repo, windowOpen: openAt, windowEnd: closeAt,
				now: func() time.Time { return tc.now },
			}

			e := echo.New()
			req := httptest.NewRequest(http.MethodGet, "/airdrop/stats", nil)
			rec := httptest.NewRecorder()
			c := e.NewContext(req, rec)

			if err := h.Handle(c); err != nil {
				t.Fatalf("handler: %v", err)
			}
			var body map[string]interface{}
			json.Unmarshal(rec.Body.Bytes(), &body)
			if body["window_state"] != tc.wantState {
				t.Errorf("window_state = %v, want %s", body["window_state"], tc.wantState)
			}
			if body["total_registrations"].(float64) != 1234 {
				t.Errorf("total = %v, want 1234", body["total_registrations"])
			}
			bySource, _ := body["by_source"].(map[string]interface{})
			if bySource["seed"].(float64) != 800 || bySource["vault_share"].(float64) != 434 {
				t.Errorf("by_source = %+v", bySource)
			}
		})
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/api/ -run TestStatsHandler -v
```

Expected: compile error.

- [ ] **Step 3: Implement the handler**

Create `internal/api/airdrop_stats.go`:

```go
package api

import (
	"context"
	"net/http"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/vultisig/agent-backend/internal/types"
)

type statsRepo interface {
	CountBySource(ctx context.Context) ([]types.SourceCount, int64, error)
}

type statsHandler struct {
	repo       statsRepo
	windowOpen time.Time
	windowEnd  time.Time
	now        func() time.Time
}

type statsResponse struct {
	TotalRegistrations int64            `json:"total_registrations"`
	BySource           map[string]int64 `json:"by_source"`
	WindowState        string           `json:"window_state"`
	WindowOpensAt      time.Time        `json:"window_opens_at"`
	WindowClosesAt     time.Time        `json:"window_closes_at"`
}

func (h *statsHandler) Handle(c echo.Context) error {
	counts, total, err := h.repo.CountBySource(c.Request().Context())
	if err != nil {
		return c.JSON(http.StatusInternalServerError, ErrorResponse{Error: "STATS_LOOKUP_FAILED"})
	}

	bySource := map[string]int64{
		string(types.SourceSeed):       0, // ensure both keys always present
		string(types.SourceVaultShare): 0,
	}
	for _, sc := range counts {
		bySource[string(sc.Source)] = sc.Count
	}

	now := h.now()
	state := "open"
	switch {
	case now.Before(h.windowOpen):
		state = "upcoming"
	case !now.Before(h.windowEnd):
		state = "closed"
	}

	return c.JSON(http.StatusOK, statsResponse{
		TotalRegistrations: total,
		BySource:           bySource,
		WindowState:        state,
		WindowOpensAt:      h.windowOpen,
		WindowClosesAt:     h.windowEnd,
	})
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/api/ -run TestStatsHandler -v
```

Expected: PASS for all subtests.

- [ ] **Step 5: Commit**

```bash
git add internal/api/airdrop_stats.go internal/api/airdrop_stats_test.go
git commit -m "feat(airdrop): add GET /airdrop/stats handler (no auth, direct DB)"
```

---

## Task 7: Wire routes in `cmd/server/main.go` and `internal/api/server.go`

**Why:** Until this task the handlers exist but aren't reachable. This task plugs them into the running service.

**Files:**
- Modify: `internal/api/server.go` — add `RegistrationRepository` field, plus three handler instances.
- Modify: `cmd/server/main.go` — instantiate the repo + handlers + register routes.

- [ ] **Step 1: Add the repo + handler instances to the Server struct**

Find the `Server` struct in `internal/api/server.go` (search: `grep -n "type Server struct" internal/api/*.go`). Add fields:

```go
type Server struct {
	// ... existing fields ...
	regRepo         *postgres.RegistrationRepository
	registerHandler *registerHandler
	statusHandler   *statusHandler
	statsHandler    *statsHandler
}
```

In whichever constructor takes `cfg *config.Config` (likely `NewServer`), add a `repo *postgres.RegistrationRepository` parameter and wire the three handlers from it:

```go
func NewServer(cfg *config.Config, /* existing args */, regRepo *postgres.RegistrationRepository) *Server {
	s := &Server{
		// ... existing field initialisers ...
		regRepo: regRepo,
	}

	now := time.Now
	s.registerHandler = &registerHandler{
		repo:       regRepo,
		windowOpen: cfg.Airdrop.TransitionWindowStart,
		windowEnd:  cfg.Airdrop.TransitionWindowEnd,
		now:        now,
	}
	s.statusHandler = &statusHandler{
		repo:      regRepo,
		threshold: cfg.Airdrop.QuestThreshold,
		windowEnd: cfg.Airdrop.TransitionWindowEnd,
		now:       now,
	}
	s.statsHandler = &statsHandler{
		repo:       regRepo,
		windowOpen: cfg.Airdrop.TransitionWindowStart,
		windowEnd:  cfg.Airdrop.TransitionWindowEnd,
		now:        now,
	}

	return s
}
```

- [ ] **Step 2: Register the routes**

Find where existing routes are registered. Per the survey, this looks like:

```go
agentGroup := e.Group("/agent", server.AuthMiddleware)
agentGroup.POST("/conversations/:id/messages", server.SendMessage, ...)
```

Add after that block:

```go
// Airdrop endpoints — JWT-scoped + public stats
airdropGroup := e.Group("/airdrop", server.AuthMiddleware)
airdropGroup.POST("/register", server.registerHandler.Handle)
airdropGroup.GET("/status", server.statusHandler.Handle)

// Stats is intentionally public — marketing polls it for the live counter
e.GET("/airdrop/stats", server.statsHandler.Handle)
```

- [ ] **Step 3: Wire the repository in `cmd/server/main.go`**

Find where the existing repos are instantiated (search: `grep -n "postgres.New" cmd/server/main.go`). Add:

```go
regRepo := postgres.NewRegistrationRepository(db.Pool())
```

Then update the `api.NewServer(...)` call to pass it:

```go
server := api.NewServer(cfg, /* existing args */, regRepo)
```

- [ ] **Step 4: Build and run locally**

```bash
make build
DATABASE_DSN="postgres://..." \
  TRANSITION_WINDOW_START="2026-04-25T00:00:00Z" \
  TRANSITION_WINDOW_END="2026-04-30T00:00:00Z" \
  INTERNAL_API_KEY="local" \
  QUEST_ACTIVATION_TIMESTAMP="2026-05-01T00:00:00Z" \
  KMS_RELAYER_KEY_ARN="arn" \
  EVM_RPC_URL="https://ethereum-rpc.publicnode.com" \
  AIRDROP_CLAIM_CONTRACT_ADDRESS="0x0000000000000000000000000000000000000000" \
  CLAIM_WINDOW_OPEN_AT="2026-05-15T00:00:00Z" \
  OPS_USERNAME="ops" \
  OPS_PASSWORD="ops" \
  ./bin/server
```

(All other env vars use envconfig defaults.)

In a second terminal, smoke test the public endpoint:

```bash
curl -s http://localhost:8084/airdrop/stats | jq .
```

Expected response shape:

```json
{
  "total_registrations": 0,
  "by_source": {"seed": 0, "vault_share": 0},
  "window_state": "open",
  "window_opens_at": "2026-04-25T00:00:00Z",
  "window_closes_at": "2026-04-30T00:00:00Z"
}
```

(Adjust the host/port to match where the local server binds.)

- [ ] **Step 5: Commit**

```bash
git add internal/api/server.go cmd/server/main.go
git commit -m "feat(airdrop): wire register/status/stats routes into server"
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
DATABASE_DSN="postgres://..." go test -tags=integration ./internal/storage/postgres/ ./internal/service/airdrop/quests/
```

(If the repo doesn't use `-tags=integration`, just run `go test ./...` with `DATABASE_DSN` set.)

Expected: PASS for the registration repo tests and the eligibility helper tests.

- [ ] **Step 3: Manual smoke against running server**

Start the server (Step 4 of Task 7). In another terminal:

```bash
# Mint a JWT — the existing /auth/token flow needs a vault signature; in dev with
# DEV_SKIP_AUTH=true any JWT-shaped token with a public_key claim works.
# Adjust to whatever the agent-backend repo's local-dev pattern is — see CLAUDE.md.
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."  # replace

curl -s -X POST http://localhost:8084/airdrop/register \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"source":"seed","recipient_address":"0x0000000000000000000000000000000000000001"}' | jq .

curl -s http://localhost:8084/airdrop/status \
  -H "Authorization: Bearer $TOKEN" | jq .

curl -s http://localhost:8084/airdrop/stats | jq .
```

Expected:
- `register` returns 200 with `registered_at` and the recipient address you sent.
- `status` returns 200 with `registered: true`, `raffle_state: "pending"`, `quests` map with all five keys = "pending", `claim_eligible: false`.
- `stats` returns 200 with `total_registrations: 1` (one registration just inserted).

Re-run `register` with a different source — should return 200 but with the original source unchanged (idempotency check).

- [ ] **Step 4: Run lint**

```bash
make lint
```

Expected: no errors.

- [ ] **Step 5: Push branch**

```bash
git push -u origin feat/airdrop-stage-1-boarding
```

- [ ] **Step 6: Open PR**

```bash
gh pr create --title "feat(airdrop): Stage 1 boarding endpoints (register, status, stats)" --body "$(cat <<'EOF'
## Summary

Adds the three Stage 1 endpoints from the Vultiverse Airdrop spec:
- \`POST /airdrop/register\` — idempotent vault registration (JWT-scoped)
- \`GET /airdrop/status\` — drives the in-app pending screen, always returns the full shape with safe defaults (JWT-scoped)
- \`GET /airdrop/stats\` — public marketing counter (no auth, direct Postgres query)

Plus the shared eligibility helper that Stage 4 will also consume.

Spec source of truth: \`docs/airdrop-specs/stage-1-boarding.md\`.

## Architecture
- Pure handler logic in \`internal/api/airdrop_*.go\` — small interfaces in front of the repo for testability
- Repository in \`internal/storage/postgres/airdrop_registration.go\` wrapping sqlc-generated queries
- Eligibility helper in \`internal/service/airdrop/quests/eligibility.go\` — shared with Stage 4
- No Redis (\`/airdrop/stats\` is a sub-ms Postgres query against a small table)
- Status response always returns the full shape; \`quests\`, \`claim_eligible\`, \`already_claimed\`, \`claim_tx_hash\` all populated with safe defaults so the mobile app integrates against one stable shape forever

## Spec coverage
- ✅ \`POST /airdrop/register\` with all error cases (\`INVALID_SOURCE\`, \`INVALID_RECIPIENT_ADDRESS\`, \`WINDOW_NOT_OPEN\`, \`WINDOW_CLOSED\`)
- ✅ Idempotency: re-registering returns existing row, source/recipient immutable
- ✅ \`GET /airdrop/status\` state machine: pending / awaiting_draw / won / lost
- ✅ \`claim_eligible\` composite computed server-side
- ✅ \`claim_tx_hash\` returned when already_claimed
- ✅ \`GET /airdrop/stats\` with by_source breakdown + window_state
- ✅ Eligibility helper covers all five (won/lost × quests/no quests × claimed/not) combinations

## Test plan
- [ ] \`go test ./...\` passes (unit + integration with DATABASE_DSN set)
- [ ] Manual smoke test against running server (register → status → stats with curl)
- [ ] Confirm response shape matches \`stage-1-boarding.md\` for each of the four \`raffle_state\` values
- [ ] Idempotent register: same public_key + different source returns the original source

## Hard deadline
All three endpoints live in staging by Day 8 (boarding window opens). See Stage 0 decisions table for the wall-clock date.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Capture and report the PR URL.

---

## Self-review notes

This plan was reviewed against `stage-1-boarding.md` after writing. Items confirmed:
- Three endpoints, all with documented request/response shapes and status codes.
- Status response always returns the full shape (per the cleanup-pass change committed in PR #67).
- `claim_tx_hash` included (per the cleanup-pass change).
- Stats endpoint uses direct Postgres, no Redis (per the cleanup-pass change).
- Eligibility helper ships here (Stage 1 is the first consumer; Stage 4 reuses it).

Items intentionally NOT in this plan:
- **Quest event ingester** (`POST /internal/quests/event`) — Stage 3.
- **Tool-result hook wiring** — Stage 3.
- **Quest validators** — Stage 3.

Stage 2 (the raffle CLI) can begin as soon as this PR merges; Stage 3 needs Stage 1's eligibility helper to be in `main` before its tests pass.
