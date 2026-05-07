---
id: v-xvcz
status: closed
deps: []
links: [va-qbzv, v-klyp, v-duhk, v-fjph, v-cgfd]
created: 2026-05-06T00:14:32Z
type: bug
priority: 2
assignee: Jibles
---
# Wire tool-result POST into quest classification — chat-driven swaps quest-eligible per Stage 3 spec

## Problem

Per Stage 3 airdrop spec (`ai-combinator/vultihub/docs/airdrop-specs/{overview,sibling-mobile-app,stage-3-quest-tracking}.md`), the mobile app's tool-result reporting is supposed to be the canonical quest-counting signal — *not* the proposal-sign endpoint. Today the implementation has drifted: quest events fire only from `SignTxProposal`, which is reachable only via scheduler-created `agent_tx_proposals` rows. **Chat-driven swaps, bridges, and DeFi actions never count toward quest eligibility today.**

va-qbzv (PR #235 + #339) is the PR that turns tool-result reporting into a server-side primitive (`tool_call_results` table + `POST /agent/messages/:msgId/tool-results/:toolCallId/execution`). It's the natural place to close the wiring gap. Without this, any airdrop campaign shipping with the current code has a visible bug for every winner who swaps via chat instead of DCA.

"Solved" means: a chat-driven `execute_swap` (or other quest-eligible tool execution) signed and broadcast by the app produces a row in `agent_quest_events` and advances `agent_user_quests` exactly as a scheduled DCA-driven swap does today.

## Background

### Findings

**Quest taxonomy already in place.** Postgres ENUM `airdrop_quest_id` declares 5 IDs in `internal/storage/postgres/migrations/20260423000003_create_agent_quest_events.sql:4`: `swap`, `bridge`, `defi_action`, `alert`, `dca`. Stage 1 reader treats only the first three as canonical (`internal/service/airdrop/quests/eligibility.go:71`); `alert` + `dca` are deliberately stubbed as TBD with Product.

**The classifier was written for chat-driven flows.** `internal/service/airdrop/quests/tracker.go:85-95` (on `main`) recognizes `execute_swap`, `execute_contract_call`, `sign_evm_tx`, `erc20_approve` — chat-driven tool names. PR #188 (v-mjfl, 2026-04-28) explicitly added the `execute_*` family to the scheduler quest_metadata extraction at `internal/service/agent/headless.go:88-94`:
> "Without this, every execute_*-driven TxProposal gets quest_metadata=nil silently and Stage 3 quest tracking goes dark for the entire signing surface."

**Spec is unambiguous on chat-driven counting.**
- `docs/airdrop-specs/overview.md`: "Every time the mobile app reports back 'the user just signed and broadcast a transaction'… If it was a swap above $10, we tick the 'swap quest' box for that user."
- `docs/airdrop-specs/sibling-mobile-app.md`: "The user just uses VultiAgent normally. They chat with the agent, build swap/bridge/DeFi-action transactions, sign and broadcast each via the existing tool-call UX. The app's only specific responsibility here is what it already does: call `POST /conversations/{id}/tool-result` after a tool-built transaction is signed and broadcast. The backend's quest-tracking layer (Stage 3) hooks off that existing notification — no new API call from the app for quests."
- vultiagent-app issue #230: "The app has no specific responsibility for quest tracking itself — the existing tool-call result reporting already fires the signal the backend needs."

**Current implementation drift.** Quest event write happens at `internal/api/tx_proposals.go:131-138`:
```
questEvent := quests.ClassifySignedProposal(proposal, req.TxHash, signedAt, s.questTracking)
if err := s.txProposalRepo.MarkSignedAndRecordQuest(ctx, id, publicKey, req.TxHash, questEvent); ...
```
Reachable only via `POST /agent/tx-proposals/:id/sign`. Walking the call graph:
- `agent_tx_proposals` rows are created **only** by the scheduler (`internal/service/scheduler/scheduler.go:355` is the sole `txProposalRepo.Create` callsite).
- The app calls `signTxProposalApi` (`src/services/network/agentApi.ts:178`) only when `result.proposalId` is set; `proposalId` comes from `pendingTx.data.proposal_id` (`src/features/agent/lib/signAndBroadcast.ts:195`), set only by `buildSchedulerPendingTx` (`src/features/agent/lib/txProposal.ts:109`).
- Chat-driven `execute_swap` flows through `useToolExecution.ts:291` with `result.proposalId === undefined`, so `signTxProposalMutation` is skipped and `postOutcome` (the va-qbzv POST at `useToolExecution.ts:304`) is the only server-side signal — and that endpoint (`internal/api/tool_results.go`) does not call the classifier.

**va-qbzv changes nothing in this picture.** `git diff main...feat/va-qbzv-tool-call-results -- internal/api/tx_proposals.go internal/storage/postgres/tx_proposal.go internal/service/airdrop/` is empty. The `ExecutionResult` struct (`internal/types/parts.go`) carries `Status / TxHash / ApprovalTxHash / ExplorerURL / Chain / Error` — no `tool_name`, no `quest_metadata`.

**The execute_swap output already carries quest_metadata.** From mcp-ts emission, the `UIMessagePart`'s `Output` field contains `quest_metadata`. The app holds it; today it just isn't forwarded to the new POST.

### External Context

- Stage 3 spec lives in `ai-combinator/vultihub/docs/airdrop-specs/`. Sean to confirm with whoever drives the airdrop mission that chat-driven counting is still the contract (private decisions in Slack/Linear/Discord aren't visible from code).
- Quest activation is `2026-05-01` (today is 2026-05-06). Quests are technically live in env-gated config; no real users yet (pre-launch).
- Idempotency on `agent_quest_events.tool_call_id` PRIMARY KEY guarantees the `INSERT ... ON CONFLICT DO NOTHING` contract Stage 3's writer already assumes (`eligibility.go` docstring, point 3).

## Current Thinking

### Where the hook lives

In `saveToolExecutionResult` (`internal/api/tool_results.go:88-170`), at the success-status branch *after* `repo.InsertToolCallResult` returns `applied=true` with terminal status `success`. We have `msgID`, `toolCallID`, `txHash`, and `chain` from the request. Need two extra pieces:

1. **`tool_name` and `quest_metadata`.** Two options:
   - **(preferred) App POSTs them alongside the result.** The app already has them in the tool's `output` payload. Adds two optional fields to `ToolExecutionResultRequest`. Simplest, no probe latency.
   - **(alternative, server-only) Look up the part by `(message_id, tool_call_id)` in `agent_messages.parts` JSON, parse `Output`, extract `quest_metadata` and `toolName`.** Keeps wire shape stable but adds a JSON probe per write.

   Lean toward app-POST. Server-only path is fallback if app-shape change is contentious.

2. **A new classifier seam.** `ClassifySignedProposal(*types.TxProposal, ...)` today. Refactor its body into `classifyByToolMetadata(toolName, questMetadata, quoteData, txHash, signedAt, cfg)`. Both paths call the inner; proposal-sign path becomes a thin wrapper that pulls the metadata off the proposal struct.

3. **A new persistence call.** `RecordQuestEventForToolResult(ctx, publicKey, msgID, toolCallID, event)`. Reuses `recordQuestEventTx`. Idempotent on `tool_call_id` PK. `tx_proposal_id` becomes NULL for chat-driven events (column is already documented as a soft reference; no FK).

### Wire shape change for the app POST

Add two optional fields to `ToolExecutionResultRequest`:
- `tool_name string` (e.g. `"execute_swap"`)
- `quest_metadata json.RawMessage` (passthrough — server doesn't need to parse to write the audit row, classifier already knows the shape)

App side: `useToolExecution.ts` already has the `Output` payload at the point it calls `postOutcome`. Forward `tool_name` from the part and `quest_metadata` from the output. If `quest_metadata` is absent on the output (tool didn't emit it), POST without the field — server treats absent metadata as "not quest-eligible" via the existing classifier.

### What this does NOT change

- Proposal-sign flow keeps working unchanged. Scheduled DCA swaps continue to write quest events via `tx_proposals.go`. Both paths converge on `recordQuestEventTx`.
- Schema doesn't change. `agent_quest_events` and `agent_user_quests` reused as-is. `tx_proposal_id` is already a soft reference (no FK) — for chat-driven events it's NULL.
- Stage 1 reader (`eligibility.go`) needs no changes; reads `agent_user_quests`, written the same way regardless of source path.

### Tests

Two new integration tests in `internal/api/tool_results_test.go`:
- `swap_happy_path_chat_driven` — POST with `tool_name=execute_swap` + above-min-USD `quest_metadata` → asserts row in `agent_quest_events` and `agent_user_quests` advanced.
- `below_min_usd_chat_driven` — same shape, sub-threshold metadata → asserts row in `agent_quest_events` (audit, `counted=false`) but `agent_user_quests` not advanced.

Plus a unit test for `classifyByToolMetadata` confirming behavioral parity with the proposal-sign path on a fixture exercising both.

### Effort

~200–300 LOC. Mostly the classifier refactor + the repo wrapper + the request-shape extension on both sides + two integration tests. No schema changes.

## Constraints

- **Both paths converge on the same classifier and the same persistence write.** Don't fork the classifier logic; extract a shared seam and have both call it. Drift between paths is the bug we're shipping if we copy-paste.
- **Idempotency on `agent_quest_events.tool_call_id`.** Re-POSTs of the same `tool_call_id` must not double-count quest progress. Existing PK guarantees this; ensure the new wrapper uses `INSERT ... ON CONFLICT DO NOTHING` like Stage 3's existing writer.
- **`tx_proposal_id` is nullable for chat-driven events.** Don't synthesize a bogus proposal row to keep the column populated; soft-reference semantics already permit NULL.
- **Don't change `agent_quest_events` or `agent_user_quests` schemas.** Reuse as-is.
- **Quest write happens on terminal `success` only.** Same semantics as the proposal-sign path. Don't fire on `submitted`, `failed`, or `cancelled`.
- **Status set must align with `v-klyp`.** `{success, failed, cancelled, submitted}`. Quest write hooks off the same terminal-`success` branch `v-klyp` enforces.

## Assumptions

- Chat-driven counting is still the contract. Sean to confirm with airdrop mission owner that the spec hasn't been privately revised. If it has and chat counting is deferred post-launch, this ticket goes on hold.
- The app reliably has `quest_metadata` on the tool output for quest-eligible tools (`execute_swap`, `execute_contract_call`, `sign_evm_tx`, `erc20_approve`). PR #188 wired the scheduler extraction; mcp-ts emits the same shape; the app receives it.
- The `public_key` resolution path used by the proposal-sign endpoint (`GetMessageOwnerPublicKey`) is the same one the tool-results endpoint already uses. No new authz seam.
- The classifier's behavior on `execute_swap` with valid `quest_metadata` matches what the proposal-sign path produces today — same classifier function, same fixture inputs.

## Non-goals

- **Resolving `alert` / `dca` quest validators.** Spec stubs as TBD with Product; out of scope.
- **Modifying Stage 1 eligibility reader.** Reads `agent_user_quests`; both paths write it the same way.
- **Backfilling historical chat-driven swaps to count retroactively.** Forward-only; historical chats stay uncounted.
- **Schema migrations.** None needed for this ticket.
- **Anything covered by `v-klyp`** (cross-tenant uniqueness + status enum). Separate concern, but `v-klyp` lands first since the quest write hooks off the terminal-`success` branch `v-klyp` is strict about.
- **Anything covered by `v-cgfd`** (review polish). The "quest fan-out follow-up code comment" open question on `v-cgfd` is now obsolete — close that as no-op when this ticket lands.

## Dead Ends

- **Defer entirely** (the original first-subagent recommendation). Closed by this ticket. Verdict shifted from "defer" to "fix" once spec citations + classifier code on `main` confirmed chat-driven counting was the contract from day one.
- **Server-side JSON probe to extract `tool_name` + `quest_metadata` from `agent_messages.parts`.** Functional but adds a per-write JSON parse and locks the data into a probe. Forwarding from the app via two optional fields is simpler; the app already has the data. Keep server-only as fallback if app-shape change is contentious.
- **Synthesize a proposal row for chat-driven events.** Tempting because both paths could share `MarkSignedAndRecordQuest` then, but it forces a bogus `tx_proposal_id` and breaks the scheduler's invariant that proposal rows correspond to scheduled tasks. Better to extract the classifier seam and have a second persistence wrapper that takes a NULL `tx_proposal_id`.

## Open Questions

1. **Who confirms chat-driven is still the contract?** Stage 3 spec says yes; private decisions could have shifted it. Sean to confirm with airdrop mission owner before plan-mode commits.
2. **`quest_metadata` as `json.RawMessage` or parsed struct on the wire?** RawMessage simpler; parsed struct gives stricter API contract. Planner picks; doesn't affect correctness.
3. **App POSTs `tool_name` + `quest_metadata` always, or only when tool is quest-eligible?** Cleanest: "always send, server filters." Alternative: app gates on a known-set, only sends when relevant. Server-side filtering is more robust to future tool additions; lean toward always-send.
4. **What happens on classifier rejection (sub-threshold, non-allowlist contract, etc.)?** Stage 3 still writes the audit row to `agent_quest_events` with `counted=false`. Ensure the chat-driven path matches that — don't skip audit writes on rejected events. Verify with the integration test.

## Repos / branches

- **agent-backend** — work on top of `feat/va-qbzv-tool-call-results` (PR #235), in the main repo dir `agent-backend/`. Touches `internal/api/tool_results.go`, `internal/service/airdrop/quests/tracker.go`, and adds a repo wrapper alongside `recordQuestEventTx`.
- **vultiagent-app** — work on top of `worktree-va-qbzv` (PR #339), in the main repo dir `vultiagent-app/`. Touches `useToolExecution.ts` (forward `tool_name` + `quest_metadata` on `postOutcome` call) and the `txOutcomeApi.ts` wire-shape definition.
- **mcp-ts, vultisig-sdk** — no changes; emission of `quest_metadata` already works.

## What this ticket closes (review threads)

- Codex MED #235-3 ("Quest fan-out missing"). The codex finding was right about the symptom; the dismissal that came out of the first research pass was wrong because it missed the spec citations confirming chat-driven counting was always intended. This ticket is the proper resolution.
