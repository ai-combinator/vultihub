---
id: va-qbzv
status: closed
deps: []
links: [v-duhk, v-fjph, v-klyp, v-cgfd, v-xvcz, v-bvin]
created: 2026-04-30T05:31:42Z
type: task
priority: 2
assignee: Jibles
---
# Persist tool execution results server-side; retire recentActions outbox

## Objective

When a chat conversation is reopened, signed transactions render as "Swap skipped (no saved data)" instead of showing the saved outcome. Fix this by persisting tool execution outcomes (txHash, status, error) as append-only sibling rows in the agent-backend database, joined onto tool message parts at read time. This becomes the single source of truth for both the historical card render and the LLM's next-turn context, retiring the existing client-side `recentActions` outbox entirely.

## Context & Findings

**Root cause of the bug** (verified by tracing the lifecycle):
- `chatSessionStore.completeAction()` writes the tx outcome to `recentActions`
- On the next message send, `takeUnconsumed()` ships it to the backend and tags it with `consumedGeneration`
- After send confirms, `confirmInFlight()` does `recentActions.filter(a => a.consumedGeneration !== generation)` — **deleting** the row from memory and SecureStore (`chatSessionStore.ts:299`)
- On conversation reopen, message parts replay (so `replayGuardIds` includes the toolCallId) but `recentActions` is empty
- `useToolExecution.historicalStep` returns `null` → `state` falls through to `{ step: 'idle' }` → `ExecutionHistoricalGuard` renders the "skipped" stub (`ExecutionHistoricalGuard.tsx:52`)

**Architectural diagnosis**: `recentActions` was designed as a client→backend outbox (delete after send) but is also being read as a permanent UI record (must persist forever). These purposes conflict.

**Decided design**: append-only sibling table on the backend, joined at read time so the wire shape exposes `executionResult` on the tool part. Both readers (UI render + LLM context) feed from the same source. `recentActions` is retired entirely — including the `takeUnconsumed`/`confirmInFlight`/`releaseInFlight`/`consumedGeneration` outbox lifecycle.

**Rejected approaches**:
- *Soft-delete in `confirmInFlight`* — keeps two systems with overlapping responsibilities; doesn't address the root coupling issue.
- *Mutate `agent_messages.parts` JSONB to add executionResult* — breaks append-only semantics of the message log; complicates replication/audit.
- *Keep `recentActions` for legacy synchronous tools* — same pattern (proposal → user action → outcome), should use the same system.
- *AsyncStorage WAL + flush-on-hydrate retry queue* — overkill; React Query mutation retry covers transient failures, app-kill-mid-POST losses are an accepted rare failure mode.
- *Client-side store keyed by toolCallId* — doesn't survive reinstalls or work across devices; conversation log is the correct durable layer.
- *Backfill old conversations* — execution result data is gone; pre-existing conversations keep showing the skipped stub.

**Save trigger**: any client-side terminal state on a tool with client-side follow-through. Three states, all written as a single row:
- `success` — signed and confirmed on-chain
- `failed` (with error string) — tx reverted, RPC error, receipt timeout, idle timeout
- `failed` with error "Transaction cancelled." — user cancelled the approval sheet

Tools with self-contained server-side `output` (read-only / informational) write nothing — the part already carries the result.

**Durability tradeoff**: React Query mutation configured with `retry: 3` and exponential backoff handles transient network failures. App-kill-mid-POST means one outcome may be lost; that case is rare and tolerable. No WAL, no AsyncStorage retry queue, no flush-on-hydrate.

## Files

### agent-backend (Go, Postgres + sqlc + Echo)

- `internal/storage/postgres/migrations/` — new goose migration creating `tool_call_results` table (PK `tool_call_id`, FK `message_id` → `agent_messages(id) ON DELETE CASCADE`, `status`, `tx_hash`, `explorer_url`, `chain`, `error`, `created_at`; index on `message_id`)
- `internal/storage/postgres/schema/schema.sql` — append the new table to the snapshot for sqlc type resolution
- `internal/storage/postgres/sqlc/messages.sql` — extend with `InsertToolCallResult` (idempotent on `tool_call_id`) and `GetToolCallResultsByMessageIDs` (batch fetch for join)
- `internal/storage/postgres/message.go` — wire the join: when reading messages, fetch `tool_call_results` for the message ids and merge `executionResult` onto matching tool parts in `Parts` before returning
- `internal/api/message.go` — new handler `POST /agent/messages/:msgId/tool-results/:toolCallId/execution` (auth ownership: join through parent message → conversation → public_key)
- `cmd/server/main.go:577–585` — register the new route inside the existing `agentGroup` (AuthMiddleware applies)
- `internal/service/agent/prompt.go:912–931` — replace `RecentActions` markdown formatter with logic that renders each tool part's `executionResult` inline after the tool's `output`
- `internal/service/agent/types.go` — remove `RecentAction` types from `MessageContext`
- `internal/service/agent/txSequence.go:333–349` — remove `actionResultFromRequest()` synthesis path
- `internal/types/parts.go` — extend tool-part struct with optional `ExecutionResult` field for serialization

### vultiagent-app (React Native, TS, React Query)

