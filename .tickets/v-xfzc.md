---
id: v-xfzc
status: closed
deps: []
links: []
created: 2026-05-03T22:43:18Z
type: task
priority: 0
assignee: Jibles
---
# Polymarket order submission fails with 'Invalid order payload'

## Problem
Polymarket order submissions fail repeatedly with `"Invalid order payload"` even after a fresh sign cycle. Multiple convs in Felix's 2026-05-01 dump:

- `48f6a28b` (NBA Championship Bet): User signs $3 bet on OKC Thunder twice. Both submissions fail. First error: "insufficient USDC.e balance. You have $7.99, but need ~$3.30+ (order + fees)" — but $7.99 > $3.30, math is suspect.
- `b41c2933` (NBA Finals Bet - Balance Issue): Same pattern, $3 USDC bet, two submissions, both blocked on insufficient-balance check.
- `8c4bc43e` (Bet on Fed Chair Warsh): User has $14.88 USDC.e — well above bet size. Two submission attempts both fail with `"Invalid order payload"`. Agent attributes it to "market price shifted significantly between building and signing" or "Order size is too small (below exchange minimum)".

This is an investigation, not a fix. Output of this ticket is a decision: fix pre-release OR hide Polymarket from LLM until post-release. Investigation must be done pre-release; fix work is tracked separately as post-v1.

## Background
### Findings
- Separate but related Polymarket bug — approval 3/5 dies on `tx_type: "evm_approve"` rejection — already has draft PR #240 (`fix(agent/prompt): build_custom_tx tx_type whitelist`). NOT this ticket; that one is on track.
- The "Invalid order payload" error comes from `polymarket_submit_order` MCP tool. Error string from CLOB API: `"mcp tool polymarket_submit_order error: ..."` (truncated in dump).
- Insufficient-balance math is suspect — conv `48f6a28b` says "$7.99 vs need ~$3.30" but rejects, and conv `8c4bc43e` user has $14.88 well above $1.88 bet and still fails.
- After-position check via `read_evm_contract` action also fails (`outputTypes.map is not a function` — separate `actions[].type` validator ticket addresses that).

### External Context
- Polymarket CLOB API: https://docs.polymarket.com/
- "Invalid order payload" is a Polymarket-side rejection — could be signature issue, salt collision, expired order, mismatched market state, fee mismatch, etc.

## Current Thinking
This is investigation, not fix. Output is a decision:

1. **Reproduce** — run the same flow against current main; capture the actual CLOB error response (the dump has it truncated).
2. **Root-cause** — figure out which plausible cause is biting (signature mismatch? stale market price? tick-size violation? fee-config drift? balance-check math bug?).
3. **Decide**:
   - If trivial (e.g. balance-math bug, one-line signing fix): land the fix pre-release.
   - If complex (CLOB integration drift, requires Polymarket support engagement): hide Polymarket from the LLM for release. Fix tracked as post-v1.

The "hide Polymarket from LLM" path is the same shape as the Rujira FIN drop — filter Polymarket tools from agent's visible toolset, add a "coming soon" prompt redirect.

## Constraints
- Investigation must complete pre-release — output gates the hide-vs-fix decision.
- Don't ship the Polymarket flow as-is. Either it works for release or it's hidden.

## Non-goals
- Fixing the bug. Fix is a separate ticket (post-v1, depends on this).
- Re-engaging Polymarket on partnership / fee terms. Engineering investigation only.

## Open Questions
- Is the failure deterministic on a clean test bet, or only intermittent?
- Does the failure happen on every market, or only on specific ones (NBA-related vs Fed Chair vs others)?
- Is there a CLOB sandbox / testnet, or are we burning real USDC.e on test bets?

## Notes

**2026-05-05T02:40:49Z**

killed: Polymarket cut from v1 scope
