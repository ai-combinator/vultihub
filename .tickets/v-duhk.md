---
id: v-duhk
status: closed
deps: []
links: [va-qbzv, v-fjph, v-klyp, v-cgfd, v-xvcz, v-bvin]
created: 2026-05-05T22:29:20Z
type: bug
priority: 2
assignee: Jibles
---
# Close durability gap in tool-result POST: critical-section the round-trip + replace synthetic-success default

## Problem

va-qbzv (vultisig/agent-backend#235 + vultisig/vultiagent-app#339) retired the local `recentActions` outbox in favor of POSTing tool execution results to the backend's new `tool_call_results` table. The POST is fire-and-forget: on broadcast/sign tools the app updates local state and dispatches the POST without awaiting the ack. If the app crashes, OTA-restarts, or is force-quit between the local action and the ack, the backend never learns the outcome. On reopen the chat card synthesizes `success` (because `useAutoRunOnMount` defaults replay state to `success` when no execution data is available), and the LLM has no row for that `tool_call_id`. Result: wrong-answer renders, and — critical for tx tools — re-suggestion of already-completed transactions, i.e. double-spend risk.

"Solved" means: every tool that produces a persisted tool-result row treats the round-trip to the backend as part of its critical section; the synthetic-success default is replaced with an honest "unknown" terminal; and a sync-failure UI affordance lives on the card itself with a Retry sync action.

## Background

### Findings

**va-qbzv shape (already shipped on the linked PRs).** Tool execution results are POSTed to `POST /agent/messages/:msgId/tool-results/:toolCallId/execution`, server-side idempotent on `tool_call_id` via upsert. The conversation read-side joins `tool_call_results` and merges `executionResult` onto matching tool parts. See `.tickets/va-qbzv.md` for the full architecture.

**Where the fire-and-forget POSTs originate (vultiagent-app):**
- `src/features/agent/hooks/useToolExecution.ts` — generic hook called by every persisted-result tool card.
- `src/features/agent/hooks/useTransactionFlow.ts` — the tx-specific orchestrator (broadcast → receipt → outcome).
- `src/services/network/txOutcomeApi.ts` — POST helper for the new endpoint.
- Per-tool components also POST directly today: `RemoveCoinTool.tsx`, `AddCoinTool.tsx`, `AddChainTool.tsx`, `RemoveChainTool.tsx`, `CreateVaultTool.tsx`, `VerifyEmailTool.tsx`, `ScheduleTaskTool.tsx`, vault import.

**Existing critical-section primitive.** `src/features/updates/unsafeToRestartStore.ts` already gates OTA during signing — currently consumed by `useToolExecution.ts`, `useTransactionFlow.ts`, `CreateVaultTool.tsx`, and `useKeygenFlow.ts`. This is the primitive to extend; do not invent a new one.

**Synthetic-success default.** `src/features/agent/hooks/useAutoRunOnMount.ts` initializes replay state as `success` when no execution data is available. This is the line that turns a recoverable-durability bug into a wrong-answer bug: when the backend has no row, the card invents success rather than showing the truth.

**Per-tool hydrate gap.** Several tool cards do not read `executionResult` on hydrate (CodeRabbit cluster on `AddChainTool`, `RemoveChainTool`, `AddCoinTool`, `CreateVaultTool`, `VerifyEmailTool` etc.). Replacing the synthetic-success default subsumes this cluster — once the default is "unknown" instead of "success", these cards stop rendering invented outcomes regardless of whether they read `executionResult` on hydrate yet.

**Reviewer pile (codex via gomes + CodeRabbit) on PRs #339 and #235:**
- codex blocking-1 on app#339 — best-effort delivery / fire-and-forget POST.
- codex blocking-2 on app#339 — OTA mid-tx safety.
- codex blocking-3 on app#339 — auto-run replay defaults to success.
- CodeRabbit threads on `useToolExecution.ts`, `useTransactionFlow.ts`, `RemoveCoinTool.tsx`, `txOutcomeApi.ts`, `chatSessionStore.ts` — variants of the same durability complaint.

**In-progress WIP on `worktree-va-qbzv`** (vultiagent-app): modified `queries.ts`, modified `txOutcomeApi.test.ts`, new `conversationQueryMerge.test.ts`. Attacks adjacent symptoms (read-side merge / cache shape), should land alongside this ticket but is not the durability fix itself.

### External Context

- The new endpoint is idempotent server-side via upsert on `tool_call_id`, so Retry sync is safe — same payload, same key, server collapses duplicates.
- `tool_call_id` uniqueness is currently scoped per-conversation in practice, but the va-qbzv schema treats it as globally unique. Backend-side cross-tenant overwrite on `tool_call_id` is a flagged-but-orthogonal concern (see Non-goals).

## Current Thinking

### The rule

**If a tool produces a persisted `tool_call_results` row, the POST is part of the tool's critical section — not fire-and-forget.**

Applies uniformly to every persisted-result tool: broadcast tx, sign-typed-data, vault create/import, add/remove chain/coin, schedule task, verify email — basically all of `useToolExecution` + `useTransactionFlow` + the per-tool components currently calling the POST helper directly. Pure read tools that don't persist are out of scope.

### Mechanism

Extend `unsafeToRestartStore` so the critical section now also covers the round-trip to the backend, not just the local signing/broadcast step:

1. Inside the section, the POST is **awaited**, not fired and forgotten.
2. Network blips retry **in-memory** (React Query mutation retry config — same shape va-qbzv landed: `retry: 3`, exponential backoff). No on-disk retry queue.
3. OTA cannot fire while the section is active.
4. App-crash mid-section is the one remaining failure mode and is handled by the UI affordance below.

### UI states (per-card)

Three states for any persisted-result tool card:

1. **Action confirmed** (instant) — the existing success/broadcast UI renders as soon as the local action completes (e.g. `txHash` known). **Do not gate this on the save** — the on-chain event is real regardless of whether we've persisted it yet.
2. **Syncing** (typically <500ms) — a subtle secondary indicator alongside the confirmed state (small spinner / faint pulse / "saving" microcopy). The critical-section restart-guard remains active throughout.
3. **Terminal:**
   - **Synced** — indicator disappears or flips to a tiny check. User typically never noticed.
   - **Sync failed** — explicit, persistent on-card affordance with a **Retry sync** button. Microcopy in the spirit of: *"We couldn't save this to your chat history. The transaction succeeded on chain (link), but it may not appear correctly when you reopen this conversation."* Retry re-fires the same POST with the same `tool_call_id` (idempotent server-side via existing upsert). The error must live on the **card** (the unit of work that didn't complete), not in a toast.

### Replace the synthetic-success default

Independently from the critical-section work — and this is the piece that turns a recoverable bug into a wrong-answer bug — replace `useAutoRunOnMount`'s replay-state default of `success` with an explicit "unknown / not confirmed" terminal state on hydrate. When the backend has no row AND there's no live execution to read from, the card surfaces honestly that we don't know what happened. This is the change that prevents card-fabricated success and double-spend re-suggestion to the LLM.

### Why one rule and not split-by-tool

Originally tempted to split tx tools (await) from non-tx tools (best-effort) on a UX-cost argument — the tx leg already has perceived latency, so adding await is "free", whereas a non-tx tool feels snappier without it. Rejected: the engineering cost of building the await/retry/UI states is the same either way; the inconsistency cost (per-tool judgment calls in review, in tests, and in users' mental model) is high; and several non-tx tools (vault create/import, schedule task) are themselves slow enough that a small "syncing" indicator is invisible. One rule defends much more cheaply than per-tool judgment.

### Why no on-disk outbox

Tempted to re-introduce a durable on-disk outbox to cover the app-crash-mid-section case. Rejected: va-qbzv explicitly retired that pattern and accepted app-kill-mid-POST as a rare, tolerable failure mode. The Retry-sync UI affordance plus an honest "unknown" terminal covers the same ground without re-adding the two-system durability problem va-qbzv just removed.

## Constraints

- **Do not gate the on-chain "Action confirmed" UI on the save round-trip.** The chain event is real regardless; a slow backend must not look like a failed broadcast.
- **Single rule across all persisted-result tools.** No per-tool special-casing of await vs best-effort.
- **No new on-disk outbox.** Retry is in-memory only; durability across app-kill is provided by the Retry-sync UI affordance, not persistence.
- **Idempotency contract preserved.** Retry sync re-fires the same POST with the same `tool_call_id`; server-side upsert deduplicates. Don't introduce client-side state that would mutate the payload between attempts.
- **`unsafeToRestartStore` is the existing OTA-blocking primitive — extend it, don't fork.** Existing call-sites (`useToolExecution.ts`, `useTransactionFlow.ts`, `CreateVaultTool.tsx`, `useKeygenFlow.ts`) shouldn't grow inconsistent semantics.
- **Sync-failed error lives on the card, not in a toast.** The card is the unit of work that didn't complete; toasts are dismissible and lose the link to the unfinished work.

## Assumptions

- The existing server-side upsert on `tool_call_id` is truly idempotent under concurrent retries (e.g., Retry sync tapped twice). Worth a spot-check at implementation time; if not, single-flight on the client.
- React Query mutation retry covers the transient-failure window adequately. If the typical broadcast → ack round-trip is the assumed <500ms, the retry budget (3 + exponential backoff) is generous; if backends regularly take longer, revisit.
- Every persisted-result tool card has access to the parent `messageId` at POST time. va-qbzv noted this as a concern for the 6 legacy synchronous tools (Add/RemoveChain, Add/RemoveCoin, Import/CreateVault) — verify it actually survived the va-qbzv merge before extending the critical section to them.
- The "syncing" indicator is visually subtle enough that a happy-path user (who lands at Synced inside ~500ms) does not perceive any delay. If the indicator is too prominent, the UX cost of one rule rises.

## Non-goals

- **Migration of pre-PR pending outbox entries** (codex blocking-4 on app#339). Separate decision about pre-launch user contract; defer to its own ticket.
- **Backend-side cross-tenant overwrite on `tool_call_id`** and **status-enum unenforced findings** on agent-backend#235. Orthogonal mechanical schema fixes; address in a separate backend-only ticket.
- **Quest fan-out into `agent_quest_events` from the new POST.** Separate ownership question; not coupled to durability.
- **The in-progress WIP on `worktree-va-qbzv`** (`queries.ts`, `txOutcomeApi.test.ts`, `conversationQueryMerge.test.ts`). Adjacent read-side cleanup; should land alongside this work but is not the durability fix.
- **Pure read tools that do not persist a `tool_call_results` row.** Out of scope — there's nothing to be durable about.

## Dead Ends

- **Split tx vs non-tx tools (await on tx, best-effort on non-tx).** Same engineering cost both ways; per-tool inconsistency makes review and tests harder; "snappy non-tx UX" is illusory because vault create/import and schedule task are themselves slow. One rule wins on defendability.
- **Durable on-disk outbox to cover app-crash-mid-section.** Re-introduces the two-system problem va-qbzv just removed. Retry-sync UI + honest "unknown" terminal cover the same risk surface without the architectural cost.
- **Surface the sync error in a global toast.** Toasts are dismissible and decouple the failure from the unit of work that produced it. The card is the unit; the error must live there.
- **Gate the "Action confirmed" UI on the save round-trip.** Makes a slow ack look like a failed broadcast and erodes user trust in the "transaction succeeded" affordance. The on-chain event is real regardless of save state.

## Open Questions

1. **Retry-sync UX wording and density.** The microcopy *"We couldn't save this to your chat history. The transaction succeeded on chain (link), but it may not appear correctly when you reopen this conversation."* is a draft — Mia (design) should weigh in on tone/length, especially for non-tx tools where the "(link)" doesn't apply.
2. **Syncing-indicator visual.** Subtle spinner vs faint pulse vs microcopy ("saving") — pick one for consistency. May need design input.
3. **Retry sync retry budget.** Tap-to-retry only, or one auto-retry on app foreground after a backgrounded failure? Auto-retry-on-foreground reduces the number of users who ever see the sync-failed card, but adds a second async path to reason about.
4. **App-crash detection on next launch.** When the app cold-starts and finds local execution state for a tool that never persisted, can we proactively re-fire the POST on launch (with the original `tool_call_id`)? Cheap to add and would close the app-kill-mid-section case for many users without needing them to interact with Retry sync. Adjacent to but not the same as a durable outbox — the local state is already there from the existing in-memory store, we'd just be reading it on launch instead of treating it as lost.
5. **What to render for the "unknown" terminal.** A neutral card shape ("We couldn't determine the outcome of this action") vs reusing the existing skipped-stub treatment. Tied to the synthetic-success replacement — the planner needs to land on a single visual contract.
6. **Where to extend `unsafeToRestartStore`'s scope.** Wrap the POST inside each existing call-site, or have `txOutcomeApi.ts` own the section and let call-sites stay unaware. The latter is cleaner but assumes all persisted-result POSTs really do route through that helper — verify.

## Repos / branches

- **vultiagent-app** — work on top of `worktree-va-qbzv` (PR vultisig/vultiagent-app#339).
- **agent-backend** — minor support only if needed (e.g. ack semantics tightening), on top of `feat/va-qbzv-tool-call-results` (PR vultisig/agent-backend#235).
- **mcp-ts** and **vultisig-sdk** — main, no changes expected.

## What this ticket closes (review threads)

Once landed, the following collapse:
- codex blocking-1 on app#339 (best-effort delivery)
- codex blocking-2 on app#339 (OTA mid-tx safety) — solved as a side effect of extending the critical section
- codex blocking-3 on app#339 (auto-run replay defaults to success) — solved by the synthetic-success replacement
- CodeRabbit threads on `useToolExecution.ts`, `useTransactionFlow.ts`, `RemoveCoinTool.tsx`, `txOutcomeApi.ts`, `chatSessionStore.ts`
- Per-tool "doesn't read `executionResult` on hydrate" cluster (`AddChainTool`, `RemoveChainTool`, `AddCoinTool`, `CreateVaultTool`, `VerifyEmailTool`)
