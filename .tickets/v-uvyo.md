---
id: v-uvyo
status: closed
deps: []
links: []
created: 2026-04-15T21:54:13Z
type: task
priority: 2
assignee: Jibles
parent: v-bauf
---
# Consolidate MCP balance fan-out server-side

## Why
After the data-tools migration on agent-backend (commit 205c038), get_balances and get_portfolio run server-side against pre-fetched EVM+Solana balances. The LLM still has per-chain MCP balance tools available (evm_get_balance, get_utxo_balance, get_sol_balance, get_xrp_balance, get_atom_balance, get_cardano_balance, get_sui_balance, get_trx_balance, get_ton_balance) and fans out to them, producing 5–10 tool calls per portfolio query. The frontend hides this with coalesceParts + AggregateToolIndicator — a UI workaround for a backend shape we no longer want.

Shapeshift-agentic has no equivalent complexity; balance fetching is one tool.

## What
- Extend execGetBalances / execGetPortfolio (agent-backend internal/service/agent/data_tools.go) to cover not_pre_fetched chains by fanning out to MCP internally and aggregating before returning to the LLM.
- Update system prompt / tool descriptions (internal/service/agent/prompt.go) to steer the LLM toward get_balances / get_portfolio and away from per-chain MCP balance tools. Consider hiding per-chain balance MCP tools from the LLM tool list for chat flows (keep them available to signing paths if needed).
- Once metrics confirm per-chain balance tools are no longer invoked in chat turns, delete:
  - vultiagent-app/src/features/agent/lib/coalesceParts.ts (+ __tests__/coalesceParts.test.ts)
  - vultiagent-app/src/features/agent/components/AggregateToolIndicator.tsx
  - coalescing branch in AssistantMessageView.tsx (currently lines 117-125)

## Acceptance
- 'What is my portfolio' produces one tool call end-to-end.
- coalesceParts + AggregateToolIndicator deleted with no visual regression.
- Tool-call metrics show no per-chain balance tools in chat turns.

## Notes

**2026-04-22T01:12:27Z**

Closing: coalesceParts.ts + __tests__/coalesceParts.test.ts + AggregateToolIndicator.tsx all deleted. Coalescing branch (~lines 117-125) removed from AssistantMessageView.tsx. Balance fan-out evidently consolidated server-side as intended.
