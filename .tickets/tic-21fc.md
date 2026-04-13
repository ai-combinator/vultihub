---
id: tic-21fc
status: open
type: chore
priority: 2
assignee: Jibles
created: 2026-04-13T18:20:21.412630787Z
---
# Clean up AgentHomeScreen orchestration and remaining hook complexity

## Objective
After tic-6c08 (tool execution lifecycle) and tic-1d38 (chat hook wiring) land, AgentHomeScreen and its remaining hooks will have less to coordinate but will still carry structural debt. This ticket cleans up the final layer: route param sync, confirmation flow, and the overall hook composition in AgentHomeScreen. The goal is to reach a state where a new developer can read the agent feature top-to-bottom and understand the data flow without tracing through refs and implicit effect ordering.

## Context & Findings
**useRouteParamSync is an 80-line async effect with manual dedup refs:**
- Watches route.params and tries to figure out what changed (conversationId, schedulerTxProposalId, prefillText)
- Each param type has its own processedXRef to prevent re-processing
- The async fetch (getTxProposalApi) can race with navigation — if the fetch completes after the user navigates away, it writes pendingTx to the wrong conversation
- ShapeShift equivalent: navigation actions call handlers directly (no effect-based param detection). When you navigate to a conversation, the navigation action itself calls loadConversation(). No refs, no dedup.
- Fix: replace the effect with explicit navigation callbacks. handleSwitchConversation, handleSchedulerProposal, handlePrefill called directly from navigation events, not derived from route.params changes.

**useConfirmationFlow has unbounded per-conversation state:**
- consumedByConv is a Record keyed by conversationId that grows indefinitely during a session
- dismissedMessageIds is a Set that also grows indefinitely
- scheduleAddedMessageIds is a Set that grows indefinitely
- None of these are cleaned up on conversation switch
- The hook also writes to chatMessages directly via chatSetMessages callback, which is a side channel that bypasses useChat's message ownership
- Fix: scope dismissal state to the active conversation and clean up on switch. Consider whether schedule approval results should be part of the message stream (via AI SDK data parts) instead of local state.

**AgentHomeScreen hook count:**
- After tic-6c08 and tic-1d38 land, several hooks will be simplified or removed. But the screen still orchestrates auth, tools/context, conversations, chat, and potentially approval context.
- The target is 4-5 clearly scoped hooks with no cross-hook store coordination: useAgentAuth, useConversations, useAgentChat (which internally owns tool rendering + confirmation), and useApproval (context provider).
- The current 11-hook call tree should reduce naturally once pendingTx/txResult store indirection is gone and conversation switching is simplified.

## Files
- src/features/agent/screens/AgentHomeScreen.tsx — reduce hook count; replace useRouteParamSync with direct navigation callbacks; remove any remaining store-based coordination between hooks
- src/features/agent/hooks/useRouteParamSync.ts (or equivalent inline logic) — replace with explicit handlers called from navigation events
- src/features/agent/hooks/useConfirmationFlow.ts — scope state to active conversation; clean up on switch; remove unbounded maps/sets
- src/features/agent/hooks/useAgentChat.ts — after tic-6c08 removes pendingTx effects, verify this hook is clean and owns only: useChat, confirmation flow, vault import

## Acceptance Criteria
- [ ] Route param handling uses direct callbacks from navigation (not effect watching route.params)
- [ ] No manual processedXRef dedup pattern for route params
- [ ] Async operations during navigation are cancellable and scoped to the target conversation
- [ ] useConfirmationFlow state (dismissed, consumed, scheduleAdded) is bounded — cleaned up on conversation switch
- [ ] useConfirmationFlow does not write to chatMessages via side channel (or if it must, this is explicit and documented)
- [ ] AgentHomeScreen calls 5 or fewer top-level hooks
- [ ] No hook reads from a store that another hook writes to in the same render cycle (no implicit coordination)
- [ ] A new developer can trace data flow from user input to rendered response by reading AgentHomeScreen top-to-bottom
- [ ] Lint and type-check pass

## Gotchas
- useRouteParamSync handles deep links and push notification navigation — make sure direct callbacks cover these entry points, not just in-app navigation
- The scheduler proposal flow (schedulerTxProposalId) fetches from the server and may need to switch conversations first — this is a multi-step navigation action, not a single callback
- useConfirmationFlow's chatSetMessages writes are used to inject schedule confirmation results into the message stream — verify the AI SDK data-parts approach works for this before removing the side channel
- After tic-6c08 removes pendingTx from the store, some of AgentHomeScreen's wiring will naturally disappear — don't duplicate work that tic-6c08 already handles

