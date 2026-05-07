---
id: v-rjfc
status: closed
deps: []
links: []
created: 2026-05-03T22:41:05Z
type: feature
priority: 0
assignee: Jibles
---
# Terra v2 via execute_swap — Skip integration end-to-end

## Problem
Terra v2 (LUNA / phoenix-1) swap support is release-critical. Currently routed to `build_skip_swap` as a separate escape-hatch tool. Felix's 2026-05-01 debug-export has 0 Skip calls (success or fail) — the path is plausibly broken since nobody pointed a Terra test at it. Goal: pull Terra into `execute_swap` so it works through the same tool as the rest of the swap surface, AND validate end-to-end before release.

## Background
### Findings
- `build_skip_swap` is a standalone tool tagged in the prompt's "Swap Tool Selection" matrix (`agent-backend/internal/service/agent/prompt.go:67-78`).
- Polymorphic tx envelope shipped on backend (commit `23d56bdf`, ~1 week ago). App-side dispatch refactor in vultiagent-app PR #320 (`parseServerTx canonical envelope dispatch`, closes `v-uneu`) — currently OPEN, not merged.
- `execute_*` tools return `ExecutePrepResult` with discriminator `txArgs.tx_encoding ∈ {'evm','utxo-psbt','cosmos-msg','solana-tx',...}` (`mcp-ts/src/tools/execute/_base.ts:26-81`). The `cosmos-msg` family covers Cosmos-SDK message bodies including `wasm_execute`.
- App-side parsers exist for all 9 chain families in `vultiagent-app/src/features/agent/lib/executePrep.ts:69-316` (PR #320). Cosmos-msg branch validates `tx.messages/body/tx_type or recipient` (lines 273-300).
- Skip Go covers `phoenix-1` (Terra v2 / LUNA) and `columbus-5` (Terra Classic / LUNC) — verified via `api.skip.build/v2/info/chains`. Multi-hop routing built in. REST + TS SDK.
- Vultisig is already a Skip integrator via `build_skip_swap`. Refactor target: emit `ExecutePrepResult` shape from the Skip builder so it folds into the `execute_swap` family.

### External Context
- Skip Go docs: https://docs.skip.build/go
- Chain registry: https://api.skip.build/v2/info/chains

## Current Thinking
Two halves, both required for release:

**Half A — Skip into execute_swap:**
- Refactor the Skip builder (currently exposed via `build_skip_swap`) to emit `ExecutePrepResult` with `tx_encoding: 'cosmos-msg'`.
- Add Terra v2 routing inside `findSwapQuote` (or wherever execute_swap fans out) so Terra source/destination hits Skip.
- Drop the standalone `build_skip_swap` tool from the agent's view; keep underlying Skip builder for direct call paths.
- Remove Terra carve-out rules from `prompt.go:67-82` once unified path works.

**Half B — Smoke test:**
- End-to-end swap: e.g. "swap 5 USDC on Base for LUNA". Confirm `execute_swap` returns Cosmos-SDK envelope, app dispatches via `parseServerTx`, signs, broadcasts, lands on Terra. Same for reverse direction.
- Don't assume the Skip path works just because the prompt routes to it — test it.

## Constraints
- PR #320 (app-side parsers) MUST land before this — the cosmos-msg parser is the dependency.
- Skip's tx envelope must conform to `ExecutePrepResult` discriminator shape; don't invent a parallel envelope.
- Backwards compat: existing `build_skip_swap` callers shouldn't break during transition. Keep-then-deprecate.
- Astroport in-chain Terra v2 swap (`build_astroport_swap`) is a separate question — out of scope unless trivial to fold in.

## Non-goals
- Astroport in-chain Terra v2 swaps. Separate ticket if it doesn't fall out for free.
- Cosmos-SDK chains beyond Terra v2 / Terra Classic (Osmosis, Cosmos Hub, etc.) — not release-critical.

## Dead Ends
- Keeping `build_skip_swap` as a separate tool indefinitely. Reason: split-tool surface is what bit Felix on Rujira FIN; one tool for swaps is the design intent.

## Open Questions
- Does the Skip API return a single signable tx or a multi-message bundle? If multi-message, does our Cosmos-SDK signer handle that, or do we need stepper config?
- Does Terra v2 LUNA need fee in LUNA or another denom? Pre-flight context implications.
- Migration path for `build_skip_swap`: deprecate-then-remove, or hard-cut?

## Notes

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/mcp-ts#78 + vultisig/agent-backend#272 + vultisig/vultiagent-app#387 (DRAFT)
