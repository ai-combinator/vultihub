---
id: v-vpeb
status: open
deps: []
links: []
created: 2026-05-07T05:10:32Z
type: task
priority: 0
assignee: Jibles
---
# Progressive disclosure for prompts and tools

## Problem
The backend already narrows the model's tool list by intent/category, but the system prompt remains global and carries instructions for unrelated workflows on every turn. Some tool-filter paths can also fail open, and some phases widen back to a larger tool set after a build tool succeeds, so the model can pay for tools/instructions it should not need. This creates a large stable prompt/tool tax: normal e2e probes measured roughly `24.7k` stable system tokens per AI call, about `75%` of prompt-side tokens. Solved means the backend progressively discloses both tools and instructions by intent/phase: reveal only the relevant prompt pack and tool set for the current turn, make tool filtering fail closed to small safe sets, move obvious bulky workflow sections out of the global prompt, and avoid widening back to broad tools after a successful build/card result.

## Background
### Findings
- Overarching context from the token-cost investigation: the agent should be made cheaper and more reliable by shaping the LLM's environment, not by piling on more prompt warnings. The model should see fewer tools, shorter instructions, smaller results, and only context relevant to the current intent/phase.
- Prompt philosophy: the global prompt should be small and stable. Workflow/protocol details belong in intent/phase packs, tool contracts, structured errors, validators, or tests.
- Tool philosophy: prompt-pack selection and tool visibility must move together; the model should not see tools without relevant instructions or instructions for unavailable tools.
- Current prompt assembly is global. `agent-backend/internal/service/agent/prompt.go` builds the large base system prompt, and `agent-backend/internal/service/agent/agent.go` appends it into every request.
- Current tool exposure already has intent/category machinery:
  - `TurnState` / build intent logic controls forced tool choice and tool narrowing for send/swap/bridge/schedule-style flows.
  - `tool_filter.go` filters MCP tools by user query categories and related context.
- Tool filtering has expensive fail-open cases that work against progressive disclosure:
  - empty query can return all tools;
  - no-match paths can return all tools if utility fallback is unavailable;
  - uncategorized tools can be included by default.
- After a successful build invariant, `TurnState` can widen the visible tool set again for the final phase. Recent loop analysis found this is why some final narration calls carry a much larger tool schema bundle even though they only need to present a completed card.
- The existing system prompt contains many workflow-specific sections that do not apply to every turn: receive/payment request, scheduling, credits/subscriptions, Rujira, Polymarket, Cosmos staking, source-asset resolution, cross-chain destination rules, quick action formatting, structured output details, historical regression examples.
- Local usage audit showed normal backend-e2e run totals around:
  - `660,503` prompt tokens across `20` AI calls.
  - `493,780` stable system tokens across those calls.
  - `159,330` tool-definition tokens.
  - Tool results were tiny in normal probes: `4,830` tool-result tokens.
- ShapeShift comparison showed their global prompt is much smaller: roughly `10,160` chars / `2,540` tokens, compared to Vultiagent's roughly `98,756` chars / `24,689` tokens per call.

## Current Thinking
The current tool-intent system partially shapes the model's action space, but not its instruction space, and not consistently across phases. We should extend that same concept into a progressive disclosure model for both prompt content and tool visibility.

Current shape:

```text
intent/category logic -> narrows tools for initial phase
global prompt          -> still includes everything
after build success    -> may widen tools again
```

Target shape:

```text
intent/category/phase resolver -> selected tool set
same resolver                 -> selected prompt packs
unknown/empty intent          -> small safe fallback, not all tools
after build/card success      -> no tools or minimal presentational tools
```

Examples:

```text
Balance query:
  global prompt
  + balance/portfolio pack
  + balance/read tools

Send:
  global prompt
  + send pack
  + execute_send/minimal support tools

Swap/bridge:
  global prompt
  + swap pack
  + execute_swap/search/price support tools

Receive/payment request:
  global prompt
  + receive pack
  + receive/payment tools

Schedule:
  global prompt
  + schedule pack
  + schedule_task only
  + no broad tools after schedule confirmation is ready

Polymarket:
  global prompt
  + polymarket pack
  + polymarket tools, with result caps

Rujira:
  global prompt
  + rujira pack
  + rujira tools
```

The global prompt should shrink to identity, concise mobile style, crypto/wallet scope, no fabrication, fund-moving confirmation, high-level routing, and card narration rule. Detailed workflow instructions should move into packs.

This ticket should include the first concrete section moves, not just abstract pack plumbing. The first implementation slice should move obvious bulky sections out of the global prompt and into packs, starting with low-risk workflow sections:

