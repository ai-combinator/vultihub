---
id: v-whzf
status: in_progress
deps: []
links: []
created: 2026-04-23T23:00:43Z
type: feature
priority: 3
assignee: Jibles
---
# mcp-ts: Port Uniswap V3 read tools from Go mcp (pool_info, position_info, tick_math)

## Objective

Port three Uniswap V3 read tools (`uniswap_pool_info`, `uniswap_position_info`, `uniswap_tick_math`) from the Go `mcp/internal/tools/uniswap_v3_*.go` source into mcp-ts. This is a straightforward language port — not net-new protocol work — but was bundled into #30 under the "consolidation" PR label which misframed it as scope creep. Raise as its own draft PR so it gets protocol-review audience, not consolidation-review audience.

## Context & Findings

### Why this is its own PR

My own audit of the #30 scope flagged this as "net-new protocol surface smuggled in." That framing was overstated — the three tools already exist in Go at `vultisig/mcp/internal/tools/uniswap_v3_*.go`. This is a TS port, not a new integration. But bundling it under "tool consolidation" hid it from protocol-review scrutiny and confused reviewers.

Commit message from `ea38d96`:
> "v-pxuw task 4. mcp-ts had no Uniswap tools; these are the three shapes called out in the design table, named to match. Ported from Go mcp/internal/tools/uniswap_v3_*.go, rewired on SDK evmCall + viem ABI codec instead of raw selectors, with the tick math in pure TypeScript. Chains supported: Ethereum, Arbitrum, Optimism, Polygon, Base, BSC, Avalanche, Blast (same set as Go parity). Tests mock @vultisig/sdk.evmCall to exercise selector encoding + ABI decode + dispatch. 20 new tests; tick-math unit tests are pure."

### What this ticket covers

**Cherry-pick from `refactor/v-pxuw-tool-consolidation`:**
- `ea38d96` — mcp-ts: Uniswap V3 read tools (pool_info / position_info / tick_math)
- Corresponding test files + vultiagent-app fixture update
- `src/tools/uniswap/_erc20.ts`, `addresses.ts`, `pool-info.ts`, `position-info.ts`, `tick-math.ts`

Also cherry-pick any subsequent typo/category fixes that touched `src/tools/uniswap/` only — but DO NOT touch `src/tools/index.ts` wholesale re-organisation from `f5c163e` (that belongs to ticket 7 or stays out entirely).

### Reference material for the port

- Go source: `vultisig/mcp/internal/tools/uniswap_v3_pool_info.go`, `uniswap_v3_position_info.go`, `uniswap_v3_tick_math.go`
- Pattern to follow in TS: `src/tools/evm/abi-encode.ts` (SDK-wrapping one-liner style), viem ABI helpers
- 8 chains parity: Ethereum, Arbitrum, Optimism, Polygon, Base, BSC, Avalanche, Blast

## Files

**New:**
- `src/tools/uniswap/_erc20.ts` — ERC20 helper shared across the three
- `src/tools/uniswap/addresses.ts` — per-chain Uniswap V3 factory / position-manager addresses
- `src/tools/uniswap/pool-info.ts`
- `src/tools/uniswap/position-info.ts`
- `src/tools/uniswap/tick-math.ts`
- Tests under `src/tools/uniswap/__tests__/`

**Modified:**
- `src/tools/index.ts` — add 3 imports + 3 registrations in an "uniswap reads" block. Register at the bottom of `allTools[]` without touching other entries.

## Acceptance Criteria

- [ ] PR opened as **draft** against mcp-ts main, title `feat(tools): port Uniswap V3 reads from Go mcp`
- [ ] PR body explicitly frames this as a **port**, not new integration, with links to the Go source
- [ ] 3 tools registered in `src/tools/index.ts`; no other registrations changed
- [ ] All 20 new tests pass; `pnpm test`, `pnpm lint`, `pnpm build` clean
- [ ] PR notes this is **independent and mergeable off main** — no dependency on ticket 1 (execute_*) or any other
- [ ] `_meta.categories: [Categories.UNISWAP]` if ticket 1's `toolCategories` has landed, otherwise plain string `"uniswap"` (will be harmonized when ticket 7 lands)

## Gotchas

- Reviewers on #30 didn't realize this was a Go-to-TS port. State it prominently in the first sentence of the PR body.
- The 8-chain address table must match Go source exactly; do not update addresses "while we're here" — that's a separate upgrade if needed.
- `tick-math` is pure math (no RPC); verify it works with BigInt in JS for sqrt_price_x96 ↔ price ↔ tick conversions. Tests should cover edge cases (tick = 0, tick = MIN_TICK, tick = MAX_TICK, price = 1, price with fee-tier snapping).
- The tools call `@vultisig/sdk.evmCall`; make sure this export exists on whatever SDK version main is pinned to. If not, use the pattern from existing `src/tools/evm/evm-call.ts`.
- Do not bundle any DeFi consolidation (ticket 3) or Polymarket work (ticket 4) into this PR.

## Notes

**2026-04-23T23:45:36Z**

Draft PR opened: https://github.com/vultisig/mcp-ts/pull/48
