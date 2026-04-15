---
id: v-bork
status: open
deps: []
links: [v-vgby]
created: 2026-04-14T02:15:54Z
type: task
priority: 2
assignee: Jibles
---
# Transaction Action Card + Input-Bar Approve Stage

## Objective
Replace the BuildTxCard timeline-style preview with a tabular Action Card (Figma spec) for all build_* MCP tool flows, and move the approve action out of the card into an input-bar stage ("Approve or edit the action…" + checkmark) that precedes the existing signing/password stage. After the user approves or rejects, the card persists with a green "APPROVED" or red "REJECTED" LCD badge.

## Context & Findings
- Post-AI-SDK-migration, all build_* tool calls route through BuildTxCard via toolUIRegistry — one component renders the card for swap, send, and every chain-specific build variant. Data is already structured from MCP output; no backend change needed.
- BuildTxCard today renders PreviewSection → buildLines → TimelineEntry list (LCD font, single column). The Figma card is a two-column label/value table with thin dividers.
- The Sign Transaction button currently lives inside BuildTxCard's preview state. The design moves approval to the input bar: text input stays editable (for edit-the-action flow), right side becomes a blue checkmark that advances to signing.
- ApprovalContext.requestApproval() jumps straight from approve() to biometric/password today. This needs to split into Stage A (action-pending: input-bar morph) and Stage B (signing: current AgentApprovalBar behavior, unchanged). The tool-execution state machine already has awaiting_approval distinct from signing — the UI just doesn't differentiate.
- On success, the card today swaps to TxResultView. Per the design, the Action Card persists and appends an "APPROVED" badge (green check + LCD text). On cancel/error, same card, "REJECTED" badge.
- Rejected: a new universal action-card component unifying BuildTxCard and AgentConfirmationCard. They can share styling primitives, but forcing one component causes schedule/tx state machines to leak into each other.

## Files
- `src/features/agent/components/tools/BuildTxCard.tsx` — replace PreviewSection layout, drop internal Sign button, render APPROVED/REJECTED persistent badges
- `src/features/agent/lib/buildTxLines.ts` — add buildTxRows() producing { label, value }[] tuples alongside existing buildLines (keep both until callers migrated)
- `src/features/agent/components/tools/TxResultView.tsx` — fold into BuildTxCard's post-success render, or style to match the new card
- `src/features/agent/components/AgentInputBar.tsx` — add confirmation mode: "Approve or edit the action…" placeholder + checkmark action button
- `src/features/agent/contexts/ApprovalContext.tsx` — split requestApproval into two-stage (action-pending → signing); expose the intermediate state
- `src/features/agent/hooks/useToolExecution.ts` — wire the checkmark tap to the existing approve() path; today the in-card button triggers it
- Reference: `src/features/agent/components/AgentApprovalBar.tsx` — existing bar-morph pattern for the password stage

## Acceptance Criteria
- [ ] BuildTxCard preview renders as a two-column Action Card (label left, value right) for swap, send, and every build_*_send variant
- [ ] Card rows for send: Action, To, Network, Est. fee. Rows for swap: Action, Route, Est. fee, You receive
- [ ] Amounts render human-readable (uses existing from_decimals / to_decimals path from buildTxLines)
- [ ] Sign Transaction button is removed from inside the card
- [ ] When a build_* tool has output and no prior approval, the input bar shows "Approve or edit the action…" placeholder with a blue checkmark on the right; text input remains editable
- [ ] Tapping the checkmark advances to the existing password/biometric AgentApprovalBar (no regression to signing flow)
- [ ] Typing in the bar and submitting while a card is pending sends the text as a new user message (edit-the-action flow)
- [ ] On successful signing, the card remains rendered with a green checkmark + "APPROVED" LCD badge at the bottom
- [ ] On cancel or sign error, the card remains rendered with a red X + "REJECTED" LCD badge
- [ ] Historical messages re-render in the final approved/rejected state
- [ ] Lint, typecheck, and existing BuildTxCard unit tests pass

## Gotchas
- Two distinct approval states must not collide: action-pending (input-bar checkmark) vs. signing (password/biometric). A tx goes action-pending → approved → signing → signed.
- The approve() callback in useToolExecution already handles everything after user consent — don't duplicate it; just reroute the trigger from the card's button to the input-bar checkmark.
- AgentConfirmationCard (schedule path) has its own approval flow — keep it untouched here; v-vgby covers its redesign separately.
- BuildTxCard already handles a "historical" state via historicalToolIds; preserve that path so scrolling back doesn't re-prompt.
- ApprovalContext currently kicks off biometric immediately on requestApproval; if biometric auto-triggers, the action-pending stage is skipped. Stage A must be explicit and separate from the biometric/password stage.
