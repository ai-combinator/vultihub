---
id: v-usxh
status: open
deps: []
links: [v-amgl]
created: 2026-05-06T22:08:59Z
type: bug
priority: 0
assignee: Jibles
external-ref: https://github.com/vultisig/vultiagent-app/issues/428
---
# Price claims require grounded market-data evidence

## Problem
The agent can quote token spot prices, 24h changes, and market caps from model priors instead of live market-data tools. Solved means final assistant text cannot show market price claims unless the same turn has explicit market-data evidence, and the prompt/tool descriptions make the intended behavior natural without adding caps-lock rule spam.

## Background
### Findings
- Original bug ticket: `v-amgl` / GitHub issue https://github.com/vultisig/vultiagent-app/issues/428.
- Confirmed bad export examples in `/home/sean/Downloads/2026-05-01-debug-export/2026-05-01-debug-export/`:
  - `6b9039ee`: user asked “What is the price of Bitcoin?” assistant replied “Bitcoin is trading at **$76,228**...” with zero tool calls.
  - `48f6a28b`: assistant converted portfolio USD to “current BTC price of $65,849/BTC” without `get_price` / `get_market_price` evidence.
  - `ec56bde0`: assistant replied “AAVE is trading at **$94.46**, down 1.64%...” before the later `search_token`; `search_token` returns metadata only, not price/24h change/market cap.
- `agent-backend/internal/service/agent/response_validator.go` only checks price consistency after an existing price tool record. If no price tool was called, it emits nothing.
- `agent-backend/internal/service/agent/agent.go` calls `ValidateResponse` only when `len(toolResults) > 0`, so “price claim + no tools” is structurally invisible to that path.
- `agent-backend/internal/service/agent/response_validator_test.go` currently codifies `response: "The current price of ETH is $3,500.00."` + `toolResults: nil` as no-op.
- The WS3 validator pipeline is wired at the terminal-turn seam, but there is no extractor for “market-price claim without market-data evidence”.
- `agent-backend/internal/service/agent/prompt.go` has nearby guidance that partially conflicts:
  - Fiat execution tools accept `$10`, `%`, and `max` directly and the LLM should not call `get_market_price` just to resolve base units for `execute_*`.
  - The next section says prices are stale and to call `get_market_price` for every asset.
- `execute_*` quote outputs can ground transaction-specific facts they return: user fiat amount, amount_in, expected/minimum output, route/provider, and execution rate. They do not ground general spot-market claims like 24h change or market cap unless those exact fields are returned by the tool.
- `search_token` returns token metadata: id, name, symbol, deployments, market_cap_rank. It is not a price oracle.
- Relevant prompt-engineering guidance from `guide:prompt-engineer`: keep prompt changes minimal, explain why, frame positively, prefer examples over long rule lists, and make tool descriptions non-overlapping and semantically precise.

### External Context
- Standard agent practice is layered: prompt/tool contracts steer behavior, output guardrails enforce evidence, and evals check both final answer and tool trajectory.
- OpenAI function/tool loop: model requests tool, app executes, tool result is passed back, model answers from the result.
- OpenAI Agents SDK has output guardrails for final-answer validation.
- Anthropic guidance emphasizes carefully written tool descriptions and tool-use workflows.
- Google ADK evals support checking tool-use trajectory and groundedness/hallucination criteria.

## Current Thinking
Where we landed: implement a layered fix, but keep prompt edits thoughtful and targeted.

1. Add a terminal-turn guardrail/extractor for market-data claims without market-data evidence.
   - Detect final text claims that look like live market data: “trading at”, “currently at”, “current price”, `BTC ... $76,228`, 24h change percentages, and market cap phrasing.
   - Treat valid evidence as successful `get_market_price` / `get_price` records for the asset in the current turn.
   - Consider `execute_*` results valid only for transaction-specific quote facts they actually return (`amount_in`, `expected_output`, `minimum_output`, `rate`, `quote_summary`, provider/route). Do not let them authorize 24h change or market cap narration.
   - A first implementation may be same-turn only. The original ticket mentions “conversation window” as desirable, but same-turn is enough to close the confirmed regressions and avoids stale-price ambiguity.

2. Decide enforcement behavior deliberately.
   - Preferred: retry once with a corrective instruction/tool requirement so the model calls `get_market_price`, then answer from the tool result.
   - Acceptable initial fallback if retry plumbing is not available: block/replace with a safe message such as “I need to fetch the live price before quoting it.”
   - Logging-only is not sufficient for this P0 bug unless the global validator enforcement path is still gated off; in that case the ticket should include enabling this guard through the existing reliability path or a targeted direct enforcement path.

