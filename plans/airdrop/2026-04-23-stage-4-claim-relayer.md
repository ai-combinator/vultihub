# Vultiverse Airdrop — Stage 4 (Claim Relayer) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Pay out VULT to claim-eligible winners on Day 28+. `POST /airdrop/claim` (JWT) builds + KMS-signs + broadcasts an EIP-1559 `claim(recipient)` tx against the deployed `AirdropClaim.sol`, recording the submission in `agent_claim_submissions` under a single `SELECT FOR UPDATE` lock on the `agent_relayer_state` singleton. A confirmation-monitor goroutine inside the existing `cmd/scheduler` flips submitted → confirmed/failed against the chain. Operator UI at `/ops/*` (Basic Auth, server-rendered HTML) covers balance, stuck-claims, and rebroadcast-with-bump.

**Architecture:** New per-stage code lives under `internal/service/airdrop/relayer/` (calldata, tx building, claim service, confirmation monitor, rebroadcast service) and `internal/storage/postgres/` (claim + relayer-state repos). Three new HTTP surfaces: `internal/api/airdrop_claim.go` (JWT), `internal/api/relayer/` (X-Internal-Token), `internal/api/ops/` (Basic Auth, HTML). The shared-infra packages from `feat/airdrop-shared-infra` (`internal/service/airdrop/kms`, `internal/service/airdrop/ethclient`, `InternalTokenMiddleware`) and the eligibility helper from `feat/airdrop-stage-1-boarding` (`internal/service/airdrop/quests/eligibility.go`) are imported as-is. The EVM client wrapper gains one helper this stage (`EstimateGas`); KMS signer is unmodified. The confirmation monitor goroutine is launched from `cmd/scheduler/main.go` via `safego.Go` alongside the existing `sched.Run(ctx)` loop — same process, same signal handler, same shutdown.

**Tech Stack:**
- Go 1.25, Echo v4
- pgx/v5 with `pool.BeginTx(ctx, pgx.TxOptions{})` + `Queries.WithTx(tx)` for the SELECT FOR UPDATE pipeline
- `github.com/ethereum/go-ethereum` v1.17.2 — `core/types` (DynamicFeeTx, LatestSignerForChainID, Transaction.WithSignature), `crypto` (Keccak256, Ecrecover), `common` (Address, Hash, hexutil)
- `github.com/aws/aws-sdk-go-v2/service/kms` — already wired through `internal/service/airdrop/kms.Signer`
- `html/template` (stdlib) for `/ops/*` pages — first use of html/template in this repo
- Anvil from Foundry (`brew install foundry`) for integration-test EVM fixture
- `safego.Go(logger, name, fn)` for the confirmation monitor and any panic-isolated background work

**Spec source of truth:** `docs/airdrop-specs/stage-4-claim-relayer.md`. Cross-reference: `shared-concerns.md` (auth model, env inventory) and `sibling-on-chain-contract.md` (`claim(address)` ABI).

**Working directory for all tasks:** `/Users/apotheosis/git/vultisig/agent-backend/`

**Branch:** `feat/airdrop-stage-4-claim-relayer`. Branch off `main` once `feat/airdrop-shared-infra` and `feat/airdrop-stage-1-boarding` have merged. Commit after every task.

**Hard deadline:** Live in production by Day 28 (see Stage 0 decisions table for the wall-clock date). The relayer is the only stage that touches mainnet — bring it up in staging behind `CLAIM_ENABLED=false` first, smoke-test against Sepolia or a Foundry-deployed fixture, then flip the kill switch on Day 28.

---

## Pre-flight (do once before Task 1)

Confirm the prerequisite branches have merged and the airdrop tables exist locally:

```bash
cd /Users/apotheosis/git/vultisig/agent-backend
git checkout main && git pull
psql "$DATABASE_DSN" -c "\d agent_claim_submissions"
psql "$DATABASE_DSN" -c "\d agent_relayer_state"
psql "$DATABASE_DSN" -c "\d agent_raffle_winners"
psql "$DATABASE_DSN" -c "SELECT * FROM agent_relayer_state;"
```

Expected: all three `\d` calls describe the tables; the `SELECT` returns one row with `id=1, next_nonce=NULL`. If any are missing, run `make migrate-up` first.

Confirm the Stage 1 eligibility helper is present:

```bash
ls internal/service/airdrop/quests/eligibility.go
```

If it isn't there, Stage 1's PR hasn't merged — stop and resolve before proceeding.

Install Foundry (Anvil) if not already on the machine:

```bash
brew install foundry
anvil --version
```

Expected: prints a version like `anvil 0.2.0 (...)`. The Anvil binary is used by Task 1 and Task 7 for EVM-side integration tests.

```bash
git checkout -b feat/airdrop-stage-4-claim-relayer
```

---

## Task 1: Extend the EVM client wrapper with `EstimateGas`, plus an Anvil-backed test helper

**Why:** The relayer needs an EIP-1559 gas-limit estimate (`eth_estimateGas`) before broadcasting; the shared-infra wrapper at `internal/service/airdrop/ethclient/client.go:18` exposes `LatestBaseFee`, `SuggestGasTipCap`, `NonceAt`, `SendRawTransaction`, `GetTransactionReceipt`, `BalanceOf`, `Balance` — but no `EstimateGas`. Add it as a small first-task extension, then introduce a shared `anviltest` helper that subsequent tasks reuse.

**Files:**
- Modify: `internal/service/airdrop/ethclient/client.go` — add `EstimateGas` method
- Create: `internal/service/airdrop/ethclient/anviltest/anvil.go` — test helper for spinning up Anvil
- Modify: `internal/service/airdrop/ethclient/client_test.go` — add round-trip test against Anvil

- [ ] **Step 1: Write the failing test against Anvil**

Append to `internal/service/airdrop/ethclient/client_test.go`:

```go
import (
	"github.com/vultisig/agent-backend/internal/service/airdrop/ethclient/anviltest"
)

// TestEstimateGasAgainstAnvil spins up an Anvil instance, deploys nothing,
// and asks for the gas estimate of a plain ETH transfer. Anvil's deterministic
// chain returns 21000 for a value-only tx with empty data.
func TestEstimateGasAgainstAnvil(t *testing.T) {
	anvil, err := anviltest.Start(t)
	if err != nil {
		t.Skipf("anvil not available (install foundry: brew install foundry): %v", err)
	}
	c, err := NewClient(anvil.URL())
	if err != nil {
		t.Fatalf("NewClient: %v", err)
	}

	from := common.HexToAddress("0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266") // anvil[0]
	to := common.HexToAddress("0x70997970C51812dc3A010C7d01b50e0d17dc79C8")   // anvil[1]

	gas, err := c.EstimateGas(context.Background(), from, to, nil, big.NewInt(1))
	if err != nil {
		t.Fatalf("EstimateGas: %v", err)
	}
	if gas != 21000 {
		t.Errorf("EstimateGas for value transfer = %d, want 21000", gas)
	}
}
```

- [ ] **Step 2: Implement the Anvil test helper**

Create `internal/service/airdrop/ethclient/anviltest/anvil.go`:

```go
// Package anviltest spins up a Foundry Anvil node for integration tests.
// Requires the `anvil` binary on PATH (install via `brew install foundry`).
package anviltest

import (
	"context"
	"fmt"
	"net"
	"os/exec"
	"testing"
	"time"
)

// Anvil wraps a running anvil subprocess.
type Anvil struct {
	cmd  *exec.Cmd
	port int
}

// Start launches anvil on a free port. Cleans up on test end.
func Start(t *testing.T) (*Anvil, error) {
	t.Helper()
	if _, err := exec.LookPath("anvil"); err != nil {
		return nil, fmt.Errorf("anvil binary not found on PATH: %w", err)
	}

	port, err := freePort()
	if err != nil {
		return nil, fmt.Errorf("alloc port: %w", err)
	}

	cmd := exec.Command("anvil",
		"--port", fmt.Sprint(port),
		"--block-time", "1",
		"--silent",
	)
	if err := cmd.Start(); err != nil {
		return nil, fmt.Errorf("start anvil: %w", err)
	}

	a := &Anvil{cmd: cmd, port: port}
	t.Cleanup(func() { _ = a.cmd.Process.Kill() })

	// Poll until the port accepts connections (anvil takes ~50ms to boot)
	deadline := time.Now().Add(5 * time.Second)
	for time.Now().Before(deadline) {
		conn, err := net.DialTimeout("tcp", fmt.Sprintf("127.0.0.1:%d", port), 100*time.Millisecond)
		if err == nil {
			_ = conn.Close()
			return a, nil
		}
		time.Sleep(50 * time.Millisecond)
	}
	_ = a.cmd.Process.Kill()
	return nil, fmt.Errorf("anvil did not become reachable on port %d within 5s", port)
}

// URL returns the JSON-RPC URL.
func (a *Anvil) URL() string {
	return fmt.Sprintf("http://127.0.0.1:%d", a.port)
}

// Port returns the bound port (handy for tests that need to point other clients at it).
func (a *Anvil) Port() int { return a.port }

// Wait waits up to dur for the anvil process to exit.
func (a *Anvil) Wait(ctx context.Context, dur time.Duration) error {
	done := make(chan error, 1)
	go func() { done <- a.cmd.Wait() }()
	select {
	case err := <-done:
		return err
	case <-time.After(dur):
		return fmt.Errorf("timeout waiting for anvil exit")
	case <-ctx.Done():
		return ctx.Err()
	}
}

func freePort() (int, error) {
	l, err := net.Listen("tcp", "127.0.0.1:0")
	if err != nil {
		return 0, err
	}
	defer l.Close()
	return l.Addr().(*net.TCPAddr).Port, nil
}
```

- [ ] **Step 3: Run test to verify it fails (no `EstimateGas` yet)**

```bash
go test ./internal/service/airdrop/ethclient/ -run TestEstimateGasAgainstAnvil -v
```

Expected: compile error — `c.EstimateGas` undefined.

- [ ] **Step 4: Add `EstimateGas` to the Client interface and implementation**

Edit `internal/service/airdrop/ethclient/client.go`. Add to the `Client` interface (near line 32, after `Balance`):

```go
	// EstimateGas returns the gas required to execute the call (eth_estimateGas).
	// Used to size the EIP-1559 tx's GasLimit before signing.
	EstimateGas(ctx context.Context, from, to common.Address, data []byte, value *big.Int) (uint64, error)
```

Add the method implementation at the bottom of the file:

```go
func (c *client) EstimateGas(ctx context.Context, from, to common.Address, data []byte, value *big.Int) (uint64, error) {
	return c.rpc.EstimateGas(ctx, ethereum.CallMsg{
		From:  from,
		To:    &to,
		Data:  data,
		Value: value,
	})
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
go test ./internal/service/airdrop/ethclient/ -run TestEstimateGasAgainstAnvil -v
```

Expected: PASS. If anvil isn't installed the test skips with the install hint.

- [ ] **Step 6: Commit**

```bash
git add internal/service/airdrop/ethclient/client.go \
         internal/service/airdrop/ethclient/client_test.go \
         internal/service/airdrop/ethclient/anviltest/anvil.go
git commit -m "feat(airdrop): add EstimateGas to ethclient + anviltest helper"
```

---

## Task 2: sqlc queries for `agent_claim_submissions` + `agent_relayer_state`

> **Schema update (2026-04-23):** the `previous_tx_hashes TEXT[]` column was dropped. Throughout this task and Tasks 3, 11, 12, 13: drop the column from the schema fixture, drop the field from the Go struct, drop it from the queries (no `array_append` in the rebroadcast query — just `UPDATE ... SET tx_hash = $newHash, max_fee_gwei = $bumpedMax, max_priority_fee_gwei = $bumpedTip`). The audit trail of prior bumps lives in the structured log lines emitted by the rebroadcast service (one line per bump tagged `old_tx_hash`, `new_tx_hash`, `bump_gwei`, `operator`).

**Why:** The claim service needs nine queries: lock-and-read state, init-nonce-if-null, increment-nonce, insert claim, get-by-public-key (idempotent retry), list-pending-for-monitor, mark-confirmed, mark-failed, mark-rebroadcast (update tx_hash + bumped fees in place), plus a stuck-claims read for ops. Bundle them in two sqlc files mirroring the existing `internal/storage/postgres/sqlc/*.sql` pattern.

**Files:**
- Create: `internal/storage/postgres/sqlc/airdrop_claims.sql`
- Create: `internal/storage/postgres/sqlc/airdrop_relayer_state.sql`
- Modify: `internal/storage/postgres/schema/schema.sql` — append the two airdrop tables so sqlc can resolve the column types
- Generated: `internal/storage/postgres/queries/airdrop_claims.sql.go` (do not hand-edit)
- Generated: `internal/storage/postgres/queries/airdrop_relayer_state.sql.go` (do not hand-edit)

- [ ] **Step 1: Append airdrop tables to the sqlc schema file**

`sqlc.yaml` points at `internal/storage/postgres/schema/schema.sql` for type resolution. Append to the bottom of `internal/storage/postgres/schema/schema.sql`:

```sql
-- ============================================================================
-- Vultiverse Airdrop tables (see internal/storage/postgres/migrations/2026042300000{1..6}_*.sql)
-- These exist in this file solely so sqlc can resolve column types when
-- generating queries against the airdrop_*.sql query files.
-- ============================================================================

CREATE TYPE airdrop_registration_source AS ENUM ('seed', 'vault_share');
CREATE TYPE airdrop_quest_id            AS ENUM ('swap', 'bridge', 'defi_action', 'alert', 'dca');
CREATE TYPE airdrop_quest_event_status  AS ENUM ('counted', 'rejected');
CREATE TYPE airdrop_claim_status        AS ENUM ('submitted', 'confirmed', 'failed');

CREATE TABLE agent_airdrop_registrations (
    public_key        TEXT                        PRIMARY KEY,
    source            airdrop_registration_source NOT NULL,
    recipient_address TEXT                        NOT NULL,
    registered_at     TIMESTAMPTZ                 NOT NULL DEFAULT NOW()
);

CREATE TABLE agent_raffle_winners (
    public_key TEXT          PRIMARY KEY,
    recipient  TEXT          NOT NULL,
    amount     NUMERIC(78,0) NOT NULL,
    loaded_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE TABLE agent_quest_events (
    tool_call_id  TEXT                       PRIMARY KEY,
    public_key    TEXT                       NOT NULL,
    quest_id      airdrop_quest_id           NOT NULL,
    tx_hash       TEXT                       NOT NULL,
    status        airdrop_quest_event_status NOT NULL,
    reject_reason TEXT,
    payload       JSONB                      NOT NULL,
    created_at    TIMESTAMPTZ                NOT NULL DEFAULT NOW()
);

CREATE TABLE agent_user_quests (
    public_key   TEXT             NOT NULL,
    quest_id     airdrop_quest_id NOT NULL,
    completed_at TIMESTAMPTZ      NOT NULL DEFAULT NOW(),
    PRIMARY KEY (public_key, quest_id)
);

CREATE TABLE agent_claim_submissions (
    public_key            TEXT                 NOT NULL,
    nonce                 BIGINT               NOT NULL UNIQUE,
    recipient             TEXT                 NOT NULL,
    amount                NUMERIC(78, 0)       NOT NULL,
    tx_hash               TEXT                 NOT NULL,
    previous_tx_hashes    TEXT[]               NOT NULL DEFAULT '{}',
    status                airdrop_claim_status NOT NULL,
    max_fee_gwei          INTEGER              NOT NULL,
    max_priority_fee_gwei INTEGER              NOT NULL,
    block_number          BIGINT,
    failure_reason        TEXT,
    submitted_at          TIMESTAMPTZ          NOT NULL,
    confirmed_at          TIMESTAMPTZ,
    failed_at             TIMESTAMPTZ
);

CREATE TABLE agent_relayer_state (
    id          INT PRIMARY KEY DEFAULT 1 CHECK (id = 1),
    next_nonce  BIGINT,
    last_synced TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
```

If Stage 1's plan already added the first four tables (registrations, winners, quest_events, user_quests) to this file, leave the existing copies in place and only append the claim_submissions + relayer_state blocks. `grep -n "agent_airdrop_registrations\|agent_relayer_state" internal/storage/postgres/schema/schema.sql` to check before pasting.

- [ ] **Step 2: Write the claims query file**

Create `internal/storage/postgres/sqlc/airdrop_claims.sql`:

```sql
-- name: InsertClaimSubmission :one
INSERT INTO agent_claim_submissions (
    public_key, nonce, recipient, amount, tx_hash,
    status, max_fee_gwei, max_priority_fee_gwei, submitted_at
) VALUES (
    $1, $2, $3, $4, $5, 'submitted', $6, $7, NOW()
)
RETURNING *;

-- name: GetActiveClaimByPublicKey :one
-- Returns the user's submitted-or-confirmed claim if any. Idempotent-retry
-- path uses this inside the SELECT FOR UPDATE transaction.
SELECT * FROM agent_claim_submissions
WHERE public_key = $1
  AND status IN ('submitted', 'confirmed')
LIMIT 1;

-- name: ListPendingClaims :many
-- Confirmation monitor scan. Index idx_claim_submissions_pending makes this
-- O(unconfirmed). Limit caps per-tick work; the next tick picks up the rest.
SELECT * FROM agent_claim_submissions
WHERE status = 'submitted'
ORDER BY submitted_at ASC
LIMIT $1;

-- name: ListStuckClaims :many
-- /internal/relayer/stuck-claims and /ops/stuck-claims source. Filters
-- submitted-state rows older than $1 minutes, sorted by oldest-first.
SELECT * FROM agent_claim_submissions
WHERE status = 'submitted'
  AND submitted_at < NOW() - make_interval(mins => $1::int)
ORDER BY submitted_at ASC;

-- name: MarkClaimConfirmed :exec
UPDATE agent_claim_submissions
SET status = 'confirmed',
    confirmed_at = NOW(),
    block_number = $2
WHERE nonce = $1;

-- name: MarkClaimFailed :exec
UPDATE agent_claim_submissions
SET status = 'failed',
    failed_at = NOW(),
    failure_reason = $2
WHERE nonce = $1;

-- name: GetClaimByPublicKey :one
-- Rebroadcast lookup. Returns the most recent claim (any status) for the user.
SELECT * FROM agent_claim_submissions
WHERE public_key = $1
ORDER BY submitted_at DESC
LIMIT 1;

-- name: UpdateClaimRebroadcast :exec
-- Same-nonce rebroadcast. Pushes the previous tx_hash onto the audit array,
-- updates the live tx_hash + bumped fee fields, leaves status='submitted'.
UPDATE agent_claim_submissions
SET previous_tx_hashes    = array_append(previous_tx_hashes, tx_hash),
    tx_hash               = $2,
    max_fee_gwei          = $3,
    max_priority_fee_gwei = $4
WHERE nonce = $1
  AND status = 'submitted';
```

- [ ] **Step 3: Write the relayer-state query file**

Create `internal/storage/postgres/sqlc/airdrop_relayer_state.sql`:

```sql
-- name: LockRelayerState :one
-- The serialising lock for every claim submission. Must run inside a tx —
-- caller does pool.BeginTx → queries.WithTx → LockRelayerState → ... → COMMIT.
SELECT * FROM agent_relayer_state
WHERE id = 1
FOR UPDATE;

-- name: GetRelayerState :one
-- Lock-free read for the balance endpoint and the auto-init "is it nil?" probe.
SELECT * FROM agent_relayer_state
WHERE id = 1;

-- name: SetRelayerNonce :exec
-- Used by both the auto-init path (first boot) and the increment path
-- (every successful claim). last_synced updates so we can spot a stuck nonce.
UPDATE agent_relayer_state
SET next_nonce  = $1,
    last_synced = NOW()
WHERE id = 1;
```

- [ ] **Step 4: Run sqlc generation**

```bash
grep -n sqlc Makefile
```

If a `make sqlc` target exists, use it. Otherwise:

```bash
sqlc generate
```

If sqlc isn't installed:

```bash
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
```

Expected: two new files appear:
- `internal/storage/postgres/queries/airdrop_claims.sql.go`
- `internal/storage/postgres/queries/airdrop_relayer_state.sql.go`

Plus updates to `internal/storage/postgres/queries/models.go` for the new table struct types and the `AirdropClaimStatus` enum.

- [ ] **Step 5: Verify the generated code compiles**

```bash
go build ./internal/storage/postgres/...
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add internal/storage/postgres/sqlc/airdrop_claims.sql \
         internal/storage/postgres/sqlc/airdrop_relayer_state.sql \
         internal/storage/postgres/schema/schema.sql \
         internal/storage/postgres/queries/airdrop_claims.sql.go \
         internal/storage/postgres/queries/airdrop_relayer_state.sql.go \
         internal/storage/postgres/queries/models.go
git commit -m "feat(airdrop): add sqlc queries for claim submissions + relayer state"
```

---

## Task 3: Repositories — `ClaimRepository` and `RelayerStateRepository`

**Why:** Wraps the sqlc Queries with domain types. Both repos expose a `WithTx(tx pgx.Tx)` variant so the claim service can run the lock+insert+update sequence in a single transaction.

**Files:**
- Create: `internal/types/airdrop_claim.go` — `ClaimSubmission`, `RelayerState`, `ClaimStatus` constants
- Create: `internal/storage/postgres/airdrop_claim.go` — `ClaimRepository`
- Create: `internal/storage/postgres/airdrop_relayer_state.go` — `RelayerStateRepository`
- Create: `internal/storage/postgres/airdrop_claim_test.go`
- Create: `internal/storage/postgres/airdrop_relayer_state_test.go`

- [ ] **Step 1: Add the domain types**

Create `internal/types/airdrop_claim.go`:

```go
package types

import (
	"math/big"
	"time"
)

type ClaimStatus string

const (
	ClaimSubmitted ClaimStatus = "submitted"
	ClaimConfirmed ClaimStatus = "confirmed"
	ClaimFailed    ClaimStatus = "failed"
)

type ClaimSubmission struct {
	PublicKey          string
	Nonce              int64
	Recipient          string
	Amount             *big.Int
	TxHash             string
	PreviousTxHashes   []string
	Status             ClaimStatus
	MaxFeeGwei         int32
	MaxPriorityFeeGwei int32
	BlockNumber        *int64 // nil unless confirmed
	FailureReason      *string
	SubmittedAt        time.Time
	ConfirmedAt        *time.Time
	FailedAt           *time.Time
}

type RelayerState struct {
	NextNonce  *int64 // nil before first boot
	LastSynced time.Time
}
```

- [ ] **Step 2: Write the claim repo test**

Create `internal/storage/postgres/airdrop_claim_test.go`:

