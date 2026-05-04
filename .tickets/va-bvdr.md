---
id: va-bvdr
status: open
deps: []
links: []
created: 2026-04-30T01:21:12Z
type: task
priority: 2
assignee: Jibles
---
# useTransactionFlow / txFlowReducer scope creep — decomposition options

## Problem Statement
`useTransactionFlow` (489 lines) and `txFlowReducer` (387 lines) own 6 distinct responsibilities for a single tool-call lifecycle. Reducer carries domain inferences and a UI-copy helper; hook hard-codes one execution shape (ERC-20 approve + main tx); persistence calls are scattered through every signing branch. "Solved" means the file owns one concern, new execution shapes can be added without editing the phase machine, and the reducer is purely about transitions.

## Research Findings

### Current State — vultiagent
- **Reducer** (`src/features/agent/lib/txFlowReducer.ts`, 387 lines): pure, discriminated-union phases — `fetching_prep | awaiting_confirm | signing | submitted | confirmed | failed | incomplete`. 11 event types: `HYDRATE_HISTORICAL`, `PREP_RECEIVED`, `IDLE_TIMEOUT_FIRED`, `SIGNING_STARTED`, `SIGN_STEP_COMPLETED`, `BROADCAST_ACCEPTED`, `RECEIPT_CONFIRMED`, `SIGN_FAILED`, `PASSWORD_INVALID`, `CANCELLED`, `RETRY`.
- **Hook** (`src/features/agent/hooks/useTransactionFlow.ts`, 489 lines) does:
  1. Reducer wiring + `phaseRef` (lines 99–110)
  2. Hydration: subscribes to `chatSessionStore.recentActions`, derives outcome, dispatches `HYDRATE_HISTORICAL` (lines 142–192)
  3. Receipt polling for `submitted → confirmed/failed` via `useReceiptPoll` (line 196)
  4. Signing driver `runSigning` — hard-codes optional ERC-20 approve + main tx under one `requestApproval` claim (lines 207–360); branches on `tx_encoding === 'evm'`; inlines four `chatSessionStore` mutations
  5. Approval-queue wiring: `setPendingApproval` / `clearPendingApprovalIf` / `requestApproval` (lines 404–472)
  6. Idle timeout + retry glue (lines 384–392, 477–480)

### Specific scope-creep symptoms
- **Reducer carries business inference**: `HYDRATE_HISTORICAL` infers `completedSteps` from `approvalTxHash` presence (txFlowReducer.ts:142–144); `cancelled` flag is set by sniffing the error string in `HYDRATE_HISTORICAL` and `CANCELLED` (txFlowReducer.ts:154, 305).
- **UI copy in the reducer file**: `getUserFriendlyError` (txFlowReducer.ts:376–387) is pure presentation, no state involvement.
- **`tx_encoding === 'evm'` knowledge split** between reducer (HYDRATE) and hook (`waitForEvmReceipt` skip, RECEIPT_CONFIRMED short-circuit on non-EVM at lines 283, 324).
- **Scattered persistence**: `useChatSessionStore.getState().{startPendingAction,submitAction,completeAction,failAction}` called from 5 different points inside `runSigning` and `requestApproval`'s `onCancelled`.
- **Hard-coded execution shape**: "optional approve, then main tx, single claim" baked in. A 3-step flow or non-`ExecutePrepResult` input requires editing `useTransactionFlow` itself.

### Validated Behaviors (current contract — must preserve)
- Single `requestApproval` claim covers BOTH approve + main tx (deliberate per v-pxuw `7fba772`; splitting forces the approval bar to remount mid-flow). Comment at lines 13–18.
- `recentActions` hydrates async from SecureStore — hook MUST subscribe reactively, not read once. A late hydration upgrades a card from `incomplete` to its real terminal state. Codex SF#1 fix at lines 129–141.
- `HYDRATE_HISTORICAL` is idempotent on stable inputs; re-firing on late prep is safe (line 189).
- `passwordVerified` flag inside `runSigning` (line 245): a transient 5xx from `verifyFastVaultPassword` between approve and main tx must not surface as "Incorrect vault password" after approve already broadcast.
- `submitted → failed` path sets `canRetry: false` (tx is on-chain; re-broadcasting same nonce conflicts). txFlowReducer.ts:261–272.
- `incomplete` is distinct from `failed` so the UI doesn't show REJECTED for a tx the user never actively rejected.
- Empty-string error and `APP_KILLED_ERROR` route to `incomplete`, not `failed` (useTransactionFlow.ts:163–168).

### External Context — shapeshift-agentic (`../shapeshift-agentic/apps/agentic-chat/src`)
A different decomposition for the same problem class:

