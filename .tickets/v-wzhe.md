---
id: v-wzhe
status: open
deps: []
links: []
created: 2026-05-03T23:44:53Z
type: feature
priority: 2
assignee: Jibles
---
# Consolidate Rujira CCL into execute_lp_* tool family

## Problem
Rujira CCL (Custom Concentrated Liquidity — personal LP range-positions on FIN pair contracts) currently exposes 10+ raw building blocks to the LLM: `get_fin_pair_address`, `get_market_price`, `getRangePositions`, `getRangePosition`, `quoteRangeDistribution`, `buildRangeCreatePosition`, `buildRangeDeposit`, `buildRangeWithdraw`, `buildRangeWithdrawAll`, `buildRangeClaim`, `buildRangeTransfer`. Same failure shape as the FIN swap path that bit Felix in conv `8ae9bf54`: asset-format conversion, pair discovery, decimal-string configs, and multi-step state management all fall on the LLM. CCL inherits 100% of the FIN tooling failure surface AND adds price-range math, position lifecycle (create / deposit / withdraw / claim / rebalance), and per-position `idx` state tracking. Felix wanted CCL as a feature; the current shape can't deliver it reliably.

"Solved" = a small `execute_lp_*` family that compresses common LP intents into single-tool calls (mirroring how `execute_swap` compresses swap intents), with raw `build_range_*` tools dropped from the LLM-facing toolset.

## Background
### Findings
- CCL tool registrations: `mcp-ts/src/tools/rujira/index.ts:61-71` — comment block reads `// CCL (Custom Concentrated Liquidity) — per-position range ops on FIN pairs`.
- Tool source: `mcp-ts/src/tools/rujira/range.ts` — header comment: `/* RUJI Trade Custom Concentrated Liquidity (CCL) — range position tools. CCL lets a user place a personal range-position (idx) on a FIN pair contract */`.
- Existing prompt trajectory at `agent-backend/internal/service/agent/prompt.go:121`: *"Create flow: resolve pair via `get_fin_pair_address` → `get_market_price` for the pair → `build_range_create_position` with EITHER preset=passive|wide|tight|lower|higher + current_price OR an explicit config. All config fields (high, low, spread, skew, fee) are Decimal strings with up to 12 fractional digits — NEVER pass numbers. Funds: base + quote coins in native denoms."*
- Strategy presets `passive | tight | wide | lower | higher` already exist in the underlying `build_range_create_position` builder. **The abstraction is half-built** — just exposed at the wrong layer (LLM-facing instead of server-internal).
- Position state lives on the FIN pair contract, identified by `idx` (string bigint, per-pair).
- Same asset-format triad as FIN swaps (l1 / thorchain / fin) — RUJI is `RUJI` / `THOR.RUJI` / `x/ruji` per `mcp-ts/packages/rujira/dist/utils/format.js:185-198`. Same failure mode the LLM hit on swaps.
- Cross-ref: Rujira FIN aggregator ticket `v-xemk` covers the swap-path primitives (pair discovery, asset format conversion, multi-tx orchestration). This ticket reuses those primitives and is downstream of it.

## Current Thinking
**4 write tools + 2 read tools.** Not a single tool with action discriminator — CCL has genuinely different intent shapes per lifecycle stage (create needs strategy + capital; withdraw needs position + percent; claim takes only position) and per-action tools keep arg-filling unambiguous for the LLM.

```ts
execute_lp_create(
  pair: string,                                      // "RUNE/USDC", server resolves contract
  capital_usd: string,                               // "$50" — server splits base/quote
  strategy: "passive"|"tight"|"wide"|"lower"|"higher",
  // current_price fetched server-side from get_market_price
)

execute_lp_withdraw(position_id: string, percent?: number)
execute_lp_claim(position_id: string)
execute_lp_rebalance(position_id: string, new_strategy: string)

// read-side (already exist as getRangePositions/getRangePosition,
// keep exposed but consider renaming to match family)
get_lp_positions
get_lp_position
```

