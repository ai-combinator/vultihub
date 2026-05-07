---
id: v-bvin
status: closed
deps: []
links: [va-qbzv, v-duhk, v-fjph]
created: 2026-05-06T03:54:12Z
type: bug
priority: 2
assignee: Jibles
---
# ScheduleTaskTool cancellation race: barrier the cancel-side write on approve-in-flight

## Problem

`ScheduleTaskTool` has a race between approve-side writes and cancel-side writes that can leave the database internally inconsistent. User taps Confirm on a recurring schedule (e.g. DCA buy), `approveMutation` fires to commit the schedule on the backend. Before approve responds, ANY trigger of `cancelAllApprovals` (back button, conversation switch, force-quit cleanup, programmatic auto-cancel) drives `onCancelled`, which POSTs `status: "cancelled"` to `/agent/messages/:msgId/tool-results/:toolCallId/execution`.

The two writes are independent: `approveMutation` lands on the scheduler tables, the cancel POST lands on `tool_call_results`. Nothing synchronizes them. Backend can have already committed the schedule by the time cancel POSTs, so the database ends up with: scheduler row active + tool_call_results status=cancelled. Chat history reads from `tool_call_results` → card renders "Schedule cancelled" → user believes it's gone → next Monday at noon scheduler fires and the user gets a tx confirmation push for a schedule they thought they killed. Reality and conversation diverge.

For non-money flows: annoying. For DCA buys: real "I told you to stop, why did you spend my money" moment.

"Solved" means: cancel can never POST a status that contradicts what approve actually did. The tool_call_results outcome reflects approve's truth.

## Background

### Findings