- `src/services/network/agentApi.ts` (or new `txOutcomeApi.ts`) — POST helper for the new endpoint
- `src/features/agent/hooks/useToolExecution.ts:99–121` — `historicalStep` reads from `data.executionResult` instead of `recentActions`; consider deleting the memo entirely since the part already carries the data
- `src/features/agent/components/tools/ExecutionHistoricalGuard.tsx` — delete the "skipped" stub branch; with `executionResult` always available when a result exists, the guard's only remaining job is replay-skip on `output-error` parts (or the file may be deletable entirely once `useToolExecution` is updated)
- `src/features/agent/stores/chatSessionStore.ts` — strip the entire `recentActions` lifecycle: `addRecentAction`, `startPendingAction`, `submitAction`, `completeAction`, `failAction`, `takeUnconsumed`, `confirmInFlight`, `releaseInFlight`, `clearRecentActions`, `consumeGeneration`, `inFlightConsumeGeneration`, the SecureStore persistence, the stale-pending sweep on hydrate. Keep only `input`, `pendingMessage`, `hydrate` (now trivial)
- `src/features/agent/hooks/useToolExecution.ts:196,213,244,258` — replace `startPendingAction`/`completeAction`/`failAction` calls with the new POST (with `messageId` from the part)
- `src/features/agent/hooks/useReceiptPoll.ts:44,53` — same swap
- `src/features/agent/hooks/useTransactionFlow.ts:308,346,424` — same swap
- `src/features/agent/components/tools/ScheduleTaskTool.tsx:67,76` — same swap
- `src/features/agent/components/tools/{Add,Remove}Chain.tsx`, `{Add,Remove}Coin.tsx`, `{Import,Create}Vault.tsx` — replace `addRecentAction(...)` with the new POST helper
- `src/services/agentContext.ts:9,83` — remove `recentActions` field from `AgentContextExtras`
- `src/features/agent/hooks/useAgentTools.ts:55` — remove `takeUnconsumed()` call
- `src/features/agent/hooks/useAgentChat.ts:108,129,134` — remove `confirmInFlight()` / `releaseInFlight()` calls
- `src/features/agent/lib/historicalToolGuard.ts` — keep; still used by auto-fire effects to prevent re-triggering on replay (read-only re-fire guard, separate concern from outcome data)
- Tests to update: `chatSessionStore.test.ts`, `recentActionsWireShape.test.ts`, `useAgentChat.terminalFlags.test.ts` — most outbox-lifecycle tests get deleted; add a new test for the POST path and the React Query cache invalidation

### Reference patterns

- agent-backend: existing route `agentGroup.POST("/conversations/:id/messages", ...)` in `cmd/server/main.go` is the closest mirror for handler shape, auth, and rate-limit pattern.
- agent-backend: `internal/storage/postgres/message.go:33–39` shows the JSONB marshal pattern; the read-side join can mirror existing batch query patterns in the same file.
- vultiagent-app: existing `useMutation` patterns elsewhere in the app for the React Query config (`retry: 3`, exponential backoff).

## Acceptance Criteria

- [ ] New `tool_call_results` table created via goose migration; schema snapshot updated
- [ ] `POST /agent/messages/:msgId/tool-results/:toolCallId/execution` accepts `{ status, txHash?, explorerUrl?, chain?, error? }`, enforces conversation ownership via parent message, and is idempotent on `tool_call_id`
- [ ] `GetMessagesByConversationID` (and any other message-read query) joins `tool_call_results` and merges `executionResult` onto the matching tool part before returning
- [ ] LLM next-turn prompt renders `executionResult` inline after each tool part's `output` (replacing the old `recentActions` markdown formatter)
- [ ] `recentActions` and its outbox lifecycle (`takeUnconsumed`/`confirmInFlight`/`releaseInFlight`/`consumeGeneration`) fully removed from client and `MessageContext`
- [ ] On reopen of a conversation that has a saved execution result, the tool card renders the saved success/error state directly from `data.executionResult` — no "skipped" stub
- [ ] Reopening conversations that pre-date this change continues to work (renders proposal preview or skipped stub for legacy data) without crashing
- [ ] All terminal client-side states (success / failed / cancelled) trigger a POST; tools with self-contained server output do not
- [ ] React Query mutation configured with `retry: 3` and exponential backoff; on success, the conversation cache is invalidated or written through so the new `executionResult` is visible immediately
- [ ] Test suite updated: outbox-lifecycle tests removed, new tests cover the POST helper and cache invalidation
- [ ] Lint, type-check, and tests pass in both repos

## Gotchas

- The client needs the parent `messageId` (UUID) for the POST URL — verify it survives the AI SDK part shape end-to-end. The toolCallId alone is not enough.
- Deploy order matters: backend ships migration + endpoint + read-side join *before* the client switches readers and writers. Backend changes are purely additive on read so they're safe to ship standalone.
- The 6 legacy synchronous tools (Add/RemoveChain, Add/RemoveCoin, Import/CreateVault) follow the same pattern but their card components currently call `addRecentAction(...)` synchronously — verify the messageId is reachable from those components too.
- `historicalToolGuard.ts` and its `replayGuardIds` are *not* the outcome system; they're a re-fire guard derived from message part state. Keep them. The `ExecutionHistoricalGuard` component is what gets simplified or deleted, not the underlying replay-id machinery.
- `tool_call_id` as PK assumes it's globally unique across conversations. The codebase already treats it that way, but worth a quick double-check before the migration commits to it (otherwise composite key on `(message_id, tool_call_id)`).
- Stale-pending sweep on hydrate (the `app_killed` marker) goes away with `recentActions`. There's no replacement — the assumption is that React Query retry covers transient failures and the app-kill-mid-POST loss case is accepted as rare.
- Cross-device parity comes for free with this change (the conversation log is now the source of truth for tx outcomes), but only for new conversations. Old ones don't backfill.

## Notes

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/vultiagent-app#339 + vultisig/agent-backend#235 (DRAFT)