```text
receive/payment request
scheduling
credits/subscriptions
```

If the plan finds those are too coupled, it should choose the smallest safe subset but still deliver at least one real section move so the token reduction is measurable. Later milestones can move Rujira, Polymarket, staking, and detailed swap/send edge cases.

This ticket should also stop the obvious progressive-disclosure regression where a successful build/card phase widens back to broad tools. That does not need to solve deterministic final responses; it only needs to preserve the narrowed/no-tool phase after success so final presentation does not resend unrelated schemas.

This ticket should also make tool filtering fail closed. Unknown, empty, or no-match intents should not return all MCP tools. They should return a small safe default set or ask a clarifying/no-tool response. Uncategorized LLM-facing tools should be treated as a startup/test failure or excluded until categorized, rather than silently included.

The target for an initial milestone should be measurable: reduce stable system tokens in normal backend-e2e by at least `40%` without increasing probe failures.

## Constraints
- Tool filtering and prompt-pack selection must stay consistent. The model should not see a tool without the relevant instructions, and should not see detailed instructions for tools/workflows that are unavailable.
- Tool filtering should fail closed. Empty/no-match/category gaps should not expose the full tool catalog.
- Any fail-closed change needs category coverage checks so genuinely useful tools do not disappear silently.
- Phase transitions must also be consistent. After successful calldata or schedule-card production, the model should not regain broad tools unless there is a specific recovery reason.
- If intent classification is ambiguous or wrong, fallback must be safe. Prefer global + small ambiguous/routing pack over restoring the full old prompt by default.
- Preserve fund-safety constraints: no auto-executing fund-moving actions, no fabricated tools/facts, confirmation required for signing flows.
- Existing behavior tests/e2e probes must remain the source of truth; exact prompt-text tests may need conversion to behavior assertions in separate work.
- This ticket should not change tool schemas, sender injection, or deterministic final-call elimination directly, though it should be compatible with those efforts. It may change final-phase tool visibility to no-tools/minimal-tools as part of progressive disclosure.

## Assumptions
- Existing intent/category signals are good enough to start with coarse prompt packs, even if later refinement is needed.
- The planner may choose to introduce prompt pack plumbing first with no behavior change, then move sections one at a time.
- Prompt budget tests will either already exist from a separate ticket or should be included as acceptance criteria if they do not.

## Non-goals
- Do not delete large prompt sections without moving or covering their behavior.
- Do not solve final LLM narration waste here; that is covered by the presentational-tool canned response ticket.
- Do not implement tool-result truncation here.
- Do not build a new LLM-based intent classifier unless existing classifier/category logic proves insufficient.
- Do not move every section in the first PR; this is expected to be phased. The first PR should still move at least one real bulky workflow section so this is not only framework work.
- Do not implement full deterministic tx-ready/schedule termination here. This ticket may prevent tool widening after success, but canned backend responses/no second LLM call belong to `v-glle`.

## Dead Ends
- Keeping only tool narrowing was deemed insufficient. It reduces tool schemas but leaves the huge global instruction prompt on every request.
- Letting post-success phases widen back to broad tools was deemed contrary to progressive disclosure. Once a card/build result is ready, broad tools are not needed for presentation and just add schema cost.
- Fail-open tool filtering was deemed contrary to progressive disclosure. If the resolver cannot confidently identify an intent/category, exposing every tool is expensive and makes the model's job harder.
- A big-bang prompt deletion is too risky because many global instructions encode real production regressions. The safer shape is pack plumbing plus behavior tests plus incremental movement.

## Open Questions
- What exact pack set should exist in the first milestone: `send`, `swap`, `receive`, `balance`, `schedule`, `credits`, `rujira`, `polymarket`, `staking`, `contract`, `ambiguous`?
- Should pack selection be driven by `TurnState.Intent`, `tool_filter` categories, MCP tool categories, or a unified resolver that returns both tools and prompt packs?
- Should the unified resolver explicitly model phases such as `initial_tool_selection`, `tool_recovery`, and `presentation`?
- What exact safe fallback tool set should be used for empty/no-match intents?
- Should uncategorized MCP tools fail startup/tests, be excluded from LLM visibility, or be allowed only through explicit overrides?
- Should prompt packs be represented as Go string builders in `prompt.go`, separate files, or structured prompt sections with token-budget metadata?
- What should the fallback pack be when user intent is unclear?
- After build/card success, should final phase be `Tools:nil`, a tiny respond-only tool set, or no model call at all when combined with `v-glle`?
- Which exact prompt tests currently pin strings that should become behavior-level tests before sections move?
