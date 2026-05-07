---
id: v-glle
status: open
deps: []
links: []
created: 2026-05-07T05:01:02Z
type: task
priority: 0
assignee: Jibles
---
# Canned responses for presentational tool outputs

## Problem
Successful presentational tool flows currently pay for an extra LLM call just to produce short UI narration like “review below and sign.” This is expensive because every loop carries the large system prompt and tool schemas again. Solved means successful `tx_ready` / schedule-card turns can terminate with deterministic backend copy and the existing card payloads, without a second full model pass, while preserving validation, persistence, SSE usage, and mobile UI behavior.

## Background
### Findings
- Overarching context from the token-cost investigation: the agent should be made cheaper and more reliable by shaping the LLM's environment, not by piling on more prompt warnings. The model should see fewer tools, shorter instructions, smaller results, and only the fields/results it needs for the next decision.
- Cost philosophy: every extra model loop resends expensive system/tool context. Successful card/presentational flows are the clearest place to remove unnecessary calls entirely.
- Product philosophy: if the UI already renders the structured card, the final text should be deterministic and minimal unless model judgment is genuinely needed.
- The current agent loop follows the generic tool-calling pattern: model calls a tool, backend executes it, appends tool results into `messages`, then loops so the model can produce final text.
- In `agent-backend/internal/service/agent/agent.go`, each streaming loop sends `SystemParts`, `Messages`, `Tools`, and `ToolChoice` again. The loop exits only when the model response has no tool calls. Tool results are appended as `ai.ToolMessage` before the next iteration.
- `TurnState` marks a build invariant as satisfied after successful calldata/schedule output, but after success `ToolsForModel` can widen back to a larger tool set. That makes the final narration call nearly as expensive as the build call.
- Recent local usage audit with `scripts/qa/backend-e2e-probes.sh usage-audit` showed common successful card flows taking two calls:
  - P2 swap: `72,268` prompt tokens over `2` calls; final narration call alone was `43,225` prompt tokens for `11` completion tokens.
  - P3 bridge: `63,287` prompt tokens over `2` calls; final narration call was `34,832` prompt tokens for `13` completion tokens.
  - P8 schedule: `64,184` prompt tokens over `2` calls; final narration call was `35,204` prompt tokens for `44` completion tokens.
- The cards/payloads are already the actual user-facing product for these flows. The final text is usually only a short presentational sentence, and the prompt already tells the model not to restate card fields.
- Relevant code paths discussed:
  - `agent-backend/internal/service/agent/agent.go` streaming and non-streaming model loops.
  - `agent-backend/internal/service/agent/turn_state.go` `ToolsForModel` / `ObserveToolCall` invariant logic.
  - Existing pending event buffers such as `pendingTxReadyEvents` and `pendingScheduleEvents` in the streaming path must keep working.

## Current Thinking
Where we landed: treat successful presentational tool outputs differently from generic information tools.

Generic info tools still need the normal tool-result → model-summary pattern. Example: balance/portfolio data may need model narration from returned data.

Presentational tools should not. For successful transaction/schedule card flows, the backend should emit deterministic text and terminate the turn after the card/payload is ready and validated. The model’s job is to select the correct build tool; once the backend has a validated `tx_ready` or schedule confirmation, asking the model again is wasteful.

Ideal successful flow:

```text
User asks for send/swap/bridge/schedule
LLM call 1 chooses the build tool
Backend executes the tool
Backend validates/security-checks/builds tx_ready or schedule card
Backend emits deterministic short text + card payload
No final LLM call
```

Suggested canned copy shape:

```text
send:     Transaction prepared — review the details below and tap ✓ to sign.
swap:     Swap prepared — review the details below and tap ✓ to approve.
bridge:   Bridge prepared — review the details below and tap ✓ to approve.
schedule: Review the scheduled transaction below.
```

Safer rollout shape if full deterministic termination is too much in one step:

```text
After build invariant success, use a tool-free final response mode first:
  Tools: nil
  small max_tokens
  no Gemini thinking
Then graduate to fully deterministic termination once probes prove card/text persistence is correct.
```

The preferred end-state is deterministic termination for successful presentational outputs, not just tool-free final narration, because the text is known and the card owns the details.

## Constraints
- Must not bypass transaction validation, Blockaid/security checks, calldata stashing, or any existing safety gate.
- Must preserve mobile/client-visible payloads and parts, including `data-tx_ready`, schedule confirmation parts, top-level `transactions`, and any existing SSE event ordering the app depends on.
- Must preserve message persistence, usage accounting, credit settlement, and final SSE `usage`/`finish` events.
- Must handle both streaming and non-streaming paths or explicitly scope the first implementation to one path with a follow-up.
- Full raw tx/card payloads must remain available to the client/signing path; any LLM-facing compaction is separate work.

## Assumptions
- For successful `execute_send`, `execute_swap`, `execute_contract_call` when it yields a presentational tx card, and `schedule_task`, the card is sufficient for the user and model-written summary is not necessary.
- Existing backend has enough information to choose a reasonable canned sentence by tool/result type.
- Existing e2e probes are adequate to catch obvious card/persistence regressions once token budget assertions are added.

## Non-goals
- Do not rewrite the global system prompt in this ticket.
- Do not implement full intent prompt packs here.
- Do not change tool schemas or sender injection here.
- Do not remove final model narration for all tools; this ticket is only for successful presentational/card outputs.
- Do not solve all error-loop behavior here, except avoiding the final narration call after success.

## Dead Ends
- Keeping the generic agent loop unchanged was rejected for this class of flow. It is standard for general tool agents, but in our tx/card flows it pays tens of thousands of prompt tokens for a canned sentence.
- Tool-free final narration is useful as a lower-risk intermediate rollout, but it still pays the large system prompt. It is not the desired final form for successful card outputs.

## Open Questions
- Which exact tool/result predicates should count as “presentational success” for the first rollout: only `produces_calldata` tools with emitted `tx_ready`, only `schedule_task`, or also some read-only card tools?
- Should this ship behind a launch flag such as `AgentDeterministicCardFinal` or `AgentToolFreeFinalAfterBuild`?
- Should streaming path be implemented first because the app uses SSE, or should non-streaming be kept in lockstep to avoid drift?
- What is the exact persisted assistant message shape when terminating deterministically: text part first, then card part, matching current client expectations?
- Should failed tool calls ever use deterministic prose in this ticket, or be deferred to a separate known-error terminal handling ticket?
