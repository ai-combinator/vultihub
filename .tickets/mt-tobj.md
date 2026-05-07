---
id: mt-tobj
status: open
deps: [mt-wjmy]
links: []
created: 2026-05-04T00:23:08Z
type: chore
priority: 3
assignee: Jibles
---
# Retire per-chain MCP balance tools, collapse agent-backend dispatch (post mt-wjmy cleanup)

## Problem

After `mt-wjmy` ships and the SDK-backed unified balance tool (`mcp-ts/src/tools/balance/get-balance.ts`, wrapping `@vultisig/sdk`'s `getCoinBalance`) proves out in production, ~900 LOC of now-redundant per-chain balance plumbing across agent-backend + mcp-ts + vultiagent-app can be deleted. Goal: collapse to a single SDK-backed balance path everywhere, retire the per-chain dispatch, eliminate the LLM-facing droplist of per-chain balance tools, and clean up the RN app's now-unreachable thinking-step labels. Net effect: MCP-ts honors its "thin wrapper" contract for the balance surface; adding a new chain becomes a one-place change in vultisig-sdk. The mcp-ts CLAUDE.md *"per-chain tools will be migrated incrementally"* statement is the contract this ticket fulfils.

## Background

### Findings

Validated by subagent against current code (post-mt-wjmy assumed merged into main).

**agent-backend deletions (~120 LOC):**
- `chainToBalanceTool` registry (`internal/service/agent/data_tools.go:392-436`, 45 LOC) — wholesale dead.
- `chainCanonicalNames` (`data_tools.go:445-471`, 27 LOC) — only consumer is `canonicalChainName`; wholesale dead.
- `parseBalanceResult` (`data_tools.go:531-612`, ~80 LOC) — only caller is `fetchChainBalance:378`; wholesale dead, along with helpers (`textBalanceRe`, `textAsOfRe`, `firstStringValue`, `balanceResultKeys`).
- `fetchChainBalance` (`data_tools.go:349-383`, 35 LOC) — collapses to a single SDK-backed call.
- `tool_filter.go llmFacingDropList:198-218` — 21 per-chain balance tool entries; exist solely to hide per-chain tools from the LLM. Comment block at `tool_filter.go:153-174` documenting the hide-from-LLM constraint becomes obsolete.
- Side note: `chainTicker()` at `data_tools.go:638` reads `.ticker` from the deleted map for multi-denom fallback. Implementation choice for preserving its behavior is in Open Questions.

**mcp-ts deletions (~815 LOC + tests):**
- 5 balance tool files in `src/tools/balance/`: `cosmos-balance.ts` (199), `other-balance.ts` (354), `evm-balance.ts` (110), `solana-balance.ts` (107), `utxo-balance.ts` (45). All exports are pure tool defs consumed only by `tools/index.ts:130-144` (verified by grep — `getAtomBalance` named export and `cosmosBalanceTools` have zero non-registry consumers).
- Their tool registrations in `src/tools/index.ts:130-144`.
- Tests: `tests/maya-balance.test.ts`, `tests/xrp-balance.test.ts`, parity entries in `tests/parity/mcp-tools.postman_collection.json` (~50-100 LOC).
- The unified replacement (`src/tools/balance/get-balance.ts`, wrapping SDK's `getCoinBalance`) is added by mt-wjmy and stays.

**vultiagent-app cleanup (~30 LOC, UI cosmetic only):**
- 11 hardcoded per-chain tool names mapped to "Checking balance" thinking-step labels at `src/features/agent/lib/coalesceParts.ts:8-18`, `toolLabels.ts:113-114`, `chipPresets.ts:178`. Once tools are retired, these are unreachable code, not breakage.

**Test-fixture impact (~140 grep hits across 11 agent-backend test files):** dominated by `tool_filter_test.go` (44), `tool_classify_test.go` (25), `validator/golden.jsonl` (29), `data_tools_test.go` (15). Most are fixture inputs, not droplist-content assertions. Each touch is a one-liner — high count, low complexity.

**External consumer search — clean:** no hits on per-chain balance tool names across `vultisig-sdk/clients`, `vultiserver`, `station`, or vultiagent-app dispatch paths. Native repos (Android/iOS/Windows) not checked out locally; flagged in Open Questions for verification before mcp-ts PR merges.

**Latent bug confirmed:** `chainToBalanceTool["polkadot"] = {name: "get_dot_balance"}` at `data_tools.go:435` references a tool that does not exist in mcp-ts. SDK migration silently fixes this.

**Stride non-issue (correction to mt-wjmy scope):** no `get_stride_balance` tool in mcp-ts. Both repos are Stride-less. Not a coverage gap.

### External Context

- `mt-wjmy` is the predecessor; this ticket is a **dependency** on its merge + ~2-week production bake.
- Pre-launch state-disposability convention applies: no feature flag / dual-emit period needed. Hard cut acceptable.
- vultisig CLAUDE.md guidance: *"Chain ops, signing, and address derivation live in vultisig-sdk. UI flow, request handling, and platform glue stay in their app/CLI/backend layer."* This cleanup is the agent-backend / mcp-ts side of that contract being honored for the balance surface.

## Current Thinking

**One ticket, three coordinated PRs, hard merge order.** Keeping it as a single ticket because (a) the work is interdependent — agent-backend's switch to the unified tool + mcp-ts retirement + vultiagent-app label cleanup all release the same dead-code wave; (b) splitting risks a half-state where mcp-ts retires tools agent-backend still calls; (c) the volume per repo is small enough that one engineer can drive all three in 1-2 days.

**PR train (merge in this order):**

1. **agent-backend (PR 1)** — migrate `gatherBalances` to use `get_unified_balance`, delete `chainToBalanceTool` / `chainCanonicalNames` / `parseBalanceResult` / `fetchChainBalance` per-chain logic, delete the 21 droplist entries + comment block, update ~140 test-fixture references. ~120 LOC delete + test fixture updates. **Mechanical.**
2. **mcp-ts (PR 2)** — delete the 5 balance tool files + their registrations + parity tests. ~815 LOC delete. **Mechanical.** Cannot merge before PR 1.
3. **vultiagent-app (PR 3)** — delete the hardcoded per-chain entries in `coalesceParts.ts`, `toolLabels.ts`, `chipPresets.ts`. ~30 LOC delete. **Mechanical.** Order-independent w.r.t. PR 1 / PR 2 (it's UI-only cosmetic cleanup) — can ship before, with, or after.

**Test verification:** for each PR, run the package's test suite. agent-backend: `go test ./internal/service/agent/...`. mcp-ts: `pnpm test`. vultiagent-app: `yarn jest`. Plus a manual end-to-end balance-fetch smoke for native + token on at least 3 chain families (EVM/cosmos/utxo) before PR 1 merges.

### Difficulty / Risk / LOC summary

| Dim | Estimate | Notes |
|---|---|---|
| **Net LOC** | **~935 delete, ~30 additive cleanup** | Verified via subagent + grep against current code |
| **Difficulty** | **MODERATE — 1-2 days for one engineer** | Volume is low-complexity but high; ~140 test-fixture touches dominate effort |
| **Risk** | **medium-low** | No external dispatch consumers, but balance flow is fund-safety-adjacent; merge ordering is non-negotiable |
| **Blast radius** | 3 repos | Balance flow only; no signing / swap / send paths touched |
| **Side-effects** | Closes latent `get_dot_balance` dead route. mt-wjmy already updated mcp-ts CLAUDE.md to flag per-chain tools as "migrated incrementally" — this ticket completes that migration |

## Constraints

- **Hard merge order: agent-backend PR 1 must merge before mcp-ts PR 2.** If mcp-ts retires the per-chain tools first, agent-backend's `mcpProvider.CallTool(ctx, "evm_get_balance", ...)` at `data_tools.go:373` fails for every balance fan-out. No flag / dual-emit acceptable per pre-launch state-disposability convention.
- **Depends on `mt-wjmy` merge + production bake.** The unified tool needs reps in production via the validator JIT path before it becomes the sole balance surface. Recommend ~2 weeks bake. This ticket should not start until then.
- Validator MUST continue to honor fail-open contracts established in mt-wjmy (`req.Context == nil`, MCP unreachable). Don't tighten safety contracts in a cleanup ticket.
- `chainTicker()` (`data_tools.go:638`) behavior MUST be preserved — multi-denom fallback still needs the ticker lookup. Implementation approach is in Open Questions.
- Don't touch unrelated tools in mcp-ts — fee, gas-estimation, swap-quote tools all remain (they have their own consumers and aren't part of this cleanup).
- vultiagent-app PR is UI cosmetic only; do not change agent-backend wire shape from the app side.

## Assumptions

- Per-chain MCP balance tools have NO external dispatch consumers outside agent-backend (verified by grep across vultisig-sdk/clients, vultiserver, station, vultiagent-app dispatch paths). If a non-checked-out repo (vultisig-android/ios/windows) calls these by name, retiring breaks them — verify against those repos before mcp-ts PR 2 merges.
- mt-wjmy's `get_unified_balance` returns a stable, parseable shape that subsumes everything `parseBalanceResult` currently handles. Spot-check before starting.
- vultiagent-app's `coalesceParts.ts` thinking-step labels are purely cosmetic; removing them does not break any chip / streaming logic.
- ~2-week production bake on mt-wjmy is sufficient signal. If the unified tool surfaces issues during bake, this ticket pauses.

## Non-goals

- Migrating fee / gas-estimation / swap-quote MCP tools to consume SDK. Same architectural smell, separate strategic ticket.
- Adding new chain coverage. Anything SDK doesn't already cover stays out of scope.
- Changing `get_balances` LLM-facing tool surface (schema, behavior). It now wraps the unified tool internally; the model sees no change.
- Changing the validator's fail-open contracts.
- Touching `as_of`-as-fetch-time semantics nuance (Neo's mcp-ts#76 follow-up). Separate ticket.
- USD-value enrichment (Neo's deferred finding on agent-backend#236). Separate ticket.

## Dead Ends

- **Two separate cleanup tickets (one per follow-up).** Considered but rejected: the agent-backend migration and mcp-ts retirement are interdependent (hard merge order), and splitting creates a half-state risk for no benefit. One engineer can drive all three PRs in 1-2 days.
- **Feature flag / dual-emit period for mcp-ts retirement.** Rejected per pre-launch state-disposability convention. Hard cut, no shim.
- **Keep the per-chain balance tools as deprecated for a release.** No external consumers identified; deprecation period adds complexity for zero benefit.

## Open Questions

- **Bake duration on mt-wjmy.** ~2 weeks is the proposed signal threshold — confirm. Should bake include at least one week of non-trivial swap-execution traffic (the validator JIT path's primary exercise).
- **`chainTicker()` rewire**: keep a slim ticker-only map, or thread ticker through unified tool response? Plan-mode call; latter is cleaner but couples the new tool's shape to a backend concern.
- **vultiagent-app PR ordering**: ship before, with, or after the agent-backend PR? UI-only so order-independent, but bundling with the agent-backend PR keeps the cleanup one cohesive change.
- **Native repos verification (Android / iOS / Windows)**: not checked out locally. Before mcp-ts PR 2 merges, grep those repos for per-chain balance tool name strings to confirm zero external dispatch consumers.
