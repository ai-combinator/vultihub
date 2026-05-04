---
id: v-llqd
status: in_progress
deps: [v-qjvi]
links: []
created: 2026-04-23T23:05:51Z
type: feature
priority: 1
assignee: Jibles
---
# vultiagent-app: Add execute_* flow cards consuming ExecutePrepResult (Send/Swap/ContractCall/Lp) + chain-id guard

## Objective

Add the four execute_* flow cards (SendFlowCard, SwapFlowCard, ContractCallCard, LpFlowCard) + `executePrep` parser + chain-id cross-check, wired to the **existing** useToolExecution hook from tic-6c08 (already on main). Register the new cards in `toolUIRegistry` alongside — do NOT remove — the legacy `BuildTxCard`. This gives the app a rendering path for the new mcp-ts execute_* tools while staying minimal-risk.

## Context & Findings

### Decision: use existing useToolExecution, not port useTransactionFlow

The original #164 replaced `useToolExecution` (tic-6c08 work, on main) with `useTransactionFlow` + `txFlowReducer`. That's a nicer architecture (pure reducer, cleaner state machine) but carries risk — the author's own in-file TODO at `useTransactionFlow.ts:6529`:
> "before absorbing useToolExecution (build_* family), port over: IDLE_TIMEOUT_MS (45s) safety-net for stream drops / hung backends; ready_to_approve branch for late-arriving output after a timeout"

Landing useTransactionFlow without those safety nets is a regression risk on flaky networks. Since tic-6c08's useToolExecution is already working on main with those safety nets, the minimum-risk path is:
- Adapt the 4 Execute cards to consume ExecutePrepResult via useToolExecution
- Keep the existing hook architecture
- Leave useTransactionFlow port as a separate follow-up ticket (not this one)

### Reviewer findings this PR must address

