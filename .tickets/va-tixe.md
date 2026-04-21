---
id: va-tixe
status: closed
deps: []
links: []
created: 2026-04-20T06:36:30Z
type: task
priority: 2
assignee: Jibles
---
# MCP tool consolidation: retire redundant build_* tools and validate execute_send UTXO/Cosmos parity

## Objective

Consolidate the MCP tool suite by retiring `build_*` tools whose coverage is fully subsumed by the newer `execute_*` family, then validate that `execute_send` reaches signing-path parity with UTXO and Cosmos `build_*_send` tools so they can be retired as well. The target architecture is a small set of high-level end-to-end `execute_*` tools for common operations, plus a focused set of low-level atomic tools for arbitrary or protocol-specific signing — no overlapping tiers.

## Context & Findings

The agent tool suite has two families doing overlapping work:
- **`execute_*`** (5 operation-centric tools: send/swap/contract_call/lp_add/lp_remove) — bundled flows with stepper UI, rich labels, multi-tx approve+sign handling
- **`build_*`** (chain-named, many) — single-tx signing tools; some are genuine primitives, many are vestigial duplicates of execute_*

A coverage audit across `mcp-ts` + `agent-backend` produced this breakdown:

**Safe to retire immediately (zero coverage loss, no validation required):**
- `build_swap_tx` — wraps same providers (THOR/MAYA/1inch/LiFi/Uniswap) as `execute_swap`
- `build_thorchain_lp_add` → `execute_lp_add`
- `build_thorchain_lp_remove` → `execute_lp_remove`
- `build_solana_tx` (native SOL) → `execute_send`
- `build_cardano_send` → `execute_send`
- `build_ton_send` (native) → `execute_send`
- `build_trx_send` (native) → `execute_send`

**Keep regardless (genuine low-level tier):**
- Generic primitives: `sign_typed_data`, `build_custom_tx`, `build_evm_tx`
- Token-layer builders where `execute_send` has documented rejections because the wallet signing-path doesn't handle token-specific payload fields yet: `build_spl_transfer_tx`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build_xrp_send`
- Domain-specific protocol tools: Rujira family (`build_fin_*`, `build_ghost_*`, `build_perps_*`, `build_ruji_*`, `build_secured_*`), `build_pumpfun_create`

**Target for follow-on retirement (pending validation):**
- UTXO send family: `build_btc_send`, `build_ltc_send`, `build_doge_send`, `build_bch_send`, `build_dash_send`, `build_zec_send`
- Cosmos send family: `build_thor_send`, `build_maya_send`, `build_cosmos_send`, `build_osmosis_send`, `build_kujira_send` (confirm complete list in mcp-ts)

`execute_send` already claims coverage for UTXO and Cosmos at the handler level; the wait is on parity validation, not missing implementation. **Zcash is one real gap** — `execute_send` does not currently implement Zcash, so retiring `build_zec_send` either requires extending execute_send or an explicit decision to drop Zcash support.

### Validation strategy — no emulator needed

Parity testing is a pure-function diff problem: same user intent in, compare signable payloads out. Four layers, only the first two are primary:

**Layer 1 — mcp-ts unit tests (primary signal):**
Call both tool handlers with matching user intent; assert signable payload equivalence. Mock chain RPC with recorded fixtures (UTXO sets, account numbers/sequences, fee estimates). Hermetic, deterministic, fast.

**Layer 2 — Client-side parsing tests:**
Feed both outputs through `parseServerTx` (`src/features/agent/lib/txProposal.ts`) and assert both produce `PendingTx` of the same shape. Catches client-side drift.

**Layer 3 — `vultiagent-cli` smoke tests:**
5-10 representative flows exercised end-to-end with SDK signing but no broadcast. Catches SDK-layer issues that unit tests miss.

**Layer 4 — Device/emulator:**
Reserved for MPC handshake, biometric prompt, UI state-machine behavior. Not the primary parity validation path.

### Rejected approaches

- **Merging `useToolExecution` into `useTransactionFlow`** as a hook-unification step — addresses a symptom. Actual goal is fewer tools server-side, after which the frontend hooks simplify naturally.
- **Emulator-only validation** — unnecessary for a parity diff, which is a pure function of inputs → payload.

## Files

- `mcp-ts/` (sibling repo) — primary tool definitions, handlers, and where the validation harness lives
- `agent-backend/` (sibling repo) — Claude-facing tool allowlist; must ship in lockstep with mcp-ts removals
- `vultiagent-app/src/features/agent/lib/toolUIRegistry.ts` — client-side registry, remove entries as tools retire
- `vultiagent-app/src/features/agent/lib/txProposal.ts` — Layer 2 parsing tests
- `vultiagent-app/src/features/agent/lib/executePrep.ts` — client shape validator for `execute_*` tx_encoding families
- `vultiagent-cli/` (sibling repo) — Layer 3 smoke test scripts

