---
id: v-rfbq
status: in_progress
deps: [v-eftp]
links: [v-eftp]
created: 2026-05-03T22:42:51Z
type: bug
priority: 0
assignee: Jibles
---
# Premature 'no liquidity' claims after single-provider failure

## Problem
Agent declares "no liquidity exists for this pair on any DEX" after `execute_swap` returns a single provider's failure. Felix's 2026-05-01 dump conv `ec56bde0`: ETH→VULT swap on Ethereum failed with `pool ETH.VULT-0XB788144DF611029C60B859DF47E79B7726C4DEBA doesn't exist` — agent told the user three times in a row that VULT had no liquidity anywhere, even suggesting "you may need to provide liquidity yourself on Uniswap." User: *"Thats just wrong. check lifi or kyber"* — model retried with explicit contract address and succeeded via Li.Fi.

## Background
### Findings
- `execute_swap` uses sequential fallback (`asyncFallbackChain`) across providers — KyberSwap, 1inch, LiFi, THORChain, MayaChain. Defined at `vultisig-sdk/packages/core/chain/swap/quote/findSwapQuote.ts:159-167`. First success wins; on all-fail, the last error is thrown.
- For EVM→EVM with `to_id` (contract address) present: general-first ordering: KyberSwap → 1inch → LiFi → THORChain → MayaChain.
- For EVM→EVM without `to_id`: native-first: THORChain → MayaChain → KyberSwap → 1inch → LiFi.
- The error string returned to the LLM is whichever provider's error happened to bubble up. THORChain's error format (`pool ETH.VULT-0X...`) is provider-specific but doesn't name the provider — model reads it as universal.
- Successful retry in dump used `to_address: "0xb788..."` explicitly. Without `to_address`, several providers can't resolve the symbol (low-cap tokens disambiguate via contract address, not symbol).
- Existing prompt rule (`agent-backend/internal/service/agent/prompt.go:285`): *"If a specific pair has no liquidity, the tool returns a clear 'no route' error — report that verbatim instead of inventing constraints."* — model violated it.

## Current Thinking
Root-cause pass narrowed this bug to the quote error surface. The exact ETH->VULT THORChain misroute has an adjacent mcp-ts guard merged in `mcp-ts#80`, and SDK parallel quote selection has landed in `vultisig-sdk#356`.

This ticket is now the user-visible bug driver. Implementation lives in `v-eftp`: provider-attributed all-fail quote errors.

Desired behavior: when every eligible provider fails, `execute_swap` surfaces a short aggregate naming each provider attempted and its individual reason. The model should see that multiple providers were attempted and must not infer "no liquidity anywhere" from one provider's error shape.

`to_address` enrichment is no longer in this ticket's implementation scope. Keep it as a possible follow-up only if aggregated all-fail errors do not close the observed behavior.

## Constraints
- Must remain a non-breaking change to `execute_swap`'s outer interface — new args/return fields fine, removed/renamed not.
- Don't surface so many provider errors that the LLM gets confused; cap at the providers actually attempted, with each line short.

## Non-goals
- Parallel fan-out + best-rate selection. Already partially delivered by `vultisig-sdk#356`; any deeper comparator work is separate.
- Changing the existing fetcher priority order or `shouldPreferGeneralSwap` heuristic.
- Backend or mcp-ts `to_address` enrichment.

## Dead Ends
- Pure prompt fix without code change. Tried in `prompt.go:285` already ("report verbatim, don't invent constraints") — model violated 3× in `ec56bde0`. Code-side change is necessary.

## Open Questions
- For (a): server-side auto-fill of `to_address` from `MessageContext.Coins` vs prompt rule alone. Server-side is safer.
- For (b): error format — joined string or structured `{ provider, error }[]`? Structured would let the validator distinguish per-provider failures cleanly but the LLM consumes a string anyway.
- Same bug class hits Rujira (conv `8ae9bf54`): `"Symbol 'RUJI' not found on THORChain"` is THORChain-only. Does (b) cover Rujira too, or is that handled by the "Drop FIN tools from chat" ticket?

## Notes

**2026-05-05T05:32:25Z**

migrated to GH issue: https://github.com/vultisig/vultiagent-app/issues/429

**2026-05-05T05:34:06Z**

kept open locally — GH issue is a duplicate, local remains canonical scratch

**2026-05-06T21:58:47Z**

Initial rootcause pass: claimed GH #429 and local ticket. Trace: app only forwards/renders AI SDK tool errorText; backend normalizes execute_swap args and forwards MCP textError; mcp-ts execute_swap resolves tokens then calls @vultisig/sdk findSwapQuote; SDK findSwapQuote now runs providers via Promise.allSettled (sdk#356) and picks best successful quote, but when all providers reject it intentionally throws the first fetcher-order rejection. That still surfaces a single provider-specific error as execute_swap: quote failed: <provider msg>, with no provider attribution/aggregate. Original ETH->VULT/THORChain misroute has an adjacent mcp-ts guard merged in #80 on 2026-05-04 that catches to_chain=THORChain plus EVM-shaped to_address/pool-id and directs retry with to_chain=Ethereum. Current unresolved system bug: all-fail path has no structured provider error aggregate, and backend does not enrich missing to_address from app MessageContext.Coins before MCP dispatch. Next validation: add failing tests around SDK all-provider rejection aggregation and backend/mcp-ts address enrichment path.

**2026-05-06T22:08:00Z**

Linked to `v-eftp` and stripped local implementation scope. `v-rfbq` remains the bug driver / acceptance context; `v-eftp` is the active implementation ticket for better all-provider failure errors.