- **`hooks/useToolExecution.ts`** (181 lines) — generic step container. Holds runtime state in zustand. Exposes mutation API: `advanceStep`, `skipStep`, `setMeta`, `setSubstatus`, `markTerminal`, `persist`, `failAndPersist`. Plus `refs` to wallet/address/conversation. No opinion on what a step *means*.
- **`hooks/useExecuteOnce.ts`** (44 lines) — single-fire guard: skips if persisted, skips if no data, skips if already running; otherwise calls `executor(data, ctx)`.
- **Per-tool callers** own the imperative flow. `components/tools/useSwapExecution.tsx` (185 lines): `withWalletLock(async () => { try { … step 0 quote, step 1 network switch (switchNetworkStep), step 2 approve (ensureAllowance + waitForConfirmedReceipt), step 3 swap (executeSwap + waitForConfirmedReceipt), markTerminal, persist, analytics, toast } catch { failAndPersist } })`. Steps are numeric (`SWAP_STEPS = { QUOTE: 0, NETWORK: 1, APPROVE: 2, SWAP: 3 }`).
- **State shape** (`lib/executionState.ts`): generic `{ currentStep: number, completedSteps: number[], skippedSteps: number[], failedStep?, error?, terminal, meta }`. Per-tool `Meta` types in a `ToolMetaMap`. Status derived by numeric position in `getStepStatus`.

### Comparison summary

|                        | vultiagent                                              | shapeshift                                                     |
|------------------------|---------------------------------------------------------|----------------------------------------------------------------|
| State                  | typed phase union                                       | numeric step + arrays                                          |
| Updates                | pure reducer + 11 events                                | direct draft mutation in zustand                               |
| Where execution lives  | inside the hook (`runSigning`)                          | per-tool caller                                                |
| Step semantics         | closed set: `'approve' \| 'sign' \| 'broadcast'`        | open numeric, per-tool enum                                    |
| Resume mid-poll        | explicit `submitted` phase + `useReceiptPoll`           | not modeled (inline await)                                     |
| Terminal model         | `failed{canRetry, cancelled, lastStep}` + `incomplete`  | `terminal: true` + optional `error`/`failedStep`               |
| Hydration              | reactive subscription                                   | one-shot `initializeRuntimeState`                              |
| Approval UX            | tightly coupled to `ApprovalContext` queue              | none (Dynamic SDK signs inline)                                |
| Per-tool extension     | edit the hook + reducer                                 | write a new caller hook                                        |

## Constraints
- Must preserve single-claim multi-tx invariant (one `requestApproval` covers both approve + main tx).
- Must preserve reactive `recentActions` subscription; late hydration upgrades phase.
- Must preserve `submitted → confirmed/failed` resume-on-reopen via `useReceiptPoll`.
- Must preserve `incomplete` vs `failed` distinction.
- Must preserve `passwordVerified` short-circuit so a transient verify-endpoint 5xx doesn't surface as a wrong-password error post-approve-broadcast.
- Cannot regress `getUserFriendlyError` copy (kept in sync with legacy helper per file comment).
- Public API of the hook (`{ phase, onRetry }`) is consumed by card components — changes here have UI fanout.

## Dead Ends
- **Wholesale port to shapeshift's generic numeric-step model**: loses the `failed{canRetry, cancelled}` distinctions, the `incomplete` phase, and the typed transitions that prevent invalid states. Vultisig's UX requirements (cancel chip vs failed chip, resume-mid-poll) are richer than what a numeric-step model expresses cleanly.
- **Splitting the single `requestApproval` claim into approve-then-sign sequential claims**: explicitly rejected upstream (v-pxuw `7fba772`); forces approval-bar remount mid-flow.

## Open Questions
- **Reducer purity**: should `getUserFriendlyError` move to a UI helper module? Should `HYDRATE_HISTORICAL` accept an explicit `cancelled: boolean` and an explicit `completedSteps: SigningStep[]` instead of inferring them, pushing the inference into a `chatSessionStore` selector?
- **Persistence boundary**: replace the 5 scattered `chatSessionStore` calls inside `runSigning`/`onCancelled` with a single `useTxPhasePersistence(phase, toolCallId)` effect that reacts to phase transitions? Or keep call-sites explicit so the order relative to `dispatch` stays obvious (current code persists *before* `BROADCAST_ACCEPTED` deliberately)?
- **Execution driver extraction**: pull `runSigning` into a `useExecutePrepSigner(prep, dispatch)` hook so `useTransactionFlow` shrinks to phase + hydration + queue wiring + idle timeout? Or extract one level higher — a generic `useTxExecutor(buildSteps)` that callers parameterize, closer to shapeshift's split?
- **Approval-queue wiring extraction**: split into `usePendingApprovalRegistration(phase, onApprove)` so consent-UX glue lives next to `ApprovalContext`? Trade-off: more files, but each focused.
- **EVM/non-EVM branching**: centralize the `tx_encoding === 'evm'` decision (currently in reducer HYDRATE + hook two places) into a single helper, e.g. `requiresReceiptPoll(prep)`?
- **Scope of this work**: one bundled refactor (touches reducer, hook, and persistence callsites) or staged (1: pull pure helpers out, 2: persistence subscriber, 3: extract executor)?
