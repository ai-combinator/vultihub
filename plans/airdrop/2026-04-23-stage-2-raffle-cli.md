# Vultiverse Airdrop — Stage 2 (Raffle Draw CLI) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A new `airdrop-raffle` CLI binary with three subcommands (`draw`, `load-winners`, `verify-onchain`) that the operator runs by hand on Day 12. `draw` reads `agent_airdrop_registrations`, picks winners with a seeded Fisher–Yates shuffle, and writes `winners.csv` + `manifest.json`. The multisig signer (a human, not this CLI) takes `winners.csv` and submits batched `setWinners(...)` calls. `load-winners` then mirrors the CSV into `agent_raffle_winners`. `verify-onchain` reads each row of `winners.csv` and asserts `allowance(recipient)` on the contract matches.

**Architecture:** One Go binary at `cmd/airdrop-raffle/` with stdlib `flag`-based subcommand routing — same dependency-free pattern as `cmd/admin/main.go` and `cmd/scheduler/main.go`. Pure selection + serialization logic in `internal/service/airdrop/raffle/` so unit tests run without Postgres or RPC. New sqlc query file `internal/storage/postgres/sqlc/airdrop_raffle.sql` for the CLI's reads/writes against the tables shipped in shared-infra. The `verify-onchain` subcommand needs `allowance(address) -> uint256` on `AirdropClaim`, which differs from the ERC-20 `balanceOf` already in the wrapper — so we extend `internal/service/airdrop/ethclient/client.go` with one more method (`AirdropAllowance`) rather than calling `CallContract` directly from `cmd/`. That keeps all ABI-encoding concerns inside the wrapper, matching the existing `BalanceOf` pattern.

**Tech Stack:**
- Go 1.25 stdlib (`flag`, `crypto/rand`, `math/rand/v2`, `encoding/csv`, `encoding/json`)
- pgx/v5 + sqlc (existing)
- `internal/service/airdrop/ethclient` from shared-infra (extended with one method)
- Logrus for structured stdout JSON logs
- No new external dependencies

**Spec source of truth:** `docs/airdrop-specs/stage-2-raffle-draw.md`

**Working directory for all tasks:** `/Users/apotheosis/git/vultisig/agent-backend/`

**Branch:** `feat/airdrop-stage-2-raffle-cli`. Branch off `main` once `feat/airdrop-shared-infra` has merged. Commit after every task.

**Hard deadline:** Operator runs `airdrop-raffle draw` on Day 12 (boarding window closes at `TRANSITION_WINDOW_END`). The binary must be built and the runbook drafted by Day 11.

---

## Pre-flight (do once before Task 1)

Confirm the shared-infra PR has merged to `main`, the airdrop migrations applied locally, and the `ethclient` + Stage 1 sqlc files exist:

```bash
cd /Users/apotheosis/git/vultisig/agent-backend
git checkout main && git pull
psql "$DATABASE_DSN" -c "\d agent_airdrop_registrations"
psql "$DATABASE_DSN" -c "\d agent_raffle_winners"
ls internal/service/airdrop/ethclient/client.go
ls internal/storage/postgres/sqlc/airdrop_registrations.sql
```

All four checks must succeed. If `agent_raffle_winners` is missing, run `make migrate-up`. If `airdrop_registrations.sql` is missing, the Stage 1 PR has not merged yet — wait for it before continuing (this plan reuses its `AirdropRegistrationSource` enum binding and its `AgentAirdropRegistration` row type).

```bash
git checkout -b feat/airdrop-stage-2-raffle-cli
```

---

## Task 1: sqlc queries for raffle reads + winner inserts

**Why:** The CLI needs four queries: read every registration (for `draw`), check if winners exist (idempotency guard for `load-winners`), batched winner insert (`load-winners` body), and read every winner (for `verify-onchain`). All four belong in one file.

**Files:**
- Create: `internal/storage/postgres/sqlc/airdrop_raffle.sql`
- Generated: `internal/storage/postgres/queries/airdrop_raffle.sql.go` (do not hand-edit)

- [ ] **Step 1: Write the queries**

Create `internal/storage/postgres/sqlc/airdrop_raffle.sql`:

```sql
-- name: ListAirdropRegistrationsForDraw :many
-- Returns every registration in deterministic order (registered_at, public_key)
-- so the same seed yields byte-identical winners.csv across runs.
SELECT * FROM agent_airdrop_registrations
ORDER BY registered_at ASC, public_key ASC;

-- name: AirdropRaffleWinnerCount :one
-- Used by load-winners to refuse if any winners are already loaded.
SELECT COUNT(*)::bigint FROM agent_raffle_winners;

-- name: InsertRaffleWinner :exec
-- Single-row insert; load-winners issues many of these inside one tx.
INSERT INTO agent_raffle_winners (public_key, recipient, amount)
VALUES ($1, $2, $3);

-- name: ListRaffleWinners :many
-- Used by verify-onchain when re-reading from DB (not used by the CSV path,
-- but useful for ad-hoc operator checks).
SELECT * FROM agent_raffle_winners
ORDER BY public_key ASC;
```

Note: `amount` is `NUMERIC(78,0)` in the schema. sqlc will bind it to `pgtype.Numeric`; the repository wrapper (Task 3) converts `*big.Int` ↔ `pgtype.Numeric`.

- [ ] **Step 2: Run sqlc generation**

```bash
grep -n sqlc Makefile
```

If `make sqlc` exists, use it. Otherwise:

```bash
sqlc generate
```

Expected: `internal/storage/postgres/queries/airdrop_raffle.sql.go` appears.

If sqlc is not installed locally:

```bash
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
```

- [ ] **Step 3: Verify generated code compiles**

```bash
go build ./internal/storage/postgres/...
```

Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add internal/storage/postgres/sqlc/airdrop_raffle.sql \
         internal/storage/postgres/queries/airdrop_raffle.sql.go
git commit -m "feat(airdrop): add sqlc queries for raffle draw + load-winners"
```

---

## Task 2: Selection logic (Fisher–Yates with seeded PRNG)

**Why:** The randomness is the heart of the raffle. Splitting it into a pure function on a `[]Registration` slice makes it trivially unit-testable for determinism, undersubscription, oversubscription, and the zero-entries hard-fail. No DB, no I/O, no logger.

**Files:**
- Create: `internal/service/airdrop/raffle/selection.go`
- Create: `internal/service/airdrop/raffle/selection_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/service/airdrop/raffle/selection_test.go`:

```go
package raffle

import (
	"testing"
	"time"
)

func mkRegs(n int) []Registration {
	out := make([]Registration, n)
	base := time.Date(2026, 4, 25, 0, 0, 0, 0, time.UTC)
	for i := 0; i < n; i++ {
		out[i] = Registration{
			PublicKey:        fmtPK(i),
			Source:           "seed",
			RecipientAddress: fmtAddr(i),
			RegisteredAt:     base.Add(time.Duration(i) * time.Second),
		}
	}
	return out
}

func fmtPK(i int) string   { return "pk-" + itoa(i) }
func fmtAddr(i int) string { return "0xaddr" + itoa(i) }
func itoa(i int) string {
	if i == 0 {
		return "0"
	}
	out := []byte{}
	for i > 0 {
		out = append([]byte{byte('0' + i%10)}, out...)
		i /= 10
	}
	return string(out)
}

func TestDraw_ZeroEntries(t *testing.T) {
	_, err := Draw(nil, 700, 12345)
	if err != ErrNoRegistrations {
		t.Errorf("err = %v, want ErrNoRegistrations", err)
	}
}

func TestDraw_Undersubscribed(t *testing.T) {
	regs := mkRegs(50)
	res, err := Draw(regs, 700, 12345)
	if err != nil {
		t.Fatalf("Draw: %v", err)
	}
	if !res.Undersubscribed {
		t.Errorf("expected Undersubscribed = true")
	}
	if len(res.Winners) != 50 {
		t.Errorf("winner count = %d, want 50 (all entries win)", len(res.Winners))
	}
}

func TestDraw_Oversubscribed(t *testing.T) {
	regs := mkRegs(1000)
	res, err := Draw(regs, 700, 12345)
	if err != nil {
		t.Fatalf("Draw: %v", err)
	}
	if res.Undersubscribed {
		t.Errorf("Undersubscribed should be false")
	}
	if len(res.Winners) != 700 {
		t.Errorf("winner count = %d, want 700", len(res.Winners))
	}
	seen := map[string]bool{}
	for _, w := range res.Winners {
		if seen[w.PublicKey] {
			t.Errorf("duplicate winner %s", w.PublicKey)
		}
		seen[w.PublicKey] = true
	}
}

func TestDraw_DeterministicForSameSeed(t *testing.T) {
	regs := mkRegs(1000)
	a, _ := Draw(regs, 700, 0xDEADBEEF)
	b, _ := Draw(regs, 700, 0xDEADBEEF)
	if len(a.Winners) != len(b.Winners) {
		t.Fatalf("len mismatch: %d vs %d", len(a.Winners), len(b.Winners))
	}
	for i := range a.Winners {
		if a.Winners[i].PublicKey != b.Winners[i].PublicKey {
			t.Errorf("idx %d: %s vs %s", i, a.Winners[i].PublicKey, b.Winners[i].PublicKey)
		}
	}
}

func TestDraw_DifferentSeedDifferentOrder(t *testing.T) {
	regs := mkRegs(1000)
	a, _ := Draw(regs, 700, 1)
	b, _ := Draw(regs, 700, 2)
	// At least one of the first 10 winners should differ — collision probability
	// across 1000 entries is astronomically low for distinct seeds.
	differ := false
	for i := 0; i < 10; i++ {
		if a.Winners[i].PublicKey != b.Winners[i].PublicKey {
			differ = true
			break
		}
	}
	if !differ {
		t.Errorf("two seeds produced identical first-10 ordering; PRNG suspect")
	}
}

