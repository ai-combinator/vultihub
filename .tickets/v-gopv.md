---
id: v-gopv
status: closed
deps: []
links: []
created: 2026-04-29T01:23:02Z
type: task
priority: 1
assignee: Jibles
---
# Replace auto-fire approval with explicit-tap input-bar approval

## Objective

Remove the auto-fire-approval anti-pattern from both transaction-flow hooks. Approval today is claimed as a side-effect of rendering — a `useEffect` calls `requestApproval` when a card enters `awaiting_confirm`. Anything that re-runs that effect (password keystroke, slot release/claim cycle, StrictMode double-mount, context churn) can re-pop the credential prompt without a fresh user tap. Replace it with an explicit-tap path: the bottom approval bar gains a render-driven "ready" state showing a per-tx summary + a tick; tapping the tick is the only thing in the codebase that claims the credential slot. This addresses gomes' R2 #1 blocker on PR #242 (vultiagent-app) and fixes the same class of bug already in production for `build_*` tools.

## Context & Findings

- **Why this is the open blocker.** gomes' R2 review (vultiagent-app PR #242, 2026-04-28T21:19) marks Concern A "STILL OPEN": the new `useTransactionFlow.ts:388` consent-arm effect reenacted the prod anti-pattern from `useToolExecution.ts:279`. R1's ask was to remove auto-fire-on-mount entirely; R2's response added more guards on top of broken inference. Recent main commits like "close re-entrancy window in auto-fire approval" and "do not re-arm auto-fire on timeout recovery" are the same pattern of guards-over-inference.

- **Architectural shape of the bug.** The system uses a UI rendering event ("prep arrived") as a substitute for user intent ("user wants to approve"). They aren't the same thing. Refs and dep-array narrowing suppress symptoms but the underlying confusion remains. The fix is to make a user tap the only trigger that claims the credential slot — never a render effect.

- **Bar visibility ≠ credential slot.** Today the AgentApprovalBar appears when `requestApproval` claims the slot. After the fix, the bar appears in a new render-driven `kind: 'ready'` state whenever a card is in `awaiting_confirm`. That's harmless display state — re-rendering it costs nothing. The credential slot (`kind: 'signing'`) is still claimed exclusively, but only from the tick's `onPress` handler. Conflating these two concepts is exactly what creates the bug class.

- **Multi-card support.** Pending approvals live as a queue in `ApprovalContext`. Queue tail = displayed pending. "Most recent in chat" falls out of React's effect-ordering naturally (cards mounted later run their effects last). When a card is signed or cancelled, its queue entry is removed and the new tail (the next-most-recent pending) automatically surfaces in the bar — no manual coordination.

- **Why the queue lives in `ApprovalContext`, not `chatSessionStore`.** Each queue entry carries an `onApprove` closure over the card's hook state. Closures fit naturally in React state; they fit awkwardly in zustand (not serialisable, dangling refs after unmount). Approval-related state belongs together — `pendingApprovals` queue alongside the existing `activeApproval` slot.

- **Summary on the bar.** Today the signing bar shows generic "Confirm Transaction" / "Enter Vault Password" copy with zero tx context. Wallet-app bad smell. While we're already touching the bar, render a per-card summary string ("Swap 0.229 USDC → ETH on Arbitrum", "Approve LINK for Kyber on Arbitrum") in **both** the new `ready` state and the existing `signing` state. The summary travels via the queue entry → `requestApproval` payload → bar props.

- **Decisions evaluated and rejected.**
  - On-card "Approve" button (instead of input-bar tick): valid architecturally but doesn't match product designs and doesn't match gomes' inferred placement. Input-bar tick is the chosen UX.
  - Lift queue to zustand `chatSessionStore`: rejected because of closure storage. Could be done with a split (metadata in store, handlers in context) but doubles the wiring with no win.
  - Retire BuildTxCard / ship unified card in this PR (gomes' Concern B): explicitly deferred. Different concern (UX consistency, not wallet safety). Different prep shape (`build_*` doesn't emit `ExecutePrepResult`). Multi-PR backend (mcp-ts) coordination needed. Ticket v-ujuc tracks it; this ticket addresses Concern A only.
  - Skip / out-of-order approval: out of scope. MVP rule is "approve in chat order"; queue model trivially extends to a skip later.