**Gomes B5 — Cancel button removed (on #164's AgentApprovalBar):**
> "preferably-blocking: cancel button removed from approval UI… Result: in password phase, a user who decides mid-flow not to sign has no visible reject path."

**This ticket does NOT touch AgentApprovalBar** — that's handled by already-in-flight PR 229 (ActionCard + 2-phase approval), which introduces an edit-to-cancel affordance. Rebase this PR on top of 229 once 229 merges. Do not duplicate 229's approval-bar changes.

**Gomes B3 — retry double-broadcasts ERC-20 approval (on `useTransactionFlow.ts:245`):**
> "preferably-blocking: retry can re-broadcast an already-confirmed ERC-20 approval… duplicate approve(spender, amount) on-chain, wasted gas, confused user."

Since we're NOT porting useTransactionFlow, this finding may not apply — but verify: the Execute cards themselves should track whether the approval leg is already confirmed and skip it on retry. Use `useToolExecution`'s existing completedSteps tracking.

**Gomes B6 — duplicate cards stack (on `AssistantMessageView.tsx:120`):**
> "`findLastBuildToolIndex` deletion removes suppression… if the model fires `build_*` twice in a turn (quote → final), both cards now stack."

This ticket does NOT delete `findLastBuildToolIndex` or `coalesceParts`. Both stay on main. Execute cards are deduplicated by `historicalToolGuard` (also tic-6c08, on main) via toolCallId, which is a different mechanism and works for execute_* tools without needing the legacy suppression.

**Gomes B7a — stepperConfig null validation:**
> "PR body claims `stepperConfig` allowed to be null with default fallback. This validator rejects missing/null `stepperConfig`."

Ensure `executePrep.validateTxArgsShape` accepts missing/null `stepperConfig` with a default fallback — do NOT reject, since ticket 1's mcp-ts always emits it but historical results may not.

**Gomes B8 — row-level crashes on malformed rows (on `DefiPricesCard.tsx:115` and 8 other cards):**
> "preferably-blocking: render crash on malformed rows (representative). Same class of issue across 8 cards. Container validated, rows not. One malformed row from any backend = render crash."

Execute cards are action cards, not query cards — different risk profile. But keep defensive per-row rendering in any row-level sub-components to avoid crashes.

**0xApot — chain_id cross-check praised:**
> "Nice — the chain_id vs chainRegistry cross-check is a real defense against a backend regression signing against the wrong network."

Include the chain-id cross-check in `executePrep.validateTxArgsShape`. Gomes agreed and proposed widening it.

### What this ticket covers

**Cherry-pick from `refactor/v-pxuw-tool-consolidation` (vultiagent-app):**
- `217bff5` — refactor(agent): align ExecutePrepResult with backend txArgs shape (v-pxuw Task 10) — includes the chain-id cross-check
- `06ac1b1` — feat(agent): add execute_* flow cards + toolUIRegistry wiring (v-pxuw Task 10) — adapt to use tic-6c08's useToolExecution
- `62f02cd` — refactor(agent): unify execute_* cards behind TxStepCard shell (v-wzgs) — closed ticket, already done
- `c6903c1` — refactor(agent): flow cards read r.labels.*, typed ExecutePrepResult (v-wldr) — labels-only rendering
- Parts of `8189f21`, `86752b2`, `6d70b99`, `ff5a0ff` — small polish commits on ExecutePrepResult / cards

**Do NOT cherry-pick:**
- `2736246` — feat(agent): add txFlowReducer state machine (PR 1) — leave useTransactionFlow out
- `7fba772` — feat(agent): useTransactionFlow + migrate SendFlowCard canary (PR 3) — leave out
- `d45e3d2` — feat(agent): migrate Swap, ContractCall, Lp cards to useTransactionFlow (PR 4) — leave out
- `ccc5d63` — feat(agent): split ApprovalContext into consent + password phases (PR 2) — PR 229 does this
- `057707f` — feat(agent): route build_* tools through consent-first approval (PR 5) — PR 229 does this
- `b321124` — feat(agent): delete legacy hooks + route stepper off phase directly (PR 6) — do NOT delete useToolExecution
- `1e9cc9d` — refactor(agent): drop coalesceParts + AggregateToolIndicator (v-pxuw Task 12) — keep those
- `76264a1` — refactor(agent): flatten Execution.* family into ExecutionStepper — skip
- `ee3776b` — Trustwallet Maven — ticket 8
- `e14f503` — WalletCore probe — ticket 9
- `7db747b` — CrudCards — ticket 13
- `5b9fb98` — simplify ExecutePrepResult + split CRUD cards per action — CRUD part is ticket 13

### Dependency chain

- **Must rebase on PR 229** (ActionCard + 2-phase approval) before opening. PR 229 must merge first.
- **Must depend on ticket 1 (mcp-ts execute_*)** being deployed before this PR is meaningful to merge (the execute_* tools must exist server-side). State this in PR body.
- **toolUIRegistry**: add new entries, keep legacy `BuildTxCard` registered under existing `build_*` keys. Dual-rendering during transition.

## Files

**New:**
- `src/features/agent/lib/executePrep.ts` + `__tests__/executePrep.test.ts`
- `src/features/agent/components/tools/ExecuteCards/SendFlowCard.tsx`, `SwapFlowCard.tsx`, `ContractCallCard.tsx`, `LpFlowCard.tsx`
- `src/features/agent/components/tools/ExecuteCards/contractCall.ts`, `index.ts`
- `src/features/agent/components/tools/__tests__/ContractCallCard.test.ts`
- `src/features/agent/components/tools/ExecutionHistoricalGuard.tsx` — small wrapper around tic-6c08's `historicalToolGuard` for execute_* cards

**Modified:**
- `src/features/agent/lib/toolUIRegistry.ts` — ADD 4 new entries; keep legacy `build_*` entries
- Possibly `src/features/agent/hooks/useToolExecution.ts` — small adaptations to accept ExecutePrepResult shape in addition to BuildTxResult
- `src/features/agent/lib/pendingAgentRow.ts` — if needed for ExecutePrepResult parsing

**Do NOT touch:**
- `src/features/agent/contexts/ApprovalContext.tsx` — PR 229 owns
- `src/features/agent/components/AgentApprovalBar.tsx` — PR 229 owns
- `src/features/agent/components/AssistantMessageView.tsx` — do NOT delete `findLastBuildToolIndex` or `coalesceParts`
- `src/features/agent/components/tools/BuildTxCard.tsx` — leave registered
- `src/features/agent/components/tools/SignTypedDataTool.tsx` — leave as-is (PR 229 adapts it)
- `plugins/withTrustwalletMavenRepo.js` — ticket 8
- `src/services/addressDerivation.ts` — ticket 9

## Acceptance Criteria

- [ ] PR opened as **draft** against vultiagent-app main (rebased on PR 229's `feat/action-card-send-swap` branch if 229 hasn't merged, else rebased on main), title `feat(agent): execute_* flow cards consuming ExecutePrepResult`
- [ ] 4 Execute cards registered in `toolUIRegistry`; legacy `BuildTxCard` also registered
- [ ] `executePrep.validateTxArgsShape` includes chain_id vs chainRegistry cross-check
- [ ] `stepperConfig` null/missing tolerated with default fallback
- [ ] `historicalToolGuard` wraps execute_* cards (no re-entry on scroll-back / conversation reopen)
- [ ] `useToolExecution` — NOT replaced — still hosts the 45s IDLE_TIMEOUT_MS safety net
- [ ] `AgentApprovalBar` — UNCHANGED in this PR (comes from PR 229)
- [ ] `findLastBuildToolIndex` + `coalesceParts` — UNCHANGED on main
- [ ] `pnpm test`, `pnpm lint`, `pnpm tsc --noEmit` clean; all 517+ jest tests pass
- [ ] PR body has **"Blockers" section**: "Do NOT merge before PR 229 merges and ticket 1 (mcp-ts execute_*) is deployed"
- [ ] PR body references tickets 1 and 11, notes Query cards (ticket 11) follow after this

## Gotchas

- `tic-6c08` is marked `in_progress`; its implementation is on main but manual verification is pending. Validate manually that historical conversations with completed tool calls still render correctly after this PR lands (that's tic-6c08's outstanding validation).
- The adapters to make useToolExecution accept ExecutePrepResult may need care. Two approaches: (a) discriminated union for `Build` vs `Execute` prep types; (b) translate ExecutePrepResult to a common internal shape. Choose whichever needs fewer call-site changes.
- ExecutePrepResult has both `labels` and `raw` (v-wldr). UI renders from `labels` only. The `raw` field echoes for LLM next-turn reasoning. Do not render `raw` in cards.
- 8 cards (query cards in ticket 11) need defensive per-row validation. This ticket is action cards only — 4 of them — so that pattern is less critical here but good practice.
- `BuildTxCard` staying registered means old tool-call parts still render. Keep a small TODO in `toolUIRegistry.ts` head pointing at `v-ujuc` for eventual build_* retirement.

## Notes

**2026-04-24T00:06:39Z**

Draft PR opened: https://github.com/vultisig/vultiagent-app/pull/242 (base=feat/action-card-send-swap, stacked on PR 229)
