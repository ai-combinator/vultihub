---
id: tic-6c08
status: closed
type: task
priority: 2
assignee: Jibles
created: 2026-04-13T12:07:22.280484066Z
---
# Unified tool execution lifecycle with historical guard

## Gotchas
- The text-based chain detection fallback (regex-matches chain from conversation text when parseServerTx returns null) must be preserved but extracted — call it once when the tool output arrives, cache the result, don't recompute per render
- Biometric prompt is wrapped in InteractionManager.runAfterInteractions() to prevent navigation race — preserve this in the new useApproval hook
- The tx_ready path (scheduled proposals via getTxProposalApi) is separate from build_* detection — both need historical guard but have different data sources
- recentActions must still flow to backend on next message send — callback pattern changes WHERE it's written, not WHETHER
- ApprovalContext needs to handle the case where multiple tool components exist in the same conversation — only one approval at a time

## Acceptance Criteria

- [ ] signAndBroadcast() exists as a pure async function: takes (parsedTx, vault, password, authToken) → returns { txHash, explorerUrl, chain } or throws
- [ ] Text-based chain detection fallback extracted from completeTxApproval into a standalone helper — not recalculated on every render
- [ ] useApproval hook (or ApprovalContext) manages password/biometric state; tool components call requestApproval() to trigger the prompt and receive the password
- [ ] ToolExecutionState type defined with step-based state machine (currentStep, completedSteps, skippedSteps, failedStep, terminal, meta, substatus)
- [ ] useToolExecution hook provides ExecutionContext with advanceStep/skipStep/failAndPersist/markTerminal
- [ ] Historical tool guard: Set of tool call IDs populated on conversation load, checked before any tool execution
- [ ] BuildTxCard receives tool output data as props (not from chatSessionStore) and owns its step machine
- [ ] pendingTx and txResult removed from chatSessionStore
- [ ] recentActions reported via callback (onActionLogged) — not written to store by signing, read by context builder
- [ ] buildToolDetection auto-fire effect in useAgentChat removed entirely
- [ ] Loading a saved conversation with completed build_* tools does NOT show approval UI
- [ ] Switching conversations within a session does NOT re-trigger historical build tools
- [ ] All existing signing flows (EVM, Solana, UTXO, XRP, Cosmos, etc.) work through the pure signAndBroadcast function
- [ ] `messages` is NOT in the dependency array of any signing-related callback
- [ ] Existing tests updated to reflect new patterns

## Notes

### 2026-04-13T18:58:36Z

Implementation complete. Created signAndBroadcast.ts, chainDetection.ts, historicalToolGuard.ts, ApprovalContext.tsx, ToolExecutionContext.tsx, useToolExecution.ts. Migrated BuildTxCard + SignTypedDataTool to step machine. Removed pendingTx/txResult from chatSessionStore. Deleted useTransactionSigning.ts. All 345 tests pass, TS clean. Fixed pre-existing gap: added cw20 chain dispatch case. Needs manual testing: new tx flow, historical conversation load, conversation switching, tx_ready proposals.


**2026-04-23T23:55:05Z**

Closing: implementation commit fee8f58 already on origin/main. Manual verification subsumed by v-llqd dogfood (execute_* cards exercise useToolExecution, ApprovalContext, historicalToolGuard).
