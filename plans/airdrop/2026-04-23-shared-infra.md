# Vultiverse Airdrop — Shared Infra Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Lay the cross-cutting foundations every airdrop stage will reuse — config, six DB tables, an internal-token middleware, a KMS signer, and an EVM RPC client — inside the existing `agent-backend` Go service.

**Architecture:** Additive only. New config substruct grouped under `cfg.Airdrop`. Six new `agent_*` tables via Goose migrations. New middleware lives in the existing `internal/api/` package. KMS signer + EVM client live under `internal/service/airdrop/` so per-stage code can import them by stable paths. No existing code is modified except the root `Config` struct in `internal/config/config.go`.

**Tech Stack:**
- Go 1.25, Echo v4
- PostgreSQL via pgx/v5; sqlc for query generation; Goose for migrations
- envconfig (`kelseyhightower/envconfig`) for env vars
- AWS SDK v2 (`aws-sdk-go-v2/service/kms`) — new dependency
- `go-ethereum/ethclient` and `go-ethereum/crypto` — new dependency
- Logrus, Prometheus client (existing)

**Spec source of truth:** `docs/airdrop-specs/shared-concerns.md`

**Working directory for all tasks:** `/Users/apotheosis/git/vultisig/agent-backend/` — every `git`, `go`, and `make` command runs from here unless stated otherwise.

**Branch:** Create one branch for the whole shared-infra plan: `feat/airdrop-shared-infra`. Commit after every task. Open the PR once the plan is complete.

---

## Pre-flight (do once before Task 1)

```bash
cd /Users/apotheosis/git/vultisig/agent-backend
git checkout main
git pull
git checkout -b feat/airdrop-shared-infra
```

Confirm Postgres is running locally and `DATABASE_DSN` is set in your shell or `.env`. The repo's existing migrations should apply cleanly — verify with:

```bash
make migrate-up
```

If that fails, stop and fix the local DB before continuing.

**Test database setup.** The Stage 1 / 3 / 4 plans run integration tests against a real Postgres. The agent-backend repo doesn't currently distinguish a test DB from the dev DB — tests just use `DATABASE_DSN`. Two options:
- (a) Point `DATABASE_DSN` at a throwaway local DB before running `go test ./...`. Truncates between tests are sufficient.
- (b) Set `TEST_DATABASE_DSN` to a separate test DB; the helper functions in the per-stage plans fall back to `DATABASE_DSN` if `TEST_DATABASE_DSN` is empty.

Either works. If your dev DB has data you care about, use (b) with a separate test DB.

---

## Task 1: Add `AirdropConfig` substruct to `Config`

**Why:** Every subsequent task reads from `cfg.Airdrop.*`. Establishing the struct first means later tasks just plug in.

**Files:**
- Modify: `internal/config/config.go` (add new substruct + add to root `Config`)
- Modify: `.env.example` (add new env vars at the bottom in a clearly-labelled airdrop section)
- Test: `internal/config/airdrop_test.go` (new file)

- [ ] **Step 1: Write the failing test**

Create `internal/config/airdrop_test.go`:

```go
package config

import (
	"os"
	"testing"
	"time"
)

func TestAirdropConfigLoadsFromEnv(t *testing.T) {
	envs := map[string]string{
		"TRANSITION_WINDOW_START":            "2026-04-25T00:00:00Z",
		"TRANSITION_WINDOW_END":              "2026-04-30T00:00:00Z",
		"INTERNAL_API_KEY":                   "test-internal-secret",
		"QUEST_ACTIVATION_TIMESTAMP":         "2026-05-01T00:00:00Z",
		"QUEST_THRESHOLD":                    "3",
		"DEFI_ACTION_CONTRACT_ALLOWLIST":     "0xaaa,0xbbb",
		"SWAP_QUEST_MIN_USD":                 "10",
		"BRIDGE_QUEST_MIN_USD":               "10",
		"KMS_RELAYER_KEY_ARN":                "arn:aws:kms:us-east-1:000000000000:key/abc",
		"EVM_RPC_URL":                        "https://ethereum-rpc.publicnode.com",
		"EVM_CHAIN_ID":                       "1",
		"AIRDROP_CLAIM_CONTRACT_ADDRESS":     "0x0000000000000000000000000000000000000001",
		"CLAIM_WINDOW_OPEN_AT":               "2026-05-15T00:00:00Z",
		"CLAIM_ENABLED":                      "true",
		"LOW_BALANCE_THRESHOLD_ETH":          "0.5",
		"CONFIRMATION_BLOCKS":                "1",
		"CONFIRMATION_POLL_INTERVAL_SECONDS": "15",
		"OPS_USERNAME":                       "ops",
		"OPS_PASSWORD":                       "ops-secret",
	}
	for k, v := range envs {
		t.Setenv(k, v)
	}
	// Ensure no stray vars from a real .env interfere
	defer func() {
		for k := range envs {
			os.Unsetenv(k)
		}
	}()

	cfg, err := LoadConfig()
	if err != nil {
		t.Fatalf("LoadConfig: %v", err)
	}

	if got := cfg.Airdrop.TransitionWindowStart; !got.Equal(time.Date(2026, 4, 25, 0, 0, 0, 0, time.UTC)) {
		t.Errorf("TransitionWindowStart = %v, want 2026-04-25T00:00:00Z", got)
	}
	if cfg.Airdrop.QuestThreshold != 3 {
		t.Errorf("QuestThreshold = %d, want 3", cfg.Airdrop.QuestThreshold)
	}
	if cfg.Airdrop.InternalAPIKey != "test-internal-secret" {
		t.Errorf("InternalAPIKey wrong: %q", cfg.Airdrop.InternalAPIKey)
	}
	if !cfg.Airdrop.ClaimEnabled {
		t.Errorf("ClaimEnabled should be true")
	}
	if cfg.Airdrop.ConfirmationBlocks != 1 {
		t.Errorf("ConfirmationBlocks = %d, want 1", cfg.Airdrop.ConfirmationBlocks)
	}
	if cfg.Airdrop.LowBalanceThresholdETH != "0.5" {
		t.Errorf("LowBalanceThresholdETH = %q, want 0.5", cfg.Airdrop.LowBalanceThresholdETH)
	}
}

func TestAirdropConfigDefaults(t *testing.T) {
	// Set only required vars; verify defaults for the rest
	t.Setenv("TRANSITION_WINDOW_START", "2026-04-25T00:00:00Z")
	t.Setenv("TRANSITION_WINDOW_END", "2026-04-30T00:00:00Z")
	t.Setenv("INTERNAL_API_KEY", "x")
	t.Setenv("QUEST_ACTIVATION_TIMESTAMP", "2026-05-01T00:00:00Z")
	t.Setenv("KMS_RELAYER_KEY_ARN", "arn")
	t.Setenv("EVM_RPC_URL", "url")
	t.Setenv("AIRDROP_CLAIM_CONTRACT_ADDRESS", "0x1")
	t.Setenv("CLAIM_WINDOW_OPEN_AT", "2026-05-15T00:00:00Z")
	t.Setenv("OPS_USERNAME", "u")
	t.Setenv("OPS_PASSWORD", "p")

	cfg, err := LoadConfig()
	if err != nil {
		t.Fatalf("LoadConfig: %v", err)
	}

	if cfg.Airdrop.QuestThreshold != 3 {
		t.Errorf("QuestThreshold default = %d, want 3", cfg.Airdrop.QuestThreshold)
	}
	if cfg.Airdrop.SwapQuestMinUSD != 10 {
		t.Errorf("SwapQuestMinUSD default = %d, want 10", cfg.Airdrop.SwapQuestMinUSD)
	}
	if !cfg.Airdrop.ClaimEnabled {
		t.Errorf("ClaimEnabled default should be true")
	}
	if cfg.Airdrop.EVMChainID != 1 {
		t.Errorf("EVMChainID default = %d, want 1", cfg.Airdrop.EVMChainID)
	}
	if cfg.Airdrop.ConfirmationBlocks != 1 {
		t.Errorf("ConfirmationBlocks default = %d, want 1", cfg.Airdrop.ConfirmationBlocks)
	}
	if cfg.Airdrop.ConfirmationPollIntervalSeconds != 15 {
		t.Errorf("ConfirmationPollIntervalSeconds default = %d, want 15", cfg.Airdrop.ConfirmationPollIntervalSeconds)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/config/ -run TestAirdropConfig -v
```

