---
id: v-aoov
status: closed
deps: []
links: []
created: 2026-05-01T05:09:38Z
type: task
priority: 2
assignee: Jibles
---
# rework tool execution results: sidecar map, not embedded on parts

## Objective

Both PR #235 (agent-backend) and PR #339 (vultiagent-app) currently embed `executionResult` directly onto each tool part in the API response and client cache. The persistence is correctly sidecar at the storage layer (`tool_call_results` table), but the wire shape and client cache mirror a merged/denormalized view. This is stored derived state — every write site has to maintain the join manually (see `writeExecutionResultToCache`'s tree walk, and the +156-line read-path merge in `internal/storage/postgres/message.go`). Rework so the wire shape, client cache, and render seam all keep results sidecar; derivation happens at render time via a lookup keyed by `toolCallId`.

## Context & Findings

- Both PRs are in development, untested in production, and have no downstream consumers — wire shape is malleable now and will be expensive to change later.
- Storage shape stays as-is: `tool_call_results` sibling table with `(message_id, tool_call_id)` join key, `ON CONFLICT DO UPDATE WHERE status = 'submitted'` upgrade rule. The schema is correct; only the response shape and client cache shape change.
- The merge-on-read pattern bred two places where the join must be maintained: backend `message.go` (+156 lines for batch-fetch + merge onto parts), and frontend `txOutcomeApi.ts::writeExecutionResultToCache` (tree-walks the cached conversation cloning the matching part). Both go away under the sidecar shape.
- POST callsites (~10 across `useToolExecution`, `useTransactionFlow`, `ScheduleTaskTool`, the 6 legacy synchronous tools, the 3 EVM cards) don't change at all — they build a `ToolExecutionResult` and call the mutation; they never cared where it gets stored.
- Read sites are small: `AssistantMessageView` (already threads `messageId` to `ToolUIProps`; would also thread the looked-up result), `useTransactionFlow` historical hydration (changes from `part.executionResult?.status` to `toolResults[toolCallId]?.status`), and `historicalToolGuard.ts`. Tool card components keep receiving `executionResult` as a prop — only the parent's lookup path changes.
- Backend `WriteToolExecutionResults` in `internal/service/agent/prompt.go` currently iterates parts and reads `part.ExecutionResult` to render the "Recent Tool Outcomes" section. Switches to looking up by `toolCallId` against the sidecar map. Same logic, different access path.
- Security-sensitive `(message_id, tool_call_id)` join key on the read-path stays — the spoofed-row guard test in `tool_call_results_test.go` still pins the same invariant.
- `partExecutionResult.ts` (added in #339) likely goes away.
- Rejected: keeping the embedded shape and just simplifying the cache patcher. The cache patcher is the symptom; the cause is storing derived state. A render-time selector (`useToolResult(toolCallId)` or a parent prop drill from the toolResults map) is the cleaner shape and removes a whole class of drift between the persisted source-of-truth and the cached merged view.
- Rejected: leaving as-is and revisiting later. The pattern this lands with will be copied for the next per-part persistence feature (message reactions, signing approvals, etc.). Cheaper to fix the shape now while there's no production data and no downstream consumers.
- Effort estimate from the discussion: ~half a day total. Backend ~1–2 hours (drop the merge step, return a sidecar map, update `prompt.go` lookups, reshape tests). Frontend ~2–3 hours (cache type change, simplify `writeExecutionResultToCache`, move lookup up to `AssistantMessageView`, update two read sites in hooks, adjust test fixtures).
- Coordination: merge order stays backend → app. No DB migration. No new failure modes.

## Files

### agent-backend (PR #235 branch: `feat/va-qbzv-tool-call-results`)
- `internal/storage/postgres/message.go` — drop the merge step in `GetWithMessages` and the message-fetching repo methods; return the batch-fetched results as a sidecar field instead of merging onto parts.
- `internal/types/parts.go` — remove `ExecutionResult` from the Part struct.
- `internal/service/agent/prompt.go` (`WriteToolExecutionResults`) — look up by `toolCallId` against the sidecar map instead of reading from each part.
- `internal/api/tool_results.go` and the conversation/message response types — add the `toolResults` field to whatever `GetWithMessages` and the read paths return.
- `internal/api/tool_results_test.go`, `internal/storage/postgres/tool_call_results_test.go`, `internal/storage/postgres/tool_call_results_upgrade_test.go` — reshape assertions to read from the sidecar map; the security-sensitive `(message_id, tool_call_id)` join key still applies.

### vultiagent-app (PR #339 branch: `worktree-va-qbzv`)
- `src/services/network/agentApi.ts` (Conversation type) — add `toolResults: Record<string, ToolExecutionResult>` to the conversation/cache entry shape.
- `src/services/network/txOutcomeApi.ts` — `writeExecutionResultToCache` collapses to a one-key spread on `cached.toolResults`. Drop the tree-walking part-mapping.
- `src/services/network/__tests__/txOutcomeApi.test.ts` — reshape cache-patch coverage to assert against the sidecar map.
- `src/features/agent/components/AssistantMessageView.tsx` — already plumbs `messageId` to `ToolUIProps`; also look up `toolResults[part.toolCallId]` and thread it as `executionResult`.
- `src/features/agent/lib/toolUIRegistry.tsx` — `ToolUIProps` keeps `executionResult` (sourced from parent now, not part).
- `src/features/agent/hooks/useTransactionFlow.ts` — historical hydration reads from the sidecar map: `toolResults[toolCallId]?.status === 'submitted'` rather than `part.executionResult?.status === 'submitted'`. Same logic.
- `src/features/agent/lib/historicalToolGuard.ts` — same shift.
- `src/features/agent/lib/partExecutionResult.ts` — likely deletable.
- Tool card test fixtures across `ContractCallCard.test.tsx`, `SendFlowCard.test.tsx`, `SwapFlowCard.test.tsx`, `SignTypedDataTool.test.tsx`, `BuildTxCard.security.test.tsx` — adjust fixtures that currently pre-set `executionResult` on the part.

## Acceptance Criteria

- [ ] Backend response shape no longer embeds `executionResult` on tool parts. Conversation/message read paths return `toolResults` as a sidecar field keyed by `toolCallId`.
- [ ] Backend storage layer is unchanged: `tool_call_results` table, idempotency rules, `(message_id, tool_call_id)` join key, FK cascade, security guard against forged `message_id` rows.
- [ ] Backend `WriteToolExecutionResults` renders the "Recent Tool Outcomes" prompt section by reading the sidecar map; output unchanged.
- [ ] Frontend cache entry shape includes `toolResults: Record<string, ToolExecutionResult>` alongside `messages`.
- [ ] `writeExecutionResultToCache` is a single-key spread on the sidecar map — no tree walk over messages/parts.
- [ ] Tool card components receive `executionResult` as a prop sourced by the parent's lookup against the sidecar map; their public API is unchanged.
- [ ] Historical hydration in `useTransactionFlow` reads from `toolResults[toolCallId]` rather than `part.executionResult`. Reopened mid-broadcast EVM flows still resume polling and upgrade to terminal correctly. Approve+swap stepper reconstruction still works.
- [ ] Backend tests green, including the spoofed-row security guard, the submitted-to-terminal upgrade test, and the `approval_tx_hash` round-trip + upgrade-preserves-approval cases.
- [ ] Frontend unit and component suites green. POST helper tests, retry/cache-patch tests, and tool card fixtures all assert against the new shape.
- [ ] Lint and typecheck pass on both repos.
- [ ] Smoke matrix from the original test plan still applies and passes: send → close → reopen renders persisted card; submitted → terminal upgrade across kill/reopen; failed-tx; retry/offline; OTA mid-tx guard.
- [ ] Merge order stays backend → app; both PRs updated in lockstep on their existing branches.

## Gotchas

- The `(message_id, tool_call_id)` join on the backend read path is security-sensitive and must stay — including `message_id` in the lookup key defends against a buggy/forged row bleeding sideways. Don't drop this when refactoring the merge step into a sidecar map fetch.
- POST helper URL and request body shape don't change. Don't touch the wire contract on the write side; this rework is read-side only.
- `partExecutionResult.ts` was added in PR #339 specifically to support the embedded shape. Verify nothing else has grown to depend on it before deleting.
- Tool card test fixtures across multiple files currently set `executionResult` on the part. Catch them all — a half-migrated fixture renders pass-by-coincidence in unit tests but fails in integration.
- The OTA mid-tx guard rewrite (`useOtaUpdateController.ts`, `unsafeToRestartStore.ts`) is unrelated to this rework. Don't conflate.
- The B2 prompt render path in `WriteToolExecutionResults` is the LLM's view of past tool outcomes — regression here is a silent prompt-quality issue, not a test failure. Verify the rendered "Recent Tool Outcomes" section is byte-identical for a fixture conversation before/after the rework.
- `submitted` is non-terminal and the only non-terminal status in the lifecycle. The upgrade rule (`ON CONFLICT DO UPDATE WHERE status = 'submitted'`) is the only place this matters — preserve it exactly.

## Notes

**2026-05-01T05:28:47Z**

Backend: commit f159efdd; bytes-parity test added; build/vet/650 tests pass.

**2026-05-01T05:28:47Z**

Frontend: commit 74b0a155; 2118 tests pass; partExecutionResult.ts deleted.

**2026-05-01T05:58:39Z**

Cross-repo integration review caught wire-shape misalignment (backend per-message vs frontend root-level). Fixed in backend commit 05171d24: ConversationWithMessages + MessagesSinceResponse expose root-level toolResults via AggregateToolResults; Message.ToolResults now json:"-" (internal-only).

**2026-05-01T05:58:39Z**

Frontend code-quality found render-thrash bug in AgentHomeScreen.tsx empty-fallback identity flip. Fixed in commit 0bb49129 with module-level EMPTY_TOOL_RESULTS constant.

**2026-05-01T05:58:39Z**

Final state: backend f159efdd→05171d24 (build/vet/test green), frontend 74b0a155→0bb49129 (tsc/biome/2118 tests green). Worktrees at .tk-aoov-backend and .tk-aoov-app — push when ready, then merge backend → app per ticket coordination.