```go
package postgres_test

import (
	"context"
	"math/big"
	"os"
	"testing"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/storage/postgres"
	"github.com/vultisig/agent-backend/internal/types"
)

func newClaimTestPool(t *testing.T) *pgxpool.Pool {
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
	_, err = pool.Exec(context.Background(), `TRUNCATE agent_claim_submissions;`)
	if err != nil {
		t.Fatalf("truncate: %v", err)
	}
	return pool
}

func TestClaimRepo_InsertAndGet(t *testing.T) {
	pool := newClaimTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	ctx := context.Background()

	row, err := repo.Insert(ctx, "pk1", 7, "0xaaa", big.NewInt(1500), "0xtx1", 50, 2)
	if err != nil {
		t.Fatalf("Insert: %v", err)
	}
	if row.Nonce != 7 || row.TxHash != "0xtx1" || row.Status != types.ClaimSubmitted {
		t.Errorf("unexpected row: %+v", row)
	}

	got, err := repo.GetActiveByPublicKey(ctx, "pk1")
	if err != nil {
		t.Fatalf("GetActiveByPublicKey: %v", err)
	}
	if got.Nonce != 7 {
		t.Errorf("nonce = %d, want 7", got.Nonce)
	}

	// Same-user second insert with a fresh nonce must violate the partial unique index
	_, err = repo.Insert(ctx, "pk1", 8, "0xaaa", big.NewInt(1500), "0xtx2", 50, 2)
	if err == nil {
		t.Errorf("second submission for same user should fail unique index")
	}
}

func TestClaimRepo_MarkConfirmed(t *testing.T) {
	pool := newClaimTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	ctx := context.Background()

	_, _ = repo.Insert(ctx, "pk1", 9, "0xaaa", big.NewInt(1500), "0xtx", 50, 2)
	if err := repo.MarkConfirmed(ctx, 9, 19400000); err != nil {
		t.Fatalf("MarkConfirmed: %v", err)
	}

	// Now GetActiveByPublicKey still finds it (confirmed counts as active for idempotency)
	got, err := repo.GetActiveByPublicKey(ctx, "pk1")
	if err != nil {
		t.Fatalf("GetActiveByPublicKey: %v", err)
	}
	if got.Status != types.ClaimConfirmed {
		t.Errorf("status = %s, want confirmed", got.Status)
	}
	if got.BlockNumber == nil || *got.BlockNumber != 19400000 {
		t.Errorf("block_number = %v, want 19400000", got.BlockNumber)
	}
}

func TestClaimRepo_ListStuck(t *testing.T) {
	pool := newClaimTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	ctx := context.Background()

	// Insert one fresh and one with submitted_at=NOW()-30min via direct UPDATE
	_, _ = repo.Insert(ctx, "pk1", 1, "0xaaa", big.NewInt(1500), "0xtx1", 50, 2)
	_, _ = repo.Insert(ctx, "pk2", 2, "0xbbb", big.NewInt(1500), "0xtx2", 50, 2)
	_, err := pool.Exec(ctx, `UPDATE agent_claim_submissions SET submitted_at = NOW() - interval '30 minutes' WHERE nonce = 2`)
	if err != nil {
		t.Fatalf("backdate: %v", err)
	}

	stuck, err := repo.ListStuck(ctx, 10) // older than 10 minutes
	if err != nil {
		t.Fatalf("ListStuck: %v", err)
	}
	if len(stuck) != 1 || stuck[0].Nonce != 2 {
		t.Errorf("ListStuck = %+v, want one row with nonce=2", stuck)
	}
}

func TestClaimRepo_Rebroadcast(t *testing.T) {
	pool := newClaimTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	ctx := context.Background()

	_, _ = repo.Insert(ctx, "pk1", 5, "0xaaa", big.NewInt(1500), "0xold", 50, 2)
	if err := repo.UpdateRebroadcast(ctx, 5, "0xnew", 100, 4); err != nil {
		t.Fatalf("UpdateRebroadcast: %v", err)
	}

	got, _ := repo.GetActiveByPublicKey(ctx, "pk1")
	if got.TxHash != "0xnew" {
		t.Errorf("tx_hash = %s, want 0xnew", got.TxHash)
	}
	if len(got.PreviousTxHashes) != 1 || got.PreviousTxHashes[0] != "0xold" {
		t.Errorf("previous_tx_hashes = %v, want [0xold]", got.PreviousTxHashes)
	}
	if got.MaxFeeGwei != 100 || got.MaxPriorityFeeGwei != 4 {
		t.Errorf("fees = (%d, %d), want (100, 4)", got.MaxFeeGwei, got.MaxPriorityFeeGwei)
	}
}
```

- [ ] **Step 3: Write the relayer-state repo test**

Create `internal/storage/postgres/airdrop_relayer_state_test.go`:

```go
package postgres_test

import (
	"context"
	"os"
	"testing"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/storage/postgres"
)

func newRelayerStateTestPool(t *testing.T) *pgxpool.Pool {
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
	// Reset to nil-nonce baseline (singleton row created by migration 6)
	_, err = pool.Exec(context.Background(), `UPDATE agent_relayer_state SET next_nonce = NULL WHERE id = 1`)
	if err != nil {
		t.Fatalf("reset: %v", err)
	}
	return pool
}

func TestRelayerStateRepo_GetReturnsNilNonceInitially(t *testing.T) {
	pool := newRelayerStateTestPool(t)
	repo := postgres.NewRelayerStateRepository(pool)

	st, err := repo.Get(context.Background())
	if err != nil {
		t.Fatalf("Get: %v", err)
	}
	if st.NextNonce != nil {
		t.Errorf("expected nil next_nonce initially, got %v", *st.NextNonce)
	}
}

func TestRelayerStateRepo_LockAndSetWithinTx(t *testing.T) {
	pool := newRelayerStateTestPool(t)
	repo := postgres.NewRelayerStateRepository(pool)
	ctx := context.Background()

	tx, err := pool.BeginTx(ctx, pgx.TxOptions{})
	if err != nil {
		t.Fatalf("BeginTx: %v", err)
	}
	defer tx.Rollback(ctx)

	st, err := repo.LockWithTx(ctx, tx)
	if err != nil {
		t.Fatalf("LockWithTx: %v", err)
	}
	if st.NextNonce != nil {
		t.Errorf("expected nil nonce, got %v", *st.NextNonce)
	}
	if err := repo.SetNonceWithTx(ctx, tx, 42); err != nil {
		t.Fatalf("SetNonceWithTx: %v", err)
	}
	if err := tx.Commit(ctx); err != nil {
		t.Fatalf("Commit: %v", err)
	}

	after, _ := repo.Get(ctx)
	if after.NextNonce == nil || *after.NextNonce != 42 {
		t.Errorf("after commit next_nonce = %v, want 42", after.NextNonce)
	}
}
```

- [ ] **Step 4: Run tests to verify they fail**

```bash
go test ./internal/storage/postgres/ -run 'TestClaimRepo|TestRelayerStateRepo' -v
```

Expected: compile errors — `NewClaimRepository`, `NewRelayerStateRepository` undefined.

- [ ] **Step 5: Implement `ClaimRepository`**

Create `internal/storage/postgres/airdrop_claim.go`:

```go
package postgres

import (
	"context"
	"errors"
	"fmt"
	"math/big"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgtype"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/storage/postgres/queries"
	"github.com/vultisig/agent-backend/internal/types"
)

type ClaimRepository struct {
	pool *pgxpool.Pool
	q    *queries.Queries
}

func NewClaimRepository(pool *pgxpool.Pool) *ClaimRepository {
	return &ClaimRepository{pool: pool, q: queries.New(pool)}
}

func (r *ClaimRepository) Insert(
	ctx context.Context,
	publicKey string,
	nonce int64,
	recipient string,
	amount *big.Int,
	txHash string,
	maxFeeGwei int32,
	maxPriorityFeeGwei int32,
) (*types.ClaimSubmission, error) {
	return r.insert(ctx, r.q, publicKey, nonce, recipient, amount, txHash, maxFeeGwei, maxPriorityFeeGwei)
}

// InsertWithTx is the txn-scoped variant the claim service calls inside the
// SELECT FOR UPDATE block.
func (r *ClaimRepository) InsertWithTx(
	ctx context.Context, tx pgx.Tx,
	publicKey string, nonce int64, recipient string, amount *big.Int,
	txHash string, maxFeeGwei int32, maxPriorityFeeGwei int32,
) (*types.ClaimSubmission, error) {
	return r.insert(ctx, r.q.WithTx(tx), publicKey, nonce, recipient, amount, txHash, maxFeeGwei, maxPriorityFeeGwei)
}

func (r *ClaimRepository) insert(
	ctx context.Context, q *queries.Queries,
	publicKey string, nonce int64, recipient string, amount *big.Int,
	txHash string, maxFeeGwei int32, maxPriorityFeeGwei int32,
) (*types.ClaimSubmission, error) {
	amountNum := pgtype.Numeric{}
	if err := amountNum.Scan(amount.String()); err != nil {
		return nil, fmt.Errorf("scan amount: %w", err)
	}
	row, err := q.InsertClaimSubmission(ctx, &queries.InsertClaimSubmissionParams{
		PublicKey:          publicKey,
		Nonce:              nonce,
		Recipient:          recipient,
		Amount:             amountNum,
		TxHash:             txHash,
		MaxFeeGwei:         maxFeeGwei,
		MaxPriorityFeeGwei: maxPriorityFeeGwei,
	})
	if err != nil {
		return nil, fmt.Errorf("insert claim submission: %w", err)
	}
	return claimFromDB(row), nil
}

// GetActiveByPublicKey returns the user's submitted-or-confirmed claim row,
// ErrNotFound if none. Used both by the claim handler (idempotent retry) and
// by the rebroadcast service to validate the user has a live submission.
func (r *ClaimRepository) GetActiveByPublicKey(ctx context.Context, publicKey string) (*types.ClaimSubmission, error) {
	return r.getActiveByPublicKey(ctx, r.q, publicKey)
}

func (r *ClaimRepository) GetActiveByPublicKeyWithTx(ctx context.Context, tx pgx.Tx, publicKey string) (*types.ClaimSubmission, error) {
	return r.getActiveByPublicKey(ctx, r.q.WithTx(tx), publicKey)
}

func (r *ClaimRepository) getActiveByPublicKey(ctx context.Context, q *queries.Queries, publicKey string) (*types.ClaimSubmission, error) {
	row, err := q.GetActiveClaimByPublicKey(ctx, publicKey)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return nil, ErrNotFound
		}
		return nil, fmt.Errorf("get active claim: %w", err)
	}
	return claimFromDB(row), nil
}

// GetByPublicKey returns the most recent claim row regardless of status.
// Used for rebroadcast and ops display.
func (r *ClaimRepository) GetByPublicKey(ctx context.Context, publicKey string) (*types.ClaimSubmission, error) {
	row, err := r.q.GetClaimByPublicKey(ctx, publicKey)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return nil, ErrNotFound
		}
		return nil, fmt.Errorf("get claim by public key: %w", err)
	}
	return claimFromDB(row), nil
}

// ListPending returns up to `limit` submitted-state rows for the confirmation
// monitor. Sorted oldest-first so we drain the backlog FIFO.
func (r *ClaimRepository) ListPending(ctx context.Context, limit int32) ([]types.ClaimSubmission, error) {
	rows, err := r.q.ListPendingClaims(ctx, limit)
	if err != nil {
		return nil, fmt.Errorf("list pending: %w", err)
	}
	out := make([]types.ClaimSubmission, 0, len(rows))
	for _, row := range rows {
		out = append(out, *claimFromDB(row))
	}
	return out, nil
}

func (r *ClaimRepository) ListStuck(ctx context.Context, olderThanMinutes int32) ([]types.ClaimSubmission, error) {
	rows, err := r.q.ListStuckClaims(ctx, olderThanMinutes)
	if err != nil {
		return nil, fmt.Errorf("list stuck: %w", err)
	}
	out := make([]types.ClaimSubmission, 0, len(rows))
	for _, row := range rows {
		out = append(out, *claimFromDB(row))
	}
	return out, nil
}

func (r *ClaimRepository) MarkConfirmed(ctx context.Context, nonce int64, blockNumber int64) error {
	if err := r.q.MarkClaimConfirmed(ctx, &queries.MarkClaimConfirmedParams{
		Nonce:       nonce,
		BlockNumber: pgtype.Int8{Int64: blockNumber, Valid: true},
	}); err != nil {
		return fmt.Errorf("mark confirmed: %w", err)
	}
	return nil
}

func (r *ClaimRepository) MarkFailed(ctx context.Context, nonce int64, reason string) error {
	if err := r.q.MarkClaimFailed(ctx, &queries.MarkClaimFailedParams{
		Nonce:         nonce,
		FailureReason: pgtype.Text{String: reason, Valid: true},
	}); err != nil {
		return fmt.Errorf("mark failed: %w", err)
	}
	return nil
}

func (r *ClaimRepository) UpdateRebroadcast(
	ctx context.Context,
	nonce int64,
	newTxHash string,
	maxFeeGwei int32,
	maxPriorityFeeGwei int32,
) error {
	if err := r.q.UpdateClaimRebroadcast(ctx, &queries.UpdateClaimRebroadcastParams{
		Nonce:              nonce,
		TxHash:             newTxHash,
		MaxFeeGwei:         maxFeeGwei,
		MaxPriorityFeeGwei: maxPriorityFeeGwei,
	}); err != nil {
		return fmt.Errorf("update rebroadcast: %w", err)
	}
	return nil
}

func claimFromDB(row *queries.AgentClaimSubmission) *types.ClaimSubmission {
	out := &types.ClaimSubmission{
		PublicKey:          row.PublicKey,
		Nonce:              row.Nonce,
		Recipient:          row.Recipient,
		TxHash:             row.TxHash,
		PreviousTxHashes:   row.PreviousTxHashes,
		Status:             types.ClaimStatus(row.Status),
		MaxFeeGwei:         row.MaxFeeGwei,
		MaxPriorityFeeGwei: row.MaxPriorityFeeGwei,
		SubmittedAt:        row.SubmittedAt.Time,
	}
	// pgtype.Numeric → *big.Int via the canonical decimal string
	if amountStr, err := row.Amount.Value(); err == nil && amountStr != nil {
		amount := new(big.Int)
		amount.SetString(fmt.Sprintf("%v", amountStr), 10)
		out.Amount = amount
	}
	if row.BlockNumber.Valid {
		v := row.BlockNumber.Int64
		out.BlockNumber = &v
	}
	if row.FailureReason.Valid {
		v := row.FailureReason.String
		out.FailureReason = &v
	}
	if row.ConfirmedAt.Valid {
		v := row.ConfirmedAt.Time
		out.ConfirmedAt = &v
	}
	if row.FailedAt.Valid {
		v := row.FailedAt.Time
		out.FailedAt = &v
	}
	return out
}
```

The exact field names on the generated struct (`AgentClaimSubmission`) and param structs (`InsertClaimSubmissionParams`, etc.) come from sqlc — open `internal/storage/postgres/queries/airdrop_claims.sql.go` after Step 4 of Task 2 to confirm. The shape above matches sqlc's `pgx/v5` defaults with `emit_params_struct_pointers: true` (per `sqlc.yaml`).

- [ ] **Step 6: Implement `RelayerStateRepository`**

Create `internal/storage/postgres/airdrop_relayer_state.go`:

```go
package postgres

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgtype"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/storage/postgres/queries"
	"github.com/vultisig/agent-backend/internal/types"
)

type RelayerStateRepository struct {
	pool *pgxpool.Pool
	q    *queries.Queries
}

func NewRelayerStateRepository(pool *pgxpool.Pool) *RelayerStateRepository {
	return &RelayerStateRepository{pool: pool, q: queries.New(pool)}
}

// Get is a lock-free read for the balance endpoint and the auto-init probe.
func (r *RelayerStateRepository) Get(ctx context.Context) (*types.RelayerState, error) {
	row, err := r.q.GetRelayerState(ctx)
	if err != nil {
		return nil, fmt.Errorf("get relayer state: %w", err)
	}
	return relayerStateFromDB(row), nil
}

// LockWithTx runs SELECT ... FOR UPDATE inside the supplied transaction.
// Caller must have BEGUN the tx and must COMMIT or ROLLBACK it.
func (r *RelayerStateRepository) LockWithTx(ctx context.Context, tx pgx.Tx) (*types.RelayerState, error) {
	row, err := r.q.WithTx(tx).LockRelayerState(ctx)
	if err != nil {
		return nil, fmt.Errorf("lock relayer state: %w", err)
	}
	return relayerStateFromDB(row), nil
}

// SetNonceWithTx writes the next_nonce inside the supplied transaction.
func (r *RelayerStateRepository) SetNonceWithTx(ctx context.Context, tx pgx.Tx, nextNonce int64) error {
	if err := r.q.WithTx(tx).SetRelayerNonce(ctx, pgtype.Int8{Int64: nextNonce, Valid: true}); err != nil {
		return fmt.Errorf("set relayer nonce: %w", err)
	}
	return nil
}

// Pool exposes the underlying pgxpool — the claim service needs it to call
// BeginTx directly. Returns the same *pgxpool.Pool the constructor was given.
func (r *RelayerStateRepository) Pool() *pgxpool.Pool { return r.pool }

func relayerStateFromDB(row *queries.AgentRelayerState) *types.RelayerState {
	out := &types.RelayerState{LastSynced: row.LastSynced.Time}
	if row.NextNonce.Valid {
		v := row.NextNonce.Int64
		out.NextNonce = &v
	}
	return out
}
```

- [ ] **Step 7: Run tests to verify they pass**

```bash
go test ./internal/storage/postgres/ -run 'TestClaimRepo|TestRelayerStateRepo' -v
```

Expected: PASS for all subtests, given `DATABASE_DSN` is set.

- [ ] **Step 8: Commit**

```bash
git add internal/types/airdrop_claim.go \
         internal/storage/postgres/airdrop_claim.go \
         internal/storage/postgres/airdrop_claim_test.go \
         internal/storage/postgres/airdrop_relayer_state.go \
         internal/storage/postgres/airdrop_relayer_state_test.go
git commit -m "feat(airdrop): add ClaimRepository + RelayerStateRepository"
```

---

## Task 4: Calldata builder for `claim(address)`

**Why:** The contract surface is `function claim(address recipient)` (`sibling-on-chain-contract.md:66`). Calldata is `keccak256("claim(address)")[:4] ++ left_pad32(recipient)` — 36 bytes total. Hand-rolled — easier to test against a fixed fixture than dragging in `accounts/abi`'s parser, and avoids any "did the ABI shape change" surprise.

**Files:**
- Create: `internal/service/airdrop/relayer/calldata.go`
- Create: `internal/service/airdrop/relayer/calldata_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/service/airdrop/relayer/calldata_test.go`:

```go
package relayer

import (
	"encoding/hex"
	"testing"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// TestClaimSelector confirms the 4-byte function selector matches keccak256("claim(address)")[:4].
// Computed independently with go-ethereum's keccak so we don't trust a hardcoded constant blindly.
func TestClaimSelector(t *testing.T) {
	want := crypto.Keccak256([]byte("claim(address)"))[:4]
	if hex.EncodeToString(claimSelector) != hex.EncodeToString(want) {
		t.Errorf("claimSelector = %x, want %x", claimSelector, want)
	}
	// Sanity-check the actual bytes — keccak256("claim(address)") is well-known
	// (matches what cast's `cast sig "claim(address)"` returns).
	wantHex := "1e83409a"
	if hex.EncodeToString(claimSelector) != wantHex {
		t.Errorf("claimSelector hex = %x, want %s", claimSelector, wantHex)
	}
}

func TestBuildClaimCalldata(t *testing.T) {
	recipient := common.HexToAddress("0x70997970C51812dc3A010C7d01b50e0d17dc79C8") // anvil[1]

	got := BuildClaimCalldata(recipient)
	if len(got) != 36 {
		t.Fatalf("calldata length = %d, want 36", len(got))
	}

	// Selector
	if hex.EncodeToString(got[:4]) != "1e83409a" {
		t.Errorf("selector bytes = %x, want 1e83409a", got[:4])
	}
	// 12 zero pad bytes
	for i := 4; i < 16; i++ {
		if got[i] != 0 {
			t.Errorf("byte %d = %d, want zero pad", i, got[i])
		}
	}
	// 20 address bytes
	if hex.EncodeToString(got[16:]) != "70997970c51812dc3a010c7d01b50e0d17dc79c8" {
		t.Errorf("address bytes = %x, want 70997970...", got[16:])
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/service/airdrop/relayer/ -v
```

Expected: compile error — package, selector, builder all undefined.

- [ ] **Step 3: Implement the calldata builder**

Create `internal/service/airdrop/relayer/calldata.go`:

```go
// Package relayer implements the Stage 4 airdrop claim relayer:
// calldata + tx building, KMS-signed broadcast under a serialising
// SELECT FOR UPDATE lock, confirmation monitoring, and same-nonce rebroadcast.
package relayer

import (
	"github.com/ethereum/go-ethereum/common"
)

// claimSelector is the 4-byte function selector for AirdropClaim.claim(address).
// Equal to keccak256("claim(address)")[:4]. Verified by TestClaimSelector.
var claimSelector = []byte{0x1e, 0x83, 0x40, 0x9a}

// BuildClaimCalldata returns the 36-byte EVM calldata payload for
// AirdropClaim.claim(recipient): 4-byte selector ++ 32-byte left-padded recipient.
//
// The contract is `function claim(address recipient) external` — see
// sibling-on-chain-contract.md. The amount lives on-chain in the contract's
// allowance(recipient) mapping, populated by the multisig's setWinners batch.
func BuildClaimCalldata(recipient common.Address) []byte {
	out := make([]byte, 0, 36)
	out = append(out, claimSelector...)
	out = append(out, make([]byte, 12)...) // left-pad address to 32 bytes
	out = append(out, recipient.Bytes()...)
	return out
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/relayer/ -v
```

Expected: PASS for both subtests.

- [ ] **Step 5: Commit**

```bash
git add internal/service/airdrop/relayer/calldata.go internal/service/airdrop/relayer/calldata_test.go
git commit -m "feat(airdrop): add claim(address) calldata builder"
```

---

## Task 5: EIP-1559 tx builder + KMS sign assembly

**Why:** Pure function: take (chainID, nonce, gasParams, to, calldata) → unsigned `types.DynamicFeeTx` → ask KMS signer for `(r, s, v)` over the EIP-155 signing hash → produce signed RLP bytes. Splitting build from sign keeps the unit tests trivial — sign exercises the KMS round-trip path tested in shared infra.

**Files:**
- Create: `internal/service/airdrop/relayer/tx.go`
- Create: `internal/service/airdrop/relayer/tx_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/service/airdrop/relayer/tx_test.go`:

