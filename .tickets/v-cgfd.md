---
id: v-cgfd
status: closed
deps: []
links: [va-qbzv, v-klyp, v-duhk, v-fjph, v-xvcz]
created: 2026-05-06T00:09:08Z
type: chore
priority: 2
assignee: Jibles
---
# va-qbzv review polish: small mechanical fixes across backend + app

## Problem

The va-qbzv PR pair (vultisig/agent-backend#235 + vultisig/vultiagent-app#339) accumulated a long tail of small, no-design-call mechanical fixes during review (codex via gomes + CodeRabbit). Each is independent and low-risk; together they're the polish layer that makes both PRs un-draftable. None require architectural decisions. Batching them avoids ticket-per-fix churn.

"Solved" means: every item below has landed; the corresponding review threads can be marked resolved or replied-to.

## Background

### Findings — backend (agent-backend#235)

1. **`/messages/since` skips tool-result hydration.** `internal/storage/postgres/message.go:GetSinceCursor` (around `message.go:150`-`:160`) returns `messagesFromDB(msgs)` directly. Unlike other read paths it does NOT call `attachToolCallResults`. Reconnect clients miss persisted tool outcomes. (CodeRabbit + codex MED #235-5.)

2. **Server-tool replay synthesizes `queued` for tools that never had rows.** `internal/service/agent/agent.go:5095` falls back to `{"status":"queued"}` when there's no `tool_call_results` row, but for MCP/server-side tools there usually never will be one (those tools don't POST through the new endpoint). Effect: completed server-executed tools look in-flight in replay context. (CodeRabbit, open thread.)

3. **Prompt-injection escaping for `txHash`, `explorerUrl`, `error` before insertion.** `internal/service/agent/prompt.go:1067, :1077` writes these fields verbatim into the system prompt. Any value containing newlines or prompt-like text could become "instructions" the LLM follows on the next turn. Mitigation: newline-strip + length-cap before insertion. (Codex LOW #235-6, CodeRabbit `prompt.go:null`.)

4. **403 body shape on owner mismatch.** `internal/api/tool_results.go:143` returns `{"error":"forbidden"}`; the contract is "shape like a missing-message response so a foreign `msgId` is indistinguishable from a non-existent one." Status 403 stays; body shape changes. Pin in `tool_results_test.go:150`. (CodeRabbit.)

   *Overlap note: also listed in `v-klyp` (backend HIGHs ticket). If `v-klyp` lands first, this item is already done — leave in here as a safety net.*

5. **Same-`tool_call_id` / different-message regression test fixture.** `internal/storage/postgres/tool_call_results_upgrade_test.go:261` only seeds one message per `tool_call_id`. Add a fixture: two messages, same `tool_call_id`, assert second write doesn't mutate the first row. (CodeRabbit.)

   *Same overlap caveat as #4 — leave in for safety.*

### Findings — app (vultiagent-app#339)

6. **`AssistantMessageView` `executionResult` spoofing.** `src/features/agent/components/AssistantMessageView.tsx:35` spreads `input` before conditionally assigning `executionResult`. If `part.executionResult` is absent, an attacker-controlled `input.executionResult` can survive into `data` and be treated as a persisted backend outcome. Fix: reorder so `executionResult` is set after the spread, with a fallback only if neither source provides it. (CodeRabbit.)

7. **`partExecutionResult.ts` array exclusion.** `src/features/agent/lib/partExecutionResult.ts:readPartExecutionResult` uses `typeof === 'object'` as a guard, which passes for arrays. An array `executionResult` would be incorrectly cast. Add an `Array.isArray` exclusion. (CodeRabbit.)

8. **`useToolExecution` cancelled-status branch.** `src/features/agent/hooks/useToolExecution.ts:117` — if the backend returns `status: 'cancelled'` without an `error`, reopened chats render "Transaction failed." instead of a cancellation message. Branch: render cancellation copy on `status === 'cancelled'` before falling through to the failure path. (CodeRabbit.)

9. **`useReceiptPoll` metadata preservation on receipt failure.** `src/features/agent/hooks/useReceiptPoll.ts:12` — when receipt polling fails after broadcast was accepted, the failure variant drops `txHash`/`explorerUrl`/`chain`. Historical hydration uses those fields to keep the explorer link and mark broadcast as completed on reopen. Preserve them in the failure path. (CodeRabbit.)

### External Context

- Each fix has at least one linked review thread (codex deep review or CodeRabbit) on the corresponding PR. Resolving each thread is part of "done."
- The backend prompt-injection LOW (#3) is a small concession to a "not a regression but worth noting" finding — acceptable defense-in-depth touch.

## Current Thinking

- **Land in any order.** No item depends on another. Reviewer can squash or split into commits as preferred.
- **Each item is a small surgical edit at the named callsite.** No new abstractions, no helpers, no shared utilities.
- **No new tests beyond what each fix needs.** Fixture changes for #5 are required; CodeRabbit-flagged tests for the others can be inlined where natural; new test files aren't a goal.
- **Pre-launch test bar applies.** One smoke per backend item where natural; mechanical app items don't all need their own dedicated test if the existing test covers the fix path.

## Constraints

- **Don't expand any item into an abstraction.** Every item is "fix the named line/file." If a planner is tempted to introduce a shared utility, a hook factory, or a sanitization library — reject; those are separate tickets.
- **Don't fix anything not on this list.** Adjacent issues spotted during work go into a follow-up ticket, not this one.
- **Items 4 and 5 may already be done by `v-klyp`.** If so, mark closed; otherwise apply here.
- **Status strings must align with `{success, failed, cancelled, submitted}`.** Same set `v-klyp` enforces. If `v-klyp` hasn't landed yet, use the same set anyway.

## Assumptions

- Each named file/line still exists at the location codex/CodeRabbit cited. If a file has moved, find it; the fix is the same.
- The companion durability + hydration tickets (`v-duhk`, `v-fjph`) and the backend HIGHs ticket (`v-klyp`) cover the structural changes; nothing in this batch is load-bearing for the larger PR pair to merge.

## Non-goals

- **Anything covered by `v-duhk`** (durability, critical section, sync UI affordance, synthetic-success-default replacement).
- **Anything covered by `v-fjph`** (per-tool `executionResult` hydration migration).
- **Anything covered by `v-klyp`** (cross-tenant uniqueness scope + status enum enforcement).
- **`ScheduleTaskTool` cancellation race** — separate ticket pending design.
- **Replay nonce, legacy outbox migration, quest fan-out** — dismissed under pre-launch policy or pending separate investigation.

## Open Questions

1. **Items #4 and #5 — overlap with `v-klyp`.** If `v-klyp` lands first, these items are already done. Coordinate to avoid duplicate work; keeping them listed here is the safety net.
2. **Quest fan-out follow-up code comment.** Pending the in-flight investigation. May add a small comment in `internal/api/tool_results.go` near the insert site pointing at `internal/storage/postgres/tx_proposal.go` as the canonical quest writer. If the investigation says "yes, add the comment," include as item #10. If it says "expand quest scope," that becomes its own ticket and this item is dropped.

## Repos / branches

- **agent-backend** — items 1–5, on top of `feat/va-qbzv-tool-call-results` (PR #235), in the main repo dir `agent-backend/`.
- **vultiagent-app** — items 6–9, on top of `worktree-va-qbzv` (PR #339), in the main repo dir `vultiagent-app/`.
- **mcp-ts, vultisig-sdk** — no changes.

## What this ticket closes (review threads)

- CodeRabbit threads on backend `message.go` (`/messages/since` hydration), `agent.go:5095` (queued synthesis), `prompt.go:null` (escaping), `tool_results.go:143` + `tool_results_test.go:150` (403 body shape), `tool_call_results_upgrade_test.go:261` (regression test).
- CodeRabbit threads on app `AssistantMessageView.tsx:35`, `partExecutionResult.ts:null`, `useToolExecution.ts:117`, `useReceiptPoll.ts:12`.
- Codex MED #235-5 (`/messages/since` hydration) and codex LOW #235-6 (prompt-injection).
