---
id: v-tfhp
status: closed
deps: []
links: []
created: 2026-04-23T23:56:08Z
type: task
priority: 2
assignee: Jibles
---
# vultiagent-app: Unify schedule approval onto shared input-bar consent flow

## Objective
Bring `ScheduleTaskTool` into the two-phase approval flow that send/swap adopted on branch `feat/action-card-send-swap`. Schedule preview becomes a read-only card; approval moves to the bottom input bar ("Approve or edit the action…"). Restores design parity with the DCA Schedule designs (FLOW 3) and finishes the migration that commit `e92dc99` explicitly deferred when it narrowed `AgentConfirmationCard` to schedule-only.

## Context & Findings
- `ScheduleConfirmationCard.tsx:54-95` currently hosts its own Approve/Edit buttons and "Looks good to you?" copy. `ScheduleTaskTool.tsx` runs `useApproveSchedule` locally and never touches `ApprovalContext`. Send/swap moved to the shared consent bar in `e550a78`; schedule was left behind.
- Schedule approval is **password-less**. `approveSchedulePreviewApi` (`src/services/network/agentApi.ts:114-125`) just hits `POST /agent/scheduled-tasks/approve/:previewId` to promote a Redis preview to a real schedule. No vault signature, no biometric. Unlike send/swap, the consent flow must skip `promptBiometricOrPassword`.
- `ApprovalContext.ConsentRequest` (`src/features/agent/contexts/ApprovalContext.tsx:17-23`) is shaped around `onPassword(password: string)`. A schedule consent needs an `onApproved()` callback that carries no password — either a new `phase: 'schedule'` branch or a no-password consent variant.
- `AgentApprovalBar.tsx:148-169` already handles the edit-text path via `onEditAction → chat.doSend(text)`; works for schedules unchanged (user edits the natural-language ask, backend rebuilds the preview).
- `tryClaimApprovalSlot` is single-occupancy. Until the paired backend ticket lands, the LLM may still emit a parallel `build_*` that wins the slot first. Mirror the graceful-degrade pattern in `useToolExecution.ts:286-293` (revert auto-fire ref on `granted === false`) — do NOT add sibling-tool render suppression, that hack was explicitly rejected.
- Reference pattern for `requestConsent` wiring: `src/features/agent/hooks/useToolExecution.ts:168-270`.

## Files
- `src/features/agent/components/ScheduleConfirmationCard.tsx` — strip Approve/Edit buttons + "Looks good to you?". Presentation-only.
- `src/features/agent/components/tools/ScheduleTaskTool.tsx` — replace in-component buttons + mutation-on-press with `requestConsent`; mutation runs inside the consent `onApproved` callback.
- `src/features/agent/contexts/ApprovalContext.tsx` — extend `ConsentRequest` for no-password consent (or add `requestScheduleConsent`); `approvalPhase` may gain `'schedule'`.
- `src/features/agent/lib/txSigning.ts` — if the phase enum changes, update reducer + initial state.
- `src/features/agent/components/AgentApprovalBar.tsx` — schedule-appropriate label copy; ensure the checkmark path renders without a password field.
- `src/features/agent/screens/AgentHomeScreen.tsx:245-273` — verify `AgentBottomBar` renders the shared bar for schedule consent with no further changes.

## Acceptance Criteria
- [ ] Schedule preview renders as a read-only card (no buttons, no "Looks good to you?" copy).
- [ ] When a schedule preview lands, the bottom bar enters consent with a schedule-appropriate label.
- [ ] Tapping the checkmark promotes the preview via `approveSchedulePreviewApi` with no biometric/password prompt.
- [ ] Cancel clears consent and leaves the schedule preview card intact but inert.
- [ ] Editing through the input bar sends a new chat message (existing `onEditAction → chat.doSend` path) and the backend rebuilds the preview.
- [ ] On approval success, `ScheduleAddedBanner` replaces the card (existing `approved` state).
- [ ] Historical/replayed turns with `output.consumed === true` still render the banner and do not re-prompt.
- [ ] Lint + type-check pass.
- [ ] Manual smoke: full DCA flow end-to-end matches the DCA Schedule designs on iOS and Android.

## Gotchas
- Schedule consent is password-less — do NOT route through `promptBiometricOrPassword`.
- Slot is single-occupancy; if a parallel `build_*` fires from the backend (until the paired ticket lands), `requestConsent` returns `granted: false` — handle gracefully, do not add sibling checks in `AssistantMessageView.tsx`.
- Preserve `output.consumed` historical branch in `ScheduleTaskTool`.

## Notes

**2026-04-24T00:13:09Z**

Implemented: discriminated-union ConsentRequest (kind: 'tx' | 'schedule'), schedule skips promptBiometricOrPassword, slot-conflict revert mirrors useToolExecution, error path clears slot to avoid stale password sheet. Label fix from review (normalized.title fallback was unreachable). 46 test suites pass, lint + typecheck clean. Commits: 66d89c0, c77ab10.
