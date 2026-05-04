---
id: v-hqxs
status: closed
deps: []
links: []
created: 2026-04-21T03:31:35Z
type: task
priority: 2
assignee: Jibles
---
# Unify agent-turn presence — one render for the active turn

## Objective

Eliminate the visual desync between UI elements that express "the agent is engaged with this turn." Today the loader and the orb (and the elapsed counter) read from different state timelines — one client-side, one server-ack'd — so they appear at different times. Restructure so there's one source of truth: the active assistant turn renders as a single UI row the moment the user sends, and the real assistant message smoothly takes over once streaming begins.

## Context & Findings

**The two-clock problem:**
- `useChat().loading` flips true synchronously when `sendMessage()` is called.
- `useChat().messages` doesn't gain an assistant entry until the AI SDK processes the first SSE frame from the backend (~1-2s later).
- Every UI element expressing "agent is working" picks one or the other. The Analyzing footer picks `loading` (instant). The orb lives on the assistant message (delayed). They visually drift apart.

**Confirmed symptoms:**
- User reported the orb appearing a couple of seconds after "Analyzing..." showed up.
- Earlier in the same session, a conditional orb-in-footer hack was shipped and then removed ("double bubble") — the team has already iterated around this shape once.
- `useMessageElapsed` sits next to the same fork; adding any new agent-presence UI will re-surface the issue.

**The restructure:**
- Introduce a "pending assistant turn" concept that's visible in the render list from the moment the user sends until it's superseded by the real assistant message.
- One row, rendered by one component, owning orb + Analyzing + (eventually) streamed text + tool cards.
- When `messages` gains the real assistant shell via the first SSE frame, the pending row steps aside and the real message takes over rendering — visually continuous because both express the same grammar (orb + content).

**Implementation shape (not prescriptive — implementer decides):**
- Likely: compose the FlashList data as `[...messages, ...(loading && lastMessageIsUser ? [pendingRow] : [])]` in `AgentChatArea`. Pending row is a synthetic UIMessage-shaped object (e.g. `{ id: 'pending', role: 'assistant', parts: [] }`) rendered by an existing or new component.
- Alternative: inject an optimistic assistant message into `useChat` via `setMessages` on send, then let the AI SDK reconcile when its own shell lands. More invasive; may fight the SDK's ID ownership.
- Either way: the Analyzing loader + orb live on that one row, not in the footer.

**Rejected approaches:**
- **Conditional orb in footer** — already tried and removed this session. Patches the symptom; drifts next time a new agent-presence element is added.
- **Hide the loader until the orb is ready** — removes immediate feedback; worse UX.
- **Server-side latency fix** — wrong layer; the client should not wait on the network to show its own UI.

## Files

- `src/features/agent/components/AgentChatArea.tsx` — primary render site; composes list data, manages footer
- `src/features/agent/components/AssistantMessageView.tsx` — owns per-message rendering (orb + parts); may need an empty-parts case
- `src/features/agent/hooks/useAgentChat.ts` — if the optimistic-injection approach is chosen, this is where it'd live
- `src/features/agent/hooks/useStreamPauseDetector.ts` — already wired; should keep working through the restructure
- `src/features/agent/hooks/useMessageElapsed.ts` — currently uses the same two-clock fork; should cleanly key off the unified turn

## Acceptance Criteria

- [ ] The orb appears in the same render cycle as the Analyzing loader when the user sends a message (no perceptible gap on a fast machine)
- [ ] Once the assistant starts streaming text, the orb and streaming text render on a single visually continuous row — no jump, no flicker, no duplicate avatars
- [ ] The Analyzing loader still reappears during mid-turn pauses (tool-call gaps > 500ms) via the existing `useStreamPauseDetector`
- [ ] The conditional orb-in-footer logic is gone (or at minimum, its complexity is not reintroduced elsewhere)
- [ ] No regression to historical message rendering (past assistant messages still show their orbs)
- [ ] No regression to tool-card rendering (existing execute_send etc. cards still render correctly)
- [ ] Lint and type-check pass

## Gotchas

- AI SDK v5 assigns message IDs server-side. If you go the optimistic-injection route, the ID at send time won't match the one the backend emits; reconcile carefully or the FlashList will re-key and visibly flicker.
- FlashList's `keyExtractor` returns `item.id` — a synthetic pending row needs a stable distinctive id (e.g. `'pending-agent'`) that doesn't collide with real message IDs.
- The `SwipeableMessageRow` and `FeatureErrorBoundary` wrap each message; ensure the pending row renders inside the same wrapping grammar so hitboxes/swipe behavior don't change.
- The `chatError` footer path must still work (can appear with no preceding message — auth failure, network drop).
- A persisted/reloaded conversation must not re-create the pending shell. The shell is purely for the currently-in-flight turn from this client.
- This change intersects with the existing `useStreamPauseDetector` + `useMessageElapsed` hooks. Don't break their existing semantics; they should key off the unified turn naturally.

## Notes

**2026-04-21T03:47:30Z**

Implemented: pending agent row composed into FlashList data when loading && lastMessageIsUser. AssistantMessageView renders both pending (empty parts + paused=true) and real assistant messages (paused driven by streamPaused for latest). Removed the awaitingContent footer branch; chatError footer preserved. Removed dead elapsedForActive + TimelineEntry/RiveIcon imports. Typecheck clean, lint clean, 673/673 tests pass.
