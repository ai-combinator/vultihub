---
id: tic-1d38
status: open
type: chore
priority: 2
assignee: Jibles
created: 2026-04-13T12:07:29.626270262Z
---
# Simplify agent chat hook wiring

## Objective
Reduce incidental complexity in the agent chat hook layer by consolidating context building, eliminating the persisted flag and pendingSwitchMessagesRef workarounds, and removing redundant cache operations. ShapeShift-agentic has the same fundamental split (AI SDK owns messages in memory, separate persistence layer stores them) but handles it cleanly: write on finish, read on load, loading state during switch. Vultiagent has accumulated workarounds (persisted flag, ref stashing, double-invalidation) that can be replaced with direct patterns.

## Context & Findings
**Context building (3 layers to 1):**
- Layer 1: agentContext.ts buildAgentContext() — derives addresses, builds coins, creates instructions
- Layer 2: useAgentTools.ts — wraps buildAgentContext, injects balances from React Query cache + recentActions from Zustand
- Layer 3: chatTransport.ts — calls the callback and packages into HTTP body
- All three serve a single consumer with no branching. agentContext.ts is not used anywhere else. buildAgentContext could take balances and recentActions as optional parameters directly, eliminating Layer 2 as a separate concern.

**Persisted flag — remove entirely, not just simplify:**
- The `persisted` flag exists because conversations are created optimistically (client-side UUID) before the server knows about them. fetchConversation branches: if unpersisted, return empty messages from cache; if persisted, fetch from server.
- This creates a race: if user switches away before onFinish marks persisted=true, fetchConversation returns empty messages and the conversation appears blank on return.
- ShapeShift avoids this because IndexedDB is instant (no async gap). But the fix for server-side persistence is the same: drop the flag, let the backend create the conversation implicitly on first message POST. If the server 404s on a conversation that hasn't been persisted yet, return the cached messages instead of branching preemptively.
- markConversationPersisted, the persisted field on Conversation type, and the merge logic in queries.ts that separates unpersisted from server conversations — all of this goes away.

**pendingSwitchMessagesRef — replace with loading state:**
- Exists because useChat resets messages when its `id` prop changes. Without the ref, setting conversationId and then calling setMessages races: the ID change wipes the messages.
- ShapeShift's IndexedDB read is instant so there's no gap. With a server fetch, the fix is: show a loading state during the fetch, set messages only after both the new conversationId AND the fetched messages are ready. No ref stashing needed.

**Double-invalidation in onFinish:**
- useAgentChat.ts calls markConversationPersisted (updates cache) then invalidateConversations (throws cache away and refetches). With persisted flag removed, markConversationPersisted is deleted entirely. Just invalidate.

**Callback capture refs:**
- useAgentChat.ts has six ref assignments to avoid stale closures in onFinish. With markConversationPersisted removed (and its ref), at least two refs go away. Remaining refs can be reduced by extracting onFinish logic into a stable callback.

## Files
- src/services/agentContext.ts — add optional balances and recentActions parameters to buildAgentContext
- src/features/agent/hooks/useAgentTools.ts — simplify to thin wrapper that gathers params and passes to buildAgentContext
- src/features/agent/hooks/useAgentChat.ts — remove markConversationPersisted call and its ref; reduce remaining callback refs; simplify onFinish to just invalidateConversations + title update
- src/features/agent/hooks/useConversations.ts — remove persisted flag entirely; remove fetchConversation branching; replace pendingSwitchMessagesRef pattern in AgentHomeScreen with a loading state
- src/features/agent/lib/queries.ts — remove persisted field from Conversation type; remove unpersisted/server merge logic in conversationsQueryOptions; remove normalizeConversation's persisted: true assignment
- src/features/agent/screens/AgentHomeScreen.tsx — replace pendingSwitchMessagesRef with loading state during conversation switch; set messages only after both conversationId and fetched messages are ready
- src/features/agent/lib/chatTransport.ts — reference only (consumer, no changes needed)

## Acceptance Criteria
- [ ] buildAgentContext accepts optional balances and recentActions parameters
- [ ] useAgentTools is simplified (passes params to buildAgentContext, no separate injection layer)
- [ ] `persisted` field removed from Conversation type
- [ ] markConversationPersisted function and all references removed
- [ ] fetchConversation does not branch on any persisted flag — single code path
- [ ] queries.ts conversationsQueryOptions does not merge unpersisted + server conversations separately
- [ ] pendingSwitchMessagesRef removed from AgentHomeScreen
- [ ] Conversation switching shows a loading state while fetching, then sets messages after both ID and data are ready
- [ ] onFinish only calls invalidateConversations (no double-write)
- [ ] Callback capture refs in useAgentChat reduced to minimum necessary
- [ ] No behavioral regressions in conversation loading, switching, or context sending
- [ ] Lint and type-check pass

## Gotchas
- balancesQueryOptions cache (60s staleTime) is used elsewhere in the app for balance display — do not remove the query, just change where it is accessed for context building
- recentActions are cleared immediately after extraction (useAgentTools line 46) — preserve this clear-on-read semantics wherever it moves
- The backend may need to handle "conversation doesn't exist yet" gracefully on the first message POST — verify this works before removing persisted flag, or add a simple "create-on-first-message" behavior server-side
- chatTransport.ts prepareSendMessagesRequest extracts only the latest user message text (not full history) — this is intentional, do not change it
- The loading state during conversation switch should be brief (server fetch is fast) — don't add a skeleton loader, just delay setMessages until data is ready