- The card lives at `src/features/agent/components/tools/ScheduleTaskTool.tsx`. It uses `useApproveSchedule` for `approveMutation` and a `useConfirmation` flow that exposes `cancelAllApprovals` / `onCancelled`.
- `onCancelled` posts via the same `usePostToolOutcome(toolCallId, messageId)` hook every other persisted-result tool uses (after `v-duhk` consolidated 10 callers behind that hook).
- The race window is `approveMutation` round-trip time, ~200–800ms typical, much worse on bad network. Triggers in real usage: slow network (user perceives stuck, bails), iOS suspend/resume during approve, programmatic auto-cancel on conversation-switch.
- The hydration half of `ScheduleTaskTool` (reading `executionResult` on historical mount via `resolveToolCardSource`) already shipped in `v-fjph` (commit `32b3c081`) — that work is complete and in scope only as the consumer of whatever status this ticket lands on.
- `usePostToolOutcome` (after `v-duhk`'s `0d714007`) is fire-and-not-quite-forget — it enters the `unsafeToRestartStore('tool_result_post')` critical section and uses `mutateAsync`, but the cancel-side handler in `ScheduleTaskTool` still calls `postOutcome({ status: 'cancelled' })` synchronously as soon as `cancelAllApprovals` fires.
- `approveMutation` is React Query — `.isPending`, `.isSuccess`, `.isError` are reliable lifecycle signals.

### External Context

- This is the last identified review concern from the va-qbzv PR pair (vultisig/agent-backend#235 + vultisig/vultiagent-app#339). All other findings have been ticketed and landed (`v-klyp`, `v-duhk`, `v-fjph`, `v-xvcz`, `v-cgfd`, `v-cgfd-followup`).
- CodeRabbit's original thread on this race is on PR #339; resolving this ticket closes that thread.
- Pre-launch — no production users with affected schedules to migrate or warn.

## Current Thinking

### Decision: the barrier shape, gated at the cancel-side write

The cancel-side handler waits for approve to settle before deciding what to POST. Approve's handler is the source of truth for the tool-call outcome; cancel can't clobber it. Plus a UI lockout so the visible cancel button can't even be tapped during the race window.

### Why the barrier (not compensating action, not state machine)

- **Simplest** — single-handler change in one component (~15 LOC). No backend changes, no reconciler, no schema work.
- **Least bug prone** — synchronous logic, predictable. The "approve already POSTed success" branch is just a noop; it can't make the database wrong. No async reconciler timing to verify.
- **Works** — eliminates the inconsistency by construction. Cancel literally cannot post a status that contradicts approve.
- **Reasonable UX in the majority case** — cancel button disabled for ~200–800ms during approve. After approve resolves: schedule is active (no cancel needed; user goes to schedule-management UI to delete) or it failed (cancel becomes redundant or the affordance offers retry).

Compensating action and state-machine were considered and rejected as overshoot for a pre-launch fix. See **Dead Ends**.

### Pseudo-shape of the cancel handler

```
on cancel (tap or programmatic):
  if approveMutation.isPending:
    await approveMutation.finally  // or await the existing in-flight promise
  if approveMutation.isSuccess:
    return  // no-op — approve's handler already POSTed success
  // approve failed or never started — safe to post cancelled
  postOutcome({ status: 'cancelled' })
```

Plus the visible-cancel button: `disabled={approveMutation.isPending}`.

### Programmatic cleanup paths

Conversation-switch and unmount paths that call into the cancel flow MUST also wait for approve before POSTing. The cleanest way is to keep the same handler — if both the visible button and cleanup hooks call the same `onCancel`, the barrier applies uniformly. Don't fork the cancel logic.

### Edge cases this handles cleanly

- **User navigates away during approve.** Cleanup hook also waits. If approve succeeds, no-op. If approve fails, POST cancelled.
- **Approve takes forever / never completes.** User can't cancel during the wait. They can force-quit, in which case the cleanup either runs at next mount (if app comes back) or never (if uninstalled). Either way: no inconsistency. The only way `cancelled` lands in the database is if approve ALSO returned a non-success.
- **Rapid cancel-tap during approve.** All taps await the same promise. Idempotent.
- **User wants to cancel a schedule that just successfully committed.** Out of scope — this is "I changed my mind about an active schedule" which belongs in the schedule-management UI, not the tool card. Card honestly shows "schedule active." User goes elsewhere to delete it. No conversation-vs-reality drift.

## Constraints

- **Cancel handler MUST wait for approve before deciding what to POST.** If approve is in-flight, the cancel-side write blocks until approve resolves. No cancel POST can race ahead of approve's outcome.
- **Approve's handler is the source of truth for the tool_call_results status.** If approve succeeded, tool_call_results gets `success` (from approve's path). Cancel does NOT POST `cancelled` over a successful approve. Period.
- **Visible cancel button disabled while approve is pending.** UI must reflect the barrier — don't let the user tap a button that won't do what they expect.
- **Single cancel handler shared between visible button and programmatic cleanup paths.** Don't fork; the barrier needs to apply uniformly.
- **Don't add a backend reconciler.** Out of scope; the barrier closes the inconsistency on the frontend.
- **Don't refactor `ScheduleTaskTool` into a state machine.** Out of scope; the barrier is point-fix sized.
- **Status set stays `{success, failed, cancelled, submitted}`** — the same set `v-klyp` enforces server-side. No new statuses introduced.
- **Don't touch the hydration half.** `v-fjph` already migrated `ScheduleTaskTool` to `resolveToolCardSource`. This ticket only changes the cancel-write path.

## Assumptions

- `approveMutation` exposes a stable promise or `.isPending`/`.isSuccess`/`.isError` signals via React Query semantics. Verify by reading the existing `useApproveSchedule` hook.
- Programmatic cleanup paths (conversation-switch, unmount) currently call into something that ultimately fires the same cancel-side write. If they call a different code path, the barrier needs to be applied there too — find every entry point that POSTs `cancelled` and gate them all behind the same await.
- The schedule lifecycle UI (separate from this card) already exists and lets the user cancel an active schedule. If it doesn't, the "user wants to cancel a successfully-committed schedule" edge case has no escape hatch from this card — but that's still out of scope; flag for product if relevant.
- Disabling the cancel button while approve is pending doesn't break any other flow that depends on the button being live.

## Non-goals

- **Backend reconciler / compensating action.** Considered (option 2 in the design discussion); rejected for complexity and async-timing fragility.
- **State-machine refactor of the schedule lifecycle.** Considered (option 3); rejected as overshoot for a point-fix.
- **Schedule-management UI changes.** Out of scope — that's where users cancel active schedules; this ticket only fixes the race.
- **Backend changes.** None needed; the barrier is frontend-only.
- **Changes to `usePostToolOutcome` itself.** The hook's contract is unchanged; this ticket changes when and how the cancel-side caller invokes it.
- **Non-schedule tools.** The race is specific to tools with a parallel approve+cancel handler shape. Other tools (`AddCoinTool`, `CreateVaultTool`, etc.) don't have this shape.

## Dead Ends

- **Compensating action (backend reconciler).** Watches for approve-then-cancel sequences and runs a delete/disable on the just-created schedule. Rejected because: more code (frontend + backend), async timing fragility (what if reconciler crashes? what if approve was already partially executed?), harder to verify end-to-end. The barrier achieves the same correctness with synchronous frontend logic.
- **State-machine refactor.** Schedule has explicit `requested → committed → cancelled` states with atomic transitions. Cleanest semantics, biggest refactor — touches schedule lifecycle across both repos. Rejected as overshoot for a point-fix and pre-launch timing.
- **Hide the cancel button entirely during approve.** Considered (option 4 in the discussion). Doesn't help with programmatic cleanup paths (conversation-switch, etc.) that fire the cancel-side write without going through the visible button. The barrier on the cancel handler itself covers both visible and programmatic paths uniformly. The disabled-button UI lockout still goes in as a presentation concern, but it's not the load-bearing piece.

## Open Questions

1. **Where exactly is the programmatic cleanup path?** The cancel-side write fires from at least the visible button and `onCancelled` from `cancelAllApprovals`. Verify whether other entry points exist (unmount cleanup, navigation hooks, etc.) by grepping for callers of whatever cancel-side mutation `ScheduleTaskTool` ultimately calls. All entry points need to share the barrier.
2. **Does `useApproveSchedule` expose a stable awaitable?** Confirm by reading its implementation. If it's a bare `useMutation` from React Query, we have `.mutateAsync()` and `.isPending` — straightforward. If it's a custom hook with different lifecycle, the barrier shape may need adjustment.
3. **What does the visible card look like during the disabled-cancel window?** Is the cancel button replaced by a spinner, greyed out with the same label, or hidden entirely? Tied to the existing `ScheduleConfirmationCard` UI vocabulary; planner picks based on what's already in the card.
4. **Approve-failed UX.** When approve fails, is the user supposed to see a "Cancel" or a "Retry" button? Today the card may already handle this via terminalKind='failed'; verify the failure path doesn't break with the new barrier.

## Repos / branches

- **vultiagent-app** — work on top of `worktree-va-qbzv` (PR #339), in the main repo dir `vultiagent-app/`. Touches `src/features/agent/components/tools/ScheduleTaskTool.tsx` and possibly the `useApproveSchedule` / `useConfirmation` hooks if they need to expose additional signals.
- **agent-backend, mcp-ts, vultisig-sdk** — no changes expected.

## What this ticket closes (review threads)

- The CodeRabbit thread on `ScheduleTaskTool` cancellation race (vultisig/vultiagent-app#339).
- The lone outstanding va-qbzv review-pile concern not closed by `v-duhk` / `v-fjph` / `v-cgfd`.