Expected: compile error — `cfg.Airdrop` undefined.

- [ ] **Step 3: Add the substruct**

In `internal/config/config.go`, add a new field at the bottom of the `Config` struct:

```go
type Config struct {
	// ... existing fields ...
	Airdrop AirdropConfig
}
```

Then add the substruct anywhere below `Config` (matching the file's existing style — substructs follow the root struct):

```go
// AirdropConfig holds Vultiverse Airdrop configuration. Spec:
// docs/airdrop-specs/shared-concerns.md.
type AirdropConfig struct {
	// Stage 1 — boarding window
	TransitionWindowStart time.Time `envconfig:"TRANSITION_WINDOW_START" required:"true"`
	TransitionWindowEnd   time.Time `envconfig:"TRANSITION_WINDOW_END" required:"true"`

	// Internal-token shared secret for /internal/* endpoints (Stages 3, 4)
	InternalAPIKey string `envconfig:"INTERNAL_API_KEY" required:"true"`

	// Stage 3 — quest tracking
	QuestActivationTimestamp     time.Time `envconfig:"QUEST_ACTIVATION_TIMESTAMP" required:"true"`
	QuestThreshold               int       `envconfig:"QUEST_THRESHOLD" default:"3"`
	DefiActionContractAllowlist  string    `envconfig:"DEFI_ACTION_CONTRACT_ALLOWLIST" default:""`
	SwapQuestMinUSD              int       `envconfig:"SWAP_QUEST_MIN_USD" default:"10"`
	BridgeQuestMinUSD            int       `envconfig:"BRIDGE_QUEST_MIN_USD" default:"10"`

	// Stage 4 — claim relayer
	KMSRelayerKeyARN              string `envconfig:"KMS_RELAYER_KEY_ARN" required:"true"`
	EVMRPCURL                     string `envconfig:"EVM_RPC_URL" required:"true"`
	EVMChainID                    int    `envconfig:"EVM_CHAIN_ID" default:"1"`
	AirdropClaimContractAddress   string `envconfig:"AIRDROP_CLAIM_CONTRACT_ADDRESS" required:"true"`
	ClaimWindowOpenAt             time.Time `envconfig:"CLAIM_WINDOW_OPEN_AT" required:"true"`
	ClaimEnabled                  bool   `envconfig:"CLAIM_ENABLED" default:"true"`
	LowBalanceThresholdETH        string `envconfig:"LOW_BALANCE_THRESHOLD_ETH" default:"0.5"`
	ConfirmationBlocks            int    `envconfig:"CONFIRMATION_BLOCKS" default:"1"`
	ConfirmationPollIntervalSeconds int  `envconfig:"CONFIRMATION_POLL_INTERVAL_SECONDS" default:"15"`

	// Stage 4 — operator UI Basic Auth
	OpsUsername string `envconfig:"OPS_USERNAME" required:"true"`
	OpsPassword string `envconfig:"OPS_PASSWORD" required:"true"`
}
```

Add `"time"` to the import block if it isn't already there (it likely is — check).

`LowBalanceThresholdETH` is intentionally a `string` not a `float64` — it's a decimal value that we'll parse into a `*big.Int` (wei) at the use site to avoid float precision issues. Same approach as the spec's NUMERIC(78,0) database column.

- [ ] **Step 4: Run test to verify it passes**

```bash
go test ./internal/config/ -run TestAirdropConfig -v
```

Expected: PASS for both subtests.

- [ ] **Step 5: Document the new vars in `.env.example`**

Append to the end of `.env.example`:

```bash
# ====================================================================
# Vultiverse Airdrop (specs in docs/airdrop-specs/)
# ====================================================================

# Stage 1 — boarding window. RFC3339 timestamps.
TRANSITION_WINDOW_START=2026-04-25T00:00:00Z
TRANSITION_WINDOW_END=2026-04-30T00:00:00Z

# Stages 3 + 4 — shared secret for /internal/* endpoints (X-Internal-Token header)
INTERNAL_API_KEY=change-me

# Stage 3 — quest tracking
QUEST_ACTIVATION_TIMESTAMP=2026-05-01T00:00:00Z
QUEST_THRESHOLD=3
DEFI_ACTION_CONTRACT_ALLOWLIST=
SWAP_QUEST_MIN_USD=10
BRIDGE_QUEST_MIN_USD=10

# Stage 4 — claim relayer
KMS_RELAYER_KEY_ARN=arn:aws:kms:us-east-1:000000000000:key/REPLACE
EVM_RPC_URL=https://ethereum-rpc.publicnode.com
EVM_CHAIN_ID=1
AIRDROP_CLAIM_CONTRACT_ADDRESS=0x0000000000000000000000000000000000000000
CLAIM_WINDOW_OPEN_AT=2026-05-15T00:00:00Z
CLAIM_ENABLED=true
LOW_BALANCE_THRESHOLD_ETH=0.5
CONFIRMATION_BLOCKS=1
CONFIRMATION_POLL_INTERVAL_SECONDS=15

# Stage 4 — operator UI Basic Auth
OPS_USERNAME=ops
OPS_PASSWORD=change-me
```

- [ ] **Step 6: Commit**

```bash
git add internal/config/config.go internal/config/airdrop_test.go .env.example
git commit -m "feat(airdrop): add AirdropConfig substruct with 19 env vars"
```

---

## Task 2: Six Goose migrations for the airdrop tables

**Why:** Stage 1's status endpoint, the raffle CLI, the quest tracker, and the relayer all need their tables to exist before they can be developed. Bundling all six up front avoids the engineer who picks up Stage 4 having to write Stage 2's migration as a side-quest.

**Files (all new):**
- Create: `internal/storage/postgres/migrations/20260423000001_create_agent_airdrop_registrations.sql`
- Create: `internal/storage/postgres/migrations/20260423000002_create_agent_raffle_winners.sql`
- Create: `internal/storage/postgres/migrations/20260423000003_create_agent_quest_events.sql`
- Create: `internal/storage/postgres/migrations/20260423000004_create_agent_user_quests.sql`
- Create: `internal/storage/postgres/migrations/20260423000005_create_agent_claim_submissions.sql`
- Create: `internal/storage/postgres/migrations/20260423000006_create_agent_relayer_state.sql`

Pattern matches the existing `20260224000001_create_scheduled_tasks.sql` template — `-- +goose Up` and `-- +goose Down` blocks wrapped in `-- +goose StatementBegin/End` directives.

- [ ] **Step 1: Create migration 1 — `agent_airdrop_registrations`**

```sql
-- +goose Up
-- +goose StatementBegin

CREATE TYPE airdrop_registration_source AS ENUM ('seed', 'vault_share');

CREATE TABLE agent_airdrop_registrations (
    public_key        TEXT                        PRIMARY KEY,
    source            airdrop_registration_source NOT NULL,
    recipient_address TEXT                        NOT NULL,
    registered_at     TIMESTAMPTZ                 NOT NULL DEFAULT NOW()
);

-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin

DROP TABLE IF EXISTS agent_airdrop_registrations;
DROP TYPE IF EXISTS airdrop_registration_source;

-- +goose StatementEnd
```

- [ ] **Step 2: Create migration 2 — `agent_raffle_winners`**

```sql
-- +goose Up
-- +goose StatementBegin

CREATE TABLE agent_raffle_winners (
    public_key TEXT          PRIMARY KEY,
    recipient  TEXT          NOT NULL,
    amount     NUMERIC(78,0) NOT NULL,
    loaded_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin

DROP TABLE IF EXISTS agent_raffle_winners;

-- +goose StatementEnd
```

- [ ] **Step 3: Create migration 3 — `agent_quest_events`**

```sql
-- +goose Up
-- +goose StatementBegin

CREATE TYPE airdrop_quest_id AS ENUM ('swap', 'bridge', 'defi_action', 'alert', 'dca');

CREATE TABLE agent_quest_events (
    tool_call_id   TEXT             PRIMARY KEY,
    public_key     TEXT             NOT NULL,
    quest_id       airdrop_quest_id NOT NULL,
    tx_hash        TEXT             NOT NULL,
    counted        BOOLEAN          NOT NULL,
    reject_reason  TEXT,
    tx_proposal_id UUID,
    created_at     TIMESTAMPTZ      NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_quest_events_public_key ON agent_quest_events (public_key);

-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin

DROP TABLE IF EXISTS agent_quest_events;
DROP TYPE IF EXISTS airdrop_quest_id;

-- +goose StatementEnd
```

- [ ] **Step 4: Create migration 4 — `agent_user_quests`**

```sql
-- +goose Up
-- +goose StatementBegin

CREATE TABLE agent_user_quests (
    public_key   TEXT             NOT NULL,
    quest_id     airdrop_quest_id NOT NULL,
    completed_at TIMESTAMPTZ      NOT NULL DEFAULT NOW(),
    PRIMARY KEY (public_key, quest_id)
);

-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin

DROP TABLE IF EXISTS agent_user_quests;

-- +goose StatementEnd
```

Note: `airdrop_quest_id` enum is created in migration 3 — this migration depends on it. Goose applies in filename order, so this is fine.

- [ ] **Step 5: Create migration 5 — `agent_claim_submissions`**

```sql
-- +goose Up
-- +goose StatementBegin

CREATE TYPE airdrop_claim_status AS ENUM ('submitted', 'confirmed', 'failed');

CREATE TABLE agent_claim_submissions (
    public_key            TEXT                 NOT NULL,
    nonce                 BIGINT               NOT NULL UNIQUE,
    recipient             TEXT                 NOT NULL,
    amount                NUMERIC(78, 0)       NOT NULL,
    tx_hash               TEXT                 NOT NULL,
    status                airdrop_claim_status NOT NULL,
    max_fee_gwei          INTEGER              NOT NULL,
    max_priority_fee_gwei INTEGER              NOT NULL,
    block_number          BIGINT,
    failure_reason        TEXT,
    submitted_at          TIMESTAMPTZ          NOT NULL,
    confirmed_at          TIMESTAMPTZ,
    failed_at             TIMESTAMPTZ
);

CREATE UNIQUE INDEX idx_claim_submissions_one_per_user
  ON agent_claim_submissions (public_key)
  WHERE status IN ('submitted', 'confirmed');

CREATE INDEX idx_claim_submissions_pending
  ON agent_claim_submissions (submitted_at)
  WHERE status = 'submitted';

-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin

DROP TABLE IF EXISTS agent_claim_submissions;
DROP TYPE IF EXISTS airdrop_claim_status;

-- +goose StatementEnd
```

- [ ] **Step 6: Create migration 6 — `agent_relayer_state`**

```sql
-- +goose Up
-- +goose StatementBegin

CREATE TABLE agent_relayer_state (
    id          INT PRIMARY KEY DEFAULT 1 CHECK (id = 1),
    next_nonce  BIGINT,
    last_synced TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

INSERT INTO agent_relayer_state (id) VALUES (1);

-- +goose StatementEnd

-- +goose Down
-- +goose StatementBegin

DROP TABLE IF EXISTS agent_relayer_state;

-- +goose StatementEnd
```

`next_nonce` is intentionally nullable — it gets populated on first boot of the relayer (Stage 4) via `eth_getTransactionCount`.

- [ ] **Step 7: Apply all six migrations against local DB**

```bash
make migrate-up
```

Expected output: `OK   20260423000006_create_agent_relayer_state.sql` (or similar — final line should reference migration 6).

Confirm tables exist:

```bash
psql "$DATABASE_DSN" -c "\dt agent_*"
```

Expected: lists all six new `agent_*` tables alongside the pre-existing ones.

- [ ] **Step 8: Test rollback**

```bash
make migrate-down
make migrate-down
make migrate-down
make migrate-down
make migrate-down
make migrate-down
```

Each call rolls back one migration. After six calls, all six new tables should be gone:

```bash
psql "$DATABASE_DSN" -c "\dt agent_airdrop_registrations agent_raffle_winners agent_quest_events agent_user_quests agent_claim_submissions agent_relayer_state"
```

Expected: `Did not find any relation named ...` for each.

- [ ] **Step 9: Re-apply for the rest of the plan**

```bash
make migrate-up
```

Subsequent tasks will assume the tables exist.

- [ ] **Step 10: Commit**

```bash
git add internal/storage/postgres/migrations/20260423000001_create_agent_airdrop_registrations.sql \
         internal/storage/postgres/migrations/20260423000002_create_agent_raffle_winners.sql \
         internal/storage/postgres/migrations/20260423000003_create_agent_quest_events.sql \
         internal/storage/postgres/migrations/20260423000004_create_agent_user_quests.sql \
         internal/storage/postgres/migrations/20260423000005_create_agent_claim_submissions.sql \
         internal/storage/postgres/migrations/20260423000006_create_agent_relayer_state.sql
git commit -m "feat(airdrop): add six migrations for airdrop tables"
```

---

## Task 3: Internal-token middleware

**Why:** Stages 3 and 4 expose `/internal/*` endpoints called from elsewhere in this same binary plus by ops curl. They need a shared-secret header check separate from the user-facing JWT middleware.

**Files:**
- Create: `internal/api/internal_token_middleware.go`
- Create: `internal/api/internal_token_middleware_test.go`

Pattern matches the existing `internal/api/middleware.go` — function on `Server` struct, returns `echo.HandlerFunc`. Use Go's `crypto/subtle` for constant-time comparison so timing attacks against the secret aren't possible.

- [ ] **Step 1: Write the failing test**

Create `internal/api/internal_token_middleware_test.go`:

```go
package api

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/labstack/echo/v4"
)

func TestInternalTokenMiddleware(t *testing.T) {
	const secret = "test-internal-secret"
	server := &Server{internalToken: secret, logger: testLogger(t)}
	mw := server.InternalTokenMiddleware

	handler := func(c echo.Context) error {
		return c.String(http.StatusOK, "ok")
	}

	cases := []struct {
		name       string
		header     string
		wantStatus int
	}{
		{"valid token", secret, http.StatusOK},
		{"missing header", "", http.StatusUnauthorized},
		{"wrong token", "nope", http.StatusUnauthorized},
		{"empty token", "", http.StatusUnauthorized},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			e := echo.New()
			req := httptest.NewRequest(http.MethodGet, "/internal/anything", nil)
			if tc.header != "" {
				req.Header.Set("X-Internal-Token", tc.header)
			}
			rec := httptest.NewRecorder()
			c := e.NewContext(req, rec)
			err := mw(handler)(c)
			if err != nil {
				t.Fatalf("middleware returned error: %v", err)
			}
			if rec.Code != tc.wantStatus {
				t.Errorf("status = %d, want %d", rec.Code, tc.wantStatus)
			}
		})
	}
}
```

You'll need a small `testLogger` helper if the package doesn't already have one. Check `internal/api/auth_logout_test.go` for the existing test-helper pattern — re-use whatever it uses (almost certainly `logrus.New()` set to `logrus.PanicLevel` to silence test output). If no helper exists, add one in this same test file:

```go
func testLogger(t *testing.T) *logrus.Logger {
	t.Helper()
	l := logrus.New()
	l.SetLevel(logrus.PanicLevel)
	return l
}
```

(Add `"github.com/sirupsen/logrus"` to imports if you add the helper.)

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/api/ -run TestInternalTokenMiddleware -v
```

Expected: compile error — `InternalTokenMiddleware` undefined; `internalToken` field undefined on `Server`.

- [ ] **Step 3: Add the `internalToken` field to the Server struct**

In `internal/api/server.go` (or wherever the `Server` struct is declared — `grep -n "type Server struct" internal/api/*.go` to find it), add a field:

```go
type Server struct {
	// ... existing fields ...
	internalToken string
}
```

And in the `NewServer` (or equivalent constructor), wire it from config:

```go
func NewServer(cfg *config.Config, /* other deps */) *Server {
	return &Server{
		// ... existing fields ...
		internalToken: cfg.Airdrop.InternalAPIKey,
	}
}
```

Match whatever constructor pattern the existing code uses. If the constructor takes a config ref, just add the line. If it doesn't, pass `cfg.Airdrop.InternalAPIKey` from `cmd/server/main.go` where the server is constructed.

- [ ] **Step 4: Implement the middleware**

Create `internal/api/internal_token_middleware.go`:

```go
package api