```go
package relayer

import (
	"bytes"
	"context"
	"math/big"
	"testing"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
)

// fakeSigner implements the kms.Signer interface using a local secp256k1 key.
// Lets us exercise BuildAndSignTx without an AWS KMS round-trip.
type fakeSigner struct {
	priv *crypto.PrivateKey
}

func newFakeSigner(t *testing.T) *fakeSigner {
	priv, err := crypto.GenerateKey()
	if err != nil {
		t.Fatalf("GenerateKey: %v", err)
	}
	return &fakeSigner{priv: (*crypto.PrivateKey)(priv)}
}

func (f *fakeSigner) Address() common.Address {
	return crypto.PubkeyToAddress((*crypto.PublicKey)(f.priv).ECDSA().PublicKey)
}

func (f *fakeSigner) SignDigest(_ context.Context, digest []byte) ([]byte, error) {
	return crypto.Sign(digest, (*crypto.PrivateKey)(f.priv).ECDSA())
}

// crypto.PrivateKey/PublicKey shims — go-ethereum uses *ecdsa.PrivateKey directly,
// no shim needed in production. Replace this scaffold with the real signer interface
// if convenient; the shape that matters is (Address() common.Address, SignDigest(...)).

func TestBuildAndSignTx(t *testing.T) {
	signer := newFakeSigner(t)
	chainID := big.NewInt(1)

	to := common.HexToAddress("0x1111111111111111111111111111111111111111")
	calldata := []byte{0xde, 0xad, 0xbe, 0xef}
	params := TxParams{
		ChainID:              chainID,
		Nonce:                7,
		GasLimit:             100000,
		MaxFeePerGasWei:      big.NewInt(50_000_000_000), // 50 gwei
		MaxPriorityFeeWei:    big.NewInt(2_000_000_000),  //  2 gwei
		To:                   to,
		Calldata:             calldata,
	}

	signedBytes, txHash, err := BuildAndSignTx(context.Background(), signer, params)
	if err != nil {
		t.Fatalf("BuildAndSignTx: %v", err)
	}
	if len(signedBytes) == 0 {
		t.Fatal("signedBytes empty")
	}
	if txHash == (common.Hash{}) {
		t.Fatal("txHash zero")
	}

	// Decode the signed bytes and confirm the recovered sender matches the signer's address.
	tx := new(types.Transaction)
	if err := tx.UnmarshalBinary(signedBytes); err != nil {
		t.Fatalf("UnmarshalBinary: %v", err)
	}
	if tx.Type() != types.DynamicFeeTxType {
		t.Errorf("tx type = %d, want %d (DynamicFeeTxType)", tx.Type(), types.DynamicFeeTxType)
	}
	if tx.Nonce() != 7 {
		t.Errorf("nonce = %d, want 7", tx.Nonce())
	}
	if tx.Gas() != 100000 {
		t.Errorf("gas = %d, want 100000", tx.Gas())
	}
	if !bytes.Equal(tx.Data(), calldata) {
		t.Errorf("data = %x, want %x", tx.Data(), calldata)
	}

	gethSigner := types.LatestSignerForChainID(chainID)
	sender, err := types.Sender(gethSigner, tx)
	if err != nil {
		t.Fatalf("recover sender: %v", err)
	}
	if sender != signer.Address() {
		t.Errorf("recovered sender = %s, want %s", sender.Hex(), signer.Address().Hex())
	}
}
```

A note on the fake-signer scaffold above: in practice the simplest fake is `func(ctx, digest) ([]byte, error) { return crypto.Sign(digest, priv) }` with `priv *ecdsa.PrivateKey`. Drop the `crypto.PrivateKey` shim and use `*ecdsa.PrivateKey` directly — go-ethereum's `crypto.Sign` returns the 65-byte `(r||s||v)` signature directly, which is exactly what `BuildAndSignTx` expects from the KMS signer too. The shim is shown above only to keep the test diff focused; replace with `*ecdsa.PrivateKey` when you write it.

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/service/airdrop/relayer/ -run TestBuildAndSignTx -v
```

Expected: compile error — `TxParams`, `BuildAndSignTx` undefined.

- [ ] **Step 3: Implement the tx builder**

Create `internal/service/airdrop/relayer/tx.go`:

```go
package relayer

import (
	"context"
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"

	"github.com/vultisig/agent-backend/internal/service/airdrop/kms"
)

// TxParams holds everything needed to build the unsigned EIP-1559 tx.
type TxParams struct {
	ChainID           *big.Int
	Nonce             uint64
	GasLimit          uint64
	MaxFeePerGasWei   *big.Int
	MaxPriorityFeeWei *big.Int
	To                common.Address
	Calldata          []byte
}

// BuildAndSignTx assembles an EIP-1559 (DynamicFeeTx) transaction with the
// supplied parameters, asks the KMS signer for a signature over the
// EIP-155 signing hash for ChainID, attaches it, and returns the
// canonical-marshalled signed bytes plus the resulting tx hash.
//
// Pure of any RPC: caller is responsible for sourcing the gas params + nonce
// and for broadcasting `signedBytes` afterward.
func BuildAndSignTx(ctx context.Context, signer kms.Signer, p TxParams) (signedBytes []byte, txHash common.Hash, err error) {
	unsigned := types.NewTx(&types.DynamicFeeTx{
		ChainID:   p.ChainID,
		Nonce:     p.Nonce,
		GasTipCap: p.MaxPriorityFeeWei,
		GasFeeCap: p.MaxFeePerGasWei,
		Gas:       p.GasLimit,
		To:        &p.To,
		Value:     big.NewInt(0),
		Data:      p.Calldata,
	})

	gethSigner := types.LatestSignerForChainID(p.ChainID)
	digest := gethSigner.Hash(unsigned).Bytes()

	sigRSV, err := signer.SignDigest(ctx, digest)
	if err != nil {
		return nil, common.Hash{}, fmt.Errorf("sign digest: %w", err)
	}
	if len(sigRSV) != 65 {
		return nil, common.Hash{}, fmt.Errorf("sign returned %d bytes, want 65", len(sigRSV))
	}

	signed, err := unsigned.WithSignature(gethSigner, sigRSV)
	if err != nil {
		return nil, common.Hash{}, fmt.Errorf("attach signature: %w", err)
	}

	out, err := signed.MarshalBinary()
	if err != nil {
		return nil, common.Hash{}, fmt.Errorf("marshal signed tx: %w", err)
	}
	return out, signed.Hash(), nil
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/relayer/ -run TestBuildAndSignTx -v
```

Expected: PASS. Recovered sender matches the fake signer's address.

- [ ] **Step 5: Commit**

```bash
git add internal/service/airdrop/relayer/tx.go internal/service/airdrop/relayer/tx_test.go
git commit -m "feat(airdrop): add EIP-1559 tx builder + KMS-signed assembly"
```

---

## Task 6: Auto-init relayer nonce on first boot

**Why:** Per cleanup pass — `agent_relayer_state.next_nonce` is NULL after migration; on first boot the relayer queries `eth_getTransactionCount(relayer_address, 'pending')` and writes the result inside a `SELECT FOR UPDATE` txn. Subsequent boots see a non-null nonce and no-op. Removes a Day 28 manual-psql footgun.

**Files:**
- Create: `internal/service/airdrop/relayer/nonce.go`
- Create: `internal/service/airdrop/relayer/nonce_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/service/airdrop/relayer/nonce_test.go`:

```go
package relayer

import (
	"context"
	"errors"
	"math/big"
	"testing"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
)

// fakeRPCNonce implements the slice of ethclient.Client InitNonceIfNeeded uses.
type fakeRPCNonce struct {
	nonceAt func(ctx context.Context, addr common.Address, block *big.Int) (uint64, error)
	called  int
}

func (f *fakeRPCNonce) NonceAt(ctx context.Context, addr common.Address, block *big.Int) (uint64, error) {
	f.called++
	return f.nonceAt(ctx, addr, block)
}

// Stub the rest of the Client interface (unused by InitNonceIfNeeded) — return errors so
// any accidental call fails loudly.
func (f *fakeRPCNonce) LatestBaseFee(_ context.Context) (*big.Int, error)               { return nil, errors.New("unused") }
func (f *fakeRPCNonce) SuggestGasTipCap(_ context.Context) (*big.Int, error)            { return nil, errors.New("unused") }
func (f *fakeRPCNonce) SendRawTransaction(_ context.Context, _ []byte) (common.Hash, error) {
	return common.Hash{}, errors.New("unused")
}
func (f *fakeRPCNonce) GetTransactionReceipt(_ context.Context, _ common.Hash) (*types.Receipt, error) {
	return nil, errors.New("unused")
}
func (f *fakeRPCNonce) BalanceOf(_ context.Context, _, _ common.Address) (*big.Int, error) {
	return nil, errors.New("unused")
}
func (f *fakeRPCNonce) Balance(_ context.Context, _ common.Address) (*big.Int, error) {
	return nil, errors.New("unused")
}
func (f *fakeRPCNonce) EstimateGas(_ context.Context, _, _ common.Address, _ []byte, _ *big.Int) (uint64, error) {
	return 0, errors.New("unused")
}

func TestInitNonceIfNeeded_PopulatesWhenNil(t *testing.T) {
	pool := newNonceTestPool(t)               // helper that resets relayer_state to nil-nonce
	repo := postgres.NewRelayerStateRepository(pool)
	rpc := &fakeRPCNonce{
		nonceAt: func(_ context.Context, _ common.Address, _ *big.Int) (uint64, error) {
			return 17, nil
		},
	}

	addr := common.HexToAddress("0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266")
	if err := InitNonceIfNeeded(context.Background(), repo, rpc, addr); err != nil {
		t.Fatalf("InitNonceIfNeeded: %v", err)
	}

	st, _ := repo.Get(context.Background())
	if st.NextNonce == nil || *st.NextNonce != 17 {
		t.Errorf("after init, next_nonce = %v, want 17", st.NextNonce)
	}
	if rpc.called != 1 {
		t.Errorf("rpc.NonceAt called %d times, want 1", rpc.called)
	}
}

func TestInitNonceIfNeeded_NoOpWhenAlreadySet(t *testing.T) {
	pool := newNonceTestPool(t)
	if _, err := pool.Exec(context.Background(), `UPDATE agent_relayer_state SET next_nonce = 99 WHERE id = 1`); err != nil {
		t.Fatalf("seed: %v", err)
	}
	repo := postgres.NewRelayerStateRepository(pool)
	rpc := &fakeRPCNonce{
		nonceAt: func(_ context.Context, _ common.Address, _ *big.Int) (uint64, error) {
			t.Fatal("rpc.NonceAt should not be called when nonce is already populated")
			return 0, nil
		},
	}

	addr := common.HexToAddress("0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266")
	if err := InitNonceIfNeeded(context.Background(), repo, rpc, addr); err != nil {
		t.Fatalf("InitNonceIfNeeded: %v", err)
	}

	st, _ := repo.Get(context.Background())
	if st.NextNonce == nil || *st.NextNonce != 99 {
		t.Errorf("nonce changed to %v, want 99", st.NextNonce)
	}
}
```

The test imports `github.com/vultisig/agent-backend/internal/storage/postgres`. Reuse the `newNonceTestPool` helper from `airdrop_relayer_state_test.go` — copy/paste it into this test file with the same body, or extract into a shared `_test_util.go` if you'd rather avoid the duplication.

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/service/airdrop/relayer/ -run TestInitNonce -v
```

Expected: compile error — `InitNonceIfNeeded` undefined.

- [ ] **Step 3: Implement the auto-init helper**

Create `internal/service/airdrop/relayer/nonce.go`:

```go
package relayer

import (
	"context"
	"fmt"

	"github.com/ethereum/go-ethereum/common"
	"github.com/jackc/pgx/v5"

	"github.com/vultisig/agent-backend/internal/service/airdrop/ethclient"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
)

// InitNonceIfNeeded populates agent_relayer_state.next_nonce on first boot.
// Called from cmd/server/main.go and cmd/scheduler/main.go right after the
// repos are constructed. Subsequent boots no-op.
//
// Concurrency note: SELECT FOR UPDATE serialises against any other process
// (including a concurrent /airdrop/claim handler) — only one writer ever
// observes the nil nonce.
func InitNonceIfNeeded(
	ctx context.Context,
	repo *postgres.RelayerStateRepository,
	rpc ethclient.Client,
	relayerAddress common.Address,
) error {
	tx, err := repo.Pool().BeginTx(ctx, pgx.TxOptions{})
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	defer func() { _ = tx.Rollback(ctx) }()

	st, err := repo.LockWithTx(ctx, tx)
	if err != nil {
		return fmt.Errorf("lock relayer state: %w", err)
	}
	if st.NextNonce != nil {
		// Already set — nothing to do. Rollback (no-op) and exit.
		return nil
	}

	chainNonce, err := rpc.NonceAt(ctx, relayerAddress, nil) // nil = latest (== pending in geth client)
	if err != nil {
		return fmt.Errorf("query chain nonce: %w", err)
	}

	if err := repo.SetNonceWithTx(ctx, tx, int64(chainNonce)); err != nil {
		return fmt.Errorf("set nonce: %w", err)
	}
	if err := tx.Commit(ctx); err != nil {
		return fmt.Errorf("commit: %w", err)
	}
	return nil
}
```

Note: the spec asks for `eth_getTransactionCount(addr, 'pending')`. go-ethereum's `NonceAt(ctx, addr, nil)` calls `eth_getTransactionCount(addr, 'latest')`. The `pending` variant exists at `PendingNonceAt`. For the auto-init case the difference doesn't matter — the relayer wallet has zero in-flight transactions on first boot, so `latest == pending`. Using the existing wrapper avoids adding a one-off `PendingNonceAt` method to the wrapper just for this call site.

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/relayer/ -run TestInitNonce -v
```

Expected: PASS for both subtests.

- [ ] **Step 5: Commit**

```bash
git add internal/service/airdrop/relayer/nonce.go internal/service/airdrop/relayer/nonce_test.go
git commit -m "feat(airdrop): add auto-init nonce helper for relayer state"
```

---

## Task 7: Claim service — full submit pipeline

**Why:** The heart of Stage 4. Single function call from the HTTP handler: `service.Submit(ctx, publicKey)` runs the entire `BEGIN → SELECT FOR UPDATE → idempotency check → load winner → gas estimate → build → sign → broadcast → INSERT → UPDATE state → COMMIT` sequence. ROLLBACK on any failure. Tests exercise both serialisation (50 distinct users → 50 distinct nonces) and idempotency (50 same-user → 1 tx + 49 retries returning the same hash).

**Files:**
- Create: `internal/service/airdrop/relayer/claim.go`
- Create: `internal/service/airdrop/relayer/claim_test.go`

- [ ] **Step 1: Write the failing tests**

Create `internal/service/airdrop/relayer/claim_test.go`:

```go
package relayer

import (
	"context"
	"crypto/ecdsa"
	"errors"
	"math/big"
	"os"
	"sync"
	"testing"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/storage/postgres"
)

// ---- helpers ----

func newClaimServiceTestPool(t *testing.T) *pgxpool.Pool {
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
		TRUNCATE agent_claim_submissions;
		UPDATE agent_relayer_state SET next_nonce = 0 WHERE id = 1;
		TRUNCATE agent_raffle_winners;
	`)
	if err != nil {
		t.Fatalf("truncate: %v", err)
	}
	return pool
}

func seedWinner(t *testing.T, pool *pgxpool.Pool, pk, recipient string, amountWei *big.Int) {
	t.Helper()
	_, err := pool.Exec(context.Background(),
		`INSERT INTO agent_raffle_winners (public_key, recipient, amount) VALUES ($1, $2, $3::numeric)`,
		pk, recipient, amountWei.String())
	if err != nil {
		t.Fatalf("seed winner: %v", err)
	}
}

// fakeRPC implements ethclient.Client with controllable responses.
type fakeRPC struct {
	mu        sync.Mutex
	sentTxs   []common.Hash
	estimateGasFunc func() (uint64, error)
}

func (f *fakeRPC) LatestBaseFee(_ context.Context) (*big.Int, error) {
	return big.NewInt(20_000_000_000), nil // 20 gwei
}
func (f *fakeRPC) SuggestGasTipCap(_ context.Context) (*big.Int, error) {
	return big.NewInt(2_000_000_000), nil // 2 gwei
}
func (f *fakeRPC) NonceAt(_ context.Context, _ common.Address, _ *big.Int) (uint64, error) {
	return 0, nil
}
func (f *fakeRPC) SendRawTransaction(_ context.Context, signed []byte) (common.Hash, error) {
	f.mu.Lock()
	defer f.mu.Unlock()
	tx := new(types.Transaction)
	if err := tx.UnmarshalBinary(signed); err != nil {
		return common.Hash{}, err
	}
	f.sentTxs = append(f.sentTxs, tx.Hash())
	return tx.Hash(), nil
}
func (f *fakeRPC) GetTransactionReceipt(_ context.Context, _ common.Hash) (*types.Receipt, error) {
	return nil, errors.New("unused")
}
func (f *fakeRPC) BalanceOf(_ context.Context, _, _ common.Address) (*big.Int, error) {
	return nil, errors.New("unused")
}
func (f *fakeRPC) Balance(_ context.Context, _ common.Address) (*big.Int, error) {
	return nil, errors.New("unused")
}
func (f *fakeRPC) EstimateGas(_ context.Context, _, _ common.Address, _ []byte, _ *big.Int) (uint64, error) {
	if f.estimateGasFunc != nil {
		return f.estimateGasFunc()
	}
	return 80_000, nil
}

// localSigner mimics kms.Signer using an in-memory ecdsa key.
type localSigner struct{ priv *ecdsa.PrivateKey }

func newLocalSigner(t *testing.T) *localSigner {
	t.Helper()
	priv, err := crypto.GenerateKey()
	if err != nil {
		t.Fatalf("GenerateKey: %v", err)
	}
	return &localSigner{priv: priv}
}

func (l *localSigner) Address() common.Address              { return crypto.PubkeyToAddress(l.priv.PublicKey) }
func (l *localSigner) SignDigest(_ context.Context, d []byte) ([]byte, error) {
	return crypto.Sign(d, l.priv)
}

// ---- tests ----

func TestClaimService_Submit_Fresh(t *testing.T) {
	pool := newClaimServiceTestPool(t)
	claimRepo := postgres.NewClaimRepository(pool)
	stateRepo := postgres.NewRelayerStateRepository(pool)

	winnerPK := "pk-fresh"
	recipient := common.HexToAddress("0x70997970C51812dc3A010C7d01b50e0d17dc79C8")
	seedWinner(t, pool, winnerPK, recipient.Hex(), big.NewInt(1_500_000_000_000_000_000))

	svc := NewClaimService(ClaimServiceConfig{
		ChainID:         big.NewInt(1),
		ContractAddress: common.HexToAddress("0xCCCCCcCCCcCCCCcCcCCCCCcCcCCCcCCcCcCCCcCC"),
		ClaimRepo:       claimRepo,
		StateRepo:       stateRepo,
		WinnerLookup:    pgxWinnerLookup{pool: pool},
		RPC:             &fakeRPC{},
		Signer:          newLocalSigner(t),
	})

	row, err := svc.Submit(context.Background(), winnerPK)
	if err != nil {
		t.Fatalf("Submit: %v", err)
	}
	if row.Nonce != 0 {
		t.Errorf("nonce = %d, want 0 (first claim)", row.Nonce)
	}
	if row.TxHash == "" {
		t.Error("tx_hash empty")
	}

	// State nonce must have advanced
	st, _ := stateRepo.Get(context.Background())
	if st.NextNonce == nil || *st.NextNonce != 1 {
		t.Errorf("after submit, next_nonce = %v, want 1", st.NextNonce)
	}
}

func TestClaimService_Submit_IdempotentRetry(t *testing.T) {
	pool := newClaimServiceTestPool(t)
	claimRepo := postgres.NewClaimRepository(pool)
	stateRepo := postgres.NewRelayerStateRepository(pool)
	winnerPK := "pk-retry"
	recipient := common.HexToAddress("0x70997970C51812dc3A010C7d01b50e0d17dc79C8")
	seedWinner(t, pool, winnerPK, recipient.Hex(), big.NewInt(1_500_000_000_000_000_000))

	rpc := &fakeRPC{}
	svc := NewClaimService(ClaimServiceConfig{
		ChainID: big.NewInt(1), ContractAddress: common.HexToAddress("0xCCCC"),
		ClaimRepo: claimRepo, StateRepo: stateRepo,
		WinnerLookup: pgxWinnerLookup{pool: pool}, RPC: rpc, Signer: newLocalSigner(t),
	})

	first, _ := svc.Submit(context.Background(), winnerPK)
	second, err := svc.Submit(context.Background(), winnerPK)
	if err != nil {
		t.Fatalf("retry: %v", err)
	}
	if first.TxHash != second.TxHash {
		t.Errorf("retry returned %s, want first tx %s", second.TxHash, first.TxHash)
	}
	if len(rpc.sentTxs) != 1 {
		t.Errorf("rpc broadcasts = %d, want 1 (second call should not broadcast)", len(rpc.sentTxs))
	}
	st, _ := stateRepo.Get(context.Background())
	if *st.NextNonce != 1 {
		t.Errorf("nonce advanced too far: %v, want 1", *st.NextNonce)
	}
}

func TestClaimService_Submit_ConcurrentDistinctUsers_DistinctNonces(t *testing.T) {
	pool := newClaimServiceTestPool(t)
	claimRepo := postgres.NewClaimRepository(pool)
	stateRepo := postgres.NewRelayerStateRepository(pool)

	const N = 50
	for i := 0; i < N; i++ {
		pk := "pk-conc-" + string(rune('a'+i))
		seedWinner(t, pool, pk, "0x70997970C51812dc3A010C7d01b50e0d17dc79C8", big.NewInt(1500))
	}

	svc := NewClaimService(ClaimServiceConfig{
		ChainID: big.NewInt(1), ContractAddress: common.HexToAddress("0xCCCC"),
		ClaimRepo: claimRepo, StateRepo: stateRepo,
		WinnerLookup: pgxWinnerLookup{pool: pool}, RPC: &fakeRPC{}, Signer: newLocalSigner(t),
	})

	var wg sync.WaitGroup
	results := make([]*ClaimResult, N)
	errs := make([]error, N)
	for i := 0; i < N; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			pk := "pk-conc-" + string(rune('a'+i))
			results[i], errs[i] = svc.Submit(context.Background(), pk)
		}(i)
	}
	wg.Wait()

	seenNonces := map[int64]bool{}
	for i := 0; i < N; i++ {
		if errs[i] != nil {
			t.Errorf("user %d: %v", i, errs[i])
			continue
		}
		if seenNonces[results[i].Nonce] {
			t.Errorf("duplicate nonce %d (users %d)", results[i].Nonce, i)
		}
		seenNonces[results[i].Nonce] = true
	}
	if len(seenNonces) != N {
		t.Errorf("got %d distinct nonces, want %d", len(seenNonces), N)
	}
}

func TestClaimService_Submit_ConcurrentSameUser_SingleTx(t *testing.T) {
	pool := newClaimServiceTestPool(t)
	claimRepo := postgres.NewClaimRepository(pool)
	stateRepo := postgres.NewRelayerStateRepository(pool)
	winnerPK := "pk-same"
	seedWinner(t, pool, winnerPK, "0x70997970C51812dc3A010C7d01b50e0d17dc79C8", big.NewInt(1500))

	rpc := &fakeRPC{}
	svc := NewClaimService(ClaimServiceConfig{
		ChainID: big.NewInt(1), ContractAddress: common.HexToAddress("0xCCCC"),
		ClaimRepo: claimRepo, StateRepo: stateRepo,
		WinnerLookup: pgxWinnerLookup{pool: pool}, RPC: rpc, Signer: newLocalSigner(t),
	})

	const N = 50
	var wg sync.WaitGroup
	hashes := make([]string, N)
	errs := make([]error, N)
	for i := 0; i < N; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			r, err := svc.Submit(context.Background(), winnerPK)
			errs[i] = err
			if r != nil {
				hashes[i] = r.TxHash
			}
		}(i)
	}
	wg.Wait()

	// All errors should be nil; all hashes should match.
	first := hashes[0]
	for i := 0; i < N; i++ {
		if errs[i] != nil {
			t.Errorf("call %d: %v", i, errs[i])
		}
		if hashes[i] != first {
			t.Errorf("call %d hash %s != first %s", i, hashes[i], first)
		}
	}
	if len(rpc.sentTxs) != 1 {
		t.Errorf("rpc broadcasts = %d, want 1 (49 must be idempotent retries)", len(rpc.sentTxs))
	}
}

func TestClaimService_Submit_NotAWinner(t *testing.T) {
	pool := newClaimServiceTestPool(t)
	svc := NewClaimService(ClaimServiceConfig{
		ChainID: big.NewInt(1), ContractAddress: common.HexToAddress("0xCCCC"),
		ClaimRepo: postgres.NewClaimRepository(pool),
		StateRepo: postgres.NewRelayerStateRepository(pool),
		WinnerLookup: pgxWinnerLookup{pool: pool},
		RPC: &fakeRPC{}, Signer: newLocalSigner(t),
	})

	_, err := svc.Submit(context.Background(), "pk-not-winner")
	if !errors.Is(err, ErrNotAWinner) {
		t.Errorf("err = %v, want ErrNotAWinner", err)
	}
}

func TestClaimService_Submit_RPCFailure_RollsBackNonce(t *testing.T) {
	pool := newClaimServiceTestPool(t)
	winnerPK := "pk-rpc-fail"
	seedWinner(t, pool, winnerPK, "0x70997970C51812dc3A010C7d01b50e0d17dc79C8", big.NewInt(1500))

	rpc := &fakeRPC{estimateGasFunc: func() (uint64, error) { return 0, errors.New("rpc down") }}
	svc := NewClaimService(ClaimServiceConfig{
		ChainID: big.NewInt(1), ContractAddress: common.HexToAddress("0xCCCC"),
		ClaimRepo: postgres.NewClaimRepository(pool),
		StateRepo: postgres.NewRelayerStateRepository(pool),
		WinnerLookup: pgxWinnerLookup{pool: pool},
		RPC: rpc, Signer: newLocalSigner(t),
	})

	_, err := svc.Submit(context.Background(), winnerPK)
	if !errors.Is(err, ErrRPCUnavailable) {
		t.Errorf("err = %v, want ErrRPCUnavailable", err)
	}
	st, _ := postgres.NewRelayerStateRepository(pool).Get(context.Background())
	if st.NextNonce == nil || *st.NextNonce != 0 {
		t.Errorf("nonce advanced after rollback: %v, want 0", st.NextNonce)
	}
}
```

The test references a `pgxWinnerLookup` adapter — see Step 4 below where it's introduced as a tiny shim around the pool so the service can take an interface (not the concrete pool) and tests can pass a mock if they need to.

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/service/airdrop/relayer/ -run TestClaimService -v
```

Expected: compile error — `NewClaimService`, `ClaimServiceConfig`, `ClaimResult`, `ErrNotAWinner`, `ErrRPCUnavailable`, `pgxWinnerLookup` undefined.

- [ ] **Step 3: Implement the claim service**

Create `internal/service/airdrop/relayer/claim.go`:

```go
package relayer

import (
	"context"
	"errors"
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/service/airdrop/ethclient"
	"github.com/vultisig/agent-backend/internal/service/airdrop/kms"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
	"github.com/vultisig/agent-backend/internal/types"
)

// Sentinel errors. The HTTP handler maps these to the documented status codes.
var (
	ErrNotAWinner       = errors.New("public key is not in agent_raffle_winners")
	ErrRPCUnavailable   = errors.New("rpc unavailable or returned a transient error")
	ErrKMSUnavailable   = errors.New("kms sign call failed")
	ErrBroadcastReject  = errors.New("rpc accepted call but rejected the tx (e.g. nonce reuse, insufficient funds)")
)

// WinnerLookup returns (recipient, amount) for a public key, or ErrNotAWinner.
// Defined as an interface so claim_test.go can pass a stub. Production uses pgxWinnerLookup.
type WinnerLookup interface {
	Lookup(ctx context.Context, publicKey string) (recipient common.Address, amount *big.Int, err error)
}

type pgxWinnerLookup struct{ pool *pgxpool.Pool }

func NewPgxWinnerLookup(pool *pgxpool.Pool) WinnerLookup { return pgxWinnerLookup{pool: pool} }

func (p pgxWinnerLookup) Lookup(ctx context.Context, publicKey string) (common.Address, *big.Int, error) {
	var recipient string
	var amountStr string
	err := p.pool.QueryRow(ctx,
		`SELECT recipient, amount::text FROM agent_raffle_winners WHERE public_key = $1`,
		publicKey,
	).Scan(&recipient, &amountStr)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return common.Address{}, nil, ErrNotAWinner
		}
		return common.Address{}, nil, fmt.Errorf("lookup winner: %w", err)
	}
	amount, ok := new(big.Int).SetString(amountStr, 10)
	if !ok {
		return common.Address{}, nil, fmt.Errorf("parse amount %q", amountStr)
	}
	return common.HexToAddress(recipient), amount, nil
}