func TestDraw_RecipientPassthrough(t *testing.T) {
	regs := mkRegs(100)
	res, _ := Draw(regs, 50, 42)
	for _, w := range res.Winners {
		// Winner's RecipientAddress must match the source registration's exactly.
		var src *Registration
		for i := range regs {
			if regs[i].PublicKey == w.PublicKey {
				src = &regs[i]
				break
			}
		}
		if src == nil {
			t.Fatalf("winner %s not in source", w.PublicKey)
		}
		if w.RecipientAddress != src.RecipientAddress {
			t.Errorf("recipient mutated: winner=%s src=%s", w.RecipientAddress, src.RecipientAddress)
		}
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/service/airdrop/raffle/ -v
```

Expected: compile error — `Draw`, `Registration`, `ErrNoRegistrations` undefined.

- [ ] **Step 3: Implement selection.go**

Create `internal/service/airdrop/raffle/selection.go`:

```go
package raffle

import (
	"errors"
	"math/rand/v2"
	"time"
)

// Registration is a row from agent_airdrop_registrations as the CLI reads it.
// Owned by this package so selection has no Postgres-types coupling.
type Registration struct {
	PublicKey        string
	Source           string // "seed" | "vault_share"
	RecipientAddress string
	RegisteredAt     time.Time
}

// DrawResult is what Draw returns: the winners (a subset of input, in pick order),
// plus a flag indicating whether every entry won by virtue of undersubscription.
type DrawResult struct {
	Winners         []Registration
	Undersubscribed bool
}

// ErrNoRegistrations is returned by Draw when the input slice is empty.
// CLI maps this to exit code 2 per the spec.
var ErrNoRegistrations = errors.New("no registrations to draw from")

// Draw seeds a deterministic PRNG with `seed`, runs Fisher–Yates against a copy
// of `regs`, and returns the first `slotCount` entries (or all of them when
// undersubscribed). Pure function — no I/O, no clock, no logger. The same
// `regs` (in any order) and the same `seed` yield the same winners; callers
// that want byte-identical output should sort `regs` first (the sqlc query
// ListAirdropRegistrationsForDraw already does this).
func Draw(regs []Registration, slotCount int, seed uint64) (DrawResult, error) {
	if len(regs) == 0 {
		return DrawResult{}, ErrNoRegistrations
	}

	// Copy so we don't mutate the caller's slice.
	pool := make([]Registration, len(regs))
	copy(pool, regs)

	if len(pool) <= slotCount {
		// Undersubscribed: every entry wins. Order is the pre-shuffle order
		// (i.e. registered_at ASC) to make the artifact human-scannable.
		return DrawResult{Winners: pool, Undersubscribed: true}, nil
	}

	// math/rand/v2's PCG seeded with (seed, 0) is deterministic and well-
	// distributed. We use shuffle (which is itself a Fisher–Yates) and then
	// take the prefix.
	r := rand.New(rand.NewPCG(seed, 0))
	r.Shuffle(len(pool), func(i, j int) { pool[i], pool[j] = pool[j], pool[i] })

	return DrawResult{
		Winners:         pool[:slotCount],
		Undersubscribed: false,
	}, nil
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/raffle/ -v
```

Expected: PASS for all six subtests.

- [ ] **Step 5: Commit**

```bash
git add internal/service/airdrop/raffle/selection.go \
         internal/service/airdrop/raffle/selection_test.go
git commit -m "feat(airdrop): add seeded Fisher-Yates draw selection"
```

---

## Task 3: Output serialization (`winners.csv` + `manifest.json`)

**Why:** Two artifact formats, both spec-defined. Single function each, file-system side-effect goes through an `io.Writer` so tests assert on bytes without temp files. CSV writer uses stdlib `encoding/csv`; manifest uses stdlib `encoding/json` with indentation for human review.

**Files:**
- Create: `internal/service/airdrop/raffle/output.go`
- Create: `internal/service/airdrop/raffle/output_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/service/airdrop/raffle/output_test.go`:

```go
package raffle

import (
	"bytes"
	"encoding/json"
	"math/big"
	"strings"
	"testing"
	"time"
)

func TestWriteWinnersCSV_HeaderAndRows(t *testing.T) {
	winners := []Registration{
		{PublicKey: "pk-a", Source: "seed", RecipientAddress: "0xaaa",
			RegisteredAt: time.Date(2026, 4, 25, 12, 0, 0, 0, time.UTC)},
		{PublicKey: "pk-b", Source: "vault_share", RecipientAddress: "0xbbb",
			RegisteredAt: time.Date(2026, 4, 25, 13, 0, 0, 0, time.UTC)},
	}
	amount := new(big.Int).SetUint64(1500)

	var buf bytes.Buffer
	if err := WriteWinnersCSV(&buf, winners, amount); err != nil {
		t.Fatalf("WriteWinnersCSV: %v", err)
	}

	got := buf.String()
	wantHeader := "public_key,recipient,amount,registered_at,source"
	if !strings.HasPrefix(got, wantHeader) {
		t.Errorf("header mismatch:\ngot:  %q\nwant: %q...", got[:len(wantHeader)+1], wantHeader)
	}
	if !strings.Contains(got, "pk-a,0xaaa,1500,2026-04-25T12:00:00Z,seed") {
		t.Errorf("missing pk-a row; got:\n%s", got)
	}
	if !strings.Contains(got, "pk-b,0xbbb,1500,2026-04-25T13:00:00Z,vault_share") {
		t.Errorf("missing pk-b row; got:\n%s", got)
	}
}

func TestWriteWinnersCSV_IsByteIdenticalForSameInput(t *testing.T) {
	winners := []Registration{
		{PublicKey: "pk-a", Source: "seed", RecipientAddress: "0xaaa",
			RegisteredAt: time.Date(2026, 4, 25, 12, 0, 0, 0, time.UTC)},
	}
	amount := new(big.Int).SetUint64(1500)
	var a, b bytes.Buffer
	_ = WriteWinnersCSV(&a, winners, amount)
	_ = WriteWinnersCSV(&b, winners, amount)
	if !bytes.Equal(a.Bytes(), b.Bytes()) {
		t.Errorf("non-deterministic output:\nA: %q\nB: %q", a.String(), b.String())
	}
}

func TestWriteManifest_Shape(t *testing.T) {
	m := Manifest{
		Seed:              0xDEADBEEF,
		SlotCount:         700,
		VultAmount:        "1500000000000000000000",
		RegistrationCount: 1500,
		WinnerCount:       700,
		DrawTimestamp:     time.Date(2026, 5, 5, 0, 0, 0, 0, time.UTC),
		Undersubscribed:   false,
	}
	var buf bytes.Buffer
	if err := WriteManifest(&buf, m); err != nil {
		t.Fatalf("WriteManifest: %v", err)
	}
	var round Manifest
	if err := json.Unmarshal(buf.Bytes(), &round); err != nil {
		t.Fatalf("re-parse: %v", err)
	}
	if round.Seed != m.Seed || round.SlotCount != m.SlotCount ||
		round.VultAmount != m.VultAmount || round.RegistrationCount != m.RegistrationCount ||
		round.WinnerCount != m.WinnerCount || !round.DrawTimestamp.Equal(m.DrawTimestamp) ||
		round.Undersubscribed != m.Undersubscribed {
		t.Errorf("round-trip mismatch: got %+v want %+v", round, m)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/service/airdrop/raffle/ -run TestWrite -v
```

Expected: compile error — `WriteWinnersCSV`, `WriteManifest`, `Manifest` undefined.

- [ ] **Step 3: Implement output.go**

Create `internal/service/airdrop/raffle/output.go`:

```go
package raffle

import (
	"encoding/csv"
	"encoding/json"
	"fmt"
	"io"
	"math/big"
	"time"
)

// Manifest is the JSON shape written to manifest.json alongside winners.csv.
// Field names match the spec (stage-2-raffle-draw.md, "Subcommands → draw").
// VultAmount is a string so a uint256 fits without precision loss.
type Manifest struct {
	Seed              uint64    `json:"seed"`
	SlotCount         int       `json:"slot_count"`
	VultAmount        string    `json:"vult_amount"`
	RegistrationCount int       `json:"registration_count"`
	WinnerCount       int       `json:"winner_count"`
	DrawTimestamp     time.Time `json:"draw_timestamp"`
	Undersubscribed   bool      `json:"undersubscribed"`
}

// WinnersCSVHeader is the canonical header row. Exported so tests + the
// load-winners reader can refer to the same string.
var WinnersCSVHeader = []string{"public_key", "recipient", "amount", "registered_at", "source"}

// WriteWinnersCSV writes the spec-defined CSV: one header row, one row per
// winner with `amount` formatted as the decimal string representation of the
// per-winner VULT amount and `registered_at` formatted as RFC3339 UTC.
func WriteWinnersCSV(w io.Writer, winners []Registration, amount *big.Int) error {
	cw := csv.NewWriter(w)
	if err := cw.Write(WinnersCSVHeader); err != nil {
		return fmt.Errorf("write header: %w", err)
	}
	amountStr := amount.String()
	for _, win := range winners {
		row := []string{
			win.PublicKey,
			win.RecipientAddress,
			amountStr,
			win.RegisteredAt.UTC().Format(time.RFC3339),
			win.Source,
		}
		if err := cw.Write(row); err != nil {
			return fmt.Errorf("write row %s: %w", win.PublicKey, err)
		}
	}
	cw.Flush()
	return cw.Error()
}

// WriteManifest serializes the manifest as indented JSON for human review.
func WriteManifest(w io.Writer, m Manifest) error {
	enc := json.NewEncoder(w)
	enc.SetIndent("", "  ")
	return enc.Encode(m)
}

// CSVRow mirrors a single parsed row of winners.csv, used by load-winners
// and verify-onchain when reading the artifact back.
type CSVRow struct {
	PublicKey        string
	Recipient        string
	Amount           *big.Int
	RegisteredAt     time.Time
	Source           string
}

// ReadWinnersCSV parses an entire winners.csv into rows. Returns an error
// if the header is missing or malformed, or if any row's `amount` is not
// a valid decimal integer.
func ReadWinnersCSV(r io.Reader) ([]CSVRow, error) {
	cr := csv.NewReader(r)
	header, err := cr.Read()
	if err != nil {
		return nil, fmt.Errorf("read header: %w", err)
	}
	if len(header) != len(WinnersCSVHeader) {
		return nil, fmt.Errorf("header has %d cols, want %d", len(header), len(WinnersCSVHeader))
	}
	for i, want := range WinnersCSVHeader {
		if header[i] != want {
			return nil, fmt.Errorf("header[%d] = %q, want %q", i, header[i], want)
		}
	}
	var out []CSVRow
	for lineNum := 2; ; lineNum++ {
		rec, err := cr.Read()
		if err == io.EOF {
			break
		}
		if err != nil {
			return nil, fmt.Errorf("read line %d: %w", lineNum, err)
		}
		amount, ok := new(big.Int).SetString(rec[2], 10)
		if !ok {
			return nil, fmt.Errorf("line %d: invalid amount %q", lineNum, rec[2])
		}
		ts, err := time.Parse(time.RFC3339, rec[3])
		if err != nil {
			return nil, fmt.Errorf("line %d: invalid registered_at %q: %w", lineNum, rec[3], err)
		}
		out = append(out, CSVRow{
			PublicKey:    rec[0],
			Recipient:    rec[1],
			Amount:       amount,
			RegisteredAt: ts,
			Source:       rec[4],
		})
	}
	return out, nil
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/service/airdrop/raffle/ -v
```

Expected: PASS for all selection + output tests.

- [ ] **Step 5: Add a CSV round-trip test**

Append to `internal/service/airdrop/raffle/output_test.go`:

```go
func TestReadWinnersCSV_RoundTrip(t *testing.T) {
	winners := []Registration{
		{PublicKey: "pk-a", Source: "seed", RecipientAddress: "0xaaa",
			RegisteredAt: time.Date(2026, 4, 25, 12, 0, 0, 0, time.UTC)},
		{PublicKey: "pk-b", Source: "vault_share", RecipientAddress: "0xbbb",
			RegisteredAt: time.Date(2026, 4, 25, 13, 0, 0, 0, time.UTC)},
	}
	amount, _ := new(big.Int).SetString("1500000000000000000000", 10)

	var buf bytes.Buffer
	if err := WriteWinnersCSV(&buf, winners, amount); err != nil {
		t.Fatalf("write: %v", err)
	}
	rows, err := ReadWinnersCSV(&buf)
	if err != nil {
		t.Fatalf("read: %v", err)
	}
	if len(rows) != 2 {
		t.Fatalf("rows = %d, want 2", len(rows))
	}
	if rows[0].PublicKey != "pk-a" || rows[0].Recipient != "0xaaa" ||
		rows[0].Amount.Cmp(amount) != 0 || rows[0].Source != "seed" ||
		!rows[0].RegisteredAt.Equal(winners[0].RegisteredAt) {
		t.Errorf("row 0 mismatch: %+v", rows[0])
	}
}

func TestReadWinnersCSV_RejectsBadHeader(t *testing.T) {
	bad := "wrong,header,row\n"
	_, err := ReadWinnersCSV(bytes.NewBufferString(bad))
	if err == nil {
		t.Errorf("expected error for bad header, got nil")
	}
}
```

- [ ] **Step 6: Run all raffle tests**

```bash
go test ./internal/service/airdrop/raffle/ -v
```

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/service/airdrop/raffle/output.go \
         internal/service/airdrop/raffle/output_test.go
git commit -m "feat(airdrop): add winners.csv + manifest.json serialization"
```

---

## Task 4: `draw` subcommand

**Why:** Wires Task 1 (DB read), Task 2 (selection), and Task 3 (CSV + manifest) together behind a `flag.FlagSet` matching the spec's flag table. Logger emits one structured JSON line per major step. Process exits non-zero with the spec's exit codes on documented failure modes.

**Files:**
- Create: `cmd/airdrop-raffle/draw.go`
- Create: `cmd/airdrop-raffle/draw_test.go`

- [ ] **Step 1: Write the failing test**

Create `cmd/airdrop-raffle/draw_test.go`:

```go
package main

import (
	"bytes"
	"context"
	"errors"
	"math/big"
	"os"
	"path/filepath"
	"testing"
	"time"

	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/service/airdrop/raffle"
)

// fakeRegLister implements registrationLister for tests without a DB.
type fakeRegLister struct {
	regs []raffle.Registration
	err  error
}

func (f *fakeRegLister) ListForDraw(ctx context.Context) ([]raffle.Registration, error) {
	return f.regs, f.err
}

func nopLogger() *logrus.Logger {
	l := logrus.New()
	l.SetOutput(&bytes.Buffer{})
	l.SetLevel(logrus.PanicLevel)
	return l
}

func TestRunDraw_WritesArtifacts(t *testing.T) {
	dir := t.TempDir()
	regs := []raffle.Registration{
		{PublicKey: "pk1", Source: "seed", RecipientAddress: "0xaaa",
			RegisteredAt: time.Date(2026, 4, 25, 12, 0, 0, 0, time.UTC)},
		{PublicKey: "pk2", Source: "seed", RecipientAddress: "0xbbb",
			RegisteredAt: time.Date(2026, 4, 25, 13, 0, 0, 0, time.UTC)},
	}
	amount, _ := new(big.Int).SetString("1500000000000000000000", 10)

	code := runDraw(context.Background(), drawOpts{
		seed:       42,
		slotCount:  1,
		vultAmount: amount,
		outputDir:  dir,
		lister:     &fakeRegLister{regs: regs},
		logger:     nopLogger(),
		now:        func() time.Time { return time.Date(2026, 5, 5, 0, 0, 0, 0, time.UTC) },
	})
	if code != 0 {
		t.Fatalf("exit code = %d, want 0", code)
	}

	csvBytes, err := os.ReadFile(filepath.Join(dir, "winners.csv"))
	if err != nil {
		t.Fatalf("winners.csv missing: %v", err)
	}
	if !bytes.Contains(csvBytes, []byte("public_key,recipient,amount,registered_at,source")) {
		t.Errorf("winners.csv missing header; got:\n%s", csvBytes)
	}

	manBytes, err := os.ReadFile(filepath.Join(dir, "manifest.json"))
	if err != nil {
		t.Fatalf("manifest.json missing: %v", err)
	}
	if !bytes.Contains(manBytes, []byte(`"seed": 42`)) {
		t.Errorf("manifest missing seed; got:\n%s", manBytes)
	}
	if !bytes.Contains(manBytes, []byte(`"winner_count": 1`)) {
		t.Errorf("manifest missing winner_count; got:\n%s", manBytes)
	}
}

func TestRunDraw_ZeroEntriesExitCode2(t *testing.T) {
	dir := t.TempDir()
	code := runDraw(context.Background(), drawOpts{
		seed: 1, slotCount: 700, vultAmount: big.NewInt(1500),
		outputDir: dir,
		lister:    &fakeRegLister{regs: nil},
		logger:    nopLogger(),
		now:       time.Now,
	})
	if code != 2 {
		t.Errorf("exit code = %d, want 2 (zero registrations)", code)
	}
}

func TestRunDraw_DBErrorExitCode1(t *testing.T) {
	dir := t.TempDir()
	code := runDraw(context.Background(), drawOpts{
		seed: 1, slotCount: 700, vultAmount: big.NewInt(1500),
		outputDir: dir,
		lister:    &fakeRegLister{err: errors.New("boom")},
		logger:    nopLogger(),
		now:       time.Now,
	})
	if code != 1 {
		t.Errorf("exit code = %d, want 1 (input/db error)", code)
	}
}

func TestRunDraw_OutputDirCreatedIfMissing(t *testing.T) {
	parent := t.TempDir()
	missing := filepath.Join(parent, "new-subdir")
	regs := []raffle.Registration{
		{PublicKey: "pk1", Source: "seed", RecipientAddress: "0xaaa",
			RegisteredAt: time.Date(2026, 4, 25, 12, 0, 0, 0, time.UTC)},
	}
	code := runDraw(context.Background(), drawOpts{
		seed: 1, slotCount: 1, vultAmount: big.NewInt(1),
		outputDir: missing,
		lister:    &fakeRegLister{regs: regs},
		logger:    nopLogger(),
		now:       time.Now,
	})
	if code != 0 {
		t.Errorf("exit code = %d, want 0", code)
	}
	if _, err := os.Stat(filepath.Join(missing, "winners.csv")); err != nil {
		t.Errorf("winners.csv not created: %v", err)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./cmd/airdrop-raffle/ -run TestRunDraw -v
```

Expected: compile error — `runDraw`, `drawOpts`, `registrationLister` undefined.

- [ ] **Step 3: Implement draw.go**

Create `cmd/airdrop-raffle/draw.go`:

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/binary"
	"errors"
	"fmt"
	"math/big"
	"os"
	"path/filepath"
	"time"

	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/service/airdrop/raffle"
)

// registrationLister is the slice of the registrations repo this subcommand
// needs. Defined as an interface so unit tests can pass a fake.
type registrationLister interface {
	ListForDraw(ctx context.Context) ([]raffle.Registration, error)
}

// drawOpts is the resolved set of inputs runDraw operates on. Constructed by
// drawMain (the flag-parsing entrypoint) or directly by tests.
type drawOpts struct {
	seed       uint64
	slotCount  int
	vultAmount *big.Int
	outputDir  string
	lister     registrationLister
	logger     *logrus.Logger
	now        func() time.Time
}

// runDraw executes the draw subcommand with the given opts. Returns the spec's
// exit code: 0 success, 1 input/db error, 2 zero registrations, 4 output write error.
func runDraw(ctx context.Context, opts drawOpts) int {
	log := opts.logger.WithFields(logrus.Fields{
		"subcommand": "draw",
		"seed":       opts.seed,
		"slot_count": opts.slotCount,
	})

	regs, err := opts.lister.ListForDraw(ctx)
	if err != nil {
		log.WithError(err).Error("list registrations failed")
		return 1
	}
	log.WithField("registration_count", len(regs)).Info("loaded registrations")

	res, err := raffle.Draw(regs, opts.slotCount, opts.seed)
	if err != nil {
		if errors.Is(err, raffle.ErrNoRegistrations) {
			log.Error("zero registrations: refusing to draw")
			fmt.Fprintln(os.Stderr, "ERROR: no registrations in agent_airdrop_registrations; nothing to draw.")
			return 2
		}
		log.WithError(err).Error("draw failed")
		return 1
	}

	if res.Undersubscribed {
		shortfall := opts.slotCount - len(res.Winners)
		fmt.Fprintln(os.Stderr, "============================================================")
		fmt.Fprintf(os.Stderr, "WARNING: undersubscribed (%d entries vs %d slots).\n", len(res.Winners), opts.slotCount)
		fmt.Fprintln(os.Stderr, "All entries will win.")
		fmt.Fprintf(os.Stderr, "Prize pool will be undersubscribed by %d × amount VULT.\n", shortfall)
		fmt.Fprintln(os.Stderr, "============================================================")
	}

	if err := os.MkdirAll(opts.outputDir, 0o755); err != nil {
		log.WithError(err).Error("mkdir output dir failed")
		return 4
	}

	csvPath := filepath.Join(opts.outputDir, "winners.csv")
	csvFile, err := os.Create(csvPath)
	if err != nil {
		log.WithError(err).Error("create winners.csv failed")
		return 4
	}
	if err := raffle.WriteWinnersCSV(csvFile, res.Winners, opts.vultAmount); err != nil {
		csvFile.Close()
		log.WithError(err).Error("write winners.csv failed")
		return 4
	}
	if err := csvFile.Close(); err != nil {
		log.WithError(err).Error("close winners.csv failed")
		return 4
	}

	manifestPath := filepath.Join(opts.outputDir, "manifest.json")
	manFile, err := os.Create(manifestPath)
	if err != nil {
		log.WithError(err).Error("create manifest.json failed")
		return 4
	}
	manifest := raffle.Manifest{
		Seed:              opts.seed,
		SlotCount:         opts.slotCount,
		VultAmount:        opts.vultAmount.String(),
		RegistrationCount: len(regs),
		WinnerCount:       len(res.Winners),
		DrawTimestamp:     opts.now().UTC(),
		Undersubscribed:   res.Undersubscribed,
	}
	if err := raffle.WriteManifest(manFile, manifest); err != nil {
		manFile.Close()
		log.WithError(err).Error("write manifest.json failed")
		return 4
	}
	if err := manFile.Close(); err != nil {
		log.WithError(err).Error("close manifest.json failed")
		return 4
	}

	log.WithFields(logrus.Fields{
		"winner_count":     len(res.Winners),
		"undersubscribed":  res.Undersubscribed,
		"output_dir":       opts.outputDir,
		"winners_csv":      csvPath,
		"manifest_json":    manifestPath,
	}).Info("draw complete")

	fmt.Printf("Draw complete:\n")
	fmt.Printf("  seed:            %d\n", opts.seed)
	fmt.Printf("  registrations:   %d\n", len(regs))
	fmt.Printf("  winners:         %d\n", len(res.Winners))
	fmt.Printf("  undersubscribed: %v\n", res.Undersubscribed)
	fmt.Printf("  artifacts:       %s\n", opts.outputDir)
	return 0
}

// freshSeed mints a 64-bit seed from crypto/rand. Used when the operator did
// not pass --seed explicitly. Logged + printed so the operator can record it
// for later reproduction.
func freshSeed() (uint64, error) {
	var b [8]byte
	if _, err := rand.Read(b[:]); err != nil {
		return 0, fmt.Errorf("crypto/rand: %w", err)
	}
	return binary.BigEndian.Uint64(b[:]), nil
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./cmd/airdrop-raffle/ -run TestRunDraw -v
```

Expected: PASS for all four subtests.

- [ ] **Step 5: Commit**

```bash
git add cmd/airdrop-raffle/draw.go cmd/airdrop-raffle/draw_test.go
git commit -m "feat(airdrop): add draw subcommand with CSV + manifest output"
```

---

## Task 5: `load-winners` subcommand

**Why:** Reads `winners.csv` from a draw artifact directory and inserts every row into `agent_raffle_winners` in a single transaction. Refuses if the table already has any rows (operator must explicitly `TRUNCATE` to re-load).

**Files:**
- Create: `cmd/airdrop-raffle/load_winners.go`
- Create: `cmd/airdrop-raffle/load_winners_test.go`

- [ ] **Step 1: Write the failing test**

Create `cmd/airdrop-raffle/load_winners_test.go`:

```go
package main

import (
	"bytes"
	"context"
	"errors"
	"math/big"
	"os"
	"path/filepath"
	"testing"
	"time"

	"github.com/vultisig/agent-backend/internal/service/airdrop/raffle"
)

// fakeWinnersStore implements winnersStore for tests.
type fakeWinnersStore struct {
	existing  int64
	inserted  []raffle.CSVRow
	insertErr error
	countErr  error
}

func (f *fakeWinnersStore) Count(ctx context.Context) (int64, error) {
	return f.existing, f.countErr
}

func (f *fakeWinnersStore) InsertAll(ctx context.Context, rows []raffle.CSVRow) error {
	if f.insertErr != nil {
		return f.insertErr
	}
	f.inserted = append(f.inserted, rows...)
	return nil
}

func writeFixtureCSV(t *testing.T, dir string) {
	t.Helper()
	winners := []raffle.Registration{
		{PublicKey: "pk-a", Source: "seed", RecipientAddress: "0xaaa",
			RegisteredAt: time.Date(2026, 4, 25, 12, 0, 0, 0, time.UTC)},
		{PublicKey: "pk-b", Source: "vault_share", RecipientAddress: "0xbbb",
			RegisteredAt: time.Date(2026, 4, 25, 13, 0, 0, 0, time.UTC)},
	}
	amount, _ := new(big.Int).SetString("1500", 10)
	var buf bytes.Buffer
	if err := raffle.WriteWinnersCSV(&buf, winners, amount); err != nil {
		t.Fatalf("write fixture: %v", err)
	}
	if err := os.WriteFile(filepath.Join(dir, "winners.csv"), buf.Bytes(), 0o644); err != nil {
		t.Fatalf("write fixture file: %v", err)
	}
}

func TestRunLoadWinners_Success(t *testing.T) {
	dir := t.TempDir()
	writeFixtureCSV(t, dir)
	store := &fakeWinnersStore{}
	code := runLoadWinners(context.Background(), loadWinnersOpts{
		outputDir: dir,
		store:     store,
		logger:    nopLogger(),
	})
	if code != 0 {
		t.Fatalf("exit = %d, want 0", code)
	}
	if len(store.inserted) != 2 {
		t.Errorf("inserted = %d rows, want 2", len(store.inserted))
	}
}

func TestRunLoadWinners_RefusesIfAlreadyLoaded(t *testing.T) {
	dir := t.TempDir()
	writeFixtureCSV(t, dir)
	store := &fakeWinnersStore{existing: 700}
	code := runLoadWinners(context.Background(), loadWinnersOpts{
		outputDir: dir,
		store:     store,
		logger:    nopLogger(),
	})
	if code == 0 {
		t.Errorf("expected non-zero exit when table already loaded")
	}
	if len(store.inserted) != 0 {
		t.Errorf("inserted should be empty; got %d rows", len(store.inserted))
	}
}

func TestRunLoadWinners_MissingCSV(t *testing.T) {
	dir := t.TempDir()
	store := &fakeWinnersStore{}
	code := runLoadWinners(context.Background(), loadWinnersOpts{
		outputDir: dir,
		store:     store,
		logger:    nopLogger(),
	})
	if code == 0 {
		t.Errorf("expected non-zero exit when winners.csv missing")
	}
}

func TestRunLoadWinners_InsertErrorPropagates(t *testing.T) {
	dir := t.TempDir()
	writeFixtureCSV(t, dir)
	store := &fakeWinnersStore{insertErr: errors.New("constraint violation")}
	code := runLoadWinners(context.Background(), loadWinnersOpts{
		outputDir: dir,
		store:     store,
		logger:    nopLogger(),
	})
	if code == 0 {
		t.Errorf("expected non-zero exit when insert fails")
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./cmd/airdrop-raffle/ -run TestRunLoadWinners -v
```

Expected: compile error — `runLoadWinners`, `loadWinnersOpts`, `winnersStore` undefined.

- [ ] **Step 3: Implement load_winners.go**

Create `cmd/airdrop-raffle/load_winners.go`:

```go
package main

import (
	"context"
	"fmt"
	"os"
	"path/filepath"

	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/service/airdrop/raffle"
)

// winnersStore is the slice of the raffle winners repo load-winners needs.
// Implemented for production by RaffleWinnersRepository (Task 7).
type winnersStore interface {
	Count(ctx context.Context) (int64, error)
	InsertAll(ctx context.Context, rows []raffle.CSVRow) error
}

type loadWinnersOpts struct {
	outputDir string
	store     winnersStore
	logger    *logrus.Logger
}

// runLoadWinners reads winners.csv from outputDir, refuses if the table is
// non-empty, and inserts every row inside a single transaction (the store's
// InsertAll is responsible for the BEGIN/COMMIT). Returns 0 on success;
// non-zero on any error so the caller's shell sees a failure.
func runLoadWinners(ctx context.Context, opts loadWinnersOpts) int {
	log := opts.logger.WithField("subcommand", "load-winners")

	csvPath := filepath.Join(opts.outputDir, "winners.csv")
	f, err := os.Open(csvPath)
	if err != nil {
		log.WithError(err).Error("open winners.csv failed")
		fmt.Fprintf(os.Stderr, "ERROR: cannot open %s: %v\n", csvPath, err)
		return 1
	}
	defer f.Close()

	rows, err := raffle.ReadWinnersCSV(f)
	if err != nil {
		log.WithError(err).Error("parse winners.csv failed")
		fmt.Fprintf(os.Stderr, "ERROR: parse %s: %v\n", csvPath, err)
		return 1
	}
	log.WithField("row_count", len(rows)).Info("parsed winners.csv")

	existing, err := opts.store.Count(ctx)
	if err != nil {
		log.WithError(err).Error("count agent_raffle_winners failed")
		return 1
	}
	if existing > 0 {
		msg := fmt.Sprintf("agent_raffle_winners already has %d rows; refusing to load. "+
			"Run `psql ... -c 'TRUNCATE agent_raffle_winners'` to re-load.", existing)
		log.Error(msg)
		fmt.Fprintln(os.Stderr, "ERROR: "+msg)
		return 1
	}

	if err := opts.store.InsertAll(ctx, rows); err != nil {
		log.WithError(err).Error("insert winners failed")
		fmt.Fprintf(os.Stderr, "ERROR: insert: %v\n", err)
		return 1
	}

	log.WithField("inserted", len(rows)).Info("load-winners complete")
	fmt.Printf("Loaded %d winners into agent_raffle_winners.\n", len(rows))
	return 0
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./cmd/airdrop-raffle/ -run TestRunLoadWinners -v
```

Expected: PASS for all four subtests.

- [ ] **Step 5: Commit**

```bash
git add cmd/airdrop-raffle/load_winners.go cmd/airdrop-raffle/load_winners_test.go
git commit -m "feat(airdrop): add load-winners subcommand"
```

---

## Task 6: `verify-onchain` subcommand (and `AirdropAllowance` on the ETH client)

**Why:** Reads each row of `winners.csv`, calls `allowance(recipient)` on the deployed `AirdropClaim` contract, asserts the value matches the CSV's `amount`. Catches multisig batch errors before users start claiming. The ethclient wrapper from shared-infra has `BalanceOf` (ERC-20 `balanceOf(address)`) but `AirdropClaim.allowance(address)` is a different selector — extending the wrapper with one airdrop-specific method keeps all ABI encoding inside the wrapper rather than scattering raw `CallContract` invocations through `cmd/`.

**Files:**
- Modify: `internal/service/airdrop/ethclient/client.go` (add `AirdropAllowance` method to interface + impl)
- Modify: `internal/service/airdrop/ethclient/client_test.go` (cover the new method's API surface)
- Create: `cmd/airdrop-raffle/verify_onchain.go`
- Create: `cmd/airdrop-raffle/verify_onchain_test.go`

- [ ] **Step 1: Extend the ethclient wrapper**

Open `internal/service/airdrop/ethclient/client.go`. Add `AirdropAllowance` to the `Client` interface alongside the existing methods:

```go
type Client interface {
	// ... existing methods ...

	// AirdropAllowance reads `allowance(address)` on AirdropClaim — the per-recipient
	// VULT amount the contract owes. Distinct from BalanceOf (ERC-20) because
	// AirdropClaim's allowance is a single-arg storage getter, not the ERC-20
	// two-arg allowance(owner, spender).
	AirdropAllowance(ctx context.Context, contract, recipient common.Address) (*big.Int, error)
}
```

Add the selector and implementation alongside the existing `erc20BalanceOfSelector` block:

```go
// airdropAllowanceSelector is keccak256("allowance(address)")[:4] — the AirdropClaim
// public mapping getter. Distinct from ERC-20 allowance(address,address).
var airdropAllowanceSelector = []byte{0xbb, 0xa0, 0xb9, 0x73}

func (c *client) AirdropAllowance(ctx context.Context, contract, recipient common.Address) (*big.Int, error) {
	calldata := make([]byte, 0, 36)
	calldata = append(calldata, airdropAllowanceSelector...)
	// pad address to 32 bytes
	calldata = append(calldata, make([]byte, 12)...)
	calldata = append(calldata, recipient.Bytes()...)

	out, err := c.rpc.CallContract(ctx, ethereum.CallMsg{
		To:   &contract,
		Data: calldata,
	}, nil)
	if err != nil {
		return nil, fmt.Errorf("eth_call allowance: %w", err)
	}
	if len(out) != 32 {
		return nil, fmt.Errorf("expected 32-byte response, got %d", len(out))
	}
	return new(big.Int).SetBytes(out), nil
}
```

The selector hex value `0xbba0b973` is `keccak256("allowance(address)")[:4]`. Confirm it after the implementation by running the test in Step 4 — if the bytes are wrong, `AirdropAllowance` will return zero against a deployed contract and the integration test in Task 8 will catch it.

- [ ] **Step 2: Update the ethclient API-surface test**

Open `internal/service/airdrop/ethclient/client_test.go` and add `AirdropAllowance` to the unreachable interface-pin block:

```go
// Inside the existing TestClientInterface, add to the `if false` block:
_, _ = iface.AirdropAllowance(ctx, common.Address{}, common.Address{})
```

- [ ] **Step 3: Add a selector regression test**

Append to `internal/service/airdrop/ethclient/client_test.go`:

```go
import "github.com/ethereum/go-ethereum/crypto"

func TestAirdropAllowanceSelector(t *testing.T) {
	got := crypto.Keccak256([]byte("allowance(address)"))[:4]
	if !bytes.Equal(got, airdropAllowanceSelector) {
		t.Errorf("airdropAllowanceSelector mismatch: got %x want %x", airdropAllowanceSelector, got)
	}
}
```

(Add `"bytes"` to imports if not already present.)

- [ ] **Step 4: Run ethclient tests**

```bash
go test ./internal/service/airdrop/ethclient/ -v
```

Expected: PASS for `TestClientInterface`, `TestNewClientRejectsBadURL`, `TestAirdropAllowanceSelector`.

- [ ] **Step 5: Write the failing verify-onchain test**

Create `cmd/airdrop-raffle/verify_onchain_test.go`:

```go
package main

import (
	"context"
	"errors"
	"math/big"
	"os"
	"path/filepath"
	"testing"

	"github.com/ethereum/go-ethereum/common"
)

// fakeAllowanceReader implements allowanceReader for tests without a real chain.
type fakeAllowanceReader struct {
	values map[common.Address]*big.Int
	err    error
}

func (f *fakeAllowanceReader) AirdropAllowance(ctx context.Context, contract, recipient common.Address) (*big.Int, error) {
	if f.err != nil {
		return nil, f.err
	}
	if v, ok := f.values[recipient]; ok {
		return v, nil
	}
	return new(big.Int), nil
}

func writeVerifyFixture(t *testing.T, dir string) (recipientA, recipientB common.Address) {
	t.Helper()
	recipientA = common.HexToAddress("0x000000000000000000000000000000000000000a")
	recipientB = common.HexToAddress("0x000000000000000000000000000000000000000b")
	csv := "public_key,recipient,amount,registered_at,source\n" +
		"pk-a," + recipientA.Hex() + ",1500,2026-04-25T12:00:00Z,seed\n" +
		"pk-b," + recipientB.Hex() + ",1500,2026-04-25T13:00:00Z,vault_share\n"
	if err := os.WriteFile(filepath.Join(dir, "winners.csv"), []byte(csv), 0o644); err != nil {
		t.Fatalf("write fixture: %v", err)
	}
	return
}

func TestRunVerifyOnchain_AllMatch(t *testing.T) {
	dir := t.TempDir()
	a, b := writeVerifyFixture(t, dir)
	reader := &fakeAllowanceReader{values: map[common.Address]*big.Int{
		a: big.NewInt(1500),
		b: big.NewInt(1500),
	}}
	code := runVerifyOnchain(context.Background(), verifyOpts{
		outputDir: dir,
		contract:  common.HexToAddress("0x1"),
		reader:    reader,
		logger:    nopLogger(),
	})
	if code != 0 {
		t.Errorf("exit = %d, want 0 (all match)", code)
	}
}

func TestRunVerifyOnchain_AmountMismatch(t *testing.T) {
	dir := t.TempDir()
	a, b := writeVerifyFixture(t, dir)
	reader := &fakeAllowanceReader{values: map[common.Address]*big.Int{
		a: big.NewInt(1500),
		b: big.NewInt(999), // wrong
	}}
	code := runVerifyOnchain(context.Background(), verifyOpts{
		outputDir: dir,
		contract:  common.HexToAddress("0x1"),
		reader:    reader,
		logger:    nopLogger(),
	})
	if code == 0 {
		t.Errorf("expected non-zero exit on mismatch, got 0")
	}
}

func TestRunVerifyOnchain_MissingAllowance(t *testing.T) {
	dir := t.TempDir()
	a, _ := writeVerifyFixture(t, dir)
	reader := &fakeAllowanceReader{values: map[common.Address]*big.Int{
		a: big.NewInt(1500),
		// recipient B intentionally absent → returns 0
	}}
	code := runVerifyOnchain(context.Background(), verifyOpts{
		outputDir: dir,
		contract:  common.HexToAddress("0x1"),
		reader:    reader,
		logger:    nopLogger(),
	})
	if code == 0 {
		t.Errorf("expected non-zero exit on missing allowance, got 0")
	}
}

func TestRunVerifyOnchain_RPCErrorPropagates(t *testing.T) {
	dir := t.TempDir()
	writeVerifyFixture(t, dir)
	reader := &fakeAllowanceReader{err: errors.New("rpc down")}
	code := runVerifyOnchain(context.Background(), verifyOpts{
		outputDir: dir,
		contract:  common.HexToAddress("0x1"),
		reader:    reader,
		logger:    nopLogger(),
	})
	if code == 0 {
		t.Errorf("expected non-zero exit on RPC error, got 0")
	}
}
```

- [ ] **Step 6: Run test to verify it fails**

```bash
go test ./cmd/airdrop-raffle/ -run TestRunVerifyOnchain -v
```

Expected: compile error — `runVerifyOnchain`, `verifyOpts`, `allowanceReader` undefined.

- [ ] **Step 7: Implement verify_onchain.go**

Create `cmd/airdrop-raffle/verify_onchain.go`:

```go
package main

import (
	"context"
	"fmt"
	"os"
	"path/filepath"

	"github.com/ethereum/go-ethereum/common"
	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/service/airdrop/raffle"
)

// allowanceReader is the slice of the ethclient.Client this subcommand needs.
// Defined here so the test can pass a fake without a live RPC.
type allowanceReader interface {
	AirdropAllowance(ctx context.Context, contract, recipient common.Address) (*big.Int, error)
}

type verifyOpts struct {
	outputDir string
	contract  common.Address
	reader    allowanceReader
	logger    *logrus.Logger
}

// runVerifyOnchain reads winners.csv, queries allowance(recipient) for each row,
// and asserts the on-chain value matches the CSV's amount. Returns 0 only when
// every row matches; non-zero on any mismatch, missing allowance, or RPC error.
func runVerifyOnchain(ctx context.Context, opts verifyOpts) int {
	log := opts.logger.WithFields(logrus.Fields{
		"subcommand": "verify-onchain",
		"contract":   opts.contract.Hex(),
	})

	csvPath := filepath.Join(opts.outputDir, "winners.csv")
	f, err := os.Open(csvPath)
	if err != nil {
		log.WithError(err).Error("open winners.csv failed")
		fmt.Fprintf(os.Stderr, "ERROR: cannot open %s: %v\n", csvPath, err)
		return 1
	}
	defer f.Close()

	rows, err := raffle.ReadWinnersCSV(f)
	if err != nil {
		log.WithError(err).Error("parse winners.csv failed")
		return 1
	}

	var matched, mismatched, errored int
	for _, row := range rows {
		recipient := common.HexToAddress(row.Recipient)
		got, err := opts.reader.AirdropAllowance(ctx, opts.contract, recipient)
		if err != nil {
			errored++
			fmt.Printf("ERROR  %s  rpc: %v\n", row.Recipient, err)
			log.WithError(err).WithField("recipient", row.Recipient).Error("allowance call failed")
			continue
		}
		if got.Cmp(row.Amount) == 0 {
			matched++
			fmt.Printf("OK     %s  %s\n", row.Recipient, got.String())
		} else {
			mismatched++
			fmt.Printf("FAIL   %s  on-chain=%s csv=%s\n", row.Recipient, got.String(), row.Amount.String())
			log.WithFields(logrus.Fields{
				"recipient":   row.Recipient,
				"on_chain":    got.String(),
				"csv_amount":  row.Amount.String(),
			}).Warn("allowance mismatch")
		}
	}

	fmt.Printf("\nSummary: %d matched, %d mismatched, %d errored, %d total.\n",
		matched, mismatched, errored, len(rows))
	log.WithFields(logrus.Fields{
		"matched":    matched,
		"mismatched": mismatched,
		"errored":    errored,
		"total":      len(rows),
	}).Info("verify-onchain complete")

	if mismatched == 0 && errored == 0 {
		return 0
	}
	return 1
}
```

(Add `"math/big"` to the imports — go's import organizer should handle it.)

- [ ] **Step 8: Run tests to verify they pass**

```bash
go test ./cmd/airdrop-raffle/ -run TestRunVerifyOnchain -v
```

Expected: PASS for all four subtests.

- [ ] **Step 9: Commit**

```bash
git add internal/service/airdrop/ethclient/client.go \
         internal/service/airdrop/ethclient/client_test.go \
         cmd/airdrop-raffle/verify_onchain.go \
         cmd/airdrop-raffle/verify_onchain_test.go
git commit -m "feat(airdrop): add verify-onchain subcommand + AirdropAllowance method"
```

---

## Task 7: `RaffleWinnersRepository` (Postgres adapter for the store interface)

**Why:** The CLI's three subcommands talk to interfaces (`registrationLister`, `winnersStore`); production needs a real Postgres implementation that wraps the sqlc queries from Task 1 and converts between `*big.Int` and `pgtype.Numeric`. `ListForDraw` lives on a separate adapter that wraps the existing Stage 1 `RegistrationRepository` row type — we add one new method there since the `agent_airdrop_registrations` table is already owned by the Stage 1 repo.

**Files:**
- Create: `internal/storage/postgres/airdrop_raffle_winners.go`
- Create: `internal/storage/postgres/airdrop_raffle_winners_test.go`
- Modify: `internal/storage/postgres/airdrop_registration.go` (add `ListForDraw` method)

- [ ] **Step 1: Add `ListForDraw` to the registration repository**

Open `internal/storage/postgres/airdrop_registration.go`. Add a new method on `RegistrationRepository`:

```go
import "github.com/vultisig/agent-backend/internal/service/airdrop/raffle"

// ListForDraw returns every registration in deterministic order
// (registered_at ASC, public_key ASC) — input to the raffle draw CLI.
func (r *RegistrationRepository) ListForDraw(ctx context.Context) ([]raffle.Registration, error) {
	rows, err := r.q.ListAirdropRegistrationsForDraw(ctx)
	if err != nil {
		return nil, fmt.Errorf("list registrations for draw: %w", err)
	}
	out := make([]raffle.Registration, 0, len(rows))
	for _, row := range rows {
		out = append(out, raffle.Registration{
			PublicKey:        row.PublicKey,
			Source:           string(row.Source),
			RecipientAddress: row.RecipientAddress,
			RegisteredAt:     pgtimestamptzToTime(row.RegisteredAt),
		})
	}
	return out, nil
}
```

(`pgtimestamptzToTime` is already used by the existing `Get`/`Insert` methods in the same file.)

- [ ] **Step 2: Write the failing test for the winners repository**

Create `internal/storage/postgres/airdrop_raffle_winners_test.go`:

```go
package postgres_test

import (
	"context"
	"math/big"
	"os"
	"testing"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/service/airdrop/raffle"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
)

func newRaffleTestPool(t *testing.T) *pgxpool.Pool {
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
	_, err = pool.Exec(context.Background(), `TRUNCATE agent_raffle_winners`)
	if err != nil {
		t.Fatalf("truncate: %v", err)
	}
	return pool
}

func TestRaffleWinnersRepo_CountAndInsertAll(t *testing.T) {
	pool := newRaffleTestPool(t)
	repo := postgres.NewRaffleWinnersRepository(pool)
	ctx := context.Background()

	count, err := repo.Count(ctx)
	if err != nil {
		t.Fatalf("Count: %v", err)
	}
	if count != 0 {
		t.Errorf("initial count = %d, want 0", count)
	}

	rows := []raffle.CSVRow{
		{PublicKey: "pk-1", Recipient: "0x0000000000000000000000000000000000000001",
			Amount: big.NewInt(1500), RegisteredAt: time.Now(), Source: "seed"},
		{PublicKey: "pk-2", Recipient: "0x0000000000000000000000000000000000000002",
			Amount: big.NewInt(1500), RegisteredAt: time.Now(), Source: "vault_share"},
	}
	if err := repo.InsertAll(ctx, rows); err != nil {
		t.Fatalf("InsertAll: %v", err)
	}

	count, err = repo.Count(ctx)
	if err != nil {
		t.Fatalf("Count after insert: %v", err)
	}
	if count != 2 {
		t.Errorf("count after insert = %d, want 2", count)
	}
}

func TestRaffleWinnersRepo_InsertAllIsAtomic(t *testing.T) {
	pool := newRaffleTestPool(t)
	repo := postgres.NewRaffleWinnersRepository(pool)
	ctx := context.Background()

	// Two rows with the same public_key — the second insert should fail and
	// the entire batch should roll back.
	rows := []raffle.CSVRow{
		{PublicKey: "pk-dup", Recipient: "0x1", Amount: big.NewInt(1500), Source: "seed"},
		{PublicKey: "pk-dup", Recipient: "0x2", Amount: big.NewInt(1500), Source: "seed"},
	}
	err := repo.InsertAll(ctx, rows)
	if err == nil {
		t.Errorf("expected error from duplicate insert")
	}
	count, _ := repo.Count(ctx)
	if count != 0 {
		t.Errorf("count after failed batch = %d, want 0 (transaction should have rolled back)", count)
	}
}
```

- [ ] **Step 3: Run test to verify it fails**

```bash
go test ./internal/storage/postgres/ -run TestRaffleWinnersRepo -v
```

Expected: compile error — `NewRaffleWinnersRepository` undefined.

- [ ] **Step 4: Implement the repository**

Create `internal/storage/postgres/airdrop_raffle_winners.go`:

```go
package postgres

import (
	"context"
	"fmt"
	"math/big"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgtype"
	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/vultisig/agent-backend/internal/service/airdrop/raffle"
	"github.com/vultisig/agent-backend/internal/storage/postgres/queries"
)

// RaffleWinnersRepository implements the load-winners subcommand's storage
// needs against agent_raffle_winners.
type RaffleWinnersRepository struct {
	pool *pgxpool.Pool
	q    *queries.Queries
}

func NewRaffleWinnersRepository(pool *pgxpool.Pool) *RaffleWinnersRepository {
	return &RaffleWinnersRepository{pool: pool, q: queries.New(pool)}
}

// Count returns the total number of rows currently in agent_raffle_winners.
// Used by load-winners as the idempotency guard.
func (r *RaffleWinnersRepository) Count(ctx context.Context) (int64, error) {
	n, err := r.q.AirdropRaffleWinnerCount(ctx)
	if err != nil {
		return 0, fmt.Errorf("count winners: %w", err)
	}
	return n, nil
}

// InsertAll inserts every row inside a single transaction. On any failure,
// the entire batch rolls back. Used by load-winners.
func (r *RaffleWinnersRepository) InsertAll(ctx context.Context, rows []raffle.CSVRow) error {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return fmt.Errorf("begin: %w", err)
	}
	defer func() { _ = tx.Rollback(ctx) }() // no-op if Commit succeeded

	qtx := r.q.WithTx(tx)
	for _, row := range rows {
		amount, err := bigIntToNumeric(row.Amount)
		if err != nil {
			return fmt.Errorf("convert amount for %s: %w", row.PublicKey, err)
		}
		err = qtx.InsertRaffleWinner(ctx, &queries.InsertRaffleWinnerParams{
			PublicKey: row.PublicKey,
			Recipient: row.Recipient,
			Amount:    amount,
		})
		if err != nil {
			return fmt.Errorf("insert %s: %w", row.PublicKey, err)
		}
	}

	if err := tx.Commit(ctx); err != nil {
		return fmt.Errorf("commit: %w", err)
	}
	return nil
}

// bigIntToNumeric converts a *big.Int into the pgtype.Numeric the sqlc
// binding expects for NUMERIC(78,0). Tested implicitly by the round-trip
// in TestRaffleWinnersRepo_CountAndInsertAll.
func bigIntToNumeric(n *big.Int) (pgtype.Numeric, error) {
	var num pgtype.Numeric
	if n == nil {
		return num, fmt.Errorf("nil amount")
	}
	if err := num.Scan(n.String()); err != nil {
		return num, err
	}
	return num, nil
}

// silence unused import when the file lives alongside helpers
var _ pgx.Tx = (pgx.Tx)(nil)
```

The exact name of `WithTx` and the param-struct (`InsertRaffleWinnerParams`) depend on what sqlc generated; open `internal/storage/postgres/queries/airdrop_raffle.sql.go` to confirm. Match whatever pattern `conversation.go` uses for transactional inserts if `WithTx` is named differently in this repo.

- [ ] **Step 5: Run tests to verify they pass**

```bash
DATABASE_DSN="$DATABASE_DSN" go test ./internal/storage/postgres/ -run TestRaffleWinnersRepo -v
```

Expected: PASS for both subtests, given a live test DB.

- [ ] **Step 6: Commit**

```bash
git add internal/storage/postgres/airdrop_raffle_winners.go \
         internal/storage/postgres/airdrop_raffle_winners_test.go \
         internal/storage/postgres/airdrop_registration.go
git commit -m "feat(airdrop): add RaffleWinnersRepository + ListForDraw on registrations"
```

---

## Task 8: `main.go` subcommand router + integration build + push PR

**Why:** Final wiring. The `flag.NewFlagSet` per-subcommand pattern (matching the rest of the agent-backend CLIs) lives in `main.go`. End-to-end: build the binary, run `--help` to confirm the surface, and push the branch.

**Files:**
- Create: `cmd/airdrop-raffle/main.go`
- Create: `docs/runbooks/raffle-day-12.md`

- [ ] **Step 1: Implement main.go**

Create `cmd/airdrop-raffle/main.go`. Mirrors the config-loading + DB-pool-construction pattern from `cmd/admin/main.go`, but with stdlib `flag.FlagSet`-based subcommand routing instead of an HTTP server bootstrap:

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"math/big"
	"os"
	"os/signal"
	"strings"
	"syscall"
	"time"

	"github.com/sirupsen/logrus"

	"github.com/vultisig/agent-backend/internal/service/airdrop/ethclient"
	"github.com/vultisig/agent-backend/internal/storage/postgres"
)

const usage = `airdrop-raffle — Vultiverse Airdrop raffle CLI

Usage:
  airdrop-raffle <subcommand> [flags]

Subcommands:
  draw            Pick winners from agent_airdrop_registrations and write winners.csv + manifest.json
  load-winners    Load winners.csv into agent_raffle_winners (refuses if non-empty)
  verify-onchain  Read winners.csv and assert allowance(recipient) on the contract matches each row

Run "airdrop-raffle <subcommand> -h" for subcommand-specific flags.
`

func main() {
	logger := logrus.New()
	logger.SetFormatter(&logrus.JSONFormatter{})
	logger.SetOutput(os.Stdout)

	if len(os.Args) < 2 {
		fmt.Fprint(os.Stderr, usage)
		os.Exit(2)
	}

	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	switch os.Args[1] {
	case "draw":
		os.Exit(drawMain(ctx, logger, os.Args[2:]))
	case "load-winners":
		os.Exit(loadWinnersMain(ctx, logger, os.Args[2:]))
	case "verify-onchain":
		os.Exit(verifyOnchainMain(ctx, logger, os.Args[2:]))
	case "-h", "--help", "help":
		fmt.Print(usage)
		os.Exit(0)
	default:
		fmt.Fprintf(os.Stderr, "unknown subcommand: %s\n\n%s", os.Args[1], usage)
		os.Exit(2)
	}
}

func drawMain(ctx context.Context, logger *logrus.Logger, args []string) int {
	fs := flag.NewFlagSet("draw", flag.ExitOnError)
	seed := fs.Uint64("seed", 0, "PRNG seed (uint64). Default: a fresh crypto/rand value, printed at start.")
	slotCount := fs.Int("slot-count", 0, "Number of winners to draw. Required.")
	vultAmountStr := fs.String("vult-amount", "", "VULT per winner, decimal string (uint256). Required.")
	outputDir := fs.String("output-dir", "", "Directory to write winners.csv + manifest.json. Created if missing. Required.")
	dbURL := fs.String("db-url", "", "Postgres connection string. Required.")
	if err := fs.Parse(args); err != nil {
		return 1
	}

	if *slotCount <= 0 || *vultAmountStr == "" || *outputDir == "" || *dbURL == "" {
		fmt.Fprintln(os.Stderr, "ERROR: --slot-count, --vult-amount, --output-dir, --db-url are required")
		fs.Usage()
		return 1
	}
	amount, ok := new(big.Int).SetString(*vultAmountStr, 10)
	if !ok || amount.Sign() <= 0 {
		fmt.Fprintf(os.Stderr, "ERROR: invalid --vult-amount: %q\n", *vultAmountStr)
		return 1
	}

	resolvedSeed := *seed
	if resolvedSeed == 0 {
		var err error
		resolvedSeed, err = freshSeed()
		if err != nil {
			fmt.Fprintf(os.Stderr, "ERROR: generate fresh seed: %v\n", err)
			return 1
		}
		fmt.Printf("No --seed passed; using fresh crypto/rand seed: %d\n", resolvedSeed)
	}

	db, err := postgres.New(ctx, *dbURL)
	if err != nil {
		logger.WithError(err).Error("connect to db")
		return 1
	}
	defer db.Close()
	regRepo := postgres.NewRegistrationRepository(db.Pool())

	return runDraw(ctx, drawOpts{
		seed:       resolvedSeed,
		slotCount:  *slotCount,
		vultAmount: amount,
		outputDir:  *outputDir,
		lister:     regRepo,
		logger:     logger,
		now:        time.Now,
	})
}

func loadWinnersMain(ctx context.Context, logger *logrus.Logger, args []string) int {
	fs := flag.NewFlagSet("load-winners", flag.ExitOnError)
	outputDir := fs.String("output-dir", "", "Directory containing winners.csv from a prior draw. Required.")
	dbURL := fs.String("db-url", "", "Postgres connection string. Required.")
	if err := fs.Parse(args); err != nil {
		return 1
	}
	if *outputDir == "" || *dbURL == "" {
		fmt.Fprintln(os.Stderr, "ERROR: --output-dir and --db-url are required")
		fs.Usage()
		return 1
	}

	db, err := postgres.New(ctx, *dbURL)
	if err != nil {
		logger.WithError(err).Error("connect to db")
		return 1
	}
	defer db.Close()
	store := postgres.NewRaffleWinnersRepository(db.Pool())

	return runLoadWinners(ctx, loadWinnersOpts{
		outputDir: *outputDir,
		store:     store,
		logger:    logger,
	})
}

func verifyOnchainMain(ctx context.Context, logger *logrus.Logger, args []string) int {
	fs := flag.NewFlagSet("verify-onchain", flag.ExitOnError)
	outputDir := fs.String("output-dir", "", "Directory containing winners.csv. Required.")
	rpcURL := fs.String("rpc-url", "", "Ethereum RPC URL (read-only, no signing). Required.")
	contractStr := fs.String("contract", "", "Deployed AirdropClaim address (0x...). Required.")
	if err := fs.Parse(args); err != nil {
		return 1
	}
	if *outputDir == "" || *rpcURL == "" || *contractStr == "" {
		fmt.Fprintln(os.Stderr, "ERROR: --output-dir, --rpc-url, --contract are required")
		fs.Usage()
		return 1
	}
	contract, err := parseAddress(*contractStr)
	if err != nil {
		fmt.Fprintf(os.Stderr, "ERROR: invalid --contract: %v\n", err)
		return 1
	}

	rpc, err := ethclient.NewClient(*rpcURL)
	if err != nil {
		logger.WithError(err).Error("dial rpc")
		return 1
	}

	return runVerifyOnchain(ctx, verifyOpts{
		outputDir: *outputDir,
		contract:  contract,
		reader:    rpc,
		logger:    logger,
	})
}

// parseAddress validates and parses a 0x-prefixed hex address. Defined here
// (small one-liner) to avoid pulling go-ethereum/common into main.go's import
// surface for one call site — both subcommand mains use it.
func parseAddress(s string) (common.Address, error) {
	s = strings.TrimSpace(s)
	if !strings.HasPrefix(s, "0x") || len(s) != 42 {
		return common.Address{}, fmt.Errorf("expected 0x-prefixed 20-byte hex, got %q", s)
	}
	return common.HexToAddress(s), nil
}
```

(Add `"github.com/ethereum/go-ethereum/common"` to imports — `parseAddress` returns `common.Address`. The import is also already pulled in by `verify_onchain.go`.)

- [ ] **Step 2: Build the binary**

```bash
go build -o bin/airdrop-raffle ./cmd/airdrop-raffle
./bin/airdrop-raffle help
./bin/airdrop-raffle draw -h
./bin/airdrop-raffle load-winners -h
./bin/airdrop-raffle verify-onchain -h
```

Expected: top-level usage prints; each subcommand prints its `flag.FlagSet` usage with the documented flags.

- [ ] **Step 3: Run the full test suite**

```bash
go test ./...
```

Expected: PASS. Investigate any failures.

- [ ] **Step 4: Manual smoke test against local Postgres**

Seed three fake registrations and run `draw` then `load-winners`:

```bash
psql "$DATABASE_DSN" <<'SQL'
TRUNCATE agent_airdrop_registrations;
TRUNCATE agent_raffle_winners;
INSERT INTO agent_airdrop_registrations (public_key, source, recipient_address)
VALUES ('pk-1', 'seed', '0x0000000000000000000000000000000000000001'),
       ('pk-2', 'seed', '0x0000000000000000000000000000000000000002'),
       ('pk-3', 'vault_share', '0x0000000000000000000000000000000000000003');
SQL

mkdir -p /tmp/raffle-test
./bin/airdrop-raffle draw \
  --seed 12345 \
  --slot-count 2 \
  --vult-amount 1500 \
  --output-dir /tmp/raffle-test \
  --db-url "$DATABASE_DSN"

cat /tmp/raffle-test/winners.csv
cat /tmp/raffle-test/manifest.json

./bin/airdrop-raffle load-winners \
  --output-dir /tmp/raffle-test \
  --db-url "$DATABASE_DSN"

psql "$DATABASE_DSN" -c "SELECT * FROM agent_raffle_winners"

# Re-running load-winners should now refuse:
./bin/airdrop-raffle load-winners \
  --output-dir /tmp/raffle-test \
  --db-url "$DATABASE_DSN"
```

Expected: `draw` writes a CSV with two rows + a manifest. `load-winners` inserts both, prints "Loaded 2 winners". The second `load-winners` exits non-zero with the "agent_raffle_winners already has 2 rows; refusing" message.

- [ ] **Step 5: Draft the operator runbook**

Create `docs/runbooks/raffle-day-12.md`:

```markdown
# Day 12 Runbook — Raffle Draw

> **Operator audience.** This runbook is the step-by-step for the human running the Day 12 raffle. Companion to `docs/airdrop-specs/stage-2-raffle-draw.md` (the spec) and `docs/airdrop-specs/sibling-on-chain-contract.md` (the contract behaviour).

## Pre-flight

- Boarding window has closed (`TRANSITION_WINDOW_END` has passed).
- `AirdropClaim.sol` is deployed; address + ABI captured.
- VULT prize pool has been transferred to the contract (`slot_count × amount × 1.05`).
- The `airdrop-raffle` binary is built and on the operator's PATH.
- A private ops repo (or `vultisig/vultisig-contract`) is cloned locally for committing the artifact directory.

## Step 1 — Draw

```bash
mkdir -p ./raffle-2026-day-12
airdrop-raffle draw \
  --slot-count <STAGE_0_SLOT_COUNT> \
  --vult-amount <STAGE_0_VULT_AMOUNT_BASE_UNITS> \
  --output-dir ./raffle-2026-day-12 \
  --db-url "$AIRDROP_PROD_DSN"
```

Captures:
- `winners.csv` — the artifact handed to the multisig signer.
- `manifest.json` — includes the seed for later verifiability.

The CLI prints the seed to stdout. Record it (operator log + post-draw note on X).

## Step 2 — Commit the artifact directory

```bash
cp -r ./raffle-2026-day-12 path/to/ops-repo/
cd path/to/ops-repo
git add raffle-2026-day-12
git commit -m "raffle: 2026 day-12 draw artifacts (seed=<SEED>)"
git push
```

## Step 3 — Hand winners.csv to the multisig signer

The multisig signer's checklist:
1. Open `winners.csv`. Eyeball: row count matches `slot_count`, no malformed addresses.
2. Split into batches of ~100 rows.
3. For each batch, construct `setWinners(addresses, amounts)` calldata and sign through the multisig UI.
4. Check on Etherscan that each batch was mined; capture each tx hash.
5. **Footgun guard:** before submitting, diff against any prior `setWinners` calls + any confirmed `Claimed` events. Re-arming a previously-claimed address re-grants their allowance (see `sibling-on-chain-contract.md`).

## Step 4 — Verify on-chain

After every batch is mined:

```bash
airdrop-raffle verify-onchain \
  --output-dir ./raffle-2026-day-12 \
  --rpc-url "$EVM_RPC_URL" \
  --contract "$AIRDROP_CLAIM_CONTRACT_ADDRESS"
```

Expected: every row prints `OK`. Summary: `N matched, 0 mismatched, 0 errored`.

If anything is `FAIL` or `ERROR`, **do not load-winners**. Investigate which batch missed the recipient and re-run the multisig batch for the missed entries before re-running `verify-onchain`.

## Step 5 — Load winners into Postgres

```bash
airdrop-raffle load-winners \
  --output-dir ./raffle-2026-day-12 \
  --db-url "$AIRDROP_PROD_DSN"
```

Single transaction. After this commits, `/airdrop/status` returns `won` / `lost` to all users.

If `agent_raffle_winners` is already populated (e.g. you ran this earlier and need to re-load), explicitly:

```bash
psql "$AIRDROP_PROD_DSN" -c "TRUNCATE agent_raffle_winners"
```

then re-run `load-winners`.

## Done

- Artifact directory committed to ops repo.
- Multisig has set every winner's allowance.
- `verify-onchain` is green.
- `agent_raffle_winners` is populated.
- Stage 1's `/airdrop/status` is now returning `won`/`lost` instead of `awaiting_draw`.

The system is ready for Stage 4's claim relayer to come up on Day 28.
```

- [ ] **Step 6: Run lint**

```bash
make lint
```

Expected: no errors.

- [ ] **Step 7: Push branch**

```bash
git add cmd/airdrop-raffle/main.go docs/runbooks/raffle-day-12.md
git commit -m "feat(airdrop): wire main.go subcommand router + Day 12 runbook"
git push -u origin feat/airdrop-stage-2-raffle-cli
```

- [ ] **Step 8: Open PR**

```bash
gh pr create --title "feat(airdrop): Stage 2 raffle draw CLI (draw, load-winners, verify-onchain)" --body "$(cat <<'EOF'
## Summary

Adds the Stage 2 raffle CLI from the Vultiverse Airdrop spec — a single `airdrop-raffle` binary with three subcommands the operator runs by hand on Day 12:

- `draw` — picks winners with a seeded Fisher–Yates shuffle and writes `winners.csv` + `manifest.json`
- `load-winners` — mirrors winners.csv into `agent_raffle_winners` in a single transaction (refuses if non-empty)
- `verify-onchain` — reads winners.csv and asserts `allowance(recipient)` on the deployed AirdropClaim matches each row

Plus the Day 12 operator runbook.

Spec source of truth: `docs/airdrop-specs/stage-2-raffle-draw.md`.

## Architecture
- One Go binary at `cmd/airdrop-raffle/` with stdlib `flag.FlagSet` per-subcommand routing — same dependency-free pattern as `cmd/admin` and `cmd/scheduler`
- Pure selection + serialization logic in `internal/service/airdrop/raffle/` so unit tests run without Postgres or RPC
- New sqlc query file `internal/storage/postgres/sqlc/airdrop_raffle.sql` for the CLI's reads/writes
- New `RaffleWinnersRepository` for batched inserts inside one transaction
- `internal/service/airdrop/ethclient/client.go` extended with `AirdropAllowance(contract, recipient)` — distinct selector from the existing `BalanceOf` (ERC-20 vs AirdropClaim's single-arg storage getter)
- No Merkle code (dropped from the spec 2026-04-23)

## Spec coverage
- `draw` — all flags, exit codes (0/1/2/4), idempotent for the same seed, undersubscription banner, output to `--output-dir`
- `load-winners` — single-transaction insert, refuses if already populated
- `verify-onchain` — per-row pass/fail print, summary line, non-zero exit on any mismatch
- Selection unit tests cover the spec's "Tests → Unit" list (zero entries hard-fail, undersubscription, oversubscription with no duplicates, idempotency, recipient passthrough)
- Repository integration tests cover atomic batch insert + idempotency guard
- Operator runbook drafted at `docs/runbooks/raffle-day-12.md`

## Test plan
- [ ] `go test ./...` passes
- [ ] Manual smoke (Step 4 of Task 8) inserts test registrations, runs all three subcommands, verifies idempotency guard
- [ ] `make lint` clean
- [ ] (Deferred to E2E test) `verify-onchain` against an Anvil-deployed `AirdropClaim` after multisig has set allowances

## Hard deadline
Operator runs `airdrop-raffle draw` on Day 12 (boarding window closes at `TRANSITION_WINDOW_END`). Binary must be built + runbook drafted by Day 11.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Capture and report the PR URL.

---

## Self-review notes

This plan was reviewed against `stage-2-raffle-draw.md` after writing. Items confirmed against the spec's "Done when" list:

- ✅ All three subcommands implemented and unit-tested (`runDraw`, `runLoadWinners`, `runVerifyOnchain` each have a test file).
- ✅ Integration test against real Postgres covered by `TestRaffleWinnersRepo_*` (Task 7).
- ✅ Operator runbook drafted at `docs/runbooks/raffle-day-12.md` (Task 8 Step 5).
- ⏭ E2E test against an Anvil-deployed `AirdropClaim` — left as the test plan checkbox in the PR body. The unit tests prove the CSV round-trip, the selector regression test proves the calldata, the integration tests prove the DB path. The end-to-end "draw → multisig sets allowances → verify-onchain → load-winners → claim succeeds" flow is the Stage 4 PR's responsibility (it owns the relayer and will be the only place an Anvil fixture is wired in).

Items intentionally NOT in this plan (in line with the scope-of-this-plan section of the brief):
- **No Merkle / proof / leaf code** — the spec explicitly dropped this on 2026-04-23. Confirmed nothing in `internal/service/airdrop/raffle/` references trees or proofs.
- **No `setWinners` signing** — multisig owns this; the CLI just produces `winners.csv`.
- **No HTTP service / goroutines / middleware** — every subcommand exits when its work is done.
- **No new env vars** — every input is a CLI flag, per the spec's "Configuration" section.

Decisions that needed to be made and why:

- **stdlib `flag` over cobra.** No other agent-backend CLI uses cobra; matching the existing pattern keeps the binary's import surface tiny.
- **Extended the ethclient wrapper rather than calling `CallContract` from `cmd/`.** ABI encoding (selector + 32-byte padding) belongs alongside the existing `BalanceOf` implementation. Adds one method + one regression test to the wrapper; saves repeating the same encoding pattern in `cmd/airdrop-raffle/`.
- **`InsertAll` does its own `tx.Begin/Commit` rather than accepting a `pgx.Tx` from the caller.** load-winners is the sole caller; nothing else needs the transactional surface; keeps the CLI side dead-simple.
- **`ListForDraw` lives on the existing `RegistrationRepository`** rather than a new repo — the table is owned by Stage 1's repo, and bolting on a single read method is cheaper than introducing a parallel adapter.