- **Verification recipe (manual).** USDC → ETH on Arbitrum with revoked Kyber allowance → approve broadcasts on-chain → bar's `ready` state surfaces with summary → tap tick → biometric/password → signs and broadcasts. On-chain: exactly 1 approve + 1 swap. Two-card test: kick off two swaps in one turn → bar shows summary of the second → approve it → bar surfaces summary of the first → approve it → both broadcast cleanly with no double-approve.

- **Already-pinned regression coverage.** `txFlowReducer.test.ts` has a test "broadcast_failed on sign leg → retry preserves approve completion (no double-broadcast)" — that one stays as-is and continues to guard the rebroadcast property after this restructure.

## Files

Repo: `vultiagent-app`. PR branch: `surgical/v-llqd-execute-cards`.

- `src/features/agent/hooks/useTransactionFlow.ts` — delete the consent-arm `useEffect` (~line 388-443) and its `consentRegisteredRef`; make the existing `requestApprovalExplicit` (~line 448-472) the only entry point that calls `requestApproval`; expose summary for queue registration
- `src/features/agent/hooks/useToolExecution.ts` — delete the auto-fire `useEffect` (~line 289-296) and `autoFiredRef`; replace with an explicit method consumed by the bar tap; preserve unrelated guards/refs (idle timeout, abort, etc.)
- `src/features/agent/lib/txFlowReducer.ts` — remove `requiresExplicitTap` from the `awaiting_confirm` variant and from the `PREP_RECEIVED` handler
- `src/features/agent/contexts/ApprovalContext.tsx` — add `pendingApprovals` queue with `setPendingApproval(p)`, `clearPendingApprovalIf(toolCallId)`, exposed `pendingApproval = queue[length-1]`; existing `requestApproval` semantics unchanged
- `src/features/agent/components/AgentApprovalBar.tsx` — extend discriminated union with `kind: 'ready'`; render summary in both `ready` and `signing`; tick `onPress` is the only `requestApproval` caller
- `src/features/agent/components/tools/BuildTxCard.tsx` — adapt to tap-driven model; register summary with the pending queue when in awaiting state
- `src/features/agent/components/tools/ExecuteCards/SendFlowCard.tsx` — register summary with queue on `awaiting_confirm`; remove on-card "Review and approve" button (was conditional on `requiresExplicitTap`, now dead)
- `src/features/agent/components/tools/ExecuteCards/SwapFlowCard.tsx` — same shape as Send
- `src/features/agent/components/tools/ExecuteCards/ContractCallCard.tsx` — same shape as Send
- `src/features/agent/contexts/__tests__/signingApproval.integration.test.tsx` — significant rewrite: existing scenarios assume auto-arm; replace with tap-arm flows that drive consent through the bar tick (vault-switch safety, wrong-password recovery, slot exclusivity all need to start from a user tap rather than render-driven claim)
- `src/features/agent/lib/__tests__/txFlowReducer.test.ts` — drop tests covering `requiresExplicitTap` transitions
- `src/features/agent/components/tools/ExecuteCards/__tests__/SendFlowCard.test.tsx` — drop `requiresExplicitTap === true` branches; add queue-registration assertions
- `src/features/agent/components/tools/ExecuteCards/__tests__/SwapFlowCard.test.tsx` — same
- `src/features/agent/components/tools/ExecuteCards/__tests__/ContractCallCard.test.tsx` — same

## Acceptance Criteria

