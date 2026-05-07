---
id: v-xkvs
status: closed
deps: []
links: []
created: 2026-05-03T22:43:44Z
type: bug
priority: 0
assignee: Jibles
---
# Replace 'couldn't complete in reasonable steps' wording + step-budget tweak

## Problem
When the agent's tool-call loop hits the step budget, it returns a generic `"I couldn't complete that in a reasonable number of steps. Please try a more specific request — e.g. include the chain, token, and amount explicitly."` message. Hit Felix 3 times in 5 messages in conv `8ae9bf54` while he was being increasingly specific about what he wanted. From his perspective, the agent was giving up on him while accusing him of being vague.

The wording is bad UX, AND the underlying step-budget burn is a real bug — the model was burning iterations on guess-and-retry against a single broken tool (`get_fin_swap_quote`), not exploring multiple intents.

## Background
### Findings
- Wording occurs 3× in conv `8ae9bf54` at messages #25, #27, #31. User input at the same points:
  - msg #24: "ok now can you swap to RUJI" (after agent said "swap RUNE to RUJI" is the path)
  - msg #26: "I want to swap the RUNE to RUJI" (more specific)
  - msg #28: "Swap RUNE on Thorchain to RUJI on thorchain" (chain-explicit)
  Each got the same generic "be more specific" rejection.
- Step budget was being burned by the model retrying `get_fin_swap_quote` with mutated asset strings (`x/ruji/thor.rune/x/ruji`, `ruji/eth-usdc-.../ruji`, etc.) — same tool, different bad inputs. ~25-30 retries before hitting cap.
- "Drop Rujira FIN tools" ticket addresses the root cause for this specific conversation (once `get_fin_swap_quote` is gone from agent's view, loop has nothing to retry against).
- But the broader bug remains: any future tool that returns errors and tempts the model into guess-and-retry will burn the loop the same way, and the user-facing message will still blame the user.

## Current Thinking
Two issues, both addressed pre-release:

1. **Wording** — replace with something that names the actual condition ("I'm hitting an internal limit on this swap path; let me know if you want to try a different route") and doesn't blame the user.

2. **Step-budget protection against retry-the-same-tool spirals** — cap retries of the same tool with similar args. Concretely: if the model has called `tool X with args ~Y` (cosine-similar / structural-match) 3+ times this turn and all failed, force a different action (different tool, ask clarification, or end the turn) instead of letting it consume the rest of the budget.

(2) is the load-bearing fix; (1) is the polish that doesn't trick users into thinking they're the problem.

## Constraints
- Must not regress legitimate multi-step flows that genuinely need many tool calls (e.g. multi-message Polymarket setup, big balance fan-outs).
- "Similar args" detection needs to be robust — string-equal too narrow (model mutates one char at a time), structural-equal probably right.

## Non-goals
- Generalizing to "model is stuck" detection across non-tool turns (hallucination loops, etc.). Tool-retry detection only.
- Increasing the step budget. Treating the symptom not the cause.

## Open Questions
- Where does step-budget enforcement live in agent-backend? Probably executor loop. Confirm during planning.
- "Similar args" definition — exact match excluding one field? Edit-distance over JSON? Some structural normalization? Pick during implementation.
- Wording — keep it short and stop blaming the user. Specific copy can be a product call.

## Notes

**2026-05-05T02:40:49Z**

killed: per Sean — dropping from active queue
