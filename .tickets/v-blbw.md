---
id: v-blbw
status: open
deps: []
links: []
created: 2026-05-07T05:42:33Z
type: task
priority: 0
assignee: Jibles
---
# Classify tool errors for bounded LLM retry vs terminal response

## Problem
The agent needs to recover when the LLM makes fixable tool-call mistakes, but it should not keep retrying when the tool result proves the request is impossible or requires a new user decision. Today those cases can blur together: a malformed argument can deserve one repair attempt, while failures like no route, unsupported provider, insufficient funds, invalid destination, or repeated same-tool/same-args/same-error should terminate with a clear user response. Solved means tool failures are classified by retryability, the loop only retries when the next attempt has a realistic path to change the outcome, and terminal failures produce deterministic user-facing guidance without burning extra model calls.

## Background
### Findings
- Overarching context from the token-cost investigation: reducing loops is a cost and reliability problem. The agent should recover from genuinely fixable model/tool mistakes, but terminal failures should stop deterministically instead of burning more prompt/tool-context cycles.
- Error-handling philosophy: hard impossibility and user-action-required states belong in structured tool errors and backend terminal responses, not in repeated prompt warnings.
- Retry philosophy: allow retries only when the next model/tool call has a realistic path to change the outcome.
- Recent loop/cost investigation found normal successful tool flows often spend an extra LLM call for final narration, while failure/recovery cases can take 3-4 calls when the model keeps trying variants.
- Loop caps alone are too blunt. Some tool errors are genuinely LLM-repairable: wrong arg shape, wrong identifier type, missing field that the tool can coach, or ambiguous token result with enough context to choose the right value.
- Some tool errors are not LLM-repairable: no route exists, unsupported chain/provider, insufficient balance, invalid recipient address, missing user decision, market/order unavailable, or same tool + same args + same error repeated.
- Example distinction from the conversation:
  - Retriable: model passes an asset ID where a symbol is required, tool returns a clear error, model retries with `USDC`.
  - Terminal: user asks for a BNB -> USDC swap and provider returns no route; model should stop and tell the user rather than trying nearby tokens/providers/random contract variants.
- This work complements, but is distinct from:
  - `v-glle`: deterministic/canned responses after successful presentational tool outputs.
  - `v-vpeb`: progressive disclosure of prompts/tools.
  - `v-nncg`: usage audit/persistence for measuring cost.

## Current Thinking
The important design distinction is not "retry or don't retry" globally. It is whether the error gives the LLM a concrete, safe next action that can produce a materially different tool call.

Target policy shape:

```text
recoverable_input_error       -> allow one bounded repair attempt
recoverable_disambiguation    -> allow one bounded repair attempt, or ask user if ambiguity is not resolvable
recoverable_missing_context   -> allow only if backend/model can obtain context with visible tools
terminal_user_action_required -> deterministic ask/stop
terminal_unsupported          -> deterministic stop
terminal_no_route             -> deterministic stop or ask if user wants different asset/chain
terminal_insufficient_funds   -> deterministic stop
terminal_invalid_destination  -> deterministic stop/ask for a valid address
terminal_provider_failure     -> usually stop; maybe one retry only if explicitly transient
terminal_repeated_same_error  -> stop immediately
unknown                       -> one retry max, then stop
```

Tools should increasingly return structured, LLM-readable error envelopes rather than generic strings. Ideal terminal shape:

```json
{
  "status": "error",
  "code": "no_route",
  "retriable": false,
  "message": "No route is available from BNB to USDC.",
  "next_action": "Tell the user this route is unavailable; ask if they want another destination asset."
}
```

Ideal repairable shape:

```json
{
  "status": "error",
  "code": "invalid_asset_identifier",
  "retriable": true,
  "message": "from_symbol must be a ticker like BNB, not an asset id.",
  "retry_hint": {
    "field": "from_symbol",
    "use": "BNB"
  }
}
```

The backend loop should consume that classification. It should not rely only on the LLM reading prose. If `retriable=false`, stop the model/tool loop and emit deterministic user-facing text. If `retriable=true`, allow a small bounded number of repair attempts, ideally one per error/tool class. If the same tool+args+error repeats, stop regardless of retryability.

This should be implemented as a focused reliability/cost control layer, not as a giant rewrite of all tools at once. Start with high-cost/high-frequency surfaces: `execute_swap`, `execute_send`, token search/resolve, and quote/route errors. Then generalize.

## Constraints
- Do not remove all LLM recovery. The model must still be able to fix malformed args or follow clear retry hints.
- Do not retry fund-moving tool calls after terminal errors unless the next call is materially different and allowed by policy.
- Terminal responses must not claim success, submission, broadcast, or confirmation. They should explain the blocker and the user's next option.
- Error classification must be observable in usage/debug data: code, retriable flag, loop iteration, tool name, and terminal reason where possible.
- Existing loop breaker remains as a last-resort safety net; this ticket should reduce how often it is reached.
- Keep unknown-error behavior conservative: at most one retry, then stop with honest uncertainty.

## Assumptions
- Many current MCP errors are plain strings. The first implementation may need a backend classifier for known strings/codes while tools are gradually migrated to structured envelopes.
- `execute_*` tools are the highest-value initial targets because they sit on signing/fund-moving flows and failures can be expensive or risky.
- Deterministic terminal copy can be backend-owned without waiting for full prompt-pack work.

## Non-goals
- Do not implement broad per-intent loop budgets here, except as a fallback if needed. This ticket is about retryability classification.
- Do not solve final LLM narration after successful card/tool output; that belongs to `v-glle`.
- Do not redesign every MCP tool schema in one PR; broader schema cleanup belongs to `v-giou` and follow-up tickets.
- Do not use another LLM call to classify errors.

## Dead Ends
- Pure loop caps were considered insufficient. They prevent runaway spend but also cut off legitimate one-step repairs.
- Pure prompt instruction was considered insufficient. The model may still retry terminal failures unless backend/tool errors carry explicit retryability and the loop enforces it.
- Treating every provider failure as retryable was rejected. Some provider responses encode real impossibility (`no_route`, unsupported chain), not transient failure.

## Open Questions
- Where should the canonical taxonomy live: MCP tool result envelope, agent-backend classifier, shared package, or both?
- What exact initial error codes should ship for `execute_swap` and `execute_send`?
- Should retry budget be per tool call, per error code, per visible user message, or per phase?
- What deterministic copy should be used for common terminal classes (`no_route`, `insufficient_funds`, `invalid_destination`, `unsupported`, `missing_user_decision`)?
- Which existing curl-replay/backend-e2e fixtures should be tightened to prove terminal errors stop after one call?
- What telemetry should usage audit show for terminal stops vs repair retries?