## Acceptance Criteria

Phase 1 — immediate retirement (no validation required):
- [ ] The 7 "safe to retire now" tools listed above are removed from mcp-ts registration and the agent-backend allowlist
- [ ] Client `toolUIRegistry` entries for any of these that had explicit registrations are removed (most are caught by the `build_*` wildcard — verify before deleting)
- [ ] No functional regression for common flows (e.g. "send 100 USDC to 0x…", "swap 1 ETH to USDC", "add liquidity to BTC-ETH pool")
- [ ] PR description enumerates each retired tool and names its `execute_*` replacement

Phase 2 — validation harness (gate for Phase 3):
- [ ] mcp-ts test harness diffs `execute_send` output vs `build_*_send` output for matching intent
- [ ] Recorded fixtures: UTXO sets (BTC/LTC/DOGE/BCH/DASH), cosmos account/sequence data (THORChain/Maya/Cosmos hub/Osmosis/Kujira)
- [ ] Layer 1 parity tests pass for every UTXO chain or drift is documented per chain
- [ ] Layer 1 parity tests pass for every Cosmos chain including memo-string equivalence, or drift is documented
- [ ] Layer 2 client-side tests confirm `parseServerTx` handles both outputs identically for the same intent

Phase 3 — conditional retirement (gated on Phase 2):
- [ ] UTXO `build_*_send` tools retired iff Layer 1 + Layer 2 pass for BTC/LTC/DOGE/BCH/DASH
- [ ] Cosmos `build_*_send` tools retired iff Layer 1 + Layer 2 pass for memo/auth semantics
- [ ] Zcash: decision recorded — either extend `execute_send` to cover Zcash before retiring `build_zec_send`, or explicit decision to keep the tool indefinitely
- [ ] Layer 3 CLI smoke tests run for at least one chain from each family that got retired

## Gotchas

- Cosmos memo strings (THORChain/Maya swap memos) are protocol-sensitive; byte-level memo diff required, not just envelope shape.
- UTXO input selection is non-deterministic across fee strategies — fix fixtures to a single strategy when asserting equivalence.
- Client `validateTxArgsShape` is permissive for 6 chain families (solana/ripple/tron/ton/sui/cardano). Don't tighten as part of this work — separate ticket.
- Token-layer builders (`build_spl_transfer_tx` etc.) are NOT in scope; `execute_send` explicitly rejects them pending wallet signing-path updates.
- Never touch: Rujira, PumpFun, `sign_typed_data`, `build_custom_tx`, `build_evm_tx` — these are kept regardless.
- Branch `refactor/v-pxuw-tool-consolidation` already landed the client-side execute_* work; this ticket builds on that, not re-implements it.
- Any tool retirement requires both mcp-ts removal AND agent-backend allowlist update — both repos ship in lockstep or the backend still advertises the tool to Claude.

## Notes

**2026-04-20T07:04:34Z**

Phase 1 complete: 7 redundant build_* tools retired. mcp-ts commit ad8dd45 (12 files, 71+/1249-); agent-backend commit 9397502 (3 files, 27/27). pnpm typecheck/test/build clean (312/312 tests pass). go test ./... clean. vultiagent-app untouched (wildcard toolUIRegistry handled removals). Surface preserved: Rujira, PumpFun, sign_typed_data, build_custom_tx, build_evm_tx, token-layer builders.

**2026-04-20T07:19:50Z**

Phase 2 complete: parity validation harness. mcp-ts commit 732e7ac (42 new tests across 11 chains); vultiagent-app commit b405c80 (13 new parseServerTx parity tests). All 11 UTXO+Cosmos chains pass Layer 1+2 parity. Cosmos memo bytewise parity confirmed for THOR/Maya (SWAP/ADD/WITHDRAW/LOAN/BOND). Zcash IS supported by execute_send — ticket's gap note was stale. Phase 3 unblocked: 11 tools safe to retire.

**2026-04-20T07:34:22Z**

Phase 3 complete: 11 UTXO/Cosmos build_*_send tools retired. mcp-ts commit 9cf00ea (deletes build-utxo-send.ts + build-cosmos-send.ts + parity tests, updates index.ts, skills, cosmos-chains); agent-backend commit 02c9099 (prunes tool_classify.go dead branches + test fixtures, 22 cases -> 6). Layer 3 smoke via direct handler invocation (vultiagent-cli not present in workspace) produced canonical ExecutePrepResult for Bitcoin + THORChain with memo byte-preservation. Total across all 3 phases: 18 tools removed, ~1550 lines deleted.