// ClaimResult is the data returned to the HTTP layer.
type ClaimResult struct {
	TxHash      string
	Status      types.ClaimStatus
	Nonce       int64
	SubmittedAt string // RFC3339 — caller renders into JSON
	ConfirmedAt string // RFC3339, empty unless idempotent retry hit a confirmed row
}

// ClaimServiceConfig wires the dependencies the service needs.
type ClaimServiceConfig struct {
	ChainID         *big.Int
	ContractAddress common.Address
	ClaimRepo       *postgres.ClaimRepository
	StateRepo       *postgres.RelayerStateRepository
	WinnerLookup    WinnerLookup
	RPC             ethclient.Client
	Signer          kms.Signer
}

type ClaimService struct {
	cfg ClaimServiceConfig
}

func NewClaimService(cfg ClaimServiceConfig) *ClaimService { return &ClaimService{cfg: cfg} }

// Submit runs the full claim pipeline under one Postgres transaction.
//
// Sequence (matches stage-4-claim-relayer.md "Behavior" steps 5-17):
//   1. BEGIN
//   2. SELECT FOR UPDATE on agent_relayer_state WHERE id=1
//   3. SELECT existing claim row for this public_key (idempotent retry → return)
//   4. Look up (recipient, amount) from agent_raffle_winners
//   5. Fetch baseFee + tipCap from RPC; maxFee = 2*baseFee + tip
//   6. Build calldata for claim(recipient)
//   7. EstimateGas
//   8. BuildAndSignTx (KMS sign)
//   9. SendRawTransaction
//  10. INSERT agent_claim_submissions row
//  11. UPDATE agent_relayer_state SET next_nonce = next_nonce + 1
//  12. COMMIT
//
// Any error from steps 4-9 → ROLLBACK (state unchanged, no claim row, user can retry).
func (s *ClaimService) Submit(ctx context.Context, publicKey string) (*ClaimResult, error) {
	tx, err := s.cfg.StateRepo.Pool().BeginTx(ctx, pgx.TxOptions{})
	if err != nil {
		return nil, fmt.Errorf("begin tx: %w", err)
	}
	defer func() { _ = tx.Rollback(ctx) }()

	// 2. Lock the state row — serialises all claim handlers.
	state, err := s.cfg.StateRepo.LockWithTx(ctx, tx)
	if err != nil {
		return nil, fmt.Errorf("lock relayer state: %w", err)
	}
	if state.NextNonce == nil {
		return nil, fmt.Errorf("relayer next_nonce is nil — InitNonceIfNeeded must run before Submit")
	}

	// 3. Idempotent-retry check (inside the same tx so we see committed rows from racing siblings).
	if existing, err := s.cfg.ClaimRepo.GetActiveByPublicKeyWithTx(ctx, tx, publicKey); err == nil {
		// Found — return the existing row, don't broadcast.
		if cErr := tx.Commit(ctx); cErr != nil {
			return nil, fmt.Errorf("commit (idempotent path): %w", cErr)
		}
		return claimResultFromRow(existing), nil
	} else if !errors.Is(err, postgres.ErrNotFound) {
		return nil, fmt.Errorf("idempotent-retry check: %w", err)
	}

	nonce := uint64(*state.NextNonce)

	// 4. Winner lookup.
	recipient, amount, err := s.cfg.WinnerLookup.Lookup(ctx, publicKey)
	if err != nil {
		return nil, err // ErrNotAWinner or wrapped
	}

	// 5. Gas estimates. Map any RPC error to ErrRPCUnavailable so the handler can return 503.
	baseFee, err := s.cfg.RPC.LatestBaseFee(ctx)
	if err != nil {
		return nil, fmt.Errorf("%w: base fee: %v", ErrRPCUnavailable, err)
	}
	tipCap, err := s.cfg.RPC.SuggestGasTipCap(ctx)
	if err != nil {
		return nil, fmt.Errorf("%w: tip cap: %v", ErrRPCUnavailable, err)
	}
	maxFee := new(big.Int).Add(new(big.Int).Mul(baseFee, big.NewInt(2)), tipCap)

	// 6. Calldata.
	calldata := BuildClaimCalldata(recipient)

	// 7. Gas estimate.
	gasLimit, err := s.cfg.RPC.EstimateGas(ctx, s.cfg.Signer.Address(), s.cfg.ContractAddress, calldata, nil)
	if err != nil {
		return nil, fmt.Errorf("%w: estimate gas: %v", ErrRPCUnavailable, err)
	}
	// Pad 20% to absorb intra-block variance.
	gasLimit = gasLimit + gasLimit/5

	// 8. Build + KMS-sign the EIP-1559 tx.
	signedBytes, txHash, err := BuildAndSignTx(ctx, s.cfg.Signer, TxParams{
		ChainID:           s.cfg.ChainID,
		Nonce:             nonce,
		GasLimit:          gasLimit,
		MaxFeePerGasWei:   maxFee,
		MaxPriorityFeeWei: tipCap,
		To:                s.cfg.ContractAddress,
		Calldata:          calldata,
	})
	if err != nil {
		return nil, fmt.Errorf("%w: %v", ErrKMSUnavailable, err)
	}

	// 9. Broadcast.
	if _, err := s.cfg.RPC.SendRawTransaction(ctx, signedBytes); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrBroadcastReject, err)
	}

	// 10. Insert the claim row inside the same tx.
	maxFeeGwei := int32(new(big.Int).Div(maxFee, big.NewInt(1_000_000_000)).Int64())
	tipGwei := int32(new(big.Int).Div(tipCap, big.NewInt(1_000_000_000)).Int64())
	row, err := s.cfg.ClaimRepo.InsertWithTx(ctx, tx,
		publicKey, int64(nonce), recipient.Hex(), amount, txHash.Hex(),
		maxFeeGwei, tipGwei,
	)
	if err != nil {
		return nil, fmt.Errorf("insert claim row: %w", err)
	}

	// 11. Advance the state nonce.
	if err := s.cfg.StateRepo.SetNonceWithTx(ctx, tx, int64(nonce)+1); err != nil {
		return nil, fmt.Errorf("advance nonce: %w", err)
	}

	// 12. Commit.
	if err := tx.Commit(ctx); err != nil {
		return nil, fmt.Errorf("commit: %w", err)
	}

	return claimResultFromRow(row), nil
}

