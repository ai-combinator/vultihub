---
id: v-rkks
status: closed
deps: []
links: []
created: 2026-05-01T05:30:33Z
type: task
priority: 2
assignee: Jibles
---
# PR #339 cleanup: 5 review concerns + PR-wide comment squash

## Objective

PR #339 in vultiagent-app (branch `worktree-va-qbzv`) retires the client-side `recentActions` outbox and POSTs tool execution outcomes to the backend. Code review surfaced five concrete concerns concentrated in the transaction lifecycle hook, plus broad violations of the project comment rule across the PR's touched files. Address all five and apply a comment-density pass before the PR ships, while the persistence layer is untested in production and the wire shape is malleable.

## Context & Findings

### The five concerns

1. **Silent data loss is acknowledged, not fixed.** `usePostToolExecutionResult` retries 3× then `console.warn`s and drops. The old `recentActions` outbox was SecureStore-backed and durable; the new write isn't. Decision: ship best-effort — the alternative (durable outbox) is exactly the SecureStore lifecycle we're retiring. Want a short comment naming the trade-off so the next reader doesn't reintroduce a queue thinking it's an oversight.

2. **`cancelled` collapses into `failed` at hydration.** The wire carries a distinct `cancelled` status, but `useTransactionFlow`'s historical hydration buckets it into `outcome: 'failed'`. The reducer's cancelled visual treatment relies on string-sniffing the error message — a fragile contract that requires every POST callsite to send `error: 'Transaction cancelled.'` exactly. Make `cancelled` a first-class reducer outcome so the wire status is the source of truth.

3. **`approvalTxHash` is read from `phaseRef.current` instead of a local closure.** The new code couples to reducer state + flush timing for no real benefit. The old local-`let` pattern was simpler and bulletproof. The resume-from-`submitted` path doesn't go through `runSigning` (it's driven by `useReceiptPoll`), so the local closure is sufficient for all callsites that fire from inside `runSigning`. Restore the local var; remove the two `phaseRef.current.kind === 'signing'` reads.

4. **Four-ref dance to keep `postOutcome` stable is defensive over-engineering.** `postExecutionResultRef`, `tokenPublicKeyRef`, `conversationIdRef`, `authTokenRef` exist to avoid recreating `postOutcome` when context churns. But React Query's `mutation.mutate` is stable across renders within one hook lifetime, and the context values *should* propagate (a fresh authToken should bind into postOutcome, not be ignored by a stale ref). Drop the refs, inline the values, depend directly on `mutation.mutate`.

5. **`useReceiptPoll` lost its `toolCallId` parameter.** The hook now takes both `dispatch` and `onOutcome` and fires both on each terminal event — two side-effect callsites doing related work, with the hook itself unable to log/diagnose what tool call it's polling for. Cleaner: drop `dispatch` from the hook's contract, expose only `onOutcome`, let the parent map outcome → state + persistence in one place. The parent has `toolCallId` in scope for any logging it wants.

### The comment squash

Project rule (`/home/sean/Repos/vultisig/CLAUDE.md`): "Comments — WHY-only, ≤2 lines. Don't restate what the code does, don't reference the task/ticket/PR — those belong in the commit/PR body and rot in code. Don't mimic existing long comment blocks; shorten them when you touch them. The bar: would removing this confuse a future reader? If no, delete it."

The PR violates this extensively across ~20 files. Worst categories:
- Multi-paragraph file headers restating what a hook/component does
- Inline blocks describing reducer flow, race fixes, or state machine phases (restatement of WHAT)
- References to tickets/PRs/codenames that rot: `gomes #3 (PR #282 r3)`, `Codex SF#1`, `va-tbdw`, `va-qbzv`, `v-pxuw Task 13`, `PR #305`, `PR #234 R1 should-fix #2`, `Codex PR#222 P1`, `Issue #316`, `BLOCKER #4`, `neomaking's #176 M2`, commit hash `7fba772`
- Multi-line JSDoc on internal helpers

