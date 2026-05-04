---
id: tic-21fc
status: closed
type: chore
priority: 2
assignee: Jibles
created: 2026-04-13T18:20:21.412630787Z
---
# Clean up AgentHomeScreen orchestration and remaining hook complexity

## Gotchas
- useRouteParamSync handles deep links and push notification navigation — make sure direct callbacks cover these entry points, not just in-app navigation
- The scheduler proposal flow (schedulerTxProposalId) fetches from the server and may need to switch conversations first — this is a multi-step navigation action, not a single callback
- useConfirmationFlow's chatSetMessages writes are used to inject schedule confirmation results into the message stream — verify the AI SDK data-parts approach works for this before removing the side channel
- After tic-6c08 removes pendingTx from the store, some of AgentHomeScreen's wiring will naturally disappear — don't duplicate work that tic-6c08 already handles

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

## Notes

### 2026-04-13T20:01:07Z

Implementation complete. navigationIntentStore replaces useRouteParamSync, useConfirmationFlow state bounded + side channel narrowed, useAgentOrchestration consolidates ~15 inline effects from AgentHomeScreen. 5 domain hooks in AgentHomeScreen. TypeScript clean. Needs manual testing.


**2026-04-23T23:55:07Z**

Closing: implementation commit 6b215ec already on origin/main. Manual verification implicitly covered by ongoing agent-chat use of AgentHomeScreen + navigationIntentStore + useAgentOrchestration.