- [ ] Zero `useEffect` in the codebase calls `approve()` or `requestApproval()` (verified by grep)
- [ ] `requiresExplicitTap` flag and all references deleted (production code + tests)
- [ ] `AgentApprovalBar` has a `kind: 'ready'` state showing a per-tx summary + a tick button; tick `onPress` is the only `requestApproval` caller in the app
- [ ] `AgentApprovalBar`'s existing `kind: 'signing'` state also renders the per-tx summary so the user sees what they're confirming during password/biometric
- [ ] `ApprovalContext` exposes a pending-approvals queue (cards register `{ toolCallId, summary, onApprove }` on entering `awaiting_confirm`, deregister on leaving); queue tail is what the bar displays
- [ ] Cleanup is id-matched: a card leaving `awaiting_confirm` (or unmounting) only removes its own queue entry; an older unmounting card cannot wipe a newer card's slot
- [ ] Multi-card flow: when the displayed pending approval is signed or dismissed, the next-most-recent card surfaces in the bar automatically
- [ ] Retry flow: a card re-entering `awaiting_confirm` after `SIGN_FAILED → RETRY` re-registers with the queue
- [ ] BuildTxCard works through the new tap-driven model — `build_*` and `morpho_prepare_*` tools no longer auto-pop the credential bar
- [ ] All updated tests pass: reducer, integration, and 4 card test files green
- [ ] Lint and type-check pass
- [ ] Manual smoke (single card): USDC → ETH on Arbitrum with revoked allowance → approve broadcasts → bar `ready` state shows summary → tap tick → password/biometric → signs/broadcasts → exactly 1 approve + 1 swap on-chain
- [ ] Manual smoke (multi-card): two swaps queued in one turn → bar shows summary of the second → approve it → bar then shows summary of the first → approve it → both broadcast cleanly with no double-approve

## Gotchas

- Closure storage: `onApprove` in queue entries is a closure over each card's hook state — keep the queue in `ApprovalContext` (React state), don't lift to `chatSessionStore` (zustand).
- Cleanup race: `clearPendingApprovalIf(toolCallId)` must match by id and only clear if the queue entry still belongs to the deregistering card.
- "Most recent" depends on React effect-ordering = render order = chat scroll order. If virtualisation/reordering ever changes that, the rule changes.
- `AgentApprovalBar` is owned by PR #229 (already merged into main). Extending its discriminated union touches that surface but doesn't break the existing `signing` / `confirmation` contracts.
- Bar visibility (`kind: 'ready'`) is render-driven and cheap; bar slot claim (`kind: 'signing'`) is exclusive and fires credentials. Don't conflate them in the bar's logic.
- `useToolExecution.ts:279`'s `autoFiredRef` and the comment block at lines 250-256 are legacy guards. Delete both — don't preserve as "just in case."
- Pass summary through `requestApproval`'s payload so the bar already has it on the ready → signing transition (no prop-drilling needed).
- gomes' Concern B (BuildTxCard retirement / unified card) is intentionally out of scope here. v-ujuc tracks it. Do not let the BuildTxCard auto-fire fix balloon into a card-rendering refactor.
- `signingApproval.integration.test.tsx` currently has scenarios assuming auto-arm. Tap-arm rewrite isn't a one-line change — budget meaningful test work.

## Notes

**2026-04-29T01:56:29Z**

Production refactor + tests complete. 1480 unit + 42 component tests pass. tsc 0 errors. biome clean. Acceptance greps empty: no useEffect calls approve/requestApproval, requiresExplicitTap removed, auto-fire refs in signing path deleted (ScheduleTaskTool's confirmation-path autoFiredRef is out-of-scope per ticket). Now running review loop.

**2026-04-29T02:13:44Z**

Committed eaefb06. 1480 unit + 43 component tests pass; tsc 0; biome clean. Approved by spec compliance + final integration review (2 review iterations: BLOCKER on retry-stranding + minor SignTypedData CTA scope, both fixed).
