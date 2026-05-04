---
id: v-amgl
status: open
deps: []
links: []
created: 2026-05-03T22:41:26Z
type: bug
priority: 0
assignee: Jibles
---
# Hallucinated price guardrail in agent responses

## Problem
Agent quotes token prices without calling `get_price` / `get_market_price`. Three confirmed cases in Felix's 2026-05-01 dump:
- BTC at $76,228 in conv `6b9039ee` (zero tool calls in conversation)
- BTC at $65,849 in conv `48f6a28b` (no get_price call)
- AAVE at $94.46 / -1.64% in conv `ec56bde0` (only `search_token` called — returns metadata, no price field)

For a crypto wallet, hallucinated prices are a non-starter — users will trust prices for trade decisions.

## Background
### Findings
- Existing prompt rule at `agent-backend/internal/service/agent/prompt.go:162` already says: *"Prices change constantly — your knowledge is stale. Call `get_market_price` for every asset. Always show live fiat values with balances."* Model is ignoring it.
- `search_token` returns metadata only (id, name, symbol, deployments, market_cap_rank). No price field. Confirmed via dump tool-calls output for conv `ec56bde0`.
- `get_price` is the canonical price tool. Successfully called many times in the dump for other tokens.
- Dump locations: `/home/sean/Downloads/2026-05-01-debug-export/2026-05-01-debug-export/conversations.jsonl` and `tool-calls.jsonl`.

## Current Thinking
Two-layer guardrail:
1. **Tighter prompt rule** — hard rule forbidding price quotes without a same-turn `get_price` call. Not just a hint.
2. **Validator on assistant messages** — if message contains a price quote pattern (`\$\d+(\.\d+)?` near a token symbol, or "trading at" / "currently at"), verify a `get_price` call was made for the named token in the conversation window. If not, re-prompt with a forced retry, or strip the price claim.

Validator catches what prompt rule can't enforce. Validator preferred long-term; prompt-only is the cheap version.

## Constraints
- Don't break legitimate cases where price is being quoted from a recent same-conversation `get_price` result still in context. Validator should look at conversation window, not just current turn.
- False-positive cost: stripping a legitimate price quote is annoying but safer than letting hallucinated ones through.

## Assumptions
- Validator can reliably tokenize "this assistant message is quoting a price for token X". Regex on `$XX.XX` near a token symbol is the obvious approach but may have edge cases.

## Non-goals
- Live price-streaming. Same-turn fresh fetch is enough.
- Generalizing to all hallucination classes (token contracts, tx hashes, etc.). One bug at a time.

## Open Questions
- Validator placement: server-side post-LLM, or client-side pre-render? Server-side is safer.
- Action on hit: re-prompt with forced `get_price`, or strip and append "[price elided]"?
- Strict vs loose match: do we flag "Bitcoin is around $65k" the same as "BTC is trading at $65,849"?
