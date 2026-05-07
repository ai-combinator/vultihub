---
id: v-zcbt
status: open
deps: []
links: []
created: 2026-05-06T04:33:47Z
type: task
priority: 2
assignee: Jibles
---
# Derive quest completion from server-persisted tool data

## Problem
Quest completion is currently being wired into the tool-result POST path in agent-backend PR #235, with the app able to submit quest-relevant fields alongside execution status. This couples a replay/reopen persistence feature to a separate campaign-tracking concern and risks making quest eligibility client-asserted. Solved means quest completion is derived from server-owned persisted data (`agent_messages.parts`, `tool_call_results`, or a dedicated server-side intent/projection), while the app/backend tool-result contract remains limited to execution outcome.

## Background
### Findings
- Backend PR #235 adds persisted client-side tool execution outcomes through `tool_call_results`, keyed by `(message_id, tool_call_id)`.
- Backend PR #235 adds `POST /agent/messages/:msgId/tool-results/:toolCallId/execution` in `cmd/server/main.go` under the authenticated `/agent` group.
- `internal/api/tool_results.go` currently accepts a `ToolExecutionResultRequest` with execution fields (`status`, `txHash`, `approvalTxHash`, `explorerUrl`, `chain`, `error`) plus quest/caller-supplied fields (`toolName`, `questMetadata`).
- `SaveToolExecutionResult` currently resolves message ownership, inserts/upgrades `tool_call_results`, then classifies quest events on `status == "success"` using client-supplied `req.ToolName`, `req.QuestMetadata`, and `req.TxHash`.
- `internal/storage/postgres/quest_events.go` materializes `agent_quest_events` and, when `event.Counted` is true, inserts into `agent_user_quests`.
- `agent_messages.parts` already persists UI/tool parts. `PartsAccumulator` stores `ToolCallID`, `ToolName`, and marshaled `Input` for `tool-input-available` events, and `Output` for `tool-output-available` events.
- Prior signed-proposal quest tracking exists via `types.TxProposal{ToolName, QuestMetadata, QuoteData}` and `ClassifySignedProposal`; scheduled/headless paths extract `quest_metadata` from MCP build tool results.
- We ran `go test ./...` and `go build -o /tmp/agent-server ./cmd/server` on PR #235; both passed before this ticket was created.

## Current Thinking
- Keep the current PR pair focused on persisted tool execution results for chat replay/reopen flows. The app should POST only execution outcome data:

```json
{
  "status": "success|submitted|failed|cancelled",
  "txHash": "0x...",
  "approvalTxHash": "0x...",
  "explorerUrl": "https://...",
  "chain": "ethereum",
  "error": "..."
}
```

- Quest tracking should be a derived projection, not part of the app/backend execution-result contract. The source of truth should be server-persisted tool/message/result data: `agent_messages.parts` plus `tool_call_results`, or a dedicated server-owned intent row created when the backend emits/records the tool call.
- The existing quest tables may still add value as a materialized campaign ledger: fast `agent_user_quests` lookup, auditable `agent_quest_events` rows with `counted`/`reject_reason`, threshold decisions captured at classification time, and idempotency. They should not be treated as unique source data.
- A future classifier/worker can materialize `agent_quest_events` / `agent_user_quests` from persisted server-owned data. If persisted parts do not contain enough canonical metadata to classify first swap/bridge/defi action, add server-side persistence for that metadata at tool emission or MCP result time rather than asking the app to provide it.
- Idempotency for the derived projection should use a server-owned key such as `(message_id, tool_call_id)` or a dedicated intent id, not only raw client-provided `tool_call_id` semantics.

## Constraints
- Do not require the mobile app to submit quest family, amount, chain IDs, or `questMetadata` for quest eligibility.
- Do not make quest completion depend on client-submitted `toolName` / `questMetadata`.
- Preserve the tool-result persistence/replay feature as a separate contract: the app reports execution outcome, backend persists and replays it.
- Quest projection must be idempotent across repeated POSTs/replays.

## Assumptions
- `agent_messages.parts` for the relevant first-swap/bridge/defi flows either already contains enough server-originated metadata in `Input` or `Output`, or can be extended to persist it server-side.
- The “first swap” quest should count only terminal successful executions, not queued/submitted/in-flight states.
- The current quest tracking schema (`agent_quest_events`, `agent_user_quests`) is still acceptable as the materialized read/audit model if fed by server-owned source data.

## Non-goals
- Do not block the PR #235 / app #339 replay feature on completing the full quest projection system.
- Do not expand the app POST body with campaign-specific fields.
- Do not verify tx hashes on-chain as part of this ticket unless product/security explicitly requires it; this ticket is about moving eligibility derivation server-side.
- Do not redesign the whole airdrop campaign system unless the persisted data proves insufficient.

## Dead Ends
- Classifying directly inside `SaveToolExecutionResult` from `req.ToolName` and `req.QuestMetadata` was rejected because it makes quest eligibility client-asserted and couples a replay persistence endpoint to campaign tracking.
- Keeping `toolName`/`questMetadata` in the app/backend PR pair as “convenient metadata” was rejected because it creates an avoidable contract that future quest logic may accidentally trust.

## Open Questions
- Which persisted server-owned record has the canonical quest metadata for chat-driven swaps: `agent_messages.parts[].Input`, `parts[].Output`, existing MCP build result output, or a new dedicated intent/projection table?
- For client-side signed tools, is `tool_call_id` always stable and unique enough for projection idempotency, or should projection key by `(message_id, tool_call_id)`?
- Should quest projection run synchronously after a successful tool-result POST using server-owned metadata, or asynchronously via a worker scanning `tool_call_results`?
- What minimum metadata is required for first swap, bridge, and defi-action classification, and does the current persisted data cover all three?
