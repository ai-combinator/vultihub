---
id: v-eftp
status: open
deps: []
links: []
created: 2026-05-03T22:44:35Z
type: feature
priority: 1
assignee: Jibles
---
# execute_swap parallel fan-out + best-rate selection

## Problem
`execute_swap` uses sequential fallback (`asyncFallbackChain`) across providers — first success wins, not best rate. Two consequences: (a) users get whatever rate the first-attempted provider returned, even if a downstream provider had a tighter spread; (b) errors from individual providers leak the underlying provider's name and shape, causing the LLM to misinterpret single-provider failures as market-wide. The minimal v1 fix (separate ticket: pass `to_address` + aggregate errors) closes the headline UX bug; this ticket is the structural fix.

Reference pattern: `shapeshift-agentic/apps/agentic-server/src/tools/initiateSwap.ts:52-99` — `Promise.all` across providers, filter successes, pick best by output amount, aggregate errors when all fail, surface selected provider to the LLM.

## Background
### Findings
- Current behavior: `vultisig-sdk/packages/core/chain/swap/quote/findSwapQuote.ts:159-167` uses `asyncFallbackChain` (`packages/lib/utils/promise/asyncFallbackChain/index.ts:3-20`) — try one, fail, try next, throw last error.
- Providers wired: KyberSwap, 1inch, LiFi (general); THORChain, MayaChain (native).
- For EVM→EVM with contract address: general-first ordering; otherwise native-first via `shouldPreferGeneralSwap` (lines 155-157).
- Quote shapes are NOT normalized today:
  - Native: `{ expected_amount_out, fees: { total, total_bps } }`
  - General: `{ dstAmount, provider, tx: { evm, solana }, ... }` (provider ∈ kyber|li.fi|1inch)
  Direct numeric comparison requires netting fees + gas, which differ per provider:
  - THORChain Ethereum inbound: gas-free (built into deposit), 30-150 bps protocol fee
  - 1inch on Ethereum: `dstAmount` is pre-gas; gas can be $5-20+
  - LiFi cross-chain: $5-50 bridge fees, embedded in quote
  - KyberSwap: typically $0.50-5 gas
- Subagent effort estimate: 140-170 LOC across 3 files, 2-3 days focused work + 1 day discovery/integration testing.
- Test coverage to update: `findSwapQuote.kyber.test.ts` (rewrite to mock all 5 providers), `getNativeSwapQuote.test.ts` (add parallel poll), new fan-out test file. ~100+ LOC of test work.
- KyberSwap affiliate config (`kyberSwapAffiliateConfig.source`) ties query volume to Vultisig's account — parallel fan-out 5×s the call rate; metering needs verifying.

### External Context
- ShapeShift reference: `/home/sean/Repos/shapeshift-agentic/apps/agentic-server/src/tools/initiateSwap.ts`.

## Current Thinking
Port ShapeShift's pattern with three Vultisig-specific adaptations:

1. **Provider-eligibility filter before fan-out**: filter the provider list based on swap pairing first (e.g. THORChain only for chains with THORChain inbound vaults; LiFi for cross-chain; etc.) — don't fan out 5× when only 2 are eligible.
2. **Affiliate-aware prioritization in the comparator**: when multiple providers return successful quotes, prioritize affiliate-enabled ones (KyberSwap, 1inch, LiFi) over THORChain in tiebreaks. Preserves the business intent of `shouldPreferGeneralSwap` without the rigid sequential ordering.
3. **Build a normalization layer** (`comparators.ts`, ~30-50 LOC) that nets out gas + protocol fees + affiliate fees so quotes are comparable. Need gas estimation per general provider — adds latency to the parallel fan-out.

When all providers fail, return aggregated error message (`"no route — KyberSwap: ..., 1inch: ..., LiFi: ..."`) so the LLM has full visibility.

## Constraints
- Filter eligible providers per swap pairing — don't blindly fan out to all 5.
- Affiliate-enabled providers (KyberSwap, 1inch, LiFi) prioritized in tiebreaks. Affiliate revenue is real and must not be lost to flat best-rate selection.
- Latency tradeoff acknowledged — Promise.all waits for slowest; net-out is on slowest-provider path. MCP timeout (likely 10s) must accommodate.
- Don't break the existing `execute_swap` external interface — quote envelope shape is `ExecutePrepResult`; this is internal to `findSwapQuote`.

## Assumptions
- Vultisig's relay can absorb 5× quote-call volume per swap without quota issues. **Verify before deployment** — KyberSwap source-account metering and LiFi SDK rate limits especially.
- Affiliate fees configured uniformly enough across general providers that a single bps adjustment in the comparator captures the business preference. If affiliate bps vary materially, tiebreak logic gets more complex.

## Non-goals
- Replacing or renaming the `execute_swap` MCP-tool surface — change scoped to `findSwapQuote` internals.
- Changing per-provider quote APIs themselves.
- Rebuilding `asyncFallbackChain` for other call sites (only `findSwapQuote` uses it for swap routing).

## Dead Ends
- Naive `parseFloat(buyAmount) > parseFloat(buyAmount)` (ShapeShift's literal pattern). Reason: Vultisig's heterogeneous mix of native + general providers has wildly different fee structures; comparing pre-gas amounts unfairly favors high-gas providers.
- Pure prompt fix to teach the model "try other providers if one fails". Sequential fallback already swallows the other failures, so the model never sees them. Plus that's the v1 minimal fix's job, not this one.

## Open Questions
- Affiliate-bps adjustment in the comparator: fixed bps per provider, or read from each provider's own fee config? Latter is cleaner but more wiring.
- Gas estimation for parallel quotes — fetch live or estimate from cached gas oracle? Live adds an RPC call; cached risks staleness.
- Per-provider error visibility — joined string or structured `{ provider, error }[]`? MCP tool consumes a string but a structured shape would help downstream.
- Cross-PR coordination with the v1 minimal fix ticket — this ticket assumes that one has shipped and aggregated-error format already exists.