Specifically problematic blocks (line ranges from the worktree-va-qbzv branch HEAD at the time of inventory):
- `useTransactionFlow.ts:1–18` — 18-line file header restating hook job + `v-pxuw` / `v-llqd` / commit hash refs
- `useTransactionFlow.ts:49–54` — 6 lines restating code flow
- `useTransactionFlow.ts` — the "Async-hydration race fix (Codex SF#1)" multi-paragraph block describes a state of the world (`recentActions` async hydration) that this PR removes; delete wholesale
- `useToolExecution.ts:92–94` — refs `neomaking's #176 M2`
- `useToolExecution.ts:477–490` — 14-line BLOCKER block, refs `PR #305`
- `useAgentChat.ts:58–64` — 7-line block on AI SDK v5 transport
- `useAgentChat.ts:119` — refs `PR #234 R1 should-fix #2 (@NeOMakinG)`
- `txFlowReducer.ts:1–13` — 13-line header refs `v-pxuw Task 13`
- `ApprovalContext.tsx:34–64` — 31-line design-notes block, refs `PR #305`
- `ApprovalContext.tsx:95–96, 383, 596` — refs `Codex PR#222 P1`, `Issue #316`, `BLOCKER #4 PR #305`
- `SwapFlowCard.tsx:1–16`, `SendFlowCard.tsx:1–6`, `ContractCallCard.tsx:1–5` — file headers restating purpose
- `ScheduleTaskTool.tsx:47–64` — 18-line block, refs `va-tbdw`
- `chatSessionStore.ts:1–20` — 20-line JSDoc on (now-removed) caching strategy
- `historicalToolGuard.ts:4–9` — 6-line block restating purpose
- `useOtaUpdateController.ts:1–24` — 24-line JSDoc restating phase machine
- `useOtaUpdateController.ts:53–66` — block on defensive parsing
- `unsafeToRestartStore.ts:1–22` — 22-line refcount restatement
- `agentContext.ts:10–17` — 8-line block on address caching
- `txOutcomeApi.ts:1–20` — 20-line file header restating caching strategy

Apply the rule per file, collapse to ≤2 lines or delete. Where the comment was restating WHAT, default to delete.

### Decisions and rejected alternatives

- **Rejected: durable client-side outbox for issue 1.** The whole point of this PR is to retire the SecureStore-backed lifecycle. Reintroducing a queue defeats it. Best-effort + warn is the design.
- **Rejected: keep the embedded executionResult shape and just simplify the cache patcher.** Different scope — captured separately under ticket `v-aoov`. This ticket is read-side cleanup of the existing PR shape, not a wire rework.
- **Rejected: add an OTA mid-keysign `enter()/leave()` defensive bump.** Orthogonal to these five. Raise as a follow-up if the author agrees DKLS keysign should be guarded; not part of this cleanup.
- **Rejected: promote the project comment rule to global `~/CLAUDE.md`.** User chose to leave it project-scoped.
- **Rejected: split the work across multiple commits/PRs.** Lands as a single follow-up commit on the existing PR branch.

## Files

### Code + comment changes
- `src/features/agent/hooks/useTransactionFlow.ts` — issues 3, 4, 5 + squash
- `src/features/agent/hooks/useReceiptPoll.ts` — issue 5 + squash
- `src/features/agent/lib/txFlowReducer.ts` — issue 2 + squash
- `src/services/network/txOutcomeApi.ts` — issue 1 (comment) + squash

### Comment squash only
- `src/features/agent/hooks/useToolExecution.ts`
- `src/features/agent/hooks/useAgentChat.ts`
- `src/features/agent/contexts/ApprovalContext.tsx`
- `src/features/agent/components/tools/ExecuteCards/SwapFlowCard.tsx`
- `src/features/agent/components/tools/ExecuteCards/SendFlowCard.tsx`
- `src/features/agent/components/tools/ExecuteCards/ContractCallCard.tsx`
- `src/features/agent/components/tools/ScheduleTaskTool.tsx`
- `src/features/agent/stores/chatSessionStore.ts`
- `src/features/agent/lib/historicalToolGuard.ts`
- `src/features/updates/useOtaUpdateController.ts`
- `src/features/updates/unsafeToRestartStore.ts`
- `src/services/agentContext.ts`

### Reference
- `/home/sean/Repos/vultisig/CLAUDE.md` — Comments rule (verbatim source of truth for the squash bar)

## Acceptance Criteria