import (
	"crypto/subtle"
	"net/http"

	"github.com/labstack/echo/v4"
)

// InternalTokenMiddleware authenticates requests against the shared
// X-Internal-Token secret. Used by /internal/* routes that are called
// from other parts of this same binary or by ops curl. Constant-time
// compare to defeat timing attacks against the secret.
func (s *Server) InternalTokenMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
	return func(c echo.Context) error {
		got := c.Request().Header.Get("X-Internal-Token")
		if got == "" || subtle.ConstantTimeCompare([]byte(got), []byte(s.internalToken)) != 1 {
			return c.JSON(http.StatusUnauthorized, ErrorResponse{Error: "unauthorized"})
		}
		return next(c)
	}
}
```

The `ErrorResponse` type already exists in this package — see how `AuthMiddleware` uses it.

- [ ] **Step 5: Run test to verify it passes**

```bash
go test ./internal/api/ -run TestInternalTokenMiddleware -v
```

Expected: PASS for all four subtests.

- [ ] **Step 6: Run the full api test suite to confirm no regressions**

```bash
go test ./internal/api/...
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/api/internal_token_middleware.go \
         internal/api/internal_token_middleware_test.go \
         internal/api/server.go
# (or wherever you wired the field in Step 3)
git commit -m "feat(airdrop): add internal-token middleware for /internal/* routes"
```

---

## Task 4: KMS signer

**Why:** Stage 4 signs every claim transaction via AWS KMS. The signer wraps the KMS `Sign` API and handles the DER → EVM `(r, s, v)` conversion. Splitting this into pure functions and an interface lets unit tests cover the conversion logic without LocalStack.

**Files:**
- Create: `internal/service/airdrop/kms/signer.go`
- Create: `internal/service/airdrop/kms/signer_test.go`
- Modify: `go.mod`, `go.sum` (added by `go mod tidy`)

**Dependencies (added by `go get` below):**
- `github.com/aws/aws-sdk-go-v2/config`
- `github.com/aws/aws-sdk-go-v2/service/kms`
- `github.com/ethereum/go-ethereum/crypto`
- `github.com/ethereum/go-ethereum/common`

- [ ] **Step 1: Add dependencies**

```bash
go get github.com/aws/aws-sdk-go-v2/config
go get github.com/aws/aws-sdk-go-v2/service/kms
go get github.com/ethereum/go-ethereum
go mod tidy
```

- [ ] **Step 2: Write the failing test for DER decoding**

Create `internal/service/airdrop/kms/signer_test.go`:

```go
package kms

