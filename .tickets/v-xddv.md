---
id: v-xddv
status: in_progress
deps: []
links: []
created: 2026-04-29T02:03:49Z
type: bug
priority: 2
assignee: Jibles
---
# Defer execute_* card render until assistant text has streamed

## Objective
When the agent emits an `execute_swap` (or `execute_send` / `execute_contract_call`) tool call BEFORE its narration text in the same assistant message, the card appears first and the narration streams *above* it — inverting the natural top-to-bottom reading order in chat. The card should not render until the model has begun narrating (or the message has finished streaming), so users always see context appear before the actionable card.

## Context & Findings
- **Repro:** Ask the agent to swap a token. The card pops in immediately with partial data, and a couple of beats later text starts streaming in *above* it ("I've prepared the swap for you. An approval transaction for LINK is required…"). User-facing reading order ends up reversed for the duration of the stream.
- **Why it happens:** `AssistantMessageView` iterates `message.parts` in emission order. The AI SDK delivers tool-call parts as soon as the model emits them; if the model emits the tool call before its text part, the card mounts first. There is no client-side reordering or gating today.
- **Why we're fixing this client-side instead of in the prompt:** prompt-engineering the model into "always narrate before tool-calling" is fragile across model versions and edits, and we already have `isLoading` plumbed into `AssistantMessageView` (see `AssistantMessageView.tsx:14, 114`).
- **Decision (lean):** simplest gate — defer execute_* card render while the message is still streaming (`isLoading === true`). When streaming ends, the card mounts in one frame with full prep already resolved.
- **Rejected — but viable upgrade if the consent delay is felt:** "render as soon as ANY text part appears earlier in the same message; otherwise defer until streaming ends." Keeps the consent affordance from being held hostage by long post-card narration. Capture the data on whether users complain before adopting; the simple gate is the first ship.
- **Cost of the simple gate:** the user can't sign until the assistant finishes streaming the rest of the message, because the card is what registers consent with `ApprovalContext`. For typical short post-card narration this is a non-issue; for long narration it's a noticeable "I see what it's doing, why can't I sign yet?" wait. Acceptable starting point because the existing pop-in is already worse.
- **Scope boundary:** apply only to the `execute_*` family (the cards that drive ApprovalContext consent + signing). Other tool cards (price, receive, build_*, etc.) keep streaming-render behavior — they don't have the "card before its own narration" problem in the same way and rendering them mid-stream is informative.

## Files
- `src/features/agent/components/AssistantMessageView.tsx` — gates per-tool-part rendering; already receives `isLoading`. The render branch around line 217 (`resolveToolUI` → `<Component {...props} />`) is the place to insert the gate.
- `src/features/agent/lib/toolUIRegistry.tsx` — defines the `execute_*` registry entries (`execute_send`, `execute_swap`, `execute_contract_call`); use these tool names (or a small predicate) to decide which cards are gated.
- `src/features/agent/components/tools/ExecuteCards/SwapFlowCard.tsx` / `SendFlowCard.tsx` / `ContractCallCard.tsx` — no changes expected; the gate happens above them in the parent.

## Acceptance Criteria
- [ ] While the assistant message is still streaming (`isLoading === true`), `execute_swap` / `execute_send` / `execute_contract_call` tool parts do NOT render their card.
- [ ] When the message finishes streaming, the card mounts in a single frame.
- [ ] On historical messages (`isLoading === false` from the start) cards render immediately as today — no regression for replayed/persisted message history.
- [ ] Other tool cards (`get_price`, `get_receive_info`, `build_*`, etc.) continue to render mid-stream as today; the gate is scoped to the `execute_*` family.
- [ ] `ApprovalContext` consent still works correctly once the card mounts post-stream — the approval bar arms after streaming ends, not during.
- [ ] No flicker / double-mount when the message transitions from streaming → done.
- [ ] Lint and type-check pass.

## Gotchas
- The card subtree owning `useTransactionFlow` must NOT mount during streaming — that hook holds reducer state and registers consent. A render-but-hide approach (e.g. `opacity: 0`) would still mount the hook and re-introduce mid-stream approval-bar pop-ups; use a true conditional render.
- Don't gate on the tool part's own `state` (e.g. `output-available`) — that's about THIS tool's lifecycle, not the surrounding message's. The signal is the message-level streaming flag.
- Watch interaction with the existing `lastBuildToolIndex` suppression and `coalesceReceiveInfo` logic — those run before the registry lookup; the new gate should sit alongside them, not replace them.
- For multi-card turns (e.g. two `execute_swap` parts in one message), the gate applies uniformly: both stay hidden until streaming ends, then both mount together. That's the desired behavior.
- If a future feature needs the card's consent surface to arm mid-stream (e.g. a "stream cancelled but you can still sign" recovery path), this gate would block it — flag in PR description as a known limitation.

## Notes

**2026-04-29T04:28:05Z**

Gate added in AssistantMessageView.tsx on worktree-v-xddv: `if (isLoading && toolName.startsWith('execute_')) return null` sits between the receive-multichain coalesce and resolveToolUI. True conditional return — useTransactionFlow + ApprovalContext don't arm mid-stream. Lint + tsc green. Worktree at .claude/worktrees/v-xddv awaiting batch merge.
