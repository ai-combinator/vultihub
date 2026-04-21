---
id: v-bpzs
status: open
deps: []
links: []
created: 2026-04-16T23:57:46Z
type: bug
priority: 2
assignee: Jibles
---
# Agent restates send-card details as bullet text after execute_send

## Symptoms

After the agent calls `execute_send` (mcp-ts) and the mobile UI renders the structured send confirmation card (From / To / Amount / Network / Estimated Fee), the assistant's chat text ALSO contains a bulleted restatement of the same fields. User sees the same information twice — once as a card, once as bullets — which clutters the mobile screen.

The model should either render the card OR describe it in prose, not both.

## Reproduction

1. Agent chat: "send 0.0001 ETH on arb to myself" (any execute_send flow).
2. Observe: send action card renders with From/To/Amount/Network/Estimated Fee rows, AND the assistant text immediately below repeats the same bulleted list verbatim.

## Suspected Cause

The system prompt at `agent-backend/internal/service/agent/prompt.go:98-122` explicitly instructs the model to emit the bulleted template AFTER the execute_* tool call:

```
## Transaction Confirmations (UI parser — load-bearing format)

The mobile UI parses the tool output to render a confirmation card, so AFTER a
successful `execute_send` / `execute_swap` / build tool call, narrate ONE response
with the structured bullet template. Every bullet MUST start with "- ". Use
HUMAN-READABLE amounts, not base units. Never emit placeholders like "[building...]".

SEND TEMPLATE:
Send [amount] [SYMBOL] on [CHAIN]. Here are the details:
- From: [from_address]
- To: [to_address]
- Amount: [amount] [SYMBOL]
- Network: [CHAIN]
- Estimated Fee: [human-readable fee]

Should I execute the send?
```

The comment claims "the mobile UI parses the tool output to render a confirmation card" — i.e. historically the app scraped the assistant text for the bullet block. With the unified card, the card is now rendered from the structured `execute_send` result (`txArgs` + `resolved` map in `mcp-ts/src/tools/execute/execute_send.ts:568-577`), not from the assistant's text. So the bullet template has become redundant instructional baggage that produces the duplicate block.

A secondary contributor to check: what `execute_send` returns as its text content. Currently it emits `jsonResult(result)` only (no human-readable narration), so the tool result itself isn't the source of the restatement — it's the system prompt instruction driving the model to regenerate it.

## Files Worth Checking

- `agent-backend/internal/service/agent/prompt.go:98-122` — the "Transaction Confirmations" block. The fix is likely to drop the bullet template and replace it with: "After a successful `execute_*`, respond with ONE short confirmation sentence naming the action ('Sending 0.0001 ETH on Arbitrum — confirm below?'). The app renders the card from the structured result." Coordinate with whoever owns the unified-card UI to ensure no other code path still scrapes the bullets.
- `vultiagent-app/src/features/agent/lib/__tests__/executePrep.test.ts` and sibling parseServerTx tests — confirm nothing app-side parses the assistant text to build the card (grep for "From:" / "Network:" literals in the app).
- `mcp-ts/src/tools/execute/execute_send.ts` — the tool returns `jsonResult` only, so no human-readable body needs trimming. Verify the same for `execute_swap` and other `produces_calldata: true` tools if we extend the fix.
- `agent-backend/internal/service/agent/prompt_test.go` — update the prompt assertion fixtures if present.

## Out of Scope

- Unified action-card UI redesign work (handled separately).
- Any changes to the underlying `execute_send` envelope / `produces_calldata` contract.
- The EVM self-send derivation failure bug (separate P1 ticket).

## Priority Rationale

P2 (normal): cosmetic / UX clutter; doesn't break functionality. Worth fixing for mobile chat readability, but not blocking.