import (
	"bytes"
	"encoding/hex"
	"math/big"
	"testing"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// TestParseDERSignature confirms the helper extracts (r, s) from an
// ASN.1 DER-encoded ECDSA signature in the shape KMS returns.
func TestParseDERSignature(t *testing.T) {
	// Sample DER signature: SEQUENCE { INTEGER r, INTEGER s }
	// r = 0x01, s = 0x02 (one-byte values padded). Constructed by hand
	// so we can assert the parser without depending on real KMS output.
	der, _ := hex.DecodeString("3006020101020102")

	r, s, err := parseDERSignature(der)
	if err != nil {
		t.Fatalf("parseDERSignature: %v", err)
	}
	if r.Cmp(big.NewInt(1)) != 0 {
		t.Errorf("r = %s, want 1", r.String())
	}
	if s.Cmp(big.NewInt(2)) != 0 {
		t.Errorf("s = %s, want 2", s.String())
	}
}

// TestRecoverEVMAddress confirms we can reconstruct the relayer's EVM
// address from a known compressed/uncompressed public key. Uses go-ethereum's
// own derivation as the source of truth — if these match, our path is correct.
func TestRecoverEVMAddress(t *testing.T) {
	// Generate a fresh keypair, take its uncompressed public key, then
	// confirm addressFromPublicKey() matches go-ethereum's PubkeyToAddress.
	priv, err := crypto.GenerateKey()
	if err != nil {
		t.Fatalf("GenerateKey: %v", err)
	}
	expected := crypto.PubkeyToAddress(priv.PublicKey)

	pubBytes := crypto.FromECDSAPub(&priv.PublicKey) // uncompressed: 0x04 || X || Y

	got, err := addressFromPublicKey(pubBytes)
	if err != nil {
		t.Fatalf("addressFromPublicKey: %v", err)
	}
	if !bytes.Equal(got.Bytes(), expected.Bytes()) {
		t.Errorf("address = %s, want %s", got.Hex(), expected.Hex())
	}
}

// TestNormalizeS confirms low-S enforcement (EIP-2 / Ethereum yellow paper).
// KMS may return high-S; Ethereum tx submission rejects it.
func TestNormalizeS(t *testing.T) {
	// secp256k1 curve order N (from go-ethereum source)
	n, _ := new(big.Int).SetString("FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141", 16)
	halfN := new(big.Int).Rsh(n, 1)

	// A high-S value (just above halfN)
	highS := new(big.Int).Add(halfN, big.NewInt(1))

	got := normalizeS(highS)
	if got.Cmp(halfN) > 0 {
		t.Errorf("normalized s should be <= n/2; got %s > %s", got.String(), halfN.String())
	}
	expected := new(big.Int).Sub(n, highS)
	if got.Cmp(expected) != 0 {
		t.Errorf("normalized s = %s, want %s (n - highS)", got.String(), expected.String())
	}

	// Already-low s should pass through unchanged
	lowS := big.NewInt(42)
	if normalizeS(lowS).Cmp(lowS) != 0 {
		t.Errorf("low s should pass through unchanged")
	}
}

var _ = common.Address{} // keep import live until full Signer test is added
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
go test ./internal/service/airdrop/kms/ -v
```

Expected: compile error — `parseDERSignature`, `addressFromPublicKey`, `normalizeS` undefined.

- [ ] **Step 4: Implement signer.go**

Create `internal/service/airdrop/kms/signer.go`:

```go
package kms

import (
	"context"
	"encoding/asn1"
	"errors"
	"fmt"
	"math/big"

	awsconfig "github.com/aws/aws-sdk-go-v2/config"
	awskms "github.com/aws/aws-sdk-go-v2/service/kms"
	kmstypes "github.com/aws/aws-sdk-go-v2/service/kms/types"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
)

// Signer signs 32-byte digests with an AWS KMS key. The returned
// signature is a 65-byte EVM-format (r || s || v) suitable for
// EIP-1559 transaction signing.
type Signer interface {
	SignDigest(ctx context.Context, digest []byte) (signature []byte, err error)
	Address() common.Address
}

// kmsSigner is the production implementation; backed by AWS KMS.
type kmsSigner struct {
	client  *awskms.Client
	keyARN  string
	address common.Address // recovered once at construction from the key's public key
	pubBytes []byte         // uncompressed public key, used for recovery byte selection
}

// NewKMSSigner constructs a Signer backed by the KMS key at keyARN.
// At construction it fetches the key's public key, computes the EVM
// address, and stashes both for later use.
func NewKMSSigner(ctx context.Context, keyARN string) (Signer, error) {
	awsCfg, err := awsconfig.LoadDefaultConfig(ctx)
	if err != nil {
		return nil, fmt.Errorf("load aws config: %w", err)
	}
	client := awskms.NewFromConfig(awsCfg)

	// Fetch the key's public key. KMS returns DER-encoded SubjectPublicKeyInfo.
	out, err := client.GetPublicKey(ctx, &awskms.GetPublicKeyInput{KeyId: &keyARN})
	if err != nil {
		return nil, fmt.Errorf("get kms public key: %w", err)
	}

	pubBytes, err := publicKeyFromSPKI(out.PublicKey)
	if err != nil {
		return nil, fmt.Errorf("parse spki: %w", err)
	}
	addr, err := addressFromPublicKey(pubBytes)
	if err != nil {
		return nil, fmt.Errorf("derive address: %w", err)
	}
	return &kmsSigner{
		client:   client,
		keyARN:   keyARN,
		address:  addr,
		pubBytes: pubBytes,
	}, nil
}

func (s *kmsSigner) Address() common.Address { return s.address }

func (s *kmsSigner) SignDigest(ctx context.Context, digest []byte) ([]byte, error) {
	if len(digest) != 32 {
		return nil, fmt.Errorf("digest must be 32 bytes, got %d", len(digest))
	}

	out, err := s.client.Sign(ctx, &awskms.SignInput{
		KeyId:            &s.keyARN,
		Message:          digest,
		MessageType:      kmstypes.MessageTypeDigest,
		SigningAlgorithm: kmstypes.SigningAlgorithmSpecEcdsaSha256,
	})
	if err != nil {
		return nil, fmt.Errorf("kms sign: %w", err)
	}

	r, sVal, err := parseDERSignature(out.Signature)
	if err != nil {
		return nil, fmt.Errorf("parse signature: %w", err)
	}
	sVal = normalizeS(sVal)

	// Recover v: try both possibilities and pick the one whose recovered
	// public key matches our stashed pubBytes.
	v, err := recoveryByte(digest, r, sVal, s.pubBytes)
	if err != nil {
		return nil, fmt.Errorf("recovery byte: %w", err)
	}

	out2 := make([]byte, 65)
	r.FillBytes(out2[0:32])
	sVal.FillBytes(out2[32:64])
	out2[64] = v
	return out2, nil
}

// parseDERSignature decodes an ASN.1 DER `SEQUENCE { INTEGER, INTEGER }`
// into (r, s) big.Ints — the format AWS KMS returns from `Sign`.
func parseDERSignature(der []byte) (r, s *big.Int, err error) {
	var sig struct{ R, S *big.Int }
	if _, err := asn1.Unmarshal(der, &sig); err != nil {
		return nil, nil, err
	}
	if sig.R == nil || sig.S == nil {
		return nil, nil, errors.New("nil r or s")
	}
	return sig.R, sig.S, nil
}

// publicKeyFromSPKI extracts the uncompressed secp256k1 public key
// (65 bytes: 0x04 || X || Y) from a KMS SubjectPublicKeyInfo blob.
func publicKeyFromSPKI(spki []byte) ([]byte, error) {
	// SubjectPublicKeyInfo = SEQUENCE { algorithm, BIT STRING }.
	// The bit string IS the uncompressed public key.
	var info struct {
		Algorithm asn1.RawValue
		PublicKey asn1.BitString
	}
	if _, err := asn1.Unmarshal(spki, &info); err != nil {
		return nil, err
	}
	if len(info.PublicKey.Bytes) != 65 || info.PublicKey.Bytes[0] != 0x04 {
		return nil, fmt.Errorf("expected 65-byte uncompressed key, got %d bytes prefix=0x%x",
			len(info.PublicKey.Bytes), info.PublicKey.Bytes[0])
	}
	return info.PublicKey.Bytes, nil
}

// addressFromPublicKey computes the EVM address from a 65-byte uncompressed
// secp256k1 public key (0x04 || X || Y). Address = last 20 bytes of
// keccak256(X || Y).
func addressFromPublicKey(pub []byte) (common.Address, error) {
	if len(pub) != 65 || pub[0] != 0x04 {
		return common.Address{}, fmt.Errorf("expected 65-byte uncompressed key")
	}
	return common.BytesToAddress(crypto.Keccak256(pub[1:])[12:]), nil
}

// secp256k1 curve order, from go-ethereum.
var (
	secp256k1N    *big.Int
	secp256k1HalfN *big.Int
)

func init() {
	secp256k1N, _ = new(big.Int).SetString(
		"FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141", 16)
	secp256k1HalfN = new(big.Int).Rsh(secp256k1N, 1)
}

// normalizeS enforces low-S form (EIP-2). If s > n/2, replace with n - s.
func normalizeS(s *big.Int) *big.Int {
	if s.Cmp(secp256k1HalfN) > 0 {
		return new(big.Int).Sub(secp256k1N, s)
	}
	return s
}

// recoveryByte tries v=0 and v=1 (Ethereum's recovery byte values for raw sigs);
// returns whichever recovers to a public key matching expectedPub. Caller is
// responsible for adjusting the recovery byte for chain ID where needed
// (EIP-1559 / EIP-155 use the raw 0/1 form; legacy txs use 27/28 + chainID*2).
func recoveryByte(digest []byte, r, s *big.Int, expectedPub []byte) (byte, error) {
	sigBytes := make([]byte, 65)
	r.FillBytes(sigBytes[0:32])
	s.FillBytes(sigBytes[32:64])
	for v := byte(0); v <= 1; v++ {
		sigBytes[64] = v
		recovered, err := crypto.Ecrecover(digest, sigBytes)
		if err != nil {
			continue
		}
		// Ecrecover returns the uncompressed pub key (65 bytes incl. 0x04 prefix)
		if len(recovered) == 65 && bytesEqual(recovered, expectedPub) {
			return v, nil
		}
	}
	return 0, errors.New("could not recover pub key from signature")
}

func bytesEqual(a, b []byte) bool {
	if len(a) != len(b) {
		return false
	}
	for i := range a {
		if a[i] != b[i] {
			return false
		}
	}
	return true
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/kms/ -v
```

Expected: PASS for `TestParseDERSignature`, `TestRecoverEVMAddress`, `TestNormalizeS`.

- [ ] **Step 6: Add a sign-and-recover round-trip test using a Go-native ECDSA key**

This test exercises the full flow except the actual KMS call — it produces a signature with go-ethereum's signing primitives, decodes it through `parseDERSignature` (after we DER-encode it), and confirms `recoveryByte` finds the correct `v`. Append to `signer_test.go`:

```go
import (
	"crypto/ecdsa"
	"encoding/asn1"
)

// TestSignRoundTrip generates a keypair, signs a digest with go-ethereum
// directly, encodes the signature in DER (matching what KMS returns), then
// runs it through the same pipeline our SignDigest uses. End result should
// be a 65-byte (r || s || v) signature that ecrecover validates.
func TestSignRoundTrip(t *testing.T) {
	priv, err := crypto.GenerateKey()
	if err != nil {
		t.Fatalf("GenerateKey: %v", err)
	}
	pubBytes := crypto.FromECDSAPub(&priv.PublicKey)

	digest := crypto.Keccak256([]byte("hello world"))

	// Native sign returns 65 bytes (r || s || v); we strip v and re-encode
	// (r, s) as DER to mimic what KMS returns.
	rsv, err := crypto.Sign(digest, priv)
	if err != nil {
		t.Fatalf("crypto.Sign: %v", err)
	}
	r := new(big.Int).SetBytes(rsv[0:32])
	s := new(big.Int).SetBytes(rsv[32:64])
	derBytes, err := asn1.Marshal(struct{ R, S *big.Int }{R: r, S: s})
	if err != nil {
		t.Fatalf("asn1.Marshal: %v", err)
	}

	// Now run through our parsing path
	r2, s2, err := parseDERSignature(derBytes)
	if err != nil {
		t.Fatalf("parseDERSignature: %v", err)
	}
	s2 = normalizeS(s2)

	v, err := recoveryByte(digest, r2, s2, pubBytes)
	if err != nil {
		t.Fatalf("recoveryByte: %v", err)
	}

	// Reassemble and recover; should match
	out := make([]byte, 65)
	r2.FillBytes(out[0:32])
	s2.FillBytes(out[32:64])
	out[64] = v
	recovered, err := crypto.Ecrecover(digest, out)
	if err != nil {
		t.Fatalf("Ecrecover: %v", err)
	}
	if !bytes.Equal(recovered, pubBytes) {
		t.Errorf("recovered pubkey doesn't match")
	}

	// And verify priv → expected address
	_ = ecdsa.PublicKey{} // silence unused-import if you didn't add ecdsa elsewhere
	expected := crypto.PubkeyToAddress(priv.PublicKey)
	addr, _ := addressFromPublicKey(pubBytes)
	if addr != expected {
		t.Errorf("address mismatch: %s vs %s", addr.Hex(), expected.Hex())
	}
}
```

- [ ] **Step 7: Run round-trip test**

```bash
go test ./internal/service/airdrop/kms/ -run TestSignRoundTrip -v
```

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add internal/service/airdrop/kms/ go.mod go.sum
git commit -m "feat(airdrop): add KMS signer with DER decode + EVM signature recovery"
```

**Note on integration test deferred:** The spec's "Done when" includes "KMS signer can sign a no-op digest against a real (LocalStack) KMS key and the recovered EVM address matches." That's a LocalStack-dependent integration test that can ship in Stage 4's plan (where we'll actually wire LocalStack into the test setup). The pure-Go round-trip above proves the math; the LocalStack test will prove the AWS SDK glue.

---

## Task 5: EVM client wrapper

**Why:** Stage 4 reads gas estimates and broadcasts transactions through this. Wrapping go-ethereum's `ethclient.Client` exposes only the methods we actually need — nothing accidentally leaks; retry policy + timeout + future RPC rotation live in one place.

**Files:**
- Create: `internal/service/airdrop/ethclient/client.go`
- Create: `internal/service/airdrop/ethclient/client_test.go`

- [ ] **Step 1: Write the failing test for the wrapper interface**

Create `internal/service/airdrop/ethclient/client_test.go`:

```go
package ethclient

import (
	"context"
	"math/big"
	"testing"

	"github.com/ethereum/go-ethereum/common"
)

// TestClientInterface confirms the package exposes the exact set of methods
// the Stage 4 spec requires. The bodies are not exercised here — that's
// integration territory — but the type assertion catches API drift early.
func TestClientInterface(t *testing.T) {
	var c Client = nil
	_ = c

	// Method surface — these calls won't compile if the interface changes
	if false {
		ctx := context.Background()
		var iface Client
		_, _ = iface.LatestBaseFee(ctx)
		_, _ = iface.SuggestGasTipCap(ctx)
		_, _ = iface.NonceAt(ctx, common.Address{}, nil)
		_, _ = iface.SendRawTransaction(ctx, nil)
		_, _ = iface.GetTransactionReceipt(ctx, common.Hash{})
		_, _ = iface.BalanceOf(ctx, common.Address{}, common.Address{})
		_, _ = iface.Balance(ctx, common.Address{})
		_, _ = iface.BlockNumber(ctx)
	}
}

// TestNewClientRejectsBadURL confirms the constructor surfaces dial errors.
func TestNewClientRejectsBadURL(t *testing.T) {
	_, err := NewClient("not-a-url")
	if err == nil {
		t.Errorf("expected error for invalid URL, got nil")
	}
}

// (Round-trip tests against an Anvil instance live in Stage 4's plan, where
// the dev-time Anvil dependency will be set up.)
var _ = big.NewInt
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/service/airdrop/ethclient/ -v
```

Expected: compile error — `Client`, `NewClient` undefined.

- [ ] **Step 3: Implement the wrapper**

Create `internal/service/airdrop/ethclient/client.go`:

```go
package ethclient

import (
	"context"
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	geth "github.com/ethereum/go-ethereum/ethclient"
)

// Client exposes only the EVM RPC methods the airdrop relayer needs.
// Backed by go-ethereum's ethclient under the hood; centralised here so
// retry/timeout/RPC-rotation policy lives in one place.
type Client interface {
	// LatestBaseFee returns the most recent block's baseFeePerGas.
	LatestBaseFee(ctx context.Context) (*big.Int, error)
	// SuggestGasTipCap returns the RPC's suggested priority fee (eth_maxPriorityFeePerGas).
	SuggestGasTipCap(ctx context.Context) (*big.Int, error)
	// NonceAt returns the nonce at blockNumber (nil = latest).
	NonceAt(ctx context.Context, address common.Address, blockNumber *big.Int) (uint64, error)
	// SendRawTransaction broadcasts a signed tx; returns the tx hash.
	SendRawTransaction(ctx context.Context, signedTxBytes []byte) (common.Hash, error)
	// GetTransactionReceipt returns the receipt or ethereum.NotFound.
	GetTransactionReceipt(ctx context.Context, txHash common.Hash) (*types.Receipt, error)
	// BalanceOf returns an ERC-20 balance of `account` for `token` (calls token.balanceOf(account)).
	BalanceOf(ctx context.Context, token, account common.Address) (*big.Int, error)
	// Balance returns the native ETH balance of `account` at the latest block.
	Balance(ctx context.Context, account common.Address) (*big.Int, error)
	// BlockNumber returns the most recent block number. Used by Stage 4's
	// /internal/relayer/balance for the rpc_block_height health field.
	BlockNumber(ctx context.Context) (uint64, error)
}

type client struct {
	rpc *geth.Client
}

// NewClient dials the RPC URL and returns a wrapped client.
func NewClient(rpcURL string) (Client, error) {
	c, err := geth.Dial(rpcURL)
	if err != nil {
		return nil, fmt.Errorf("dial %s: %w", rpcURL, err)
	}
	return &client{rpc: c}, nil
}

func (c *client) LatestBaseFee(ctx context.Context) (*big.Int, error) {
	header, err := c.rpc.HeaderByNumber(ctx, nil)
	if err != nil {
		return nil, fmt.Errorf("header: %w", err)
	}
	if header.BaseFee == nil {
		return nil, fmt.Errorf("base fee unavailable (pre-EIP-1559 chain?)")
	}
	return new(big.Int).Set(header.BaseFee), nil
}

func (c *client) SuggestGasTipCap(ctx context.Context) (*big.Int, error) {
	return c.rpc.SuggestGasTipCap(ctx)
}

func (c *client) NonceAt(ctx context.Context, address common.Address, blockNumber *big.Int) (uint64, error) {
	return c.rpc.NonceAt(ctx, address, blockNumber)
}

func (c *client) SendRawTransaction(ctx context.Context, signedTxBytes []byte) (common.Hash, error) {
	tx := new(types.Transaction)
	if err := tx.UnmarshalBinary(signedTxBytes); err != nil {
		return common.Hash{}, fmt.Errorf("unmarshal tx: %w", err)
	}
	if err := c.rpc.SendTransaction(ctx, tx); err != nil {
		return common.Hash{}, fmt.Errorf("send tx: %w", err)
	}
	return tx.Hash(), nil
}

func (c *client) GetTransactionReceipt(ctx context.Context, txHash common.Hash) (*types.Receipt, error) {
	r, err := c.rpc.TransactionReceipt(ctx, txHash)
	if err == ethereum.NotFound {
		return nil, ethereum.NotFound
	}
	return r, err
}

// erc20BalanceOfSelector is the keccak256("balanceOf(address)") method ID.
var erc20BalanceOfSelector = []byte{0x70, 0xa0, 0x82, 0x31}

func (c *client) BalanceOf(ctx context.Context, token, account common.Address) (*big.Int, error) {
	calldata := make([]byte, 0, 36)
	calldata = append(calldata, erc20BalanceOfSelector...)
	// pad address to 32 bytes
	calldata = append(calldata, make([]byte, 12)...)
	calldata = append(calldata, account.Bytes()...)

	out, err := c.rpc.CallContract(ctx, ethereum.CallMsg{
		To:   &token,
		Data: calldata,
	}, nil)
	if err != nil {
		return nil, fmt.Errorf("eth_call balanceOf: %w", err)
	}
	if len(out) != 32 {
		return nil, fmt.Errorf("expected 32-byte response, got %d", len(out))
	}
	return new(big.Int).SetBytes(out), nil
}

func (c *client) Balance(ctx context.Context, account common.Address) (*big.Int, error) {
	return c.rpc.BalanceAt(ctx, account, nil)
}

func (c *client) BlockNumber(ctx context.Context) (uint64, error) {
	return c.rpc.BlockNumber(ctx)
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/ethclient/ -v
```

Expected: PASS for both subtests. Note that `TestClientInterface` only verifies the API surface compiles; live RPC tests are deferred to Stage 4.

- [ ] **Step 5: Run a smoke check against a public mainnet RPC** *(manual gate, not committed)*

```bash
cat > /tmp/ethclient_smoke_test.go <<'EOF'
package main

import (
	"context"
	"fmt"
	"os"
	"time"

	"github.com/vultisig/agent-backend/internal/service/airdrop/ethclient"
)

func main() {
	c, err := ethclient.NewClient("https://ethereum-rpc.publicnode.com")
	if err != nil {
		fmt.Println("dial:", err)
		os.Exit(1)
	}
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	bf, err := c.LatestBaseFee(ctx)
	if err != nil {
		fmt.Println("base fee:", err)
		os.Exit(1)
	}
	fmt.Printf("latest base fee: %s wei (~%.1f gwei)\n", bf.String(), float64(bf.Int64())/1e9)
}
EOF
go run /tmp/ethclient_smoke_test.go
rm /tmp/ethclient_smoke_test.go
```

Expected: prints a sensible base-fee value (typically 0.5–50 gwei). Confirms the wrapper works against a real public RPC. **Don't commit this file.**

- [ ] **Step 6: Commit**

```bash
git add internal/service/airdrop/ethclient/ go.mod go.sum
git commit -m "feat(airdrop): add EVM RPC client wrapper for the relayer"
```

---

## Task 6: Push branch and open PR

- [ ] **Step 1: Run the full test suite**

```bash
go test ./...
```

Expected: PASS. If anything outside `internal/service/airdrop/...` and `internal/api/internal_token_middleware_test.go` and `internal/config/airdrop_test.go` fails, you've regressed something — investigate before pushing.

- [ ] **Step 2: Run linter**

```bash
make lint
```

Expected: no errors. Fix any lints flagged in the new files.

- [ ] **Step 3: Push the branch**

```bash
git push -u origin feat/airdrop-shared-infra
```

- [ ] **Step 4: Open the PR**

```bash
gh pr create --title "feat(airdrop): shared infrastructure for the Vultiverse Airdrop pipeline" --body "$(cat <<'EOF'
## Summary

Lays down the cross-cutting foundations for the Vultiverse Airdrop campaign — config substruct, six DB migrations, internal-token middleware, KMS signer, EVM RPC client wrapper. No business logic; subsequent stage-specific PRs build on this.

Spec source of truth: \`docs/airdrop-specs/shared-concerns.md\`.

## What's added
- \`AirdropConfig\` substruct under \`Config\` covering all 19 airdrop env vars
- Six Goose migrations under \`internal/storage/postgres/migrations/2026042300000{1..6}_*.sql\` for the airdrop \`agent_*\` tables
- \`internal/api/internal_token_middleware.go\` — \`X-Internal-Token\` shared-secret check for /internal/* routes
- \`internal/service/airdrop/kms/signer.go\` — KMS-backed signer with DER → EVM (r, s, v) conversion
- \`internal/service/airdrop/ethclient/client.go\` — go-ethereum/ethclient wrapper exposing only what the relayer needs

## Spec coverage
Maps to the "Done when" section of \`shared-concerns.md\`:
- ✅ Three of the four shared Go packages (KMS signer, ETH client wrapper, internal-token middleware). Eligibility helper deferred to Stage 1's plan since it depends on tables consumed there.
- ✅ Six migrations apply cleanly on a fresh DB in order; rollback works.
- ✅ 19 env vars loadable via envconfig.
- ✅ Internal-token middleware: 401 on missing/wrong, OK on correct.
- ⏭ KMS LocalStack integration test — deferred to Stage 4 plan where LocalStack will be wired into the test setup. Pure-Go round-trip test in this PR proves the DER + recovery math.
- ✅ ETH client reads latest base fee against a real public RPC (manual smoke test).

## Test plan
- [ ] \`go test ./...\` passes
- [ ] \`make migrate-up && make migrate-down ...×6 && make migrate-up\` round-trips cleanly
- [ ] Manual ETH client smoke test against a public RPC reports a sensible base fee
- [ ] \`make lint\` clean

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Capture and report the PR URL.

---

## Self-review notes

This plan was reviewed against `shared-concerns.md` after writing. Items confirmed:
- All 19 env vars present in Task 1's struct + `.env.example` block.
- All six migrations present in Task 2.
- Internal-token middleware in Task 3, with the constant-time compare the spec implies.
- KMS signer in Task 4 — DER decoding + low-S normalisation + recovery byte. LocalStack integration test deferred (noted in PR body).
- EVM client in Task 5 — exact method set from `shared-concerns.md`.

Items intentionally NOT in this plan:
- **Eligibility helper** (`internal/service/airdrop/quests/eligibility.go`) — depends on Stage 2/3 tables AND is the first consumer in Stage 1's status endpoint. Lives in Stage 1's plan.
- **Inline Basic Auth check** for ops UI — single-use code, lives in Stage 4's plan with the rest of the ops handler.

Stage 1's plan can begin once this PR merges — it consumes `cfg.Airdrop.TransitionWindowStart/End` and the `agent_airdrop_registrations` + `agent_raffle_winners` tables, all delivered here.