func claimResultFromRow(row *types.ClaimSubmission) *ClaimResult {
	out := &ClaimResult{
		TxHash:      row.TxHash,
		Status:      row.Status,
		Nonce:       row.Nonce,
		SubmittedAt: row.SubmittedAt.Format("2006-01-02T15:04:05Z07:00"),
	}
	if row.ConfirmedAt != nil {
		out.ConfirmedAt = row.ConfirmedAt.Format("2006-01-02T15:04:05Z07:00")
	}
	return out
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/relayer/ -run TestClaimService -v -timeout 60s
```

Expected: PASS for all subtests, given `DATABASE_DSN` is set. The two concurrent tests (`...DistinctUsers...` and `...SameUser...`) take a few seconds each because they actually serialise through Postgres locks.

- [ ] **Step 5: Commit**

```bash
git add internal/service/airdrop/relayer/claim.go internal/service/airdrop/relayer/claim_test.go
git commit -m "feat(airdrop): add claim service with FOR UPDATE serialisation + idempotent retry"
```

---

## Task 8: `POST /airdrop/claim` HTTP handler

**Why:** Wraps the service. Window check, kill-switch check, eligibility check (in-process call to `quests.IsClaimEligible`), then delegates to `service.Submit`. Maps service errors to the documented status codes (`stage-4-claim-relayer.md:74-85`).

**Files:**
- Create: `internal/api/airdrop_claim.go`
- Create: `internal/api/airdrop_claim_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/api/airdrop_claim_test.go`:

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

	"github.com/vultisig/agent-backend/internal/service/airdrop/relayer"
	"github.com/vultisig/agent-backend/internal/types"
)

type fakeClaimSvc struct {
	submit func(ctx context.Context, publicKey string) (*relayer.ClaimResult, error)
}

func (f *fakeClaimSvc) Submit(ctx context.Context, publicKey string) (*relayer.ClaimResult, error) {
	return f.submit(ctx, publicKey)
}

type fakeEligibility struct {
	eligible bool
	reason   string
	err      error
}

func (f *fakeEligibility) IsClaimEligible(_ context.Context, _ string, _ int) (bool, string, error) {
	return f.eligible, f.reason, f.err
}

func TestClaimHandler_StatusCodeMatrix(t *testing.T) {
	openAt := time.Date(2026, 5, 15, 0, 0, 0, 0, time.UTC)
	now := func(after time.Duration) time.Time { return openAt.Add(after) }

	cases := []struct {
		name        string
		clock       time.Time
		killSwitch  bool
		elig        *fakeEligibility
		submitErr   error
		submitOK    *relayer.ClaimResult
		wantStatus  int
		wantErrKey  string
	}{
		{
			name: "before window → 403 CLAIM_WINDOW_NOT_OPEN",
			clock: openAt.Add(-time.Hour), killSwitch: true,
			elig: &fakeEligibility{eligible: true},
			wantStatus: 403, wantErrKey: "CLAIM_WINDOW_NOT_OPEN",
		},
		{
			name: "kill switch flipped → 403 CLAIM_DISABLED",
			clock: now(time.Hour), killSwitch: false,
			elig: &fakeEligibility{eligible: true},
			wantStatus: 403, wantErrKey: "CLAIM_DISABLED",
		},
		{
			name: "not eligible → 403 NOT_CLAIM_ELIGIBLE",
			clock: now(time.Hour), killSwitch: true,
			elig: &fakeEligibility{eligible: false, reason: "only_2_quests_complete"},
			wantStatus: 403, wantErrKey: "NOT_CLAIM_ELIGIBLE",
		},
		{
			name: "RPC error → 503 RPC_UNAVAILABLE",
			clock: now(time.Hour), killSwitch: true,
			elig: &fakeEligibility{eligible: true},
			submitErr: relayer.ErrRPCUnavailable,
			wantStatus: 503, wantErrKey: "RPC_UNAVAILABLE",
		},
		{
			name: "KMS error → 503 KMS_UNAVAILABLE",
			clock: now(time.Hour), killSwitch: true,
			elig: &fakeEligibility{eligible: true},
			submitErr: relayer.ErrKMSUnavailable,
			wantStatus: 503, wantErrKey: "KMS_UNAVAILABLE",
		},
		{
			name: "broadcast rejected → 500 BROADCAST_REJECTED",
			clock: now(time.Hour), killSwitch: true,
			elig: &fakeEligibility{eligible: true},
			submitErr: relayer.ErrBroadcastReject,
			wantStatus: 500, wantErrKey: "BROADCAST_REJECTED",
		},
		{
			name: "happy path → 200 with tx_hash",
			clock: now(time.Hour), killSwitch: true,
			elig: &fakeEligibility{eligible: true},
			submitOK: &relayer.ClaimResult{TxHash: "0xabc", Status: types.ClaimSubmitted, Nonce: 7, SubmittedAt: "2026-05-15T18:00:00Z"},
			wantStatus: 200,
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			svc := &fakeClaimSvc{submit: func(_ context.Context, _ string) (*relayer.ClaimResult, error) {
				if tc.submitErr != nil {
					return nil, tc.submitErr
				}
				return tc.submitOK, nil
			}}
			h := &claimHandler{
				svc:               svc,
				eligibility:       tc.elig,
				questThreshold:    3,
				claimWindowOpenAt: openAt,
				claimEnabled:      tc.killSwitch,
				now:               func() time.Time { return tc.clock },
			}

			e := echo.New()
			req := httptest.NewRequest(http.MethodPost, "/airdrop/claim", nil)
			rec := httptest.NewRecorder()
			c := e.NewContext(req, rec)
			c.Set("public_key", "pk-test")

			if err := h.Handle(c); err != nil {
				t.Fatalf("handler: %v", err)
			}
			if rec.Code != tc.wantStatus {
				t.Errorf("status = %d, want %d (body=%s)", rec.Code, tc.wantStatus, rec.Body.String())
			}
			if tc.wantErrKey != "" {
				var body map[string]any
				_ = json.Unmarshal(rec.Body.Bytes(), &body)
				if body["error"] != tc.wantErrKey {
					t.Errorf("error = %v, want %s", body["error"], tc.wantErrKey)
				}
			}
			if tc.wantStatus == 200 {
				var body map[string]any
				_ = json.Unmarshal(rec.Body.Bytes(), &body)
				if body["tx_hash"] != "0xabc" {
					t.Errorf("tx_hash = %v, want 0xabc", body["tx_hash"])
				}
				if body["status"] != "submitted" {
					t.Errorf("status = %v, want submitted", body["status"])
				}
			}
		})
	}

	// Defensive: ensure no unused imports
	_ = errors.New
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/api/ -run TestClaimHandler -v
```

Expected: compile error — `claimHandler` undefined.

- [ ] **Step 3: Implement the handler**

Create `internal/api/airdrop_claim.go`:

```go
package api

import (
	"context"
	"errors"
	"net/http"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/vultisig/agent-backend/internal/service/airdrop/relayer"
)

// claimSubmitter is the slice of ClaimService the handler depends on.
type claimSubmitter interface {
	Submit(ctx context.Context, publicKey string) (*relayer.ClaimResult, error)
}

// eligibilityChecker is the slice of quests.IsClaimEligible the handler uses.
// Wrapped behind an interface so tests don't need a real DB.
type eligibilityChecker interface {
	IsClaimEligible(ctx context.Context, publicKey string, threshold int) (bool, string, error)
}

type claimHandler struct {
	svc               claimSubmitter
	eligibility       eligibilityChecker
	questThreshold    int
	claimWindowOpenAt time.Time
	claimEnabled      bool
	now               func() time.Time
}

type claimResponse struct {
	TxHash      string `json:"tx_hash"`
	Status      string `json:"status"`
	Nonce       int64  `json:"nonce"`
	SubmittedAt string `json:"submitted_at"`
	ConfirmedAt string `json:"confirmed_at,omitempty"`
}

type broadcastRejectedResponse struct {
	Error  string `json:"error"`
	Detail string `json:"detail"`
}

func (h *claimHandler) Handle(c echo.Context) error {
	publicKey := GetPublicKey(c)
	if publicKey == "" {
		return c.JSON(http.StatusUnauthorized, ErrorResponse{Error: "missing public key"})
	}

	// 1. Window check.
	if h.now().Before(h.claimWindowOpenAt) {
		return c.JSON(http.StatusForbidden, ErrorResponse{Error: "CLAIM_WINDOW_NOT_OPEN"})
	}

	// 2. Kill switch.
	if !h.claimEnabled {
		return c.JSON(http.StatusForbidden, ErrorResponse{Error: "CLAIM_DISABLED"})
	}

	// 3. Eligibility (DB read; in-process; not an HTTP roundtrip).
	eligible, _, err := h.eligibility.IsClaimEligible(c.Request().Context(), publicKey, h.questThreshold)
	if err != nil {
		return c.JSON(http.StatusInternalServerError, ErrorResponse{Error: "ELIGIBILITY_LOOKUP_FAILED"})
	}
	if !eligible {
		return c.JSON(http.StatusForbidden, ErrorResponse{Error: "NOT_CLAIM_ELIGIBLE"})
	}

	// 4. Submit.
	result, err := h.svc.Submit(c.Request().Context(), publicKey)
	if err != nil {
		switch {
		case errors.Is(err, relayer.ErrNotAWinner):
			return c.JSON(http.StatusForbidden, ErrorResponse{Error: "NOT_CLAIM_ELIGIBLE"})
		case errors.Is(err, relayer.ErrRPCUnavailable):
			return c.JSON(http.StatusServiceUnavailable, ErrorResponse{Error: "RPC_UNAVAILABLE"})
		case errors.Is(err, relayer.ErrKMSUnavailable):
			return c.JSON(http.StatusServiceUnavailable, ErrorResponse{Error: "KMS_UNAVAILABLE"})
		case errors.Is(err, relayer.ErrBroadcastReject):
			return c.JSON(http.StatusInternalServerError, broadcastRejectedResponse{
				Error:  "BROADCAST_REJECTED",
				Detail: err.Error(),
			})
		default:
			return c.JSON(http.StatusInternalServerError, ErrorResponse{Error: "INTERNAL_ERROR"})
		}
	}

	return c.JSON(http.StatusOK, claimResponse{
		TxHash:      result.TxHash,
		Status:      string(result.Status),
		Nonce:       result.Nonce,
		SubmittedAt: result.SubmittedAt,
		ConfirmedAt: result.ConfirmedAt,
	})
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/api/ -run TestClaimHandler -v
```

Expected: PASS for all subtests.

- [ ] **Step 5: Commit**

```bash
git add internal/api/airdrop_claim.go internal/api/airdrop_claim_test.go
git commit -m "feat(airdrop): add POST /airdrop/claim handler"
```

---

## Task 9: Confirmation monitor — goroutine inside `cmd/scheduler`

**Why:** A single background loop that polls submitted-but-unconfirmed claims every `CONFIRMATION_POLL_INTERVAL_SECONDS` and flips them to confirmed/failed against chain receipts. The spec is explicit: *"Lives in `cmd/scheduler/` alongside the existing `agent_scheduled_tasks` poller (reuses the existing scheduler binary; no new process)"* (`stage-4-claim-relayer.md:203`). Restart-safety comes free: the monitor's first tick on each boot picks up any rows still in `submitted` from before the crash.

**Files:**
- Create: `internal/service/airdrop/relayer/monitor.go`
- Create: `internal/service/airdrop/relayer/monitor_test.go`
- Modify: `cmd/scheduler/main.go` — wire and launch the monitor via `safego.Go`

- [ ] **Step 1: Write the failing test**

Create `internal/service/airdrop/relayer/monitor_test.go`:

```go
package relayer

import (
	"context"
	"errors"
	"math/big"
	"os"
	"testing"
	"time"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/storage/postgres"
	tt "github.com/vultisig/agent-backend/internal/types"
)

func newMonitorTestPool(t *testing.T) *pgxpool.Pool {
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
	_, err = pool.Exec(context.Background(), `TRUNCATE agent_claim_submissions;`)
	if err != nil {
		t.Fatalf("truncate: %v", err)
	}
	return pool
}

// fakeReceiptRPC drives the monitor's GetTransactionReceipt path.
type fakeReceiptRPC struct {
	receipts map[common.Hash]*types.Receipt
}

func (f *fakeReceiptRPC) LatestBaseFee(_ context.Context) (*big.Int, error) { return nil, errors.New("unused") }
func (f *fakeReceiptRPC) SuggestGasTipCap(_ context.Context) (*big.Int, error) { return nil, errors.New("unused") }
func (f *fakeReceiptRPC) NonceAt(_ context.Context, _ common.Address, _ *big.Int) (uint64, error) { return 0, errors.New("unused") }
func (f *fakeReceiptRPC) SendRawTransaction(_ context.Context, _ []byte) (common.Hash, error) { return common.Hash{}, errors.New("unused") }
func (f *fakeReceiptRPC) GetTransactionReceipt(_ context.Context, h common.Hash) (*types.Receipt, error) {
	if r, ok := f.receipts[h]; ok {
		return r, nil
	}
	return nil, ethereum.NotFound
}
func (f *fakeReceiptRPC) BalanceOf(_ context.Context, _, _ common.Address) (*big.Int, error) { return nil, errors.New("unused") }
func (f *fakeReceiptRPC) Balance(_ context.Context, _ common.Address) (*big.Int, error) { return nil, errors.New("unused") }
func (f *fakeReceiptRPC) EstimateGas(_ context.Context, _, _ common.Address, _ []byte, _ *big.Int) (uint64, error) { return 0, errors.New("unused") }

func seedSubmittedClaim(t *testing.T, pool *pgxpool.Pool, repo *postgres.ClaimRepository, pk string, nonce int64, txHash string) {
	t.Helper()
	_, err := repo.Insert(context.Background(), pk, nonce, "0xaaa", big.NewInt(1500), txHash, 50, 2)
	if err != nil {
		t.Fatalf("seed: %v", err)
	}
}

func TestMonitor_OneTick_FlipsConfirmed(t *testing.T) {
	pool := newMonitorTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	hash := common.HexToHash("0xabc1")
	seedSubmittedClaim(t, pool, repo, "pk1", 1, hash.Hex())

	rpc := &fakeReceiptRPC{
		receipts: map[common.Hash]*types.Receipt{
			hash: {Status: types.ReceiptStatusSuccessful, BlockNumber: big.NewInt(19_400_000), TxHash: hash},
		},
	}
	mon := NewMonitor(MonitorConfig{
		ClaimRepo:           repo,
		RPC:                 rpc,
		PollInterval:        100 * time.Millisecond,
		BatchLimit:          100,
		Logger:              logrus.New(),
	})

	if err := mon.Tick(context.Background()); err != nil {
		t.Fatalf("Tick: %v", err)
	}

	row, _ := repo.GetActiveByPublicKey(context.Background(), "pk1")
	if row.Status != tt.ClaimConfirmed {
		t.Errorf("status = %s, want confirmed", row.Status)
	}
	if row.BlockNumber == nil || *row.BlockNumber != 19_400_000 {
		t.Errorf("block_number = %v, want 19_400_000", row.BlockNumber)
	}
}

func TestMonitor_OneTick_FlipsFailedOnRevert(t *testing.T) {
	pool := newMonitorTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	hash := common.HexToHash("0xabc2")
	seedSubmittedClaim(t, pool, repo, "pk2", 2, hash.Hex())

	rpc := &fakeReceiptRPC{
		receipts: map[common.Hash]*types.Receipt{
			hash: {Status: types.ReceiptStatusFailed, BlockNumber: big.NewInt(19_400_001), TxHash: hash},
		},
	}
	mon := NewMonitor(MonitorConfig{ClaimRepo: repo, RPC: rpc, PollInterval: 100 * time.Millisecond, BatchLimit: 100, Logger: logrus.New()})

	if err := mon.Tick(context.Background()); err != nil {
		t.Fatalf("Tick: %v", err)
	}

	// Failed rows are no longer "active" (status != submitted/confirmed)
	if _, err := repo.GetActiveByPublicKey(context.Background(), "pk2"); !errors.Is(err, postgres.ErrNotFound) {
		t.Errorf("expected ErrNotFound for failed row, got %v", err)
	}
}

func TestMonitor_OneTick_NoReceiptYet_LeavesAsSubmitted(t *testing.T) {
	pool := newMonitorTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	seedSubmittedClaim(t, pool, repo, "pk3", 3, "0xpending")

	mon := NewMonitor(MonitorConfig{ClaimRepo: repo, RPC: &fakeReceiptRPC{}, PollInterval: 100 * time.Millisecond, BatchLimit: 100, Logger: logrus.New()})

	if err := mon.Tick(context.Background()); err != nil {
		t.Fatalf("Tick: %v", err)
	}
	row, _ := repo.GetActiveByPublicKey(context.Background(), "pk3")
	if row.Status != tt.ClaimSubmitted {
		t.Errorf("status = %s, want submitted (no receipt yet)", row.Status)
	}
}

func TestMonitor_RestartSafety(t *testing.T) {
	// Simulates: service crashed after a broadcast left a submitted row in DB.
	// On next boot, monitor's first Tick must catch the row even though no
	// "live" handler is around to nudge it.
	pool := newMonitorTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	hash := common.HexToHash("0xrestart")
	seedSubmittedClaim(t, pool, repo, "pk-restart", 99, hash.Hex())

	rpc := &fakeReceiptRPC{receipts: map[common.Hash]*types.Receipt{
		hash: {Status: types.ReceiptStatusSuccessful, BlockNumber: big.NewInt(19_400_002), TxHash: hash},
	}}
	mon := NewMonitor(MonitorConfig{ClaimRepo: repo, RPC: rpc, PollInterval: 100 * time.Millisecond, BatchLimit: 100, Logger: logrus.New()})

	// Run() with a short-lived ctx — should pick up the row on its initial boot tick.
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()
	_ = mon.Run(ctx) // Run blocks until ctx done; returns ctx.Err()

	row, _ := repo.GetActiveByPublicKey(context.Background(), "pk-restart")
	if row.Status != tt.ClaimConfirmed {
		t.Errorf("after Run, status = %s, want confirmed", row.Status)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/service/airdrop/relayer/ -run TestMonitor -v
```

Expected: compile error — `Monitor`, `MonitorConfig`, `NewMonitor` undefined.

- [ ] **Step 3: Implement the monitor**

Create `internal/service/airdrop/relayer/monitor.go`:

```go
package relayer

import (
	"context"
	"errors"
	"fmt"
	"time"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/service/airdrop/ethclient"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
)

// MonitorConfig wires the dependencies the confirmation monitor needs.
type MonitorConfig struct {
	ClaimRepo    *postgres.ClaimRepository
	RPC          ethclient.Client
	PollInterval time.Duration // CONFIRMATION_POLL_INTERVAL_SECONDS, e.g. 15s
	BatchLimit   int32         // max rows scanned per tick (spec: 100)
	Logger       *logrus.Logger
}

// Monitor is the background goroutine that flips submitted claims to
// confirmed/failed by polling chain receipts. Lives inside cmd/scheduler.
type Monitor struct {
	cfg MonitorConfig
}

func NewMonitor(cfg MonitorConfig) *Monitor {
	if cfg.BatchLimit == 0 {
		cfg.BatchLimit = 100
	}
	return &Monitor{cfg: cfg}
}

// Run loops forever. Runs Tick once at startup (catches anything left over
// from a crash) then on each tick of cfg.PollInterval.
func (m *Monitor) Run(ctx context.Context) error {
	if m.cfg.PollInterval <= 0 {
		return fmt.Errorf("invalid poll interval: %v", m.cfg.PollInterval)
	}
	m.cfg.Logger.WithField("poll_interval", m.cfg.PollInterval).Info("airdrop confirmation monitor started")

	// Boot tick — guarantees no submitted row is forgotten across restarts.
	if err := m.Tick(ctx); err != nil {
		m.cfg.Logger.WithError(err).Warn("airdrop monitor: initial tick failed")
	}

	ticker := time.NewTicker(m.cfg.PollInterval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			m.cfg.Logger.Info("airdrop confirmation monitor stopping")
			return ctx.Err()
		case <-ticker.C:
			if err := m.Tick(ctx); err != nil {
				m.cfg.Logger.WithError(err).Warn("airdrop monitor: tick failed")
			}
		}
	}
}

// Tick runs one scan of submitted-state rows. Exposed for tests + the boot tick.
func (m *Monitor) Tick(ctx context.Context) error {
	pending, err := m.cfg.ClaimRepo.ListPending(ctx, m.cfg.BatchLimit)
	if err != nil {
		return fmt.Errorf("list pending: %w", err)
	}
	if len(pending) == 0 {
		return nil
	}

	for _, row := range pending {
		if ctx.Err() != nil {
			return ctx.Err()
		}
		txHash := common.HexToHash(row.TxHash)
		receipt, err := m.cfg.RPC.GetTransactionReceipt(ctx, txHash)
		if errors.Is(err, ethereum.NotFound) {
			// No receipt yet — try again next tick. Common case.
			continue
		}
		if err != nil {
			m.cfg.Logger.WithError(err).WithFields(logrus.Fields{
				"nonce":   row.Nonce,
				"tx_hash": row.TxHash,
			}).Warn("airdrop monitor: receipt lookup failed")
			continue
		}
		m.processReceipt(ctx, row.Nonce, receipt)
	}
	return nil
}

func (m *Monitor) processReceipt(ctx context.Context, nonce int64, r *types.Receipt) {
	log := m.cfg.Logger.WithFields(logrus.Fields{
		"nonce":        nonce,
		"tx_hash":      r.TxHash.Hex(),
		"block_number": r.BlockNumber.Int64(),
	})
	switch r.Status {
	case types.ReceiptStatusSuccessful:
		if err := m.cfg.ClaimRepo.MarkConfirmed(ctx, nonce, r.BlockNumber.Int64()); err != nil {
			log.WithError(err).Error("airdrop monitor: mark confirmed failed")
			return
		}
		log.Info("airdrop monitor: confirmed")
	case types.ReceiptStatusFailed:
		if err := m.cfg.ClaimRepo.MarkFailed(ctx, nonce, "reverted"); err != nil {
			log.WithError(err).Error("airdrop monitor: mark failed failed")
			return
		}
		log.Warn("airdrop monitor: tx reverted on chain")
	}
}
```

- [ ] **Step 4: Wire into `cmd/scheduler/main.go`**

The existing scheduler entrypoint (`cmd/scheduler/main.go:181-183`) blocks on `sched.Run(ctx)` until shutdown. Add the monitor as a sibling goroutine launched via `safego.Go` *before* `sched.Run`. Both share the same ctx + signal handler — when SIGTERM cancels ctx, both loops return cleanly.

Add imports:

```go
import (
	"github.com/vultisig/agent-backend/internal/service/airdrop/ethclient"
	"github.com/vultisig/agent-backend/internal/service/airdrop/relayer"
	"github.com/vultisig/agent-backend/internal/util/safego"
)
```

Insert this block just before `sched.SetAnalytics(analyticsClient)` (around line 169 of the current file):

```go
	// Airdrop confirmation monitor — slots into the same scheduler binary so
	// the airdrop pipeline doesn't need a third process. ctx is the same one
	// that cancels sched.Run on SIGTERM, so both stop together.
	rpc, err := ethclient.NewClient(cfg.Airdrop.EVMRPCURL)
	if err != nil {
		logger.WithError(err).Fatal("airdrop: failed to dial EVM RPC")
	}
	claimRepo := postgres.NewClaimRepository(db.Pool())
	confMonitor := relayer.NewMonitor(relayer.MonitorConfig{
		ClaimRepo:    claimRepo,
		RPC:          rpc,
		PollInterval: time.Duration(cfg.Airdrop.ConfirmationPollIntervalSeconds) * time.Second,
		BatchLimit:   100,
		Logger:       logger,
	})
	safego.Go(logger, "airdrop.confirmation_monitor", func() {
		if err := confMonitor.Run(ctx); err != nil && !errors.Is(err, context.Canceled) {
			logger.WithError(err).Error("airdrop confirmation monitor exited with error")
		}
	})
```

(`errors` is already imported in the file.)

- [ ] **Step 5: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/relayer/ -run TestMonitor -v -timeout 30s
```

Expected: PASS for all four monitor tests, given `DATABASE_DSN` is set. The `TestMonitor_RestartSafety` runs `mon.Run(ctx)` with a 500ms timeout — boot tick fires immediately; Run returns `context.DeadlineExceeded` after 500ms.

- [ ] **Step 6: Verify the scheduler binary still builds**

```bash
go build ./cmd/scheduler/
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/service/airdrop/relayer/monitor.go \
         internal/service/airdrop/relayer/monitor_test.go \
         cmd/scheduler/main.go
git commit -m "feat(airdrop): add confirmation monitor goroutine inside cmd/scheduler"
```

---

## Task 10: Balance read service (no internal HTTP handler)

> **Scope update (2026-04-23):** the `/internal/relayer/balance` HTTP route is dropped. Build the data-loading function as a method on the relayer service (`internal/service/airdrop/relayer/`) returning a typed `BalanceSnapshot`. Task 13 (ops UI) calls it directly to render `/ops/balance`. Skip the `internal/api/relayer/` package, the InternalTokenMiddleware wiring, and the route registration steps below. Keep the Anvil-backed integration test — repurpose it to exercise the service method instead of the HTTP handler.

**Files:**
- Create: `internal/api/relayer/balance.go`
- Create: `internal/api/relayer/balance_test.go`
- Create: `internal/api/relayer/doc.go` — package comment

- [ ] **Step 1: Add the package doc**

Create `internal/api/relayer/doc.go`:

```go
// Package relayer holds the /internal/relayer/* HTTP handlers — ops-only
// surfaces guarded by InternalTokenMiddleware. The /ops UI calls these
// endpoints in-process to render its pages.
package relayer
```

- [ ] **Step 2: Write the failing test**

Create `internal/api/relayer/balance_test.go`:

```go
package relayer

import (
	"context"
	"encoding/json"
	"errors"
	"math/big"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/labstack/echo/v4"
	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/types"
	tt "github.com/vultisig/agent-backend/internal/types"
)

type fakeBalanceRPC struct {
	balance        *big.Int
	balanceOf      *big.Int
	tipCap         *big.Int
	baseFee        *big.Int
	headBlock      uint64
}

func (f *fakeBalanceRPC) Balance(_ context.Context, _ common.Address) (*big.Int, error) { return f.balance, nil }
func (f *fakeBalanceRPC) BalanceOf(_ context.Context, _, _ common.Address) (*big.Int, error) { return f.balanceOf, nil }
func (f *fakeBalanceRPC) SuggestGasTipCap(_ context.Context) (*big.Int, error) { return f.tipCap, nil }
func (f *fakeBalanceRPC) LatestBaseFee(_ context.Context) (*big.Int, error) { return f.baseFee, nil }
func (f *fakeBalanceRPC) NonceAt(_ context.Context, _ common.Address, _ *big.Int) (uint64, error) { return f.headBlock, nil }
func (f *fakeBalanceRPC) SendRawTransaction(_ context.Context, _ []byte) (common.Hash, error) { return common.Hash{}, errors.New("unused") }
func (f *fakeBalanceRPC) GetTransactionReceipt(_ context.Context, _ common.Hash) (*types.Receipt, error) { return nil, errors.New("unused") }
func (f *fakeBalanceRPC) EstimateGas(_ context.Context, _, _ common.Address, _ []byte, _ *big.Int) (uint64, error) { return 0, errors.New("unused") }

type fakeStateRepo struct {
	state *tt.RelayerState
}

func (f *fakeStateRepo) Get(_ context.Context) (*tt.RelayerState, error) { return f.state, nil }

func TestBalanceHandler_HappyPath(t *testing.T) {
	five := new(big.Int)
	five.SetString("5000000000000000000", 10) // 5 ETH
	vult := new(big.Int)
	vult.SetString("1417500000000000000000000", 10) // 1,417,500 VULT
	nonce := int64(42)

	h := &BalanceHandler{
		RPC:                 &fakeBalanceRPC{balance: five, balanceOf: vult, tipCap: big.NewInt(2_000_000_000), baseFee: big.NewInt(20_000_000_000), headBlock: 19_400_000},
		StateRepo:           &fakeStateRepo{state: &tt.RelayerState{NextNonce: &nonce}},
		RelayerAddress:      common.HexToAddress("0x0000000000000000000000000000000000000001"),
		ContractAddress:     common.HexToAddress("0x0000000000000000000000000000000000000002"),
		LowBalanceWei:       big.NewInt(500_000_000_000_000_000), // 0.5 ETH
		LowBalanceEthString: "0.5",
		Logger:              logrus.New(),
	}

	e := echo.New()
	req := httptest.NewRequest(http.MethodGet, "/internal/relayer/balance", nil)
	rec := httptest.NewRecorder()
	c := e.NewContext(req, rec)
	if err := h.Handle(c); err != nil {
		t.Fatalf("handler: %v", err)
	}
	if rec.Code != 200 {
		t.Fatalf("status = %d, want 200 (body=%s)", rec.Code, rec.Body.String())
	}

	var body map[string]any
	_ = json.Unmarshal(rec.Body.Bytes(), &body)
	if body["eth_balance_wei"] != "5000000000000000000" {
		t.Errorf("eth_balance_wei = %v", body["eth_balance_wei"])
	}
	if body["vult_contract_balance_wei"] != "1417500000000000000000000" {
		t.Errorf("vult_contract_balance_wei = %v", body["vult_contract_balance_wei"])
	}
	if body["low_balance_warning"] != false {
		t.Errorf("low_balance_warning = %v, want false (5 ETH > 0.5)", body["low_balance_warning"])
	}
	if body["next_nonce"].(float64) != 42 {
		t.Errorf("next_nonce = %v, want 42", body["next_nonce"])
	}
	if _, ok := body["estimated_claims_remaining"]; !ok {
		t.Errorf("missing estimated_claims_remaining")
	}
}

func TestBalanceHandler_LowBalanceFlag(t *testing.T) {
	tinyBalance := big.NewInt(100_000_000_000_000_000) // 0.1 ETH
	nonce := int64(0)

	h := &BalanceHandler{
		RPC:                 &fakeBalanceRPC{balance: tinyBalance, balanceOf: big.NewInt(0), tipCap: big.NewInt(2_000_000_000), baseFee: big.NewInt(20_000_000_000), headBlock: 19_400_000},
		StateRepo:           &fakeStateRepo{state: &tt.RelayerState{NextNonce: &nonce}},
		RelayerAddress:      common.HexToAddress("0x0001"),
		ContractAddress:     common.HexToAddress("0x0002"),
		LowBalanceWei:       big.NewInt(500_000_000_000_000_000),
		LowBalanceEthString: "0.5",
		Logger:              logrus.New(),
	}

	e := echo.New()
	req := httptest.NewRequest(http.MethodGet, "/internal/relayer/balance", nil)
	rec := httptest.NewRecorder()
	c := e.NewContext(req, rec)
	_ = h.Handle(c)

	var body map[string]any
	_ = json.Unmarshal(rec.Body.Bytes(), &body)
	if body["low_balance_warning"] != true {
		t.Errorf("low_balance_warning = %v, want true (0.1 ETH < 0.5)", body["low_balance_warning"])
	}
}
```

- [ ] **Step 3: Run test to verify it fails**

```bash
go test ./internal/api/relayer/ -v
```

Expected: compile error — `BalanceHandler` undefined.

- [ ] **Step 4: Implement the handler**

Create `internal/api/relayer/balance.go`:

```go
package relayer

import (
	"context"
	"fmt"
	"math/big"
	"net/http"

	"github.com/ethereum/go-ethereum/common"
	"github.com/labstack/echo/v4"
	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/types"
)

// rpcBalance is the slice of ethclient.Client BalanceHandler depends on.
type rpcBalance interface {
	Balance(ctx context.Context, account common.Address) (*big.Int, error)
	BalanceOf(ctx context.Context, token, account common.Address) (*big.Int, error)
	LatestBaseFee(ctx context.Context) (*big.Int, error)
	SuggestGasTipCap(ctx context.Context) (*big.Int, error)
	NonceAt(ctx context.Context, addr common.Address, block *big.Int) (uint64, error)
}

type stateGetter interface {
	Get(ctx context.Context) (*types.RelayerState, error)
}

// BalanceHandler implements GET /internal/relayer/balance.
type BalanceHandler struct {
	RPC                 rpcBalance
	StateRepo           stateGetter
	RelayerAddress      common.Address
	ContractAddress     common.Address
	LowBalanceWei       *big.Int
	LowBalanceEthString string // raw string for echo back ("0.5")
	Logger              *logrus.Logger
}

type balanceResponse struct {
	RelayerAddress           string `json:"relayer_address"`
	EthBalanceWei            string `json:"eth_balance_wei"`
	EthBalanceHuman          string `json:"eth_balance_human"`
	VultContractBalanceWei   string `json:"vult_contract_balance_wei"`
	VultContractBalanceHuman string `json:"vult_contract_balance_human"`
	LowBalanceWarning        bool   `json:"low_balance_warning"`
	LowBalanceThresholdEth   string `json:"low_balance_threshold_eth"`
	EstimatedClaimsRemaining int64  `json:"estimated_claims_remaining"`
	NextNonce                int64  `json:"next_nonce"`
	RpcBlockHeight           uint64 `json:"rpc_block_height"`
}

// avgGasPerClaim is the per-claim gas estimate from the contract spec
// (sibling-on-chain-contract.md:111). Used for estimated_claims_remaining math.
const avgGasPerClaim = 62_000

func (h *BalanceHandler) Handle(c echo.Context) error {
	ctx := c.Request().Context()

	ethBal, err := h.RPC.Balance(ctx, h.RelayerAddress)
	if err != nil {
		h.Logger.WithError(err).Error("balance: rpc Balance failed")
		return c.JSON(http.StatusServiceUnavailable, map[string]string{"error": "RPC_UNAVAILABLE"})
	}

	vultBal, err := h.RPC.BalanceOf(ctx, h.ContractAddress, h.ContractAddress)
	if err != nil {
		h.Logger.WithError(err).Error("balance: rpc BalanceOf failed")
		return c.JSON(http.StatusServiceUnavailable, map[string]string{"error": "RPC_UNAVAILABLE"})
	}

	baseFee, _ := h.RPC.LatestBaseFee(ctx)
	tipCap, _ := h.RPC.SuggestGasTipCap(ctx)
	gasPrice := new(big.Int).Add(baseFee, tipCap) // approximate per-gas cost
	estimatedRemaining := int64(0)
	if gasPrice.Sign() > 0 {
		costPerClaim := new(big.Int).Mul(gasPrice, big.NewInt(avgGasPerClaim))
		if costPerClaim.Sign() > 0 {
			estimatedRemaining = new(big.Int).Quo(ethBal, costPerClaim).Int64()
		}
	}

	st, err := h.StateRepo.Get(ctx)
	if err != nil {
		h.Logger.WithError(err).Error("balance: state lookup failed")
		return c.JSON(http.StatusInternalServerError, map[string]string{"error": "STATE_LOOKUP_FAILED"})
	}
	nextNonce := int64(0)
	if st.NextNonce != nil {
		nextNonce = *st.NextNonce
	}

	// rpc block height — read via the BlockNumber method on the ethclient wrapper
	// (added in shared-infra plan, Task 5). Cheap, latest-block read.
	headHeight, err := h.RPC.BlockNumber(ctx)
	if err != nil {
		h.Logger.WithError(err).Warn("balance: rpc block height fetch failed; reporting 0")
		headHeight = 0
	}

	resp := balanceResponse{
		RelayerAddress:           h.RelayerAddress.Hex(),
		EthBalanceWei:            ethBal.String(),
		EthBalanceHuman:          formatWeiToEth(ethBal, 3),
		VultContractBalanceWei:   vultBal.String(),
		VultContractBalanceHuman: formatWeiToEth(vultBal, 3), // VULT has 18 decimals
		LowBalanceWarning:        ethBal.Cmp(h.LowBalanceWei) < 0,
		LowBalanceThresholdEth:   h.LowBalanceEthString,
		EstimatedClaimsRemaining: estimatedRemaining,
		NextNonce:                nextNonce,
		RpcBlockHeight:           headHeight,
	}
	return c.JSON(http.StatusOK, resp)
}

// formatWeiToEth divides wei by 1e18 and formats with `decimals` precision.
func formatWeiToEth(wei *big.Int, decimals int) string {
	if wei == nil {
		return "0"
	}
	// integer / fractional split via DivMod against 1e18
	one := new(big.Int).Exp(big.NewInt(10), big.NewInt(18), nil)
	intPart := new(big.Int)
	frac := new(big.Int)
	intPart.QuoRem(wei, one, frac)
	// Pad fractional to 18 digits, then truncate to `decimals`
	fracStr := frac.String()
	for len(fracStr) < 18 {
		fracStr = "0" + fracStr
	}
	if decimals > len(fracStr) {
		decimals = len(fracStr)
	}
	return fmt.Sprintf("%s.%s", intPart.String(), fracStr[:decimals])
}
```

The `RpcBlockHeight` field is populated via the wrapper's `BlockNumber(ctx)` method (added in shared-infra Task 5). On RPC error the field reports 0 with a warning log — the rest of the balance response still renders so ops aren't blocked by a transient RPC blip.

- [ ] **Step 5: Run tests to verify they pass**

```bash
go test ./internal/api/relayer/ -v
```

Expected: PASS for both subtests.

- [ ] **Step 6: Commit**

```bash
git add internal/api/relayer/doc.go internal/api/relayer/balance.go internal/api/relayer/balance_test.go
git commit -m "feat(airdrop): add GET /internal/relayer/balance handler"
```

---

## Task 11: Stuck-claims read service (no internal HTTP handler)

> **Scope update (2026-04-23):** the `/internal/relayer/stuck-claims` HTTP route is dropped. Build the listing as a method on the relayer service returning `[]StuckClaim`. Task 13 (ops UI) calls it directly to render `/ops/stuck-claims`. Skip the `internal/api/relayer/` package and the route registration. Repurpose any handler-level tests to exercise the service method.

**Files:**
- Create: `internal/api/relayer/stuck_claims.go`
- Create: `internal/api/relayer/stuck_claims_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/api/relayer/stuck_claims_test.go`:

```go
package relayer

import (
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"github.com/labstack/echo/v4"
	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/types"
)

type fakeStuckRepo struct {
	rows []types.ClaimSubmission
}

func (f *fakeStuckRepo) ListStuck(_ context.Context, _ int32) ([]types.ClaimSubmission, error) {
	return f.rows, nil
}

func TestStuckClaimsHandler_DefaultThreshold(t *testing.T) {
	now := time.Now().UTC()
	repo := &fakeStuckRepo{rows: []types.ClaimSubmission{
		{PublicKey: "pkA", Nonce: 7, TxHash: "0x111", SubmittedAt: now.Add(-23 * time.Minute), MaxFeeGwei: 50, MaxPriorityFeeGwei: 2},
		{PublicKey: "pkB", Nonce: 9, TxHash: "0x222", SubmittedAt: now.Add(-15 * time.Minute), MaxFeeGwei: 50, MaxPriorityFeeGwei: 2, PreviousTxHashes: []string{"0xprior"}},
	}}

	h := &StuckClaimsHandler{Repo: repo, Logger: logrus.New()}
	e := echo.New()
	req := httptest.NewRequest(http.MethodGet, "/internal/relayer/stuck-claims", nil)
	rec := httptest.NewRecorder()
	c := e.NewContext(req, rec)

	if err := h.Handle(c); err != nil {
		t.Fatalf("handler: %v", err)
	}
	if rec.Code != 200 {
		t.Fatalf("status = %d", rec.Code)
	}
	var body map[string]any
	_ = json.Unmarshal(rec.Body.Bytes(), &body)
	if body["count"].(float64) != 2 {
		t.Errorf("count = %v, want 2", body["count"])
	}
	rows := body["stuck_claims"].([]any)
	if len(rows) != 2 {
		t.Fatalf("got %d rows, want 2", len(rows))
	}
	first := rows[0].(map[string]any)
	if first["public_key"] != "pkA" {
		t.Errorf("first row public_key = %v", first["public_key"])
	}
	if _, ok := first["minutes_pending"]; !ok {
		t.Error("missing minutes_pending field")
	}
	second := rows[1].(map[string]any)
	prev := second["previous_tx_hashes"].([]any)
	if len(prev) != 1 || prev[0] != "0xprior" {
		t.Errorf("previous_tx_hashes = %v, want [0xprior]", prev)
	}
}

func TestStuckClaimsHandler_CustomThreshold(t *testing.T) {
	repo := &fakeStuckRepo{rows: nil}
	h := &StuckClaimsHandler{Repo: repo, Logger: logrus.New()}

	e := echo.New()
	req := httptest.NewRequest(http.MethodGet, "/internal/relayer/stuck-claims?older_than_minutes=30", nil)
	rec := httptest.NewRecorder()
	c := e.NewContext(req, rec)
	_ = h.Handle(c)

	var body map[string]any
	_ = json.Unmarshal(rec.Body.Bytes(), &body)
	if body["count"].(float64) != 0 {
		t.Errorf("count = %v, want 0", body["count"])
	}
	if _, ok := body["stuck_claims"]; !ok {
		t.Error("missing stuck_claims field even when empty (must be [], not absent)")
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/api/relayer/ -run TestStuckClaims -v
```

Expected: compile error — `StuckClaimsHandler` undefined.

- [ ] **Step 3: Implement the handler**

Create `internal/api/relayer/stuck_claims.go`:

```go
package relayer

import (
	"context"
	"net/http"
	"sort"
	"strconv"
	"time"

	"github.com/labstack/echo/v4"
	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/types"
)

type stuckRepo interface {
	ListStuck(ctx context.Context, olderThanMinutes int32) ([]types.ClaimSubmission, error)
}

type StuckClaimsHandler struct {
	Repo   stuckRepo
	Logger *logrus.Logger
}

type stuckClaimRow struct {
	PublicKey          string   `json:"public_key"`
	TxHash             string   `json:"tx_hash"`
	Nonce              int64    `json:"nonce"`
	SubmittedAt        string   `json:"submitted_at"`
	MinutesPending     int64    `json:"minutes_pending"`
	MaxFeeGwei         int32    `json:"max_fee_gwei"`
	MaxPriorityFeeGwei int32    `json:"max_priority_fee_gwei"`
	PreviousTxHashes   []string `json:"previous_tx_hashes"`
}

type stuckClaimsResponse struct {
	StuckClaims []stuckClaimRow `json:"stuck_claims"`
	Count       int             `json:"count"`
}

func (h *StuckClaimsHandler) Handle(c echo.Context) error {
	threshold := int32(10) // default per spec
	if raw := c.QueryParam("older_than_minutes"); raw != "" {
		n, err := strconv.Atoi(raw)
		if err != nil || n <= 0 {
			return c.JSON(http.StatusBadRequest, map[string]string{"error": "invalid older_than_minutes"})
		}
		threshold = int32(n)
	}

	rows, err := h.Repo.ListStuck(c.Request().Context(), threshold)
	if err != nil {
		h.Logger.WithError(err).Error("stuck-claims: list failed")
		return c.JSON(http.StatusInternalServerError, map[string]string{"error": "STUCK_LOOKUP_FAILED"})
	}

	// Sort by minutes_pending DESC (oldest first) per spec.
	sort.SliceStable(rows, func(i, j int) bool {
		return rows[i].SubmittedAt.Before(rows[j].SubmittedAt)
	})

	out := stuckClaimsResponse{StuckClaims: make([]stuckClaimRow, 0, len(rows)), Count: len(rows)}
	now := time.Now().UTC()
	for _, r := range rows {
		prev := r.PreviousTxHashes
		if prev == nil {
			prev = []string{}
		}
		out.StuckClaims = append(out.StuckClaims, stuckClaimRow{
			PublicKey:          r.PublicKey,
			TxHash:             r.TxHash,
			Nonce:              r.Nonce,
			SubmittedAt:        r.SubmittedAt.Format(time.RFC3339),
			MinutesPending:     int64(now.Sub(r.SubmittedAt).Minutes()),
			MaxFeeGwei:         r.MaxFeeGwei,
			MaxPriorityFeeGwei: r.MaxPriorityFeeGwei,
			PreviousTxHashes:   prev,
		})
	}
	return c.JSON(http.StatusOK, out)
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/api/relayer/ -v
```

Expected: PASS for all four `TestBalanceHandler*` and `TestStuckClaims*` subtests.

- [ ] **Step 5: Commit**

```bash
git add internal/api/relayer/stuck_claims.go internal/api/relayer/stuck_claims_test.go
git commit -m "feat(airdrop): add GET /internal/relayer/stuck-claims handler"
```

---

## Task 12: Rebroadcast service — same nonce + bumped fees

**Why:** When a tx gets stuck (RPC throttling, low fee at the time of submit), ops bumps the gas and re-broadcasts. The new tx must reuse the same nonce so the original is replaced in the mempool — only one of the two can confirm, and the previous tx_hash is appended to the audit array.

**Files:**
- Create: `internal/service/airdrop/relayer/rebroadcast.go`
- Create: `internal/service/airdrop/relayer/rebroadcast_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/service/airdrop/relayer/rebroadcast_test.go`:

```go
package relayer

import (
	"context"
	"errors"
	"math/big"
	"testing"

	"github.com/ethereum/go-ethereum/common"

	"github.com/vultisig/agent-backend/internal/storage/postgres"
	tt "github.com/vultisig/agent-backend/internal/types"
)

func TestRebroadcastService_BumpsFeesAndAppendsHash(t *testing.T) {
	pool := newClaimServiceTestPool(t)
	repo := postgres.NewClaimRepository(pool)

	// Seed a submitted claim at 50/2 gwei
	winnerPK := "pk-rebroadcast"
	seedWinner(t, pool, winnerPK, "0x70997970C51812dc3A010C7d01b50e0d17dc79C8", big.NewInt(1500))
	original, err := repo.Insert(context.Background(),
		winnerPK, 5, "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
		big.NewInt(1500), "0xORIG", 50, 2)
	if err != nil {
		t.Fatalf("seed: %v", err)
	}

	rpc := &fakeRPC{}
	signer := newLocalSigner(t)
	svc := NewRebroadcastService(RebroadcastConfig{
		ChainID: big.NewInt(1), ContractAddress: common.HexToAddress("0xCCCC"),
		ClaimRepo: repo, RPC: rpc, Signer: signer,
	})

	result, err := svc.Rebroadcast(context.Background(), winnerPK, 50)
	if err != nil {
		t.Fatalf("Rebroadcast: %v", err)
	}
	if result.NewTxHash == original.TxHash {
		t.Errorf("new tx_hash = old tx_hash; expected new bumped tx")
	}
	if result.OldTxHash != "0xORIG" {
		t.Errorf("OldTxHash = %s, want 0xORIG", result.OldTxHash)
	}

	// Verify DB state
	row, _ := repo.GetActiveByPublicKey(context.Background(), winnerPK)
	if row.Nonce != 5 {
		t.Errorf("nonce changed: %d, want 5", row.Nonce)
	}
	if row.TxHash == "0xORIG" {
		t.Errorf("tx_hash unchanged after rebroadcast")
	}
	if len(row.PreviousTxHashes) != 1 || row.PreviousTxHashes[0] != "0xORIG" {
		t.Errorf("previous_tx_hashes = %v, want [0xORIG]", row.PreviousTxHashes)
	}
	if row.MaxFeeGwei <= 50 {
		t.Errorf("max_fee_gwei = %d, want > 50 (bumped by 50)", row.MaxFeeGwei)
	}
	if row.MaxPriorityFeeGwei != 2+50 {
		t.Errorf("max_priority_fee_gwei = %d, want 52", row.MaxPriorityFeeGwei)
	}
	if len(rpc.sentTxs) != 1 {
		t.Errorf("rpc broadcasts = %d, want 1", len(rpc.sentTxs))
	}
}

func TestRebroadcastService_RejectsConfirmedClaim(t *testing.T) {
	pool := newClaimServiceTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	winnerPK := "pk-confirmed"
	seedWinner(t, pool, winnerPK, "0x70997970C51812dc3A010C7d01b50e0d17dc79C8", big.NewInt(1500))
	_, _ = repo.Insert(context.Background(), winnerPK, 8, "0x70997970C51812dc3A010C7d01b50e0d17dc79C8", big.NewInt(1500), "0xCONF", 50, 2)
	if err := repo.MarkConfirmed(context.Background(), 8, 19_400_000); err != nil {
		t.Fatalf("MarkConfirmed: %v", err)
	}

	svc := NewRebroadcastService(RebroadcastConfig{
		ChainID: big.NewInt(1), ContractAddress: common.HexToAddress("0xCCCC"),
		ClaimRepo: repo, RPC: &fakeRPC{}, Signer: newLocalSigner(t),
	})
	_, err := svc.Rebroadcast(context.Background(), winnerPK, 50)
	if !errors.Is(err, ErrAlreadyConfirmed) {
		t.Errorf("err = %v, want ErrAlreadyConfirmed", err)
	}
	_ = tt.ClaimConfirmed // keep import live
}

func TestRebroadcastService_NoSubmissionForUser(t *testing.T) {
	pool := newClaimServiceTestPool(t)
	repo := postgres.NewClaimRepository(pool)
	svc := NewRebroadcastService(RebroadcastConfig{
		ChainID: big.NewInt(1), ContractAddress: common.HexToAddress("0xCCCC"),
		ClaimRepo: repo, RPC: &fakeRPC{}, Signer: newLocalSigner(t),
	})
	_, err := svc.Rebroadcast(context.Background(), "nobody", 50)
	if !errors.Is(err, ErrNoSubmission) {
		t.Errorf("err = %v, want ErrNoSubmission", err)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/service/airdrop/relayer/ -run TestRebroadcastService -v
```

Expected: compile error — `RebroadcastService`, `RebroadcastConfig`, `ErrAlreadyConfirmed`, `ErrNoSubmission` undefined.

- [ ] **Step 3: Implement the rebroadcast service**

Create `internal/service/airdrop/relayer/rebroadcast.go`:

```go
package relayer

import (
	"context"
	"errors"
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum/common"

	"github.com/vultisig/agent-backend/internal/service/airdrop/ethclient"
	"github.com/vultisig/agent-backend/internal/service/airdrop/kms"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
	"github.com/vultisig/agent-backend/internal/types"
)

var (
	ErrNoSubmission     = errors.New("no claim submission for this public_key")
	ErrAlreadyConfirmed = errors.New("claim already confirmed; rebroadcast not needed")
)

type RebroadcastConfig struct {
	ChainID         *big.Int
	ContractAddress common.Address
	ClaimRepo       *postgres.ClaimRepository
	RPC             ethclient.Client
	Signer          kms.Signer
}

type RebroadcastService struct {
	cfg RebroadcastConfig
}

func NewRebroadcastService(cfg RebroadcastConfig) *RebroadcastService {
	return &RebroadcastService{cfg: cfg}
}

type RebroadcastResult struct {
	OldTxHash string
	NewTxHash string
	Nonce     int64
	NewMaxFeeGwei int32
	NewTipGwei    int32
}

// Rebroadcast re-signs an EIP-1559 tx for the same nonce with bumped fees and broadcasts it.
//   bumpGwei is added to BOTH max_fee and max_priority_fee.
// The previous tx_hash is appended to previous_tx_hashes; the row's tx_hash + fee
// fields are updated. Status stays 'submitted'.
func (s *RebroadcastService) Rebroadcast(ctx context.Context, publicKey string, bumpGwei int32) (*RebroadcastResult, error) {
	// 1. Find the row.
	row, err := s.cfg.ClaimRepo.GetByPublicKey(ctx, publicKey)
	if err != nil {
		if errors.Is(err, postgres.ErrNotFound) {
			return nil, ErrNoSubmission
		}
		return nil, fmt.Errorf("lookup claim: %w", err)
	}
	if row.Status == types.ClaimConfirmed {
		return nil, ErrAlreadyConfirmed
	}
	if row.Status != types.ClaimSubmitted {
		return nil, fmt.Errorf("claim status is %s; can only rebroadcast 'submitted' rows", row.Status)
	}

	// 2. Bump fees. Bump in gwei → wei for the tx; bumped values written back as gwei.
	newMaxFeeGwei := row.MaxFeeGwei + bumpGwei
	newTipGwei := row.MaxPriorityFeeGwei + bumpGwei
	maxFeeWei := new(big.Int).Mul(big.NewInt(int64(newMaxFeeGwei)), big.NewInt(1_000_000_000))
	tipWei := new(big.Int).Mul(big.NewInt(int64(newTipGwei)), big.NewInt(1_000_000_000))

	// 3. Rebuild calldata + estimate gas.
	calldata := BuildClaimCalldata(common.HexToAddress(row.Recipient))
	gasLimit, err := s.cfg.RPC.EstimateGas(ctx, s.cfg.Signer.Address(), s.cfg.ContractAddress, calldata, nil)
	if err != nil {
		return nil, fmt.Errorf("estimate gas: %w", err)
	}
	gasLimit = gasLimit + gasLimit/5 // 20% pad, same as initial submit

	// 4. Sign + broadcast at the SAME nonce.
	signedBytes, txHash, err := BuildAndSignTx(ctx, s.cfg.Signer, TxParams{
		ChainID:           s.cfg.ChainID,
		Nonce:             uint64(row.Nonce),
		GasLimit:          gasLimit,
		MaxFeePerGasWei:   maxFeeWei,
		MaxPriorityFeeWei: tipWei,
		To:                s.cfg.ContractAddress,
		Calldata:          calldata,
	})
	if err != nil {
		return nil, fmt.Errorf("build+sign: %w", err)
	}
	if _, err := s.cfg.RPC.SendRawTransaction(ctx, signedBytes); err != nil {
		return nil, fmt.Errorf("broadcast: %w", err)
	}

	// 5. Update DB — append old hash, set new hash + fees.
	if err := s.cfg.ClaimRepo.UpdateRebroadcast(ctx, row.Nonce, txHash.Hex(), newMaxFeeGwei, newTipGwei); err != nil {
		return nil, fmt.Errorf("db update: %w", err)
	}

	return &RebroadcastResult{
		OldTxHash:     row.TxHash,
		NewTxHash:     txHash.Hex(),
		Nonce:         row.Nonce,
		NewMaxFeeGwei: newMaxFeeGwei,
		NewTipGwei:    newTipGwei,
	}, nil
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/relayer/ -run TestRebroadcastService -v
```

Expected: PASS for all three subtests, given `DATABASE_DSN` is set.

- [ ] **Step 5: Commit**

```bash
git add internal/service/airdrop/relayer/rebroadcast.go internal/service/airdrop/relayer/rebroadcast_test.go
git commit -m "feat(airdrop): add rebroadcast service (same nonce + bumped fees)"
```

---

## Task 13: Operator UI — `/ops/*` HTML pages with inline Basic Auth

**Why:** Server-rendered HTML, no JS build, CSS inlined. Auto-refresh via `<meta http-equiv="refresh" content="30">`. Per the cleanup pass: **inline the Basic Auth check** — `c.Request().BasicAuth()` plus `subtle.ConstantTimeCompare` against `cfg.Airdrop.OpsUsername` + `cfg.Airdrop.OpsPassword`. No shared middleware.

**Files:**
- Create: `internal/api/ops/handler.go` — Basic Auth wrapper + struct
- Create: `internal/api/ops/landing.go` — `GET /ops`
- Create: `internal/api/ops/balance.go` — `GET /ops/balance`
- Create: `internal/api/ops/stuck_claims.go` — `GET /ops/stuck-claims`
- Create: `internal/api/ops/rebroadcast.go` — `POST /ops/rebroadcast`
- Create: `internal/api/ops/templates/layout.html`
- Create: `internal/api/ops/templates/landing.html`
- Create: `internal/api/ops/templates/balance.html`
- Create: `internal/api/ops/templates/stuck_claims.html`
- Create: `internal/api/ops/templates/rebroadcast_result.html`
- Create: `internal/api/ops/handler_test.go` — Basic Auth + landing render

- [ ] **Step 1: Write the failing test**

Create `internal/api/ops/handler_test.go`:

```go
package ops

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/labstack/echo/v4"
	"github.com/sirupsen/logrus"
)

func TestOpsBasicAuth_Rejects(t *testing.T) {
	h := &Handler{Username: "admin", Password: "s3cret", Logger: logrus.New()}
	if err := h.LoadTemplates(); err != nil {
		t.Fatalf("LoadTemplates: %v", err)
	}

	cases := []struct {
		name        string
		header      string
		wantStatus  int
		wantWWWAuth bool
	}{
		{"no header", "", http.StatusUnauthorized, true},
		{"wrong creds", basicAuth("admin", "wrong"), http.StatusUnauthorized, true},
		{"wrong user", basicAuth("nope", "s3cret"), http.StatusUnauthorized, true},
		{"correct creds", basicAuth("admin", "s3cret"), http.StatusOK, false},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			e := echo.New()
			req := httptest.NewRequest(http.MethodGet, "/ops", nil)
			if tc.header != "" {
				req.Header.Set("Authorization", tc.header)
			}
			rec := httptest.NewRecorder()
			c := e.NewContext(req, rec)
			_ = h.Landing(c)
			if rec.Code != tc.wantStatus {
				t.Errorf("status = %d, want %d", rec.Code, tc.wantStatus)
			}
			if tc.wantWWWAuth {
				if got := rec.Header().Get("WWW-Authenticate"); got == "" {
					t.Error("missing WWW-Authenticate challenge header on 401")
				}
			}
		})
	}
}

func basicAuth(user, pass string) string {
	// Manually construct Basic auth header value
	// (avoid importing encoding/base64 just to keep the test tight).
	const b64 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz+/"
	raw := user + ":" + pass
	enc := make([]byte, 0, ((len(raw)+2)/3)*4)
	src := []byte(raw)
	for i := 0; i < len(src); i += 3 {
		var b [3]byte
		n := copy(b[:], src[i:])
		enc = append(enc, b64[b[0]>>2])
		enc = append(enc, b64[((b[0]&0x03)<<4)|(b[1]>>4)])
		if n > 1 {
			enc = append(enc, b64[((b[1]&0x0f)<<2)|(b[2]>>6)])
		} else {
			enc = append(enc, '=')
		}
		if n > 2 {
			enc = append(enc, b64[b[2]&0x3f])
		} else {
			enc = append(enc, '=')
		}
	}
	return "Basic " + string(enc)
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/api/ops/ -v
```

Expected: compile error — `Handler`, `LoadTemplates`, `Landing` undefined.

- [ ] **Step 3: Implement the handler skeleton + Basic Auth**

Create `internal/api/ops/handler.go`:

```go
// Package ops implements the /ops/* HTML operator UI for the airdrop relayer.
// All routes use HTTP Basic Auth checked inline (see basicAuthOK) — no shared
// middleware, since this is the only route group that uses Basic Auth.
package ops

import (
	"crypto/subtle"
	"embed"
	"fmt"
	"html/template"
	"net/http"

	"github.com/labstack/echo/v4"
	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/service/airdrop/relayer"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
)

//go:embed templates/*.html
var templatesFS embed.FS

// Handler bundles the deps + templates the four /ops routes share.
type Handler struct {
	Username string
	Password string

	// Data sources for the pages
	BalanceLoader      func(echo.Context) (BalanceView, error) // Task-13 wires to the in-process /internal/relayer/balance call
	StuckClaimsLoader  func(echo.Context) (StuckClaimsView, error)
	ClaimRepo          *postgres.ClaimRepository
	RebroadcastService *relayer.RebroadcastService

	Logger    *logrus.Logger
	templates *template.Template
}

// LoadTemplates parses every .html file under templates/. Call once at boot.
func (h *Handler) LoadTemplates() error {
	t, err := template.ParseFS(templatesFS, "templates/*.html")
	if err != nil {
		return fmt.Errorf("parse ops templates: %w", err)
	}
	h.templates = t
	return nil
}

// basicAuthOK enforces HTTP Basic Auth inline. Returns true on success;
// on failure it has already written the 401 + WWW-Authenticate response.
func (h *Handler) basicAuthOK(c echo.Context) bool {
	user, pass, ok := c.Request().BasicAuth()
	if !ok {
		h.challenge(c)
		return false
	}
	uOK := subtle.ConstantTimeCompare([]byte(user), []byte(h.Username)) == 1
	pOK := subtle.ConstantTimeCompare([]byte(pass), []byte(h.Password)) == 1
	if !uOK || !pOK {
		h.challenge(c)
		return false
	}
	return true
}

func (h *Handler) challenge(c echo.Context) {
	c.Response().Header().Set("WWW-Authenticate", `Basic realm="airdrop-ops"`)
	_ = c.String(http.StatusUnauthorized, "401 Unauthorized\n")
}

// Render is the shared template-render path. Wraps in <html><body> via layout.html.
func (h *Handler) Render(c echo.Context, name string, data any) error {
	c.Response().Header().Set("Content-Type", "text/html; charset=utf-8")
	c.Response().WriteHeader(http.StatusOK)
	return h.templates.ExecuteTemplate(c.Response().Writer, name, data)
}

// View structs — handed to templates by the page-specific handlers below.

type BalanceView struct {
	RelayerAddress           string
	EthBalanceHuman          string
	VultContractBalanceHuman string
	LowBalanceWarning        bool
	LowBalanceThresholdEth   string
	EstimatedClaimsRemaining int64
	NextNonce                int64
}

type StuckClaimRow struct {
	PublicKey          string
	TxHash             string
	Nonce              int64
	SubmittedAt        string
	MinutesPending     int64
	MaxFeeGwei         int32
	MaxPriorityFeeGwei int32
	PreviousTxHashes   []string
}

type StuckClaimsView struct {
	Rows  []StuckClaimRow
	Count int
}
```

- [ ] **Step 4: Implement the four route handlers**

Create `internal/api/ops/landing.go`:

```go
package ops

import "github.com/labstack/echo/v4"

type landingData struct {
	Title string
}

func (h *Handler) Landing(c echo.Context) error {
	if !h.basicAuthOK(c) {
		return nil
	}
	return h.Render(c, "landing.html", landingData{Title: "Airdrop Ops"})
}
```

Create `internal/api/ops/balance.go`:

```go
package ops

import (
	"github.com/labstack/echo/v4"
)

func (h *Handler) Balance(c echo.Context) error {
	if !h.basicAuthOK(c) {
		return nil
	}
	view, err := h.BalanceLoader(c)
	if err != nil {
		h.Logger.WithError(err).Error("ops balance loader failed")
		return h.Render(c, "balance.html", map[string]any{"Err": err.Error()})
	}
	return h.Render(c, "balance.html", map[string]any{"V": view})
}
```

Create `internal/api/ops/stuck_claims.go`:

```go
package ops

import (
	"github.com/labstack/echo/v4"
)

func (h *Handler) StuckClaims(c echo.Context) error {
	if !h.basicAuthOK(c) {
		return nil
	}
	view, err := h.StuckClaimsLoader(c)
	if err != nil {
		h.Logger.WithError(err).Error("ops stuck-claims loader failed")
		return h.Render(c, "stuck_claims.html", map[string]any{"Err": err.Error()})
	}
	return h.Render(c, "stuck_claims.html", map[string]any{"V": view})
}
```

Create `internal/api/ops/rebroadcast.go`:

```go
package ops

import (
	"errors"
	"net/http"
	"strconv"

	"github.com/labstack/echo/v4"

	"github.com/vultisig/agent-backend/internal/service/airdrop/relayer"
)

func (h *Handler) Rebroadcast(c echo.Context) error {
	if !h.basicAuthOK(c) {
		return nil
	}
	publicKey := c.FormValue("public_key")
	bumpStr := c.FormValue("bump_gwei")
	if publicKey == "" {
		return c.String(http.StatusBadRequest, "missing public_key")
	}
	bump := int32(50)
	if bumpStr != "" {
		n, err := strconv.Atoi(bumpStr)
		if err != nil || n <= 0 {
			return c.String(http.StatusBadRequest, "invalid bump_gwei")
		}
		bump = int32(n)
	}

	result, err := h.RebroadcastService.Rebroadcast(c.Request().Context(), publicKey, bump)
	switch {
	case errors.Is(err, relayer.ErrNoSubmission):
		c.Response().WriteHeader(http.StatusNotFound)
		return h.templates.ExecuteTemplate(c.Response().Writer, "rebroadcast_result.html",
			map[string]any{"Err": "No submission for this public_key"})
	case errors.Is(err, relayer.ErrAlreadyConfirmed):
		c.Response().WriteHeader(http.StatusBadRequest)
		return h.templates.ExecuteTemplate(c.Response().Writer, "rebroadcast_result.html",
			map[string]any{"Err": "Claim already confirmed — rebroadcast not needed"})
	case err != nil:
		h.Logger.WithError(err).Error("ops rebroadcast failed")
		c.Response().WriteHeader(http.StatusServiceUnavailable)
		return h.templates.ExecuteTemplate(c.Response().Writer, "rebroadcast_result.html",
			map[string]any{"Err": err.Error()})
	}
	return h.Render(c, "rebroadcast_result.html", map[string]any{"OK": true, "R": result})
}
```

- [ ] **Step 5: Add the templates**

Create `internal/api/ops/templates/layout.html`:

```html
{{ define "layout" }}
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{{ .Title }}</title>
  <style>
    body { font-family: -apple-system, system-ui, sans-serif; background: #0d1117; color: #c9d1d9; padding: 24px; max-width: 1200px; margin: 0 auto; }
    h1 { color: #58a6ff; }
    a { color: #58a6ff; }
    table { width: 100%; border-collapse: collapse; margin-top: 16px; }
    th, td { padding: 8px 12px; text-align: left; border-bottom: 1px solid #30363d; font-family: monospace; }
    th { background: #161b22; }
    .warning { background: #f85149; color: white; padding: 12px; border-radius: 6px; margin: 12px 0; font-weight: bold; }
    .ok { background: #238636; color: white; padding: 12px; border-radius: 6px; margin: 12px 0; }
    .err { background: #f85149; color: white; padding: 12px; border-radius: 6px; margin: 12px 0; }
    .nav { margin-bottom: 24px; padding-bottom: 12px; border-bottom: 1px solid #30363d; }
    .nav a { margin-right: 16px; }
    button { background: #238636; color: white; border: 0; padding: 6px 12px; border-radius: 4px; cursor: pointer; font-family: monospace; }
    button:hover { background: #2ea043; }
    form { display: inline; }
  </style>
</head>
<body>
  <div class="nav">
    <a href="/ops">home</a>
    <a href="/ops/balance">balance</a>
    <a href="/ops/stuck-claims">stuck-claims</a>
  </div>
  {{ template "content" . }}
</body>
</html>
{{ end }}
```

Create `internal/api/ops/templates/landing.html`:

```html
{{ define "landing.html" }}{{ template "layout" . }}{{ end }}
{{ define "content" }}
  <h1>Airdrop Ops</h1>
  <p>Operator console for the Vultiverse airdrop relayer.</p>
  <ul>
    <li><a href="/ops/balance">Balance</a> — relayer ETH balance, VULT contract balance, low-balance warning</li>
    <li><a href="/ops/stuck-claims">Stuck claims</a> — submitted txs older than 10 minutes; rebroadcast button per row</li>
  </ul>
{{ end }}
```

Create `internal/api/ops/templates/balance.html`:

```html
{{ define "balance.html" }}{{ template "layout" (dict "Title" "Balance" "Body" .) }}{{ end }}
{{ define "content" }}
  <meta http-equiv="refresh" content="30">
  <h1>Relayer Balance</h1>
  {{ with .Err }}
    <div class="err">Error: {{ . }}</div>
  {{ else }}
    {{ with .V }}
      {{ if .LowBalanceWarning }}
        <div class="warning">⚠ LOW BALANCE: relayer below {{ .LowBalanceThresholdEth }} ETH — top up immediately</div>
      {{ end }}
      <table>
        <tr><th>Relayer address</th><td>{{ .RelayerAddress }}</td></tr>
        <tr><th>ETH balance</th><td>{{ .EthBalanceHuman }} ETH</td></tr>
        <tr><th>VULT in contract</th><td>{{ .VultContractBalanceHuman }} VULT</td></tr>
        <tr><th>Estimated claims remaining</th><td>{{ .EstimatedClaimsRemaining }}</td></tr>
        <tr><th>Next nonce</th><td>{{ .NextNonce }}</td></tr>
        <tr><th>Low-balance threshold</th><td>{{ .LowBalanceThresholdEth }} ETH</td></tr>
      </table>
    {{ end }}
  {{ end }}
{{ end }}
```

Note on the layout/content split: Go's html/template makes the "layout-with-named-content-block" pattern slightly awkward when defining everything in one file. The simplest version that works: define the `layout` block with a `{{ template "content" . }}` placeholder; each page file defines both a top-level template (matching the file name passed to ExecuteTemplate) that calls layout, AND a `content` template that renders the page body. The `dict` helper above is illustrative — Go's stdlib template doesn't include it. Replace with this simpler shape that uses the data directly:

Replace the contents of `internal/api/ops/templates/landing.html` with:

```html
{{ define "landing.html" }}
<!DOCTYPE html><html><head><meta charset="utf-8"><title>Airdrop Ops</title>
<style>body{font-family:-apple-system,system-ui,sans-serif;background:#0d1117;color:#c9d1d9;padding:24px;max-width:1200px;margin:0 auto}h1{color:#58a6ff}a{color:#58a6ff;margin-right:16px}.nav{margin-bottom:24px;padding-bottom:12px;border-bottom:1px solid #30363d}</style>
</head><body>
<div class="nav"><a href="/ops">home</a><a href="/ops/balance">balance</a><a href="/ops/stuck-claims">stuck-claims</a></div>
<h1>Airdrop Ops</h1>
<p>Operator console for the Vultiverse airdrop relayer.</p>
<ul>
  <li><a href="/ops/balance">Balance</a> — relayer ETH balance, VULT contract balance, low-balance warning</li>
  <li><a href="/ops/stuck-claims">Stuck claims</a> — submitted txs older than 10 minutes; rebroadcast button per row</li>
</ul>
</body></html>
{{ end }}
```

Replace `internal/api/ops/templates/balance.html` with:

```html
{{ define "balance.html" }}
<!DOCTYPE html><html><head><meta charset="utf-8"><title>Balance</title>
<meta http-equiv="refresh" content="30">
<style>body{font-family:-apple-system,system-ui,sans-serif;background:#0d1117;color:#c9d1d9;padding:24px;max-width:1200px;margin:0 auto}h1{color:#58a6ff}a{color:#58a6ff;margin-right:16px}.nav{margin-bottom:24px;padding-bottom:12px;border-bottom:1px solid #30363d}table{width:100%;border-collapse:collapse;margin-top:16px}th,td{padding:8px 12px;text-align:left;border-bottom:1px solid #30363d;font-family:monospace}th{background:#161b22;width:30%}.warning{background:#f85149;color:white;padding:12px;border-radius:6px;margin:12px 0;font-weight:bold}.err{background:#f85149;color:white;padding:12px;border-radius:6px;margin:12px 0}</style>
</head><body>
<div class="nav"><a href="/ops">home</a><a href="/ops/balance">balance</a><a href="/ops/stuck-claims">stuck-claims</a></div>
<h1>Relayer Balance</h1>
{{ with .Err }}<div class="err">Error: {{ . }}</div>{{ end }}
{{ with .V }}
{{ if .LowBalanceWarning }}<div class="warning">⚠ LOW BALANCE: relayer below {{ .LowBalanceThresholdEth }} ETH — top up immediately</div>{{ end }}
<table>
  <tr><th>Relayer address</th><td>{{ .RelayerAddress }}</td></tr>
  <tr><th>ETH balance</th><td>{{ .EthBalanceHuman }} ETH</td></tr>
  <tr><th>VULT in contract</th><td>{{ .VultContractBalanceHuman }} VULT</td></tr>
  <tr><th>Estimated claims remaining</th><td>{{ .EstimatedClaimsRemaining }}</td></tr>
  <tr><th>Next nonce</th><td>{{ .NextNonce }}</td></tr>
  <tr><th>Low-balance threshold</th><td>{{ .LowBalanceThresholdEth }} ETH</td></tr>
</table>
{{ end }}
</body></html>
{{ end }}
```

Replace `internal/api/ops/templates/stuck_claims.html` with:

```html
{{ define "stuck_claims.html" }}
<!DOCTYPE html><html><head><meta charset="utf-8"><title>Stuck Claims</title>
<meta http-equiv="refresh" content="30">
<style>body{font-family:-apple-system,system-ui,sans-serif;background:#0d1117;color:#c9d1d9;padding:24px;max-width:1400px;margin:0 auto}h1{color:#58a6ff}a{color:#58a6ff;margin-right:16px}.nav{margin-bottom:24px;padding-bottom:12px;border-bottom:1px solid #30363d}table{width:100%;border-collapse:collapse;margin-top:16px}th,td{padding:8px 12px;text-align:left;border-bottom:1px solid #30363d;font-family:monospace;font-size:12px}th{background:#161b22}button{background:#238636;color:white;border:0;padding:6px 12px;border-radius:4px;cursor:pointer;font-family:monospace}button:hover{background:#2ea043}.ok{background:#238636;color:white;padding:12px;border-radius:6px}.err{background:#f85149;color:white;padding:12px;border-radius:6px}</style>
</head><body>
<div class="nav"><a href="/ops">home</a><a href="/ops/balance">balance</a><a href="/ops/stuck-claims">stuck-claims</a></div>
<h1>Stuck Claims</h1>
{{ with .Err }}<div class="err">Error: {{ . }}</div>{{ end }}
{{ with .V }}
{{ if eq .Count 0 }}<div class="ok">✓ No stuck claims (everything submitted has confirmed within 10 minutes)</div>
{{ else }}
<p>{{ .Count }} claim(s) submitted but unconfirmed for &gt;10 min. Each rebroadcast bumps fees by 50 gwei and reuses the same nonce.</p>
<table>
  <tr><th>Public key</th><th>Nonce</th><th>Tx hash</th><th>Min pending</th><th>Max fee (gwei)</th><th>Tip (gwei)</th><th>Prior hashes</th><th>Action</th></tr>
  {{ range .Rows }}
  <tr>
    <td>{{ .PublicKey }}</td>
    <td>{{ .Nonce }}</td>
    <td>{{ .TxHash }}</td>
    <td>{{ .MinutesPending }}</td>
    <td>{{ .MaxFeeGwei }}</td>
    <td>{{ .MaxPriorityFeeGwei }}</td>
    <td>{{ len .PreviousTxHashes }}</td>
    <td>
      <form method="POST" action="/ops/rebroadcast">
        <input type="hidden" name="public_key" value="{{ .PublicKey }}">
        <input type="hidden" name="bump_gwei" value="50">
        <button type="submit">Rebroadcast (+50 gwei)</button>
      </form>
    </td>
  </tr>
  {{ end }}
</table>
{{ end }}
{{ end }}
</body></html>
{{ end }}
```

Replace `internal/api/ops/templates/rebroadcast_result.html` with:

```html
{{ define "rebroadcast_result.html" }}
<!DOCTYPE html><html><head><meta charset="utf-8"><title>Rebroadcast Result</title>
<style>body{font-family:-apple-system,system-ui,sans-serif;background:#0d1117;color:#c9d1d9;padding:24px;max-width:1200px;margin:0 auto}h1{color:#58a6ff}a{color:#58a6ff;margin-right:16px}.nav{margin-bottom:24px;padding-bottom:12px;border-bottom:1px solid #30363d}table{width:100%;border-collapse:collapse;margin-top:16px}th,td{padding:8px 12px;text-align:left;border-bottom:1px solid #30363d;font-family:monospace}th{background:#161b22;width:30%}.ok{background:#238636;color:white;padding:12px;border-radius:6px;margin:12px 0}.err{background:#f85149;color:white;padding:12px;border-radius:6px;margin:12px 0}</style>
</head><body>
<div class="nav"><a href="/ops">home</a><a href="/ops/balance">balance</a><a href="/ops/stuck-claims">stuck-claims</a></div>
<h1>Rebroadcast Result</h1>
{{ if .OK }}
  {{ with .R }}
  <div class="ok">✓ Submitted new tx at same nonce {{ .Nonce }}. Old tx and new tx race in the mempool — only one (the higher-fee new one) should confirm.</div>
  <table>
    <tr><th>Old tx hash</th><td>{{ .OldTxHash }} (<a href="https://etherscan.io/tx/{{ .OldTxHash }}">etherscan</a>)</td></tr>
    <tr><th>New tx hash</th><td>{{ .NewTxHash }} (<a href="https://etherscan.io/tx/{{ .NewTxHash }}">etherscan</a>)</td></tr>
    <tr><th>Nonce (unchanged)</th><td>{{ .Nonce }}</td></tr>
    <tr><th>New max fee (gwei)</th><td>{{ .NewMaxFeeGwei }}</td></tr>
    <tr><th>New tip (gwei)</th><td>{{ .NewTipGwei }}</td></tr>
  </table>
  {{ end }}
{{ else }}
  <div class="err">{{ .Err }}</div>
{{ end }}
<p><a href="/ops/stuck-claims">← back to stuck claims</a></p>
</body></html>
{{ end }}
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
go test ./internal/api/ops/ -v
```

Expected: PASS for `TestOpsBasicAuth_Rejects` (4 subtests). The "correct creds" subtest hits `Landing` which requires `LoadTemplates` was called — confirmed by the test's setup.

- [ ] **Step 7: Commit**

```bash
git add internal/api/ops/
git commit -m "feat(airdrop): add /ops HTML console with inline Basic Auth"
```

---

## Task 14: Wire all routes into `cmd/server/main.go` + push PR

**Why:** Until now the new code exists but is unreachable. This task plugs the four new HTTP surfaces (claim, balance, stuck-claims, ops) into the running service, runs the auto-init nonce step at boot, and opens the PR.

**Files:**
- Modify: `cmd/server/main.go` — repos, services, KMS signer, RPC client, route registration
- Modify: `internal/api/server.go` — add fields if needed for cross-handler dependencies

- [ ] **Step 1: Wire dependencies in `cmd/server/main.go`**

Add imports near the top of the file (alongside existing service imports):

```go
import (
	// ... existing imports ...
	"math/big"

	"github.com/ethereum/go-ethereum/common"

	airdropops "github.com/vultisig/agent-backend/internal/api/ops"
	airdroprelayerapi "github.com/vultisig/agent-backend/internal/api/relayer"
	airdropethclient "github.com/vultisig/agent-backend/internal/service/airdrop/ethclient"
	airdropkms "github.com/vultisig/agent-backend/internal/service/airdrop/kms"
	"github.com/vultisig/agent-backend/internal/service/airdrop/quests"
	airdroprelayer "github.com/vultisig/agent-backend/internal/service/airdrop/relayer"
)
```

Insert this block after the existing `txProposalRepo` instantiation (around line 210, just before `server := api.NewServer(...)`):

```go
	// ---- Airdrop Stage 4 wiring ----
	airdropClaimRepo := postgres.NewClaimRepository(db.Pool())
	airdropStateRepo := postgres.NewRelayerStateRepository(db.Pool())

	airdropRPC, err := airdropethclient.NewClient(cfg.Airdrop.EVMRPCURL)
	if err != nil {
		logger.WithError(err).Fatal("airdrop: failed to dial EVM RPC")
	}
	airdropSigner, err := airdropkms.NewKMSSigner(ctx, cfg.Airdrop.KMSRelayerKeyARN)
	if err != nil {
		logger.WithError(err).Fatal("airdrop: failed to construct KMS signer")
	}

	// Auto-init the relayer nonce if it's NULL (first boot). No-op on subsequent boots.
	if err := airdroprelayer.InitNonceIfNeeded(ctx, airdropStateRepo, airdropRPC, airdropSigner.Address()); err != nil {
		logger.WithError(err).Fatal("airdrop: failed to init relayer nonce")
	}
	logger.WithField("relayer_address", airdropSigner.Address().Hex()).Info("airdrop relayer ready")

	contractAddr := common.HexToAddress(cfg.Airdrop.AirdropClaimContractAddress)
	chainID := big.NewInt(int64(cfg.Airdrop.EVMChainID))

	airdropClaimSvc := airdroprelayer.NewClaimService(airdroprelayer.ClaimServiceConfig{
		ChainID:         chainID,
		ContractAddress: contractAddr,
		ClaimRepo:       airdropClaimRepo,
		StateRepo:       airdropStateRepo,
		WinnerLookup:    airdroprelayer.NewPgxWinnerLookup(db.Pool()),
		RPC:             airdropRPC,
		Signer:          airdropSigner,
	})
	airdropRebroadcastSvc := airdroprelayer.NewRebroadcastService(airdroprelayer.RebroadcastConfig{
		ChainID: chainID, ContractAddress: contractAddr,
		ClaimRepo: airdropClaimRepo, RPC: airdropRPC, Signer: airdropSigner,
	})

	// Parse low-balance threshold (string → wei big.Int) once.
	lowBalanceWei := parseEthString(cfg.Airdrop.LowBalanceThresholdETH, logger)
```

Append the following helper anywhere in `main.go` (above `main()` works):

```go
// parseEthString parses a decimal-ETH string like "0.5" into a *big.Int wei.
// Format: an integer part, optional "." plus up to 18 decimal digits.
// Used once at boot for the airdrop low-balance threshold; reasonable to keep here
// rather than introduce a money helper package for one call site.
func parseEthString(s string, logger *logrus.Logger) *big.Int {
	dot := -1
	for i, r := range s {
		if r == '.' {
			dot = i
			break
		}
	}
	intStr, fracStr := s, ""
	if dot >= 0 {
		intStr = s[:dot]
		fracStr = s[dot+1:]
	}
	for len(fracStr) < 18 {
		fracStr += "0"
	}
	if len(fracStr) > 18 {
		fracStr = fracStr[:18]
	}
	combined := intStr + fracStr
	wei, ok := new(big.Int).SetString(combined, 10)
	if !ok {
		logger.WithField("raw", s).Fatal("airdrop: failed to parse LOW_BALANCE_THRESHOLD_ETH")
	}
	return wei
}
```

After `server.SetCreditRepo(...)` (around line 238), add:

```go
	server.SetInternalToken(cfg.Airdrop.InternalAPIKey)
```

(Already exists if shared infra fully wired — `grep -n SetInternalToken cmd/server/main.go` to check; if absent, add it.)

- [ ] **Step 2: Register the routes**

Replace the `agentGroup` route block (around line 309) with the following extension. Add **after** the existing `/agent` and `/auth` routes:

```go
	// ---- Airdrop Stage 4 routes ----

	// JWT-scoped: POST /airdrop/claim
	airdropEligibility := quests.NewEligibilityChecker(db.Pool()) // small wrapper exposing IsClaimEligible(ctx, pk, threshold)
	claimH := &api.AirdropClaimHandler{
		Svc:               airdropClaimSvc,
		Eligibility:       airdropEligibility,
		QuestThreshold:    cfg.Airdrop.QuestThreshold,
		ClaimWindowOpenAt: cfg.Airdrop.ClaimWindowOpenAt,
		ClaimEnabled:      cfg.Airdrop.ClaimEnabled,
		Now:               time.Now,
	}
	airdropGroup := e.Group("/airdrop", server.AuthMiddleware)
	airdropGroup.POST("/claim", claimH.Handle)

	// X-Internal-Token-scoped: GET /internal/relayer/balance, GET /internal/relayer/stuck-claims
	balanceH := &airdroprelayerapi.BalanceHandler{
		RPC:                 airdropRPC,
		StateRepo:           airdropStateRepo,
		RelayerAddress:      airdropSigner.Address(),
		ContractAddress:     contractAddr,
		LowBalanceWei:       lowBalanceWei,
		LowBalanceEthString: cfg.Airdrop.LowBalanceThresholdETH,
		Logger:              logger,
	}
	stuckH := &airdroprelayerapi.StuckClaimsHandler{Repo: airdropClaimRepo, Logger: logger}

	internalRelayer := e.Group("/internal/relayer", server.InternalTokenMiddleware)
	internalRelayer.GET("/balance", balanceH.Handle)
	internalRelayer.GET("/stuck-claims", stuckH.Handle)

	// Basic Auth-scoped: /ops/* HTML pages
	opsHandler := &airdropops.Handler{
		Username: cfg.Airdrop.OpsUsername,
		Password: cfg.Airdrop.OpsPassword,
		BalanceLoader: func(c echo.Context) (airdropops.BalanceView, error) {
			// In-process call: invoke the JSON handler against a captured ResponseRecorder
			// would be silly; instead, build the view directly.
			ctx := c.Request().Context()
			ethBal, _ := airdropRPC.Balance(ctx, airdropSigner.Address())
			vultBal, _ := airdropRPC.BalanceOf(ctx, contractAddr, contractAddr)
			st, _ := airdropStateRepo.Get(ctx)
			nonce := int64(0)
			if st != nil && st.NextNonce != nil {
				nonce = *st.NextNonce
			}
			estimated := int64(0)
			baseFee, _ := airdropRPC.LatestBaseFee(ctx)
			tip, _ := airdropRPC.SuggestGasTipCap(ctx)
			if baseFee != nil && tip != nil {
				gasPrice := new(big.Int).Add(baseFee, tip)
				if gasPrice.Sign() > 0 {
					cost := new(big.Int).Mul(gasPrice, big.NewInt(62_000))
					if ethBal != nil && cost.Sign() > 0 {
						estimated = new(big.Int).Quo(ethBal, cost).Int64()
					}
				}
			}
			view := airdropops.BalanceView{
				RelayerAddress:           airdropSigner.Address().Hex(),
				EthBalanceHuman:          formatWeiToEthShort(ethBal),
				VultContractBalanceHuman: formatWeiToEthShort(vultBal),
				LowBalanceWarning:        ethBal != nil && ethBal.Cmp(lowBalanceWei) < 0,
				LowBalanceThresholdEth:   cfg.Airdrop.LowBalanceThresholdETH,
				EstimatedClaimsRemaining: estimated,
				NextNonce:                nonce,
			}
			return view, nil
		},
		StuckClaimsLoader: func(c echo.Context) (airdropops.StuckClaimsView, error) {
			rows, err := airdropClaimRepo.ListStuck(c.Request().Context(), 10)
			if err != nil {
				return airdropops.StuckClaimsView{}, err
			}
			view := airdropops.StuckClaimsView{Count: len(rows)}
			now := time.Now().UTC()
			for _, r := range rows {
				prev := r.PreviousTxHashes
				if prev == nil {
					prev = []string{}
				}
				view.Rows = append(view.Rows, airdropops.StuckClaimRow{
					PublicKey:          r.PublicKey,
					TxHash:             r.TxHash,
					Nonce:              r.Nonce,
					SubmittedAt:        r.SubmittedAt.Format(time.RFC3339),
					MinutesPending:     int64(now.Sub(r.SubmittedAt).Minutes()),
					MaxFeeGwei:         r.MaxFeeGwei,
					MaxPriorityFeeGwei: r.MaxPriorityFeeGwei,
					PreviousTxHashes:   prev,
				})
			}
			return view, nil
		},
		ClaimRepo:          airdropClaimRepo,
		RebroadcastService: airdropRebroadcastSvc,
		Logger:             logger,
	}
	if err := opsHandler.LoadTemplates(); err != nil {
		logger.WithError(err).Fatal("ops: failed to load templates")
	}
	e.GET("/ops", opsHandler.Landing)
	e.GET("/ops/balance", opsHandler.Balance)
	e.GET("/ops/stuck-claims", opsHandler.StuckClaims)
	e.POST("/ops/rebroadcast", opsHandler.Rebroadcast)
```

Add `formatWeiToEthShort` helper near `parseEthString`:

```go
// formatWeiToEthShort divides wei by 1e18, formats with 3 decimal places.
func formatWeiToEthShort(wei *big.Int) string {
	if wei == nil {
		return "0"
	}
	one := new(big.Int).Exp(big.NewInt(10), big.NewInt(18), nil)
	intPart := new(big.Int)
	frac := new(big.Int)
	intPart.QuoRem(wei, one, frac)
	fracStr := frac.String()
	for len(fracStr) < 18 {
		fracStr = "0" + fracStr
	}
	return fmt.Sprintf("%s.%s", intPart.String(), fracStr[:3])
}
```

The `claimHandler` struct from Task 8 is package-private. Either export it (`AirdropClaimHandler`) or expose a constructor `func NewAirdropClaimHandler(...) echo.HandlerFunc`. The latter keeps the type private; recommended:

In `internal/api/airdrop_claim.go`, export the constructor:

```go
type AirdropClaimHandler struct {
	Svc               claimSubmitter
	Eligibility       eligibilityChecker
	QuestThreshold    int
	ClaimWindowOpenAt time.Time
	ClaimEnabled      bool
	Now               func() time.Time
}

func (h *AirdropClaimHandler) Handle(c echo.Context) error {
	// Re-uses the existing claimHandler.Handle logic — copy or adapt.
	// (Easiest: rename claimHandler → AirdropClaimHandler in Task 8 and skip this.)
}
```

Equivalently, rename `claimHandler` → `AirdropClaimHandler` in Task 8's commit — cleaner. Either approach works; pick one and stay consistent.

The `quests.NewEligibilityChecker(pool)` shim is a 5-line adapter — Stage 1 ships `quests.IsClaimEligible` as a free function, and the handler wants an interface. Add this to `internal/service/airdrop/quests/eligibility.go` if it isn't already there:

```go
import "github.com/jackc/pgx/v5/pgxpool"

type EligibilityChecker struct{ pool *pgxpool.Pool }

func NewEligibilityChecker(pool *pgxpool.Pool) *EligibilityChecker {
	return &EligibilityChecker{pool: pool}
}

func (e *EligibilityChecker) IsClaimEligible(ctx context.Context, publicKey string, threshold int) (bool, string, error) {
	return IsClaimEligible(ctx, e.pool, publicKey, threshold)
}
```

If Stage 1 already defined this, skip.

- [ ] **Step 3: Build and run locally**

```bash
go build ./cmd/server ./cmd/scheduler
```

Expected: PASS for both.

```bash
make build
DATABASE_DSN="postgres://..." \
  REDIS_URI="redis://localhost:6379" \
  AI_API_KEY="..." \
  JWT_SECRET="..." \
  MCP_SERVER_URL="..." \
  TRANSITION_WINDOW_START="2026-04-25T00:00:00Z" \
  TRANSITION_WINDOW_END="2026-04-30T00:00:00Z" \
  INTERNAL_API_KEY="local-internal-token" \
  QUEST_ACTIVATION_TIMESTAMP="2026-05-01T00:00:00Z" \
  KMS_RELAYER_KEY_ARN="arn:aws:kms:..." \
  EVM_RPC_URL="https://ethereum-rpc.publicnode.com" \
  AIRDROP_CLAIM_CONTRACT_ADDRESS="0x0000000000000000000000000000000000000000" \
  CLAIM_WINDOW_OPEN_AT="2026-05-15T00:00:00Z" \
  CLAIM_ENABLED="true" \
  OPS_USERNAME="ops" \
  OPS_PASSWORD="ops-secret" \
  ./bin/server
```

In another terminal:

```bash
# Internal endpoint smoke (no JWT, just the shared secret)
curl -s -H "X-Internal-Token: local-internal-token" \
  http://localhost:8080/internal/relayer/balance | jq .

curl -s -H "X-Internal-Token: local-internal-token" \
  http://localhost:8080/internal/relayer/stuck-claims | jq .

# Ops UI
curl -u ops:ops-secret -i http://localhost:8080/ops
curl -u ops:ops-secret -i http://localhost:8080/ops/balance
```

Expected:
- `/internal/relayer/balance` returns the JSON shape from the spec (with `eth_balance_wei`, `vult_contract_balance_wei`, etc.). Note: against a real KMS key, ETH balance is whatever's on the relayer wallet; against LocalStack KMS the GetPublicKey call fails — staging is the real test bed.
- `/internal/relayer/stuck-claims` returns `{"stuck_claims": [], "count": 0}`.
- `/ops` returns 200 with HTML (links to balance + stuck-claims).
- Without `-u ops:ops-secret`, all `/ops/*` URLs return 401 with `WWW-Authenticate: Basic realm="airdrop-ops"` header.

- [ ] **Step 4: Commit the wiring**

```bash
git add cmd/server/main.go internal/api/airdrop_claim.go internal/service/airdrop/quests/eligibility.go
git commit -m "feat(airdrop): wire stage 4 routes into cmd/server"
```

- [ ] **Step 5: Run the full test suite + lint**

```bash
go test ./... -timeout 120s
make lint
```

Expected: PASS. Investigate any failures before pushing.

- [ ] **Step 6: Push branch**

```bash
git push -u origin feat/airdrop-stage-4-claim-relayer
```

- [ ] **Step 7: Open the PR**

```bash
gh pr create --title "feat(airdrop): Stage 4 claim relayer (POST /airdrop/claim, /ops console, monitor)" --body "$(cat <<'EOF'
## Summary

Stage 4 of the Vultiverse Airdrop pipeline — the on-chain claim relayer.

- \`POST /airdrop/claim\` (JWT) — synchronous KMS-signed EIP-1559 broadcast under a SELECT FOR UPDATE lock. Idempotent retries return the existing tx_hash.
- \`/ops/*\` (Basic Auth) — server-rendered HTML console: landing, balance (auto-refresh, with low-balance banner), stuck-claims with per-row "Rebroadcast (+50 gwei)" form.
- \`POST /ops/rebroadcast\` — re-signs at the same nonce with bumped fees, updates the row in place. Audit trail in structured logs.
- Confirmation monitor goroutine inside cmd/scheduler — polls every \`CONFIRMATION_POLL_INTERVAL_SECONDS\` (default 15s), flips submitted → confirmed/failed.
- Auto-init relayer nonce on first boot via SELECT FOR UPDATE (no manual psql UPDATE needed).
- No \`/internal/relayer/*\` HTTP routes — balance + stuck-claims are served directly from /ops over Basic Auth (cleanup pass simplification).

Spec source of truth: \`docs/airdrop-specs/stage-4-claim-relayer.md\`.

## Architecture

- **Concurrency model:** the spec is explicit. SELECT FOR UPDATE on \`agent_relayer_state WHERE id=1\` serialises every claim handler. Inside the txn: load nonce → fetch winner → build/sign/broadcast → INSERT claim row → increment nonce → COMMIT. ROLLBACK on any failure (so nonce stays unchanged, no claim_submissions row, user can retry).
- **Idempotency:** the partial unique index \`agent_claim_submissions (public_key) WHERE status IN ('submitted','confirmed')\` is the DB-level guard. Concurrent same-user calls: one wins the index, the rest read the existing row.
- **Restart-safety:** the monitor's first tick on boot picks up any submitted rows from before a crash. Tested by \`TestMonitor_RestartSafety\`.
- **Operator UI:** server-rendered HTML, no JS build, CSS inlined. Auto-refresh via \`<meta http-equiv="refresh" content="30">\`. Basic Auth check inlined in the handler — no shared middleware (per cleanup pass).
- **Confirmation monitor lives in cmd/scheduler** alongside the existing tasks poller; both share ctx + signal handler.

## Spec coverage (vs "Done when" checklist)

- ✅ \`/airdrop/claim\` returns documented shapes against documented status codes (\`TestClaimHandler_StatusCodeMatrix\`)
- ✅ DB unique partial index enforces "one claim per user" — concurrency test passes (\`TestClaimService_Submit_ConcurrentSameUser_SingleTx\`: 50 concurrent → 1 tx + 49 idempotent retries)
- ✅ 50 concurrent distinct-user requests submit all 50 with distinct nonces (\`TestClaimService_Submit_ConcurrentDistinctUsers_DistinctNonces\`)
- ✅ Confirmation monitor flips submitted → confirmed (\`TestMonitor_OneTick_FlipsConfirmed\`)
- ✅ Restart test (\`TestMonitor_RestartSafety\`)
- ✅ Rebroadcast endpoint succeeds with bumped gas (\`TestRebroadcastService_BumpsFeesAndAppendsHash\`)
- ✅ \`/internal/relayer/balance\` returns sane values; low-balance flag flips at the configured threshold (\`TestBalanceHandler_LowBalanceFlag\`)
- ✅ \`/internal/relayer/stuck-claims\` lists pending-too-long rows; empty when none (\`TestStuckClaimsHandler_DefaultThreshold\`, \`...CustomThreshold\`)
- ✅ \`/ops\` UI reachable behind Basic Auth (\`TestOpsBasicAuth_Rejects\`)
- ⏭ E2E test against Foundry-deployed AirdropClaim.sol — runbook'd but not in CI (depends on contract repo's deploy script being checked in; cross-repo E2E lives in the contract team's PR)
- ⏭ Day 28 runbook (\`docs/runbooks/relayer-day-28.md\`) — write before the Day 28 deadline; not blocking this PR

## Test plan
- [ ] \`go test ./... -timeout 120s\` passes (with DATABASE_DSN set; integration tests skip otherwise)
- [ ] \`make lint\` clean
- [ ] \`make build\` produces \`bin/server\` and \`bin/scheduler\`
- [ ] Local smoke: server boots; \`/internal/relayer/balance\` + \`/ops\` return expected shapes (against a public mainnet RPC)
- [ ] Anvil-backed integration: spin up Anvil, deploy AirdropClaim fixture, exercise full claim → monitor → confirmed loop

## Hard deadline
Live in production by Day 28. Bring up in staging behind \`CLAIM_ENABLED=false\`, smoke-test against Sepolia, then flip the switch on Day 28.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Capture and report the PR URL.

---

## Self-review notes

This plan was reviewed against `stage-4-claim-relayer.md` after writing. Each "Done when" bullet, mapped to where it's covered:

- **`/airdrop/claim` returns documented shapes against documented status codes** — Task 8 (`TestClaimHandler_StatusCodeMatrix` covers all seven status codes from `stage-4-claim-relayer.md:74-85`).
- **DB unique partial index enforces "one claim per user" — 50 concurrent same-user requests yield exactly one tx** — Task 7 (`TestClaimService_Submit_ConcurrentSameUser_SingleTx`, with `len(rpc.sentTxs) == 1` assertion) plus the migration in shared infra (`idx_claim_submissions_one_per_user`).
- **50 concurrent distinct-user requests submit all 50 in ~serialised order, all with distinct nonces** — Task 7 (`TestClaimService_Submit_ConcurrentDistinctUsers_DistinctNonces`).
- **Confirmation monitor flips submitted → confirmed against Anvil within seconds** — Task 9 (`TestMonitor_OneTick_FlipsConfirmed` covers the DB transition; full Anvil-backed E2E runs in the integration suite via Task 1's `anviltest` helper, not in CI).
- **Restart test: kill mid-pipeline, restart, observe monitor pick up unfinished work** — Task 9 (`TestMonitor_RestartSafety` runs `Run()` cold and confirms the boot tick catches a pre-seeded submitted row).
- **Rebroadcast endpoint succeeds with bumped gas; original tx is replaced** — Task 12 (`TestRebroadcastService_BumpsFeesAndAppendsHash` checks new tx_hash, prior in array, fees bumped, nonce unchanged).
- **`/internal/relayer/balance` returns sane values; low-balance flag flips at the configured threshold** — Task 10 (both `TestBalanceHandler_HappyPath` and `TestBalanceHandler_LowBalanceFlag`).
- **`/internal/relayer/stuck-claims` lists pending-too-long rows; empty when none** — Task 11 (`TestStuckClaimsHandler_DefaultThreshold` populated, `...CustomThreshold` empty).
- **`/ops` UI is reachable behind Basic Auth and renders the three pages correctly** — Task 13 (`TestOpsBasicAuth_Rejects` covers all four auth states; the templates render via `LoadTemplates`). Manual smoke in Task 14 Step 3 confirms HTML output.
- **E2E test against Foundry-deployed AirdropClaim.sol** — deferred. The contract repo's deploy script + an integration test that boots Anvil, deploys the contract, calls setWinners, runs a claim through this relayer end-to-end is a cross-repo concern. Task 1's `anviltest` helper is the seed for that work; the plan calls it out as a follow-up runbook test, not in this PR.
- **Day 28 runbook (`docs/runbooks/relayer-day-28.md`)** — out of scope for this implementation PR. Spec asks for it as a deliverable; recommend writing it as a separate PR before Day 28 (no engineering risk; pure operational).
- **Stuck-tx runbook (`docs/runbooks/relayer-stuck-tx.md`)** — same as above. Not in this PR.

Cleanup-pass items honoured:

- **No separate `/internal/relayer/rebroadcast` endpoint** — collapsed into `POST /ops/rebroadcast` (Task 13). The spec at line 167-168 makes this explicit.
- **Inline Basic Auth check, no shared middleware** — Task 13's `Handler.basicAuthOK` uses `subtle.ConstantTimeCompare` directly inside the ops package.
- **Auto-init relayer nonce on first boot** — Task 6 (`InitNonceIfNeeded`) wired into `cmd/server/main.go` Step 1 of Task 14. No manual psql UPDATE in any runbook.
- **Status response addition (`claim_tx_hash`)** — out of scope here. Stage 1's status handler reads the same `agent_claim_submissions` rows this stage writes; no Stage 4 code touches it.

Items intentionally NOT in this plan:

- **Prometheus metrics for the relayer** (counter histogram set listed at `stage-4-claim-relayer.md:275-289`). The repo's `internal/metrics/` package is the right home; pattern is already established in `scheduler.go`. Adding a `relayer.go` metrics file with the listed counters is mechanical and ~80 lines — left as a follow-up PR to keep this one focused on functionality. Recommend a small `feat(airdrop): add relayer Prometheus metrics` PR before Day 28.
- **`BlockNumber` ethclient method** — added in shared-infra Task 5; consumed here in Task 10's balance handler to populate `RpcBlockHeight`. No follow-up needed.
- **LocalStack-backed KMS integration test** — shared infra's plan flagged this as a Stage 4 deliverable, but in practice the production path needs a real KMS key (the relayer wallet); the LocalStack round-trip test would only confirm what `TestSignRoundTrip` in shared infra already proves (DER decoding + recovery byte selection). Skip in favour of staging-environment smoke tests against the real KMS key.
- **Runbooks** (covered above).

Stage 4 has no follow-on stage. Once this PR ships and the runbooks are written, the airdrop pipeline is complete from boarding through claim.
