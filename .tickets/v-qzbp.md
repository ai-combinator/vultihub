---
id: v-qzbp
status: open
deps: []
links: []
created: 2026-05-07T06:21:19Z
type: task
priority: 0
assignee: Jibles
---
# Compact search and market tool outputs

## Problem
Search, market, and read/list tools can return large or raw third-party/registry payloads even though the model usually needs only a small deterministic choice set or summary. This creates outlier token-risk and makes the model reason over noisy data. Solved means high-volume read/search tools return compact, capped, deterministic LLM-visible outputs with rich details available through sidecars/refs when needed.

## Background
### Findings
- Overarching context from the token-cost investigation: the agent should be made cheaper and more reliable by shaping the LLM's environment, not by piling on more prompt warnings. The model should see fewer tools, shorter instructions, smaller results, and only fields/results it can actually use for the next decision.
- Core philosophy: tools should return decision-shaped outputs to the LLM, while rich raw/provider/UI payloads live in sidecars, refs, metadata, or app-facing payloads.
- Cost philosophy: this ticket is mostly outlier protection and model-clarity work. Normal path spend was dominated by prompt/tool context, but broad search/list payloads can still create expensive individual turns.
- Product philosophy: search/list tools should help the model choose safely from a small deterministic set, not force it to reason over raw provider/registry dumps.
- `v-giou` identified search/market tools as a major output-side contract cleanup area.
- Polymarket/search-style tools were flagged during token-cost investigation as plausible historical outlier sources, even though Polymarket is disabled for v1 under `v-tjwm`.
- `search_token` and market/price tools are core model choice helpers. They should provide enough data to choose correctly, but not dump full registry/provider payloads into the prompt.
- Balance/portfolio outputs can also become large for wallets with many assets/chains and should be considered if they naturally fit this policy.
- `v-hykj` should provide the shared result-envelope mechanism; this ticket defines concrete policies/summarizers for read/search/list tools.

## Current Thinking
This ticket is about tools that help the model choose, not signing/card transaction payloads.

Target output shape for list/search tools:

```json
{
  "status": "success",
  "query": "usdc",
  "items_returned": 5,
  "items_total": 42,
  "truncated": true,
  "choices": [
    { "ref": "token:1", "label": "USDC on Ethereum", "symbol": "USDC", "chain": "Ethereum", "contract": "0x...", "reason": "top market-cap exact symbol match" }
  ],
  "next_action": "Use the matching chain requested by the user, or ask if ambiguous."
}
```

Candidate policies:

- Token search: cap choices, rank exact symbol + requested chain + vault-held tokens first, use refs for full candidate metadata.
- Polymarket search/market/my: no raw API payloads; return ranked markets/positions/orders with ids, titles, outcomes, liquidity/volume/status, and continuation refs.
- Price lookup: structured, minimal result that is clearly market-data evidence and does not include irrelevant provider dump.
- Balance/portfolio: cap or summarize large holdings while preserving top assets and requested chains.
- Search/list continuation: deterministic ref or cursor behavior when the user asks for more.

Suggested phases:

1. Define shared read/search compaction policy and caps.
2. Apply first to `search_token` because it is core and high-frequency.
3. Apply to Polymarket read tools before any graduation from experimental.
4. Evaluate balances/portfolio and price tools for the same treatment.
5. Add token-budget probes for broad search queries and large result sets.

## Constraints
- Do not hide facts the model needs to choose correctly or answer truthfully.
- Capping must be deterministic and explain truncation/continuation.
- Preserve rich payloads for UI cards/debug where needed via sidecar/ref, not model transcript.
- Do not make Polymarket release-critical; this supports `v-tjwm`'s graduation gate.
- Treat `search_token` as metadata, not price evidence; price claims must remain grounded in price/market-data tools.

## Assumptions
- Some compaction can happen in backend result projection before every individual tool is rewritten.
- Search/list tools should generally return structured JSON, not prose.
- Usage/perf reporting from `v-nncg` can verify reductions in `tool_result_tokens`.

## Non-goals
- Do not solve signing/card payload compaction here; that belongs to `v-hykj` and flow-specific tickets.
- Do not re-enable Polymarket here.
- Do not redesign token registry matching semantics beyond ranking/capping output unless necessary.

## Open Questions
- What caps should apply by default: top 5, top 10, token budget, or tool-specific values?
- What fields are mandatory in token search choices so the model can disambiguate safely?
- Should continuation refs be stored server-side, embedded as compact ids, or derived from deterministic query params?
- Which broad-search fixtures should become token-budget regression tests?
