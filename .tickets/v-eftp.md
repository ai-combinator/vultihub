---
id: v-eftp
status: in_progress
deps: []
links: [v-rfbq]
created: 2026-05-03T22:44:35Z
type: bug
priority: 0
assignee: Jibles
---
# execute_swap: provider-attributed all-fail quote errors

## Problem
When every eligible `execute_swap` quote provider fails, the SDK currently throws a single provider-order error. mcp-ts wraps that as `execute_swap: quote failed: <provider message>`, and the LLM can overgeneralize one provider-specific failure into "no liquidity exists anywhere."

This is the remaining work after the broader fan-out/best-rate ticket was partially completed. The aim is narrow: make all-provider failure errors explicit, short, and provider-attributed.

## Background
- `vultisig-sdk#356` already changed `findSwapQuote` to `Promise.allSettled` across eligible providers and select the best successful quote by comparable destination output. This means a failing provider no longer hides a succeeding provider.
- The current SDK test explicitly pins the remaining bad behavior: `packages/core/chain/swap/quote/findSwapQuote.selection.test.ts` has "when all providers fail, propagates the first fetcher-order rejection".
- The original ETH->VULT misroute has an adjacent mcp-ts guard merged in `vultisig/mcp-ts#80` on 2026-05-04. It catches `to_chain=THORChain` plus EVM-shaped `to_address` / pool-id `to_symbol` and directs the model to retry with `to_chain=<fromChain>`.
- `v-rfbq` / GH #429 is the bug driver for the user-visible "no liquidity anywhere" hallucination.

## Current Thinking
Add provider labels to `findSwapQuote` fetchers and replace the all-rejected loop with an aggregate error such as:

`No swap route found after trying: KyberSwap: <short reason>; 1inch: <short reason>; LiFi: <short reason>; THORChain: <short reason>; MayaChain: <short reason>.`

Keep this as a plain thrown `Error` for now so the external interface remains unchanged. The mcp-ts wrapper will continue surfacing it as `execute_swap: quote failed: ...`.

Preserve the current dust-threshold special case: if any provider fails with the dust threshold signal, keep returning `Swap amount too small. Please increase the amount to proceed.`

## Constraints
- Do not change `execute_swap` or `findSwapQuote` call signatures.
- Keep provider reasons short enough for an LLM and a card error. Cap or truncate noisy API dumps.
- Only include providers actually attempted.
- Do not add more provider calls. Fan-out already exists.

## Non-goals
- Best-net-output comparator beyond what `vultisig-sdk#356` already shipped.
- Gas / fee normalization across providers.
- Affiliate-aware tie-breaking changes.
- New `to_address` enrichment in agent-backend or mcp-ts. That can be a follow-up if the all-fail error remains insufficient.
- Replacing or renaming `execute_swap`.

## Acceptance Criteria
- [ ] `findSwapQuote.selection.test.ts` no longer expects a first provider-order rejection when all providers fail.
- [ ] New/updated SDK test asserts all-fail error names every provider actually attempted.
- [ ] Error messages include friendly provider names: `KyberSwap`, `1inch`, `LiFi`, `THORChain`, `MayaChain`.
- [ ] Error reason text is sanitized/truncated so a single provider cannot dump a huge API response into the LLM.
- [ ] Existing dust-threshold tests still pass unchanged.
- [ ] Existing successful-provider tests still pass: a failing provider must not hide a succeeding one.

## Open Questions
- Joined string vs custom error class. Default to joined string unless implementation gets cleaner with a tiny internal class.
- Truncation length: start around 160 chars/provider unless surrounding patterns suggest another limit.

## Notes

**2026-05-05T05:33:00Z**

migrated to GH issue: https://github.com/vultisig/vultiagent-app/issues/436

**2026-05-05T05:34:06Z**

kept open locally — GH issue is a duplicate, local remains canonical scratch

**2026-05-06T22:08:00Z**

Scope stripped after root-cause pass for `v-rfbq` / GH #429. `vultisig-sdk#356` already delivered parallel all-settled quote selection and best successful quote by comparable destination output. `mcp-ts#80` already addresses the headline ETH->VULT same-chain EVM token misrouted to THORChain. Remaining actionable work here is only provider-attributed all-fail quote errors.
