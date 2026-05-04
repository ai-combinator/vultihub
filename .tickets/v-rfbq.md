---
id: v-rfbq
status: open
deps: []
links: []
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
Two-part minimal fix (don't tackle full parallel fan-out here — that's a separate post-v1 ticket):

**(a) Always pass `to_address` when the model has a contract address.** When `search_token` has resolved a contract address for the destination token, the model should pass it as `to_address` to `execute_swap`, not just the symbol. Prompt rule + possibly server-side enrichment that auto-fills `to_address` from `MessageContext.Coins` when the symbol matches a tracked-coin contract.

**(b) Aggregate provider errors in the surfaced message.** Instead of returning the last single provider's error verbatim, return an aggregated string naming each provider attempted: `"no route — THORChain: pool not found. KyberSwap: ..."`. Model then knows the failure was tried across multiple providers, not just one.

Together these close the headline conversation without the deeper rate-comparison / fan-out work.

## Constraints
- Must remain a non-breaking change to `execute_swap`'s outer interface — new args/return fields fine, removed/renamed not.
- Don't surface so many provider errors that the LLM gets confused; cap at the providers actually attempted, with each line short.

## Non-goals
- Parallel fan-out + best-rate selection. Separate post-v1 ticket; depends on affiliate-fee policy.
- Changing the existing fetcher priority order or `shouldPreferGeneralSwap` heuristic.

## Dead Ends
- Pure prompt fix without code change. Tried in `prompt.go:285` already ("report verbatim, don't invent constraints") — model violated 3× in `ec56bde0`. Code-side change is necessary.

## Open Questions
- For (a): server-side auto-fill of `to_address` from `MessageContext.Coins` vs prompt rule alone. Server-side is safer.
- For (b): error format — joined string or structured `{ provider, error }[]`? Structured would let the validator distinguish per-provider failures cleanly but the LLM consumes a string anyway.
- Same bug class hits Rujira (conv `8ae9bf54`): `"Symbol 'RUJI' not found on THORChain"` is THORChain-only. Does (b) cover Rujira too, or is that handled by the "Drop FIN tools from chat" ticket?
