---
id: v-xtub
status: open
deps: []
links: []
created: 2026-04-29T01:30:17Z
type: task
priority: 2
assignee: Jibles
---
# Retire BuildTxCard / unify transaction card render path

## Objective

Close the production-divergence window where the same logical transaction operation renders through two different cards depending on which tool family the agent emitted: legacy `build_*` and `morpho_prepare_*` tools render through `BuildTxCard.tsx`, while the new `execute_*` tools render through the `ExecuteCards/` family. Ship a unified `TransactionCard` wrapper that accepts either prep shape and produces the same UX. Addresses gomes' R2 Concern B (R1 #3) on PR #242 — explicitly out of scope for that PR but tracked here as fast-follow.

## Context & Findings

- Two render paths exist today, each driven by a different reducer/hook/state machine:
  - `BuildTxCard.tsx` ← `useToolExecution` (older state machine; `state.step === 'idle' | 'preview' | 'awaiting_approval' | 'signing' | 'success' | 'error'`)
  - `ExecuteCards/{Send,Swap,ContractCall}FlowCard.tsx` ← `useTransactionFlow` (new reducer; `kind: 'fetching_prep' | 'awaiting_confirm' | 'signing' | 'done' | 'failed' | 'incomplete'`)
- Different prep shape: `execute_*` tools emit `ExecutePrepResult` (structured envelope with `txArgs`, `approvalTxArgs`, `stepperConfig`, `resolved.labels`); `build_*` and `morpho_prepare_*` emit raw transaction data + a derived preview function (`buildTxPreview`). Don't try to harmonize the data layer — that's backend's job. Unify only at the rendering layer.
- User-visible divergence: different stepper styles, different fee presentation, different error classifications (`classifyBuildError` vs `getUserFriendlyError`), different button styling. Per gomes: "same UX should look the same regardless of which tool the agent picked."
- Asymmetric workarounds: `AssistantMessageView.tsx:119, 180-188` has a dedupe special-case that only applies to `build_*` (suppresses non-final occurrences when the older state machine emits quote → final). `execute_*` doesn't have or need this — confirms the divergence isn't theoretical.
- Registry split: `toolUIRegistry.tsx:180-185` exact entries for `execute_*`; `toolUIRegistry.tsx:207-212` `BUILD_TX_PREFIXES = ['build_', 'morpho_prepare_']` fallthrough to `BuildTxCard`.
- Tools rendered today through BuildTxCard: `build_solana_tx`, `build_spl_transfer_tx`, `build_thor_send`, `build_sui_send`, `build_xrp_send`, `build_trc20_transfer`, `build_ton_jetton_transfer`, plus all `morpho_prepare_*`. Production-active path for many real flows — not a minor fallback.

- **Two options evaluated:**
  - **(a) Full retirement** — migrate every remaining `build_*` / `morpho_prepare_*` tool to emit `ExecutePrepResult` on the mcp-ts side, then delete `BuildTxCard.tsx`. Has multi-PR backend dependencies. Ticket `v-ujuc` already tracks part of this for token-layer tools (SPL, TRC-20, jetton, Sui token, XRP) — but NOT for `morpho_prepare_*`, `build_thor_send`, or other build_* families. So full retirement waits on backend coordination across multiple teams.
  - **(b) Unified wrapper now, retirement later** — ship `TransactionCard` that accepts either prep shape and delegates internally. Lower risk, no backend dependency, faster. Adds a small translation layer that option (a) later removes once mcp-ts is fully migrated.
- **Recommended:** ship (b) first to close the divergence window without blocking on backend. Track BuildTxCard deletion as a follow-up gated on `v-ujuc` + any additional backend tickets for `morpho_prepare_*` etc.

- **Auto-fire-approval fix (`v-gopv`) lands FIRST in PR #242.** That ticket replaces the auto-fire `useEffect` in BOTH `useToolExecution` and `useTransactionFlow` with an explicit-tap input-bar trigger. By the time this ticket starts, both card families share the same approval-surface mechanics — so the wrapper's approval surface already converges via v-gopv's work. This ticket inherits that and unifies the visual + label + error-classification surfaces.

- **Stakeholder forcing function:** gomes' R2 review on PR #242 calls this divergence STILL OPEN: "Same logical operation ('send a token') still renders through 2 different card/state machines depending on which tool the model picked. R1 ask was either (a) ship BuildTxCard retirement here, or (b) ship a unified card. v-ujuc 'deferred upstream' doesn't close the production-divergence window." Landing the unified card answers his framing without waiting on backend.

