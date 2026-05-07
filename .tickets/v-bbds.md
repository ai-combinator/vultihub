---
id: v-bbds
status: open
deps: []
links: []
created: 2026-04-30T01:35:21Z
type: chore
priority: 3
assignee: Jibles
---
# execute_* card flow: collapse parallel-truth state model into pointer + steps

## Objective

Replace the discriminated-union `TransactionPhase` model with a single uniform shape: a static `steps` list, a `currentStep` pointer, and a small `outcome` enum. The current 7-kind union encodes the same lifecycle twice (macro `phase.kind` and micro `step`/`completedSteps`/`lastStep`), which has produced load-bearing fixes (the "completed wins over lastStep" order-of-checks rule, hydrate inferring `completedSteps` from `meta` field shape) and forced `getEffectiveSteps` and `stepStatusForKind` to special-case the `confirm` dot and the `fetching_prep` no-data branch. PR #296 surfaced this; collapsing the model removes those smells structurally.

## Context & Findings

**Why now:** PR #296 introduced the `submitted/confirmed/failed/cancelled` vocabulary. Doing the model rewrite next, while that vocabulary is still fresh, is cheaper than doing it concurrently — keeping #296 focused on the user-visible va-tbdw fix.

**Target shape:**
```ts
type TxFlow = {
  prep: ExecutePrepResult | null              // null = stream not yet parseable (replaces 'fetching_prep' kind)
  steps: StepKind[]                            // ordered, set once when prep arrives
  currentStep: StepKind | null                 // active step; null = not-started or terminal
  outcome: 'pending' | 'confirmed' | 'failed' | 'cancelled' | 'incomplete'
  meta: ExecutionMeta
  error?: string
  canRetry?: boolean
  timedOut?: boolean
}
```

Per-dot status derives from cursor position + outcome:
- index < indexOf(currentStep) → COMPLETE
- index === indexOf(currentStep) && outcome === 'pending' → IN_PROGRESS
- index === indexOf(currentStep) && outcome === 'failed' → FAILED
- index > indexOf(currentStep) → NOT_STARTED
- currentStep === null && outcome === 'confirmed' → all COMPLETE

Structural invariant: at most one step is non-COMPLETE/NOT_STARTED at any time. The "completed wins over lastStep" rule becomes structurally unrepresentable.

**What collapses:**
- `getSigningStepStatus`, `stepStatusForKind`, `phaseToStatusKind` → trivial cursor lookups
- `getEffectiveSteps` → disappears; `confirm` is just another entry in `steps`, decided once when prep arrives based on `hasReceiptPoll(chain)`
- `completedSteps`, `step`, `lastStep` → all collapse into `currentStep`
- `HydrateOutcome` and runtime outcome → merge into one type
- `getPrep(phase)` helper → disappears; `prep` is always at the top level
- `incomplete` phase → just an outcome value

**Rejected approaches considered:**
- `{ id, status }[]` per-step array: over-encodes — type allows two IN_PROGRESS or COMPLETE-after-IN_PROGRESS. Cursor model is tighter.
- Folding into PR #296: adds ~1200 lines to an already-large PR (+1786/-239, 37 files), entangles the user-visible bug fix with model rework, and worsens bisect surface for any production regression.

**Two real risks:**
1. **Persistence backward-compat.** `recentActions` is persisted on-device in the old shape. Hydrate path must migrate old payloads (the existing inference — `txHash` set → broadcast completed, `approvalTxHash` set + no txHash → only approve completed — becomes the migration logic). Test against real on-device payloads from main before merging.
2. **Behavior preservation in transitions.** PASSWORD_INVALID preserves `completedSteps` so a successful approve isn't re-broadcast — re-encode as "PASSWORD_INVALID preserves currentStep, doesn't rewind cursor." Same observable behavior.

## Files

- `src/features/agent/lib/txFlowReducer.ts` — full type + reducer rewrite (~10 transition cases)
- `src/features/agent/lib/__tests__/txFlowReducer.test.ts` — sweep all `phase.kind === '...'` patterns
- `src/features/agent/hooks/useTransactionFlow.ts` — translate ~20-30 phase reads; dispatches stay the same
- `src/features/agent/components/tools/ExecuteCards/shared.tsx` — delete `getEffectiveSteps`, `stepStatusForKind`; collapse `phaseToStatusKind` to outcome lookup; `confirm` step appended at prep-arrival time in the reducer
- `src/features/agent/components/tools/ExecuteCards/SwapFlowCard.tsx`, `SendFlowCard.tsx`, `ContractCallCard.tsx` — phase-read translations
- `src/features/agent/components/tools/SignTypedDataTool.tsx`, `BuildTxCard.tsx`, `ScheduleConfirmationCard.tsx` — same
- `src/features/agent/stores/chatSessionStore.ts` — persistence shape change + hydrate migration
- `src/features/agent/lib/classifyExecutionError.ts`, `errorRowFor.ts` — small helper updates
- All affected component test files (SwapFlowCard, SendFlowCard, ContractCallCard, shared)

## Acceptance Criteria

- [ ] New `TxFlow` shape lives where `TransactionPhase` did; old type removed
- [ ] At most one step has IN_PROGRESS/FAILED status at any time, enforced by construction (not convention)
- [ ] `getEffectiveSteps` deleted; `confirm` step decided once at prep-arrival, not on each render
- [ ] `getSigningStepStatus` and `stepStatusForKind` collapse into a single cursor-based helper
- [ ] Hydrate path migrates old persisted `recentActions` payloads to the new shape — verified against a real on-device payload exported from a main-branch build
- [ ] PASSWORD_INVALID, RETRY, CANCELLED, BROADCAST_ACCEPTED, RECEIPT_CONFIRMED, SIGN_FAILED transitions all preserve current observable behavior (existing reducer tests pass after pattern translation)
- [ ] `submitted → failed` (post-broadcast revert) still renders Broadcast green / Confirm red without any "completed wins over lastStep" workaround
- [ ] `tsc --noEmit` clean across the agent feature
- [ ] All component tests pass (SwapFlowCard, SendFlowCard, ContractCallCard, shared)
- [ ] All reducer tests pass
- [ ] Manual smoke: real Arbitrum swap (success and OOG-revert paths) renders identically to PR #296 receipts

## Gotchas

- `BROADCAST_ACCEPTED` currently rolls both 'sign' and 'broadcast' into completedSteps to allow dispatch from either signing(sign) or signing(broadcast) — under the cursor model this becomes "advance currentStep past 'broadcast'", same effect, simpler.
- `SIGN_FAILED` with phase.kind === 'submitted' must produce outcome: 'failed' with currentStep: 'confirm' (NOT 'broadcast') so the Confirm dot picks up FAILED. This is the structural fix for the order-of-checks bug.
- Cancellation sentinel: `error === 'Transaction cancelled.'` exact-match still applies for hydrate-classification (don't substring-match).
- `IDLE_TIMEOUT_FIRED` only applies when `prep === null` — the `timedOut` flag stays a sibling of prep, not folded into outcome.
- Tests assert observable transitions; codemod or careful sed gets most of them, but watch for tests that pattern-match on `phase.kind` field shape rather than just transition outcomes.

## Notes

**2026-05-05T00:19:15Z**

demoted p2 → p3 (meh bucket: internal refactor / polish, no user-facing payoff pre-launch)
