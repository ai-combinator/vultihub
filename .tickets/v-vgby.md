---
id: v-vgby
status: open
deps: []
links: [v-bork]
created: 2026-04-07T01:38:37Z
type: task
priority: 2
assignee: Jibles
---
# Vulti Agent App — UI Rework: Schedule Confirmation Flow & Polish

## Objective

Generalize the non-tx confirmation flow (schedule previews today; DCA / price alerts / yield / policies tomorrow) so `AgentConfirmationCard` supports the same three-state pattern (pending → APPROVED / REJECTED badges) as the tx Action Card, and fix two unrelated polish items: the SidePanel new-chat button and the no-vault empty state.

**Scope note:** Tx (swap/send) action cards and the input-bar approve stage are covered separately by `v-bork`. That work already rides on the AI-SDK migration's structured MCP tool output. This ticket focuses on the schedule/preview confirmation path (still SSE `confirmation` events with `schedulePreviewId`) plus the polish items from the original brief.

## Context & Findings

### Schedule confirmation path

**Current rendering (AgentConfirmationCard):**
- Has the key-value detail layout with Approve/Edit buttons — visually close to the designs
- Wired only for schedule confirmations via `schedulePreviewId`
- After approval, the card is **hidden** (via messageOverrides setting `confirmation: undefined`) — the designs show APPROVED/REJECTED badges persisted on the card
- Rejection silently removes the card — no backend notification, no visual state

**Flow (useConfirmationFlow):**
- Deeply coupled to schedules: auto-detection filters on `schedulePreviewId`, approval calls `useApproveSchedule` mutation, consumption tracked by schedule preview ID
- Needs to become generic so any confirmation type works with a universal confirmation ID
- `ConfirmationData` type already has a `details` array and flexible fields — the type system is fine, the hook logic is the bottleneck

### Design Patterns (from Figma mockups)

All action types use one card component with these patterns:
- Dark card with border, key-value rows separated by thin dividers
- Labels left-aligned (grey `#8295AE`), values right-aligned (white `#F0F4FC`)
- Three states: pending (with Approve/Edit buttons), approved (green checkmark + "APPROVED" in LCD font at bottom, no buttons), rejected (red X + "REJECTED" in LCD font, no buttons)
- Some values have special formatting (e.g., "Blocked" in red for policy withdrawals)
- Plugin install cards have a distinct header variant with icon circle — **skip plugin cards for now**

The styling should match the tx Action Card shipped under `v-bork` (share primitives where sensible; do not force a single unified component — schedule and tx state machines should not leak into each other).

### Small polish

- SidePanel new-chat button today looks like an "add vault" icon — should be compose/pencil glyph with increased spacing from the vault selector
- No-vault empty state is generic "Welcome to Vulti Agent" — should show a vault icon + "No vault connected" + connect/create CTA
- Credits badge: keep as-is for now

### Rejected Approaches

- Unifying `AgentConfirmationCard` and `BuildTxCard` into one component — forces the schedule approval state (useApproveSchedule mutation) and tx signing state (biometric/password) into one reducer; keep them separate, share styling only
- Removing the schedule-only path entirely — schedule previews still ride a different backend event (`schedulePreviewId`) than MCP-driven tx flows; the path exists for a reason

## Files

Key files to modify:
- `vultiagent-app/src/features/agent/hooks/useConfirmationFlow.ts` — generalize from schedule-only to universal confirmation ID; keep `schedulePreviewId` as one variant
- `vultiagent-app/src/features/agent/types.ts` — add confirmation status field, generic confirmation ID
- `vultiagent-app/src/features/agent/components/AgentConfirmationCard.tsx` — add APPROVED/REJECTED state rendering, remove buttons when resolved
- `vultiagent-app/src/features/agent/components/AgentChatArea.tsx` — persist card after approval/rejection instead of hiding (replace `confirmation: undefined` override with a status field)
- `vultiagent-app/src/features/agent/lib/helpers.ts` — update `normalizeConfirmation` for generic confirmation IDs
- `vultiagent-app/src/features/agent/components/SidePanel.tsx` — new chat button icon/spacing
- `vultiagent-app/src/features/agent/components/AgentEmptyState.tsx` — no-vault empty state

Reference patterns:
- `vultiagent-app/src/features/agent/components/tools/BuildTxCard.tsx` — tabular Action Card styling landed (or landing) under v-bork; match its row layout and APPROVED/REJECTED badge treatment
- `vultiagent-app/src/features/agent/components/TimelineEntry.tsx` — LCD font styling, category colors

## Acceptance Criteria

- [ ] `useConfirmationFlow` works with any confirmation type, not just schedules — uses a generic confirmation ID with schedule preview ID as one variant
- [ ] Confirmation auto-detection finds confirmations with or without `schedulePreviewId`
- [ ] `AgentConfirmationCard` renders three states: pending (with Approve/Edit buttons), approved (green checkmark + "APPROVED" badge, no buttons), rejected (red X + "REJECTED" badge, no buttons)
- [ ] After approval or rejection, the card persists in the message with its final state badge — not hidden
- [ ] Dismissing/rejecting a confirmation shows the REJECTED state on the card
- [ ] New chat button uses compose/pencil icon with increased spacing from vault selector
- [ ] No-vault empty state shows vault icon, "No vault connected" message, and connect/create vault CTA
- [ ] Existing schedule confirmation flow continues to work (backward compatible)

## Gotchas

- The confirmation flow currently uses `messageOverrides` to set `confirmation: undefined` after approval — change this to set a status field instead of removing the confirmation
- `ConfirmationData.schedulePreviewId` must remain supported alongside a new generic ID field for backward compat
- The `useApproveSchedule` mutation is the only approval endpoint today — for non-schedule confirmations, the approval action may just be sending a message (e.g., "approved") rather than calling a specific endpoint. Check backend capabilities.
- Keep schedule and tx approval state machines separate; share styling primitives only. The tx path (v-bork) uses `ApprovalContext` + MCP tool output; the schedule path uses `useConfirmationFlow` + SSE `confirmation` events. Do not cross-wire.