## Files

Repo: `vultiagent-app`. New branch suggested: `feat/unify-tx-cards`. Lands AFTER PR #242 (v-gopv).

- `src/features/agent/components/tools/TransactionCard.tsx` — NEW. Wrapper component. Accepts either prep shape, delegates to ExecuteCards' family or BuildTxCard internally.
- `src/features/agent/lib/toolUIRegistry.tsx` — point both `build_*`/`morpho_prepare_*` and `execute_*` registry entries at `TransactionCard`; remove `BUILD_TX_PREFIXES` fallthrough as a separate render path
- `src/features/agent/components/AssistantMessageView.tsx` — at lines 119 and 180-188, either remove the `build_*`-only dedupe (if both families converge to the same behavior under the wrapper) or generalize it to cover both families. Confirm during implementation.
- `src/features/agent/components/tools/BuildTxCard.tsx` — keep as internal implementation behind the wrapper for now. Full deletion is a follow-up gated on backend tool retirement.
- `src/features/agent/components/tools/ExecuteCards/{Send,Swap,ContractCall}FlowCard.tsx` — keep as internal implementations; reachable through the wrapper.
- `src/features/agent/components/tools/__tests__/TransactionCard.test.tsx` — NEW. Cover prep-shape branching, both internal cards reachable, visual parity assertions.
- Existing card tests stay as internal-component tests, no rewrite needed.

## Acceptance Criteria

- [ ] One registry entry per `build_*`, `morpho_prepare_*`, and `execute_*` tool, all pointing at `TransactionCard` — no `BUILD_TX_PREFIXES` fallthrough as a separate render path
- [ ] `TransactionCard` accepts both prep shapes (`ExecutePrepResult` and the legacy build output) and produces the same visual stepper, row layout, and approval surface for both
- [ ] Single fee-presentation surface across both prep shapes
- [ ] Single error-classification surface across both (one `getUserFriendlyError` path or equivalent unified API)
- [ ] `AssistantMessageView.tsx`'s `build_*` dedupe special-case either removed (if both families now behave identically) or generalized to cover `execute_*`
- [ ] Visual parity: a `build_send_tx` card and an `execute_send` card render identically when given equivalent inputs (compare side-by-side screenshots)
- [ ] Existing card-level tests continue to pass; new `TransactionCard` tests cover prep-shape branching and historical-replay correctness for both paths
- [ ] Manual smoke: `build_solana_tx`, `build_thor_send`, a `morpho_prepare_*`, `execute_send`, `execute_swap`, `execute_contract_call` all render the unified card with no UX regressions
- [ ] No `BuildTxCard` or `ExecuteCards/*` references outside the wrapper internals (consumers only import `TransactionCard`)
- [ ] PR description links back to this ticket and to gomes' R2 Concern B
- [ ] Lint and type-check pass

## Gotchas

- v-gopv (auto-fire fix in PR #242) MUST land first. This ticket builds on top: TransactionCard's approval surface uses the new tap-driven model uniformly; do not reintroduce auto-fire for the BuildTxCard path.
- Don't try to merge the hooks. `useToolExecution` and `useTransactionFlow` have different state shapes — keep them separate, internal to each card. Wrapper picks the inner card by prep shape; each runs its own hook.
- `morpho_prepare_*` is NOT covered by `v-ujuc` (token-layer only). Full BuildTxCard deletion will need additional backend tickets before it's possible.
- `AssistantMessageView.tsx:119, 180-188` dedupe logic exists because the older state machine emits multiple "build" outputs per turn (quote → final). Verify whether the unified TransactionCard needs the same behavior for `execute_*` — currently doesn't appear to.
- `ExecutionHistoricalGuard` wraps `execute_*` cards but not `build_*`. The wrapper needs to apply the right historical-replay semantics per inner path; don't blindly apply the guard to the build path or you may break legacy replay.
- Out of scope: full deletion of `BuildTxCard.tsx`, full backend retirement of `build_*` tools. Both deferred to follow-up tickets gated on backend (mcp-ts) coordination.
- Out of scope: the auto-fire-approval anti-pattern. Tracked separately under `v-gopv` and lands in PR #242.