**What gets hidden server-side inside `execute_lp_create`:**
1. Pair discovery — `get_fin_pair_address` runs internally.
2. Asset-format conversion (l1 / thorchain / fin) — server picks the right form per operation.
3. Decimal-string config building — strategy preset maps to `{high, low, spread, skew, fee}` server-side.
4. Current-price fetching — auto-fetched, passed to position builder.
5. Wasm-execute payload encoding — opaque, returned in `ExecutePrepResult` envelope.
6. Multi-tx approval flows — bundled into stepper config (same pattern Polymarket uses for its 5-approval onboarding).
7. Position `idx` tracking — server-side state, looked up by `(chain, address, pair)`.

**What stays user-facing because it can't be auto-decided:**
1. Strategy choice — passive vs tight vs wide is a real product decision with yield/risk tradeoffs. Default to `passive` and let the model only ask if the user said something specific ("tight LP", "active management").
2. Out-of-range deposits — when current price is outside the range, only one asset is needed. Tool returns a clear quote-side response ("you need RUNE not USDC for this position"); model handles disambiguation with the user.
3. Position lifecycle disambiguation — "withdraw my LP" with 3 positions needs a chip / clarification. Same shape as "send to which contact?".

## Constraints
- Output envelope must match `ExecutePrepResult` (`mcp-ts/src/tools/execute/_base.ts:26-81`) so the app-side cosmos-msg parser from PR #320 handles it without additional plumbing.
- Raw `build_range_*` tools MUST be dropped from the LLM-facing toolset after consolidation — that's the failure mode being avoided. Keep them registered for direct `mcpProvider.CallTool` paths.
- Reuse the FIN-format-conversion / pair-discovery / multi-tx primitives that the Rujira FIN aggregator ticket (`v-xemk`) is building. Don't duplicate them.

## Assumptions
- The Rujira FIN aggregator (`v-xemk`) lands first and provides the FIN primitives this ticket reuses. If it doesn't, scope expands to include the primitives.
- Strategy presets in `build_range_create_position` cover ~90% of user intent. The 10% needing custom configs can fall back to a separate power-user tool or a lower-priority extension — out of scope here.
- Vultisig vault signing already supports Cosmos-SDK `wasm_execute` envelopes (true after PR #320 lands; verify before starting).

## Non-goals
- LP / range positions on non-Rujira venues (Uniswap V3, etc.). Rujira FIN only.
- GHOST lending, RUJI staking, FIN orderbook limit orders. Separate Rujira products with their own tickets.
- Re-exposing raw `build_range_*` tools to the LLM as a "power-user mode". The whole point is consolidation.
- Custom (non-preset) range configs. The 5 presets cover the headline use cases; custom configs are a follow-up.

## Dead Ends
- **Single tool with action discriminator** (`execute_lp({action: "create"|"withdraw"|"claim"|"rebalance", ...})`). Reason: heterogeneous input shape per action makes it harder for the model to fill args correctly even though it's easier to choose. Per-action tools are cleaner — same reasoning Vultisig already applied to `execute_send` vs `execute_swap` vs `execute_contract_call`.
- **Letting the LLM orchestrate raw `build_range_*` tools (status quo).** Reason: same failure mode as `get_fin_swap_quote` in Felix's dump — LLM can't reliably do graph search + format conversion + decimal-string config + multi-step state tracking. 10-tool surface area is the bug, not the symptom.

## Open Questions
- Default strategy = `passive` — confirmed in this conversation but worth product input on whether `tight` or `wide` should be the default for the typical Vultisig user.
- Rebalance UX — does it close the old position and open a new one (2 txs visible to user), or migrate in-place (single tx with combined messages, if FIN supports it)? Affects stepper config.
- Out-of-range deposit handling — single-sided fill, or rejection with quote-feedback "you need RUNE not USDC"? Latter is clearer; former is less friction.
- Position labelling — use `idx` as user-facing ID, or generate a friendlier handle (e.g. "RUNE/USDC #1")? `idx` is what the contract uses but is opaque ("idx 47821938").
- Read-side renaming — `getRangePositions` → `get_lp_positions` for naming consistency, or keep current names? Cheap rename if done now; expensive later.
- Sequencing — strict dependency on `v-xemk` (FIN aggregator), or could this start in parallel reusing the same primitives as they're built?

## Notes

**2026-05-05T05:33:11Z**

migrated to GH issue: https://github.com/vultisig/vultiagent-app/issues/439

**2026-05-05T05:34:06Z**

kept open locally — GH issue is a duplicate, local remains canonical scratch
