---
id: v-vgby
status: open
deps: []
links: []
created: 2026-04-07T01:38:37Z
type: task
priority: 2
assignee: Jibles
---
# Vulti Agent App — UI Rework: Action Cards, Confirmation Flow & Polish

## Objective

Rework the agent chat UI to match the Figma design system: a unified Action Card component for all confirmable actions (swaps, sends, DCA, price alerts, yield, policies), a generalized confirmation flow that isn't schedule-only, input bar transformation for pending confirmations, and small polish fixes (new chat button, empty state). The goal is a consistent approval UX across every action type.

## Context & Findings

### Current State

**Action Card rendering (AgentConfirmationCard):**
- Has the key-value detail layout with Approve/Edit buttons — visually close to the designs
- But only wired for schedule confirmations via `schedulePreviewId`
- After approval, the card is **hidden** (via messageOverrides setting `confirmation: undefined`) — the designs show APPROVED/REJECTED badges persisted on the card
- Rejection just silently removes the card — no backend notification, no visual state

**Confirmation flow (useConfirmationFlow):**
- Deeply coupled to schedules: auto-detection filters on `schedulePreviewId` (line 84), approval calls `useApproveSchedule` mutation, consumption tracked by schedule preview ID
- Needs to become generic: any confirmation type should work with a universal confirmation ID
- `ConfirmationData` type already has a `details` array and flexible fields — the type system is fine, the hook logic is the bottleneck

**TX details display (AgentThinkingBlock):**
- Swap/send details are **regex-parsed from message body text** via `extractTxDetails()` in `thinkingBlockUtils.ts` and rendered as blue `TimelineEntry` components
- The designs show these as structured Action Cards instead
- The backend `confirmation` SSE event schema already supports `details` array — if the backend sends swap/send confirmations as structured `confirmation` events (not just body text), the frontend can render them as Action Cards without regex parsing
- For now, the frontend should support both paths: structured `confirmation.details` (preferred) and fallback extraction from body text

**Input bar (AgentInputBar / AgentApprovalBar):**
- `AgentInputBar` is always a text input + send/stop/mic button — no confirmation mode
- `AgentApprovalBar` exists but is exclusively for transaction **signing** (password/biometric) — it replaces the input bar when `pendingTx` + `approvalMode` is set
- The designs show a third mode: "Approve or edit the action..." with a checkmark button when a confirmation card is pending (before signing)
- This is a distinct state from signing — it's the user reviewing/approving the action proposal, not authenticating

**Small polish (from original ticket):**
- New chat `+` button in SidePanel reads as "add vault" — should be compose/pencil icon with more spacing from vault selector
- No-vault empty state is generic "Welcome to Vulti Agent" — should show vault icon + "No vault connected" + connect/create CTA
- Credits badge: keep as-is for now

### Design Patterns (from Figma mockups)

All action types use **one universal card component** with these patterns:
- Dark card with border, key-value rows separated by thin dividers
- Labels left-aligned (grey `#8295AE`), values right-aligned (white `#F0F4FC`)
- Three states: pending (with Approve/Edit buttons), approved (green checkmark + "APPROVED" in LCD font at bottom, no buttons), rejected (red X + "REJECTED" in LCD font, no buttons)
- Some values have special formatting (e.g., "Blocked" in red for policy withdrawals)
- The "Action" row determines the card type visually (SWAP 5 ETH → USDC, SEND 0.5 ETH, etc.)

Action types that use this card: swaps, sends, DCA schedules, price alerts, yield collection, policy/limits.

Plugin install cards have a distinct header variant with icon circle — **skip plugin cards for now**.

When a card is pending, the input bar transforms to "Approve or edit the action..." placeholder with a blue checkmark button (approve) — the text input remains editable so users can type a modification request.

### Rejected Approaches

- Separate card components per action type — the designs show one universal card, type determined by row content
- Removing `extractTxDetails` entirely — keep as fallback for backward compat until backend fully sends structured confirmations
- Making the approval bar replace the input bar entirely — the designs show the input stays editable for "edit the action" flow

## Files

Key files to modify:
- `vultiagent-app/src/features/agent/hooks/useConfirmationFlow.ts` — generalize from schedule-only to universal confirmation
- `vultiagent-app/src/features/agent/types.ts` — add confirmation status field, generic confirmation ID
- `vultiagent-app/src/features/agent/components/AgentConfirmationCard.tsx` — add APPROVED/REJECTED state rendering, remove buttons when resolved
- `vultiagent-app/src/features/agent/components/AgentInputBar.tsx` — add confirmation approval mode
- `vultiagent-app/src/features/agent/components/AgentChatArea.tsx` — persist card after approval/rejection instead of hiding
- `vultiagent-app/src/features/agent/components/AgentThinkingBlock.tsx` — render structured confirmations as Action Cards when `confirmation.details` exists instead of timeline entries
- `vultiagent-app/src/features/agent/lib/helpers.ts` — update `normalizeConfirmation` for generic confirmation IDs
- `vultiagent-app/src/features/agent/components/SidePanel.tsx` — new chat button icon/spacing
- `vultiagent-app/src/features/agent/components/AgentEmptyState.tsx` — no-vault empty state

Reference patterns:
- `vultiagent-app/src/features/agent/components/AgentApprovalBar.tsx` — existing approval bar pattern for signing
- `vultiagent-app/src/features/agent/lib/toolUIRegistry.ts` — existing tool component registry
- `vultiagent-app/src/features/agent/components/TimelineEntry.tsx` — LCD font styling, category colors

## Acceptance Criteria

- [ ] `useConfirmationFlow` works with any confirmation type, not just schedules — uses a generic confirmation ID with schedule preview ID as one variant
- [ ] Confirmation auto-detection finds confirmations with or without `schedulePreviewId`
- [ ] `AgentConfirmationCard` renders three states: pending (with Approve/Edit buttons), approved (green checkmark + "APPROVED" badge, no buttons), rejected (red X + "REJECTED" badge, no buttons)
- [ ] After approval or rejection, the card persists in the message with its final state badge — not hidden
- [ ] Dismissing/rejecting a confirmation shows the REJECTED state on the card
- [ ] When a confirmation is pending, `AgentInputBar` transforms to show "Approve or edit the action..." placeholder with checkmark button — text input remains editable
- [ ] Tapping the checkmark in confirmation mode triggers approval; typing and sending triggers the edit/modify flow (sends user text as a message)
- [ ] Action cards render correctly for swap details (Action, Route, Est. fee, You receive), send details (Action, To, Network, Est. fee), and schedule/DCA details
- [ ] New chat button uses compose/pencil icon with increased spacing from vault selector
- [ ] No-vault empty state shows vault icon, "No vault connected" message, and connect/create vault CTA
- [ ] Existing schedule confirmation flow continues to work (backward compatible)

## Gotchas

- The confirmation flow currently uses `messageOverrides` to set `confirmation: undefined` after approval — this needs to change to set a status field instead of removing the confirmation
- `AgentApprovalBar` (for signing) and the new input bar confirmation mode are **different states**: confirmation mode is "do you approve this action?", signing mode is "authenticate to execute". They should not conflict — a swap goes: confirmation pending → approved → tx_ready → signing
- The backend may or may not already send structured `confirmation` events for swaps/sends — if it only sends body text, `extractTxDetails` output should be renderable as an Action Card as a fallback
- `ConfirmationData.schedulePreviewId` must remain supported alongside a new generic ID field for backward compat
- The `useApproveSchedule` mutation is the only approval endpoint today — for non-schedule confirmations, the approval action may just be sending a message (e.g., "approved") rather than calling a specific endpoint. Check backend capabilities.