3. Make prompt changes by subtraction and clarification, not caps-lock accumulation.
   - Rework the Fiat Amounts and Fiat Values sections so they no longer read as conflicting instructions.
   - Preserve the load-bearing rule that `execute_*` accepts fiat strings directly and the model must not pre-convert `$10` into token units.
   - Add a short “what quote tools ground” distinction: execution quote tools ground the transaction preview they return; market-data tools ground spot price, 24h change, and market cap.
   - Include 2-3 compact examples rather than a new wall of prohibitions.

Suggested prompt shape, not exact final prose:

```text
## Fiat Amounts And Market Prices

For transaction tools, pass fiat amounts directly: `amount: "$10"`, `"50%"`, or `"max"`. The tool resolves the executable amount and returns the quote details the user should review. Do not pre-convert fiat into token units yourself.

Use the returned quote fields for transaction-specific narration: input amount, estimated output, minimum output, provider/route, and quoted execution rate. For spot-market facts such as “BTC is trading at $X”, 24h change, or market cap, call `get_market_price` and answer from that result.

<example>
User: “Swap $10 of ETH to AAVE”
Assistant action: call `execute_swap` with `amount: "$10"`; narrate the returned quote details. Do not add AAVE 24h change or market cap unless a market-price tool also returned them.
</example>

<example>
User: “What is the price of Bitcoin?”
Assistant action: call `get_market_price` for BTC, then answer from the returned price data.
</example>
```

4. Tighten tool descriptions so the model does not infer false authority.
   - `get_market_price`: describe as the source for spot price / 24h change / market cap if those fields are returned.
   - `search_token`: describe as token metadata only; it does not return current price, 24h change, or market cap.
   - `execute_*` descriptions or surrounding prompt: describe them as quote/execution tools, not market-data tools.

5. Add eval/test coverage before or with the fix.
   - Unit tests for the new guard/extractor:
     - BTC price claim with no tool results -> finding.
     - AAVE price/24h/market-cap claim after only `search_token` -> finding.
     - Price claim grounded by `get_market_price`/`get_price` -> no finding.
     - Swap quote narration from `execute_swap` output -> no finding for returned transaction quote facts.
     - Market cap / 24h change after only `execute_swap` -> finding.
   - Prompt/trajectory evals if the repo has a suitable harness:
     - “What is the price of Bitcoin?” must call `get_market_price` before answering.
     - “Swap $10 of ETH to AAVE” must pass `"$10"` into `execute_swap` and not pre-convert to ETH units.
     - `search_token`-only turns must not narrate price/24h/market-cap facts.

## Constraints
- Do not break fiat execution safety: `$10` must remain `$10` in `execute_*` calls; the LLM must not pre-convert fiat into token units.
- Do not treat `search_token.market_cap_rank` as market cap or price evidence.
- Do not add broad, noisy prompt spam. The prompt adjustment should remove conflict and add examples/precise semantics.
- Server-side enforcement must live in backend/reliability path, not client-side rendering. Client-side hiding is not a sufficient source of truth.
- Keep scope to market-price claims. Do not generalize this ticket into every hallucination class.

## Assumptions
- `get_market_price` / `get_price` is the canonical live market-data source for spot prices.
- Same-turn evidence is acceptable for v1. If a recent-conversation-window cache is needed, that should be explicit and should include freshness semantics.
- `execute_*` result shapes consistently expose quote labels such as `amount_in`, `expected_output`, `minimum_output`, `rate`, and provider/route for transaction preview narration.
- The existing validator pipeline or response validation path can be extended without a large architecture change.

## Non-goals
- Live price streaming.
- General fact-checking for contracts, tx hashes, token deployments, balances, or sports/Polymarket odds.
- Changing the swap/execute tool contract unless needed to make existing returned quote fields clearer.
- Adding a new market-data provider.

## Dead Ends
- Prompt-only fix: ruled out because confirmed examples already violated existing prompt guidance. Prompt changes are still useful to remove conflict and improve tool selection, but they are not enforcement.
- Client-side pre-render sanitizer: ruled out as primary fix because the backend is the source of truth and the response may be stored, streamed, evaluated, or consumed outside one client.
- Treating every dollar amount as a price: too broad. Transaction amounts (`$10 swap`, fees, portfolio totals, betting stakes) are legitimate without being spot-price claims. The detector needs market-claim context.

## Open Questions
- Should the first implementation be same-turn evidence only, or should it accept recent same-conversation market-data calls with an explicit freshness window?
- Should guardrail action be retry-first or block/replace-first under current reliability flag rollout?
- What exact category name should the validator use: `ungrounded_price_claim`, `market_price_unverified`, or fit into an existing taxonomy?
- Should `execute_*` tools return a typed `quote_facts` envelope to make quote-grounding easier, or is parsing existing labels enough for v1?