- [ ] `usePostToolExecutionResult` carries a ≤2-line comment naming the best-effort trade-off (no durable outbox, retries-then-drop). No behavior change.
- [ ] `cancelled` is a first-class outcome in the historical-hydration path: the reducer has a dedicated `cancelled` branch in `HYDRATE_HISTORICAL`, and `useTransactionFlow` routes `executionResult.status === 'cancelled'` into it. No string-sniffing the error message to detect cancellation.
- [ ] `approvalTxHash` is read from a local closure variable inside `runSigning`, not from `phaseRef.current.meta`. The two `phaseRef.current.kind === 'signing'` reads in `runSigning`'s broadcast and catch blocks are removed. Reopened approve+swap mid-broadcast still reconstructs the `[approve, sign, broadcast]` stepper (regression check).
- [ ] `postExecutionResultRef`, `tokenPublicKeyRef`, `conversationIdRef`, `authTokenRef` are removed from `useTransactionFlow`. Values are inlined in `postOutcome`'s `useCallback` with proper deps. A token refresh mid-flow rebinds `postOutcome` to the new value.
- [ ] `useReceiptPoll`'s contract is `{ phase, onOutcome }` — no `dispatch`, no `toolCallId`. The parent in `useTransactionFlow` maps `onOutcome` to dispatch + `postOutcome` in one place.
- [ ] All long comment blocks listed in Context & Findings are squashed to ≤2 lines or deleted, with all ticket/PR/codename refs removed. Each remaining comment passes the rule: "would removing this confuse a future reader?"
- [ ] No new comments introduced by these fixes restate WHAT or carry ticket refs.
- [ ] `npm run typecheck` and `npm run lint` clean.
- [ ] Full Jest suite green; specifically `txOutcomeApi.test.ts`, `useOtaUpdateController.test.ts`, and the EVM card test suites (`SendFlowCard.test.tsx`, `SwapFlowCard.test.tsx`, `ContractCallCard.test.tsx`).
- [ ] Manual smoke (with agent-backend #235 deployed): approve+swap killed mid-broadcast → reopen reconstructs Approve dot; cancel-at-consent → card renders distinctly as cancelled, not as a generic failure; plain confirmed swap end-to-end.

## Gotchas

- The "Async-hydration race fix (Codex SF#1)" block in `useTransactionFlow.ts` is describing the now-deleted `chatSessionStore` async hydration. It's actively misleading post-strip — delete wholesale, not trim.
- React Query's `mutation.mutate` is stable across renders within one hook lifetime; safe to put in dep arrays. The mutation **object** churns each render, the **method** does not. If anyone tells you otherwise, verify against the version in `package.json`.
- `useReceiptPoll` is the only consumer of the `submitted` phase's receipt-wait. When collapsing to `onOutcome`, ensure the parent's callback handles BOTH `success` (dispatch RECEIPT_CONFIRMED) and `failed` (dispatch SIGN_FAILED with `step: 'broadcast'`). Both paths must also call `postOutcome` for server-side persistence.
- `cancelled` POSTs today carry `error: 'Transaction cancelled.'` exactly. Don't drop the error string from the wire — display layers may still want it. The fix is making the **status** authoritative for the visual treatment, not removing the error.
- Resume-from-`submitted` does NOT enter `runSigning`. Issue 3's local-`let` approach is safe specifically because of this — the resume path lives in `useReceiptPoll`'s effect, with its own `onOutcome`. Verify before assuming a hydrated `approvalTxHash` needs to flow through `runSigning`.
- Comment squash should never delete a comment that names a non-obvious constraint, invariant, or workaround. If the comment is the only place a future reader would learn that some library has a quirk, keep it. Default-delete only applies to restatements of WHAT.
- Some files (e.g. `chatSessionStore.ts`) are mostly *deletions* in this PR. Squash any surviving comments to match the new (much smaller) shape — don't leave 20-line headers explaining a system that's been ripped out.
- All work is on `worktree-va-qbzv` in vultiagent-app. No backend (#235) changes. No cross-repo coordination. Lands as a follow-up commit on the existing PR.

## Notes

**2026-05-01T07:05:02Z**

All 5 issues + comment squash committed as d0310afc on worktree-va-qbzv. Spec + code-quality + final integration reviews all approved. Receipt-poll meta-carry MEDIUM finding deferred to a separate concurrent fix per author. Manual smoke (approve+swap mid-broadcast / cancel-at-consent / plain swap) still owed before PR ships.
