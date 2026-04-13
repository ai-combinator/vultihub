---
id: tic-6c08
status: open
type: task
priority: 2
assignee: Jibles
created: 2026-04-13T12:07:22.280484066Z
---
# Unified tool execution lifecycle with historical guard

## Objective
Consolidate the fragmented tool execution state (pendingTx/txResult/useToolLifecycle/useTransactionSigning/buildToolDetection) into a unified step-based execution model where tool components own their signing lifecycle — the same pattern shapeshift-agentic uses. Both apps have the same fundamental split (backend builds unsigned tx, frontend signs), but shapeshift handles it in one straight line (component gets output as prop → signs inline) while vultiagent routes through 4 hops via a global store. This ticket eliminates those hops.

## Context & Findings
Comparative analysis with shapeshift-agentic revealed that vultiagent-app's tool execution is split across 4-5 state containers and hooks doing what a single ToolExecutionState + useToolExecution pattern handles in ShapeShift. The ShapeShift pattern is signing-method-agnostic — TSS/MPC signing works fine inside a step machine; the "sign" step just takes longer.

**Current flow (store-based indirection):**
buildToolDetection scans messages → writes to chatSessionStore.pendingTx → BuildTxCard reads from store → useTransactionSigning reads from store → signs via 343-line callback → writes txResult/recentActions to store → BuildTxCard reads txResult from store

**Target flow (component-owned lifecycle, matches shapeshift-agentic):**
BuildTxCard receives tool output as prop from toolUIRegistry → calls useToolExecution(toolCallId) for step machine → calls requestApproval() from context to get password → calls signAndBroadcast() pure function → reports result via callback

**Key findings from deep analysis:**
- useTransactionSigning already accepts pendingTx as a parameter (90% decoupled) — only writes txResult/recentActions directly to store
- AgentApprovalBar is already fully controlled (no store access) — works with either pattern
- BuildTxCard is purely display-driven — reads pendingTx/txResult from store but could take props
- useConfirmationFlow (scheduled proposals) has ZERO overlap with tx signing — separate concern, no unification needed
- recentActions CANNOT be eliminated (backend doesn't receive message history) but can be callback-based
- A real bug exists: loading a conversation can re-trigger build_* tool detection because buildToolDetection scans ALL messages without distinguishing historical from new. The buildToolPendingTxIdRef guard only prevents the same tool from firing twice in a session — not historical tools from prior sessions.
- completeTxApproval is 343 lines with `messages` in its dependency array — every chat stream tick recreates it, thrashing the entire signing flow even when not signing. The `messages` dependency is only there for text-based chain detection fallback.

**Rejected approaches:**
- Promise-based approval modal: React Native doesn't have true blocking modals; use a context-based requestApproval() that sets state and resolves when user completes
- Eliminating recentActions: blocked by backend architecture (only receives current message, not history)

## Files
- src/features/agent/hooks/useTransactionSigning.ts — break into two pieces: (1) pure signAndBroadcast(parsedTx, vault, password) function with chain dispatch, (2) thin useApproval hook that manages password/biometric state and calls the pure function. Remove `messages` from deps (extract text-based chain fallback into a separate helper called once, not on every render).
- src/features/agent/hooks/useAgentChat.ts — remove auto-setPendingTx effect entirely; add historical tool ID tracking (Set populated on conversation load)
- src/features/agent/lib/buildToolDetection.ts — no changes needed (detection logic is fine, consumption changes)
- src/features/agent/stores/chatSessionStore.ts — remove pendingTx and txResult fields; these move to component-local state or tool execution context
- src/features/agent/components/tools/BuildTxCard.tsx — receive tool output as props from registry; own signing lifecycle via useToolExecution step machine; call requestApproval() from context for password
- src/features/agent/components/tools/SignTypedDataTool.tsx — same pattern: props, step machine, context-based approval
- src/features/agent/screens/AgentHomeScreen.tsx — provide ApprovalContext that tool components call into; remove pendingTx/txResult store reads and the hook wiring between useAgentChat and useTransactionSigning
- src/features/agent/components/AgentApprovalBar.tsx — driven by ApprovalContext state instead of parent prop drilling from store reads

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

## Gotchas
- The text-based chain detection fallback (regex-matches chain from conversation text when parseServerTx returns null) must be preserved but extracted — call it once when the tool output arrives, cache the result, don't recompute per render
- Biometric prompt is wrapped in InteractionManager.runAfterInteractions() to prevent navigation race — preserve this in the new useApproval hook
- The tx_ready path (scheduled proposals via getTxProposalApi) is separate from build_* detection — both need historical guard but have different data sources
- recentActions must still flow to backend on next message send — callback pattern changes WHERE it's written, not WHETHER
- ApprovalContext needs to handle the case where multiple tool components exist in the same conversation — only one approval at a time
