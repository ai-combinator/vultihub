---
id: v-mjfl
status: closed
deps: []
links: [ab-dflp, mcp-ts#49]
created: 2026-04-27T08:09:38Z
type: task
priority: 2
assignee: Jibles
---
# agent-backend: extend knownCalldataToolPrefixes to cover execute_* family

## Why

mcp-ts PR #49 (ticket ab-dflp) hard-cuts the legacy `build_swap_tx` / `build_evm_tx` tools and ports `quest_metadata` emission to their successors `execute_swap` and `execute_contract_call`. The wire shape is identical and the data flows from the MCP correctly.

But the **agent-backend scheduler's extraction won't see it.** `internal/service/agent/headless.go` has:

```go
var knownCalldataToolPrefixes = []string{"build_", "sign_"}
var knownCalldataToolNames = map[string]bool{
    "erc20_approve":        true,
    "polymarket_place_bet": true,
    "polymarket_sign_bet":  true,
    "build_custom_tx":      true,
}
```

`execute_*` tools fall through both checks → `IsKnownCalldataToolName("execute_swap")` returns `false` → `extractBuildToolMetadata` (in `internal/service/scheduler/scheduler.go`) skips them → `agent_tx_proposals.quest_metadata` stays nil for any tx_proposal driven by an execute_* tool call.

Forward-compat contract says missing `quest_metadata` is "never an error" (per `internal/types/tx_proposal.go:55`), so nothing crashes — but Stage 3 quest tracking goes silent for the entire execute_* surface.

## What to do

1. **Predicate update** — `internal/service/agent/headless.go`:
   - Add `"execute_"` to `knownCalldataToolPrefixes`. (Cleanest: prefix-based covers all 5 current execute_* tools and any future ones.)
   - Alternative: enumerate `"execute_send"` / `"execute_swap"` / `"execute_contract_call"` / `"execute_lp_add"` / `"execute_lp_remove"` in `knownCalldataToolNames`. Less drift-resistant.

2. **Pin update** — `internal/service/agent/headless_test.go` `TestIsKnownCalldataToolName` cases:
   - Add `{"execute_send", true}`, `{"execute_swap", true}`, `{"execute_contract_call", true}`, `{"execute_lp_add", true}`, `{"execute_lp_remove", true}`

3. **Integration test fixtures** — `internal/service/scheduler/`:
   - `quest_metadata_integration_test.go` currently loads fixtures for `build_swap_tx_native_thor.json`, `build_swap_tx_same_chain_1inch.json`, `build_swap_tx_cross_chain_bridge.json`, `build_evm_tx_erc20_transfer.json`. Add equivalents for the execute_* shape:
     - `execute_swap_native_thor.json` (THOR ETH→BTC)
     - `execute_swap_same_chain_1inch.json` (LiFi/1inch ETH→USDC same-chain)
     - `execute_swap_cross_chain_bridge.json` (LiFi cross-chain ETH→USDC Arbitrum)
     - `execute_contract_call_erc20_approve.json` (USDC.approve)
   - Capture from a live mcp-ts run on `surgical/v-qjvi-execute-foundation` (or main once that PR lands). Compactness check passes (`txData["quest_metadata"]` reads from top-level JSON).
   - Update test cases' `tool_name` assertion to expect `execute_swap` / `execute_contract_call`.

4. **MarshalJSON serialization-seam scrub** — `internal/service/agent/headless.go` `isMarshalJSONScrubbable`: already takes the union of `isKnownCalldataToolName` + scheduled-preview names, so the predicate update at #1 automatically extends coverage. Verify the existing tests still pass.

## Acceptance criteria

- [ ] `IsKnownCalldataToolName("execute_swap")` returns true (and same for the other 4 execute_* tools)
- [ ] Stage 3 quest tracking for an `execute_swap`-driven `tx_proposal` writes a non-nil `agent_tx_proposals.quest_metadata` row to Postgres in a live local run
- [ ] `quest_metadata_integration_test.go` covers all 5 execute_* tool names with fixtures
- [ ] Headless serialization scrub on a tainted (breaker-tripped) turn drops `execute_*` tool logs the same way it drops `build_*`

## Coordination

- Land after mcp-ts PR #49 (which emits the field) merges to mcp-ts main. Until that merge, `execute_*` tool results have no `quest_metadata` to read regardless of the predicate.
- Talk to the Stage 3 owner before merge so they know the dark-period closes.

## Notes

PR #49 already documents this gap explicitly in its body and test plan ("Reviewer: confirm agent-backend knownCalldataToolPrefixes follow-up ticket is in flight before merge"). This ticket IS that follow-up.

The mcp-ts in-flight PR #50 (`feat/quest-metadata-amount-usd-fallback`) layers price-oracle USD fallback on top of build_swap_tx / build_evm_tx — both deleted by #49. PR #50 must rebase to target the new emission sites in `execute_swap` / `execute_contract_call` before it can merge. Not a blocker for this ticket; just note for sequencing.

**2026-05-04T23:09:10Z**

auto-closed: merged via vultisig/agent-backend#188
