---
id: v-fjph
status: closed
deps: []
links: [v-duhk, va-qbzv, v-klyp, v-cgfd, v-xvcz, v-bvin]
created: 2026-05-05T23:18:42Z
type: task
priority: 2
assignee: Jibles
---
# Migrate frontend tool cards to hydrate from executionResult; retire useAutoRunOnMount historical-state branch

## Problem

va-qbzv (vultisig/agent-backend#235 + vultisig/vultiagent-app#339) migrated tool execution outcomes from a local outbox to a backend `tool_call_results` table. The **write** path is built. The **read-back-on-hydrate** path is only built for the swap/send/sign flow (`useTransactionFlow` + `ExecuteCards/*Card`). Every other frontend-executing tool component still ignores the persisted `executionResult` on historical mount and falls through to `useAutoRunOnMount`, whose default render-state is synthetic `success` — so historically-failed actions render as successes when the chat is reopened.

"Solved" means: every persisted-result tool card hydrates from `executionResult` first and only falls back to the live execution hook when no row exists; once that's true for every card, `useAutoRunOnMount`'s historical-state branch is dead code and can be deleted (or kept as a defensive `unknown` fallback only). Pairs with `v-duhk` (durability/defensive default) as the sequential consumer-side half — v-duhk floors the lying at "unknown", this ticket retires the floor entirely by making every card read the truth.

## Background

### Findings

**va-qbzv shape (already shipped on the linked PRs).** Tool execution results are POSTed to `POST /agent/messages/:msgId/tool-results/:toolCallId/execution`, server-side idempotent on `tool_call_id` via upsert. The conversation read-side joins `tool_call_results` and merges `executionResult` onto matching tool parts. See `.tickets/va-qbzv.md` for the full architecture and `.tickets/v-duhk.md` for the durability companion.

**Reference implementation (already correct).** The swap/send/sign cards (`SendFlowCard`, `SwapFlowCard`, `ContractCallCard`) hydrate from `executionResult` correctly via `useTransactionFlow`. They are the shape to copy — not work to redo.

**The hook with two jobs.** `src/features/agent/hooks/useAutoRunOnMount.ts` today does:
  (a) auto-execute the underlying action when a tool card mounts in a live conversation, and
  (b) provide a render-state to the card on historical mount, defaulting to synthetic `success` when no execution data is available.
Job (a) is real and stays. Job (b) is the source of the wrong-answer bug.

**Cards still on the wrong path** (named explicitly by reviewers — codex via gomes + CodeRabbit, multiple threads on app#339):
- `AddChainTool` (CodeRabbit, `AddChainTool.tsx:59`)
- `RemoveChainTool` (CodeRabbit, `RemoveChainTool.tsx:43`)
- `AddCoinTool` (CodeRabbit, `AddCoinTool.tsx:65`)
- `RemoveCoinTool` (CodeRabbit)
- `CreateVaultTool` (CodeRabbit, `CreateVaultTool.tsx:46`)
- `ImportVaultTool` (CodeRabbit)
- `VerifyEmailTool` (CodeRabbit)
- `ScheduleTaskTool` (partial — has a separate cancellation-race bug tracked elsewhere; only the hydration half is in scope here)

**Reviewer pile this closes** (after both v-duhk and this ticket land):
- Per-tool "doesn't read executionResult on hydrate" CodeRabbit cluster on app#339 (open threads on each file above).
- codex blocking-3 on app#339 ("auto-run replay defaults to success") — fully closed once both tickets land.

### External Context

- `tool_call_results` rows carry `status` + `error` + tool-specific result fields. Cards must render based on those, not synthesize.
- The backend upsert is idempotent on `tool_call_id`; reads of `executionResult` are repeatable and stable across reopens.

## Current Thinking

### The contract being codified

For any tool card whose tool produces a persisted `tool_call_results` row, the render decision is:

1. **`executionResult` present** (historical mount, hydrated from backend) → render based on its `status` + `error` + tool-specific fields. **No synthetic states.**
2. **`executionResult` absent + live execution active** (live mount, first time the tool is firing) → render based on the live execution hook's state machine.
3. **`executionResult` absent + no live execution** (genuinely unknown — should be vanishingly rare after `v-duhk` lands) → render an honest "unknown / not confirmed" terminal. **Don't render success.**

### Shape of the migration

The contract should be enforced **structurally**, not by copy-paste convention across seven components. The swap/send/sign path uses `useTransactionFlow` as that locus. The migrated tools likely don't need anything as heavy — a small shared hook of the shape `useToolCardState(executionResult, liveHookState)` (or similar) is the kind of locus that makes the contract a structural guarantee rather than a per-file convention. Planner picks the exact API; the **load-bearing** call is "shared, not copy-pasted".

### Retire the historical-state branch in `useAutoRunOnMount`

Once every card in the list above hydrates from `executionResult` directly, job (b) of `useAutoRunOnMount` is unreachable. Two options:
- **Delete outright.** Cleanest; planner verifies no remaining call-site reads the historical-state output.
- **Keep as defensive `unknown` fallback only.** Cheaper to land if a residual call-site is found late. Acceptable, but only if the default is `unknown`, never synthetic `success`.

Either is fine. The non-negotiable is that no path produces synthetic `success` on historical mount.

### Failure mode this closes

User runs `add_chain(Solana)` yesterday → fails (RPC timeout, etc.) → failure persisted to `tool_call_results` as `{status: "failed", error: "..."}` → user reopens chat today → backend returns the row → `AddChainTool` ignores it → card renders "Solana chain added" → user believes the chain is added → reality says otherwise. Same shape for every tool in the list above.

### Why sequential after v-duhk, not interleaved

- **v-duhk** changes the synthetic-success **default** in `useAutoRunOnMount` to `unknown`. Defensive. Stops the lying for cards that haven't been migrated yet. Also adds the critical-section-await-the-POST mechanism for write-path durability.
- **This ticket** retires the need for the default by migrating the consumer side — every card reads `executionResult` directly. After this lands the hook's historical-state branch is dead code.

Land v-duhk first (or in parallel — they don't conflict). Without v-duhk's "unknown" floor, a half-migrated state during this ticket's rollout means some cards are honest and some still lie. With v-duhk landed, an unmigrated card during this ticket renders honestly rather than fabricating success.

## Constraints

- **Read `executionResult` first, fall back to live hooks second.** Card render decision is hydration-first; live execution is the second-class path that runs only when there is no row to read.
- **No synthetic `success` on historical mount, in any card or any helper.** The whole point of the migration; if any card or shared helper ever renders `success` without an `executionResult.status === "success"` row, the migration didn't happen.
- **Encapsulate the contract once, not seven times.** A shared hook/helper is preferred over per-component conditionals. Whatever the locus, the rule "executionResult-first, live-second, unknown-third" must be enforced in one place that the migrated cards consume.
- **Don't touch the reference implementation.** `SendFlowCard`/`SwapFlowCard`/`ContractCallCard` already hydrate correctly via `useTransactionFlow`; they're not in scope.
- **Don't fix the cancellation race in `ScheduleTaskTool`.** Out of scope; tracked elsewhere. Only its hydration half migrates here.
- **Per-tool `status` + `error` + tool-specific fields drive the render.** Don't introduce a single "did it succeed?" boolean adapter that loses the shape — different tools surface different signals on `executionResult` and the cards must read the ones they need.

## Assumptions

- Every persisted-result tool's `executionResult` row contains the fields each card needs to render its full state set (success / failure / tool-specific success-but-with-warnings, etc.). If a card finds it can't reproduce its render contract from the persisted shape alone, the schema needs a follow-up — flag it; don't paper over it with a synthetic state.
- v-duhk lands first (or alongside). If it doesn't, this ticket is partial — unmigrated cards continue to lie until each is migrated, and the rollout has a window where some cards are honest and some aren't.
- The list of frontend-executing tool components in scope is complete. If a card surfaces during implementation that produces a persisted `tool_call_results` row but isn't on the list, it's in scope — extend the migration rather than excluding it.

## Non-goals

- **Cancellation-race fix in `ScheduleTaskTool`** (separate ticket — only the hydration half is in scope here).
- **`AssistantMessageView` `executionResult` spoofing fix** (separate, mechanical).
- **`useReceiptPoll` metadata-loss fix** (separate, mechanical).
- **`partExecutionResult.ts` `Array.isArray` exclusion** (separate, mechanical).
- **Cancelled-replay-renders-failed branch in `useToolExecution`** (separate, mechanical).
- **Anything in the durability mechanism** (covered by `v-duhk`: critical-section-await-the-POST, sync-failed UI affordance, OTA gating).
- **Backend-side mirror bug at `agent.go:5095`** (server-tool replay synthesizing `queued` for tools that never had rows) — different file, different concern, separate backend ticket.
- **Migration of pre-PR pending outbox entries** (codex blocking-4 on app#339).
- **The HIGH backend findings on agent-backend#235** (cross-tenant overwrite + status enum).

## Dead Ends

- **Migrate one card at a time, no shared hook.** Tempting because each card's state-machine differs slightly, but: it's literally the bug we're fixing — a contract enforced by per-file convention is a contract that drifts. Seven components copy-pasting "if executionResult then ... else liveHookState ... else unknown" will drift; one shared hook will not. The planner's call on the exact API surface, but "shared locus" is settled.
- **Keep the synthetic-success default as a "harmless" fallback after migration.** Once every card hydrates correctly, the branch is unreachable in normal operation — but if it stays as `success`, any future card that forgets to migrate silently regresses to lying. Either delete the branch or keep it as `unknown` only. Synthetic `success` is never an acceptable default.
- **Reuse `useTransactionFlow` for non-tx tools.** It carries broadcast/receipt/outcome semantics that don't map onto Add/RemoveChain, Add/RemoveCoin, vault create/import, etc. Build a smaller shared hook for the non-tx persisted-result cards rather than dragging tx machinery into them.

## Open Questions

1. **Exact API surface of the shared hook.** `useToolCardState(executionResult, liveHookState)` is the rough shape; planner decides whether it's a hook, a render-prop helper, or an HOC. Whatever the form, the contract (executionResult-first, live-second, unknown-third) lives in one place.
2. **Visual contract for the "unknown" terminal.** Tied to v-duhk's open-question 5 — the planner there should land on a single visual treatment ("We couldn't determine the outcome of this action" vs reusing skipped-stub). This ticket inherits whatever v-duhk picks; if v-duhk hasn't picked one, surface the dependency.
3. **Status enum and per-tool field surface on `executionResult`.** Verify each migrated card's `status`/`error`/tool-specific fields are actually populated by the existing va-qbzv write path. If a card needs a field that isn't being persisted today, the schema needs a follow-up — don't synthesize the missing state.
4. **Delete vs keep-as-`unknown` for the historical-state branch in `useAutoRunOnMount`.** Either is acceptable; planner picks based on what the migration footprint looks like at the end. The non-negotiable is no synthetic `success` path remaining.

## Repos / branches

- **vultiagent-app** — work on top of `worktree-va-qbzv` (PR vultisig/vultiagent-app#339), in the main repo dir `vultiagent-app/` (user moved everything to main repo dirs; not a worktree).
- **agent-backend, mcp-ts, vultisig-sdk** — main, no changes expected for this ticket.

## What this ticket closes (review threads)

Once landed alongside v-duhk:
- Per-tool "doesn't read executionResult on hydrate" CodeRabbit cluster on app#339:
  - `AddChainTool.tsx:59`, `RemoveChainTool.tsx:43`, `AddCoinTool.tsx:65`, `RemoveCoinTool.tsx`, `CreateVaultTool.tsx:46`, `ImportVaultTool.tsx`, `VerifyEmailTool.tsx`.
- codex blocking-3 on app#339 (auto-run replay defaults to success) — fully closed once both tickets land.
- The `ScheduleTaskTool` cancellation-race thread is **not** closed by this ticket (separate cancellation barrier work).
