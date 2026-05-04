---
id: v-ugzl
status: in_progress
deps: []
links: []
created: 2026-04-28T02:19:05Z
type: task
priority: 1
assignee: Jibles
---
# Port useTransactionFlow + txFlowReducer from v-pxuw into v-llqd ExecuteCards

## Objective

Port `useTransactionFlow` (323 LOC) + `txFlowReducer` (302 LOC) + reducer test suite (587 LOC) from the closed v-pxuw branch into v-llqd's ExecuteCards so that multi-step transaction flows (ERC-20 approve + swap, broadcast → confirmation, per-step retry) render with distinct phases on the stepper instead of collapsing into a single misleading "signing" blob.

## Context & Findings

The current ExecuteCards consume `useToolExecution`, whose state machine is a single latch (`idle → preview → signing → success/error`). That works for native sends but actively misinforms during multi-tx flows:

- **Lying stepper today** (`shared.tsx:76`): when phase===`signing`, picks `broadcastIndex` as the active dot — so during MPC signing (10-30s), the stepper shows `Quote ✓  Sign ✓  Broadcast ●` even though nothing has been signed or broadcast yet. Sign-time failures surface at the broadcast position.
- **Duplicate signing indicator**: the same `state.step==='signing'` flag drives both the stepper's "active" dot AND a separate `<TimelineEntry text="SIGNING TRANSACTION">` block inside each ExecuteCard (`SendFlowCard:114`, `SwapFlowCard:90`, `LpFlowCard:84`, `ContractCallCard:87`).
- **No multi-tx support**: ERC-20 swaps require two MPC signs (approve, then swap) — both txs ship from mcp-ts as `txArgs` + `approvalTxArgs` in the prep payload (see `mcp-ts/src/tools/execute/execute_swap.ts`). With `useToolExecution` they collapse into one undifferentiated blob.
- **No broadcast → confirm distinction**: card claims APPROVED the moment broadcast returns, before chain mines.

The fix is a state machine that models the real phases the user is going through. v-pxuw built that hook; PR #164 (CLOSED) bundled it but didn't ship. The hook + reducer are battle-tested through six v-pxuw review iterations (`7fba772` initial, then `b321124`, `0238652`, `f7c4817`, `5b9fb98`, `130ac0b`).

**Reference implementation** lives on locally-available branch `refactor/v-pxuw-tool-consolidation`:
- `src/features/agent/hooks/useTransactionFlow.ts` (323 LOC)
- `src/features/agent/lib/txFlowReducer.ts` (302 LOC)
- `src/features/agent/lib/__tests__/txFlowReducer.test.ts` (587 LOC)
- 4 ExecuteCards in v-pxuw shape (each ~30-40 LOC SHORTER than current because state-machine logic moves into the hook)

**Architectural gotchas from PR #229's rework** (the hook was written before #229 landed — these renames and reshapings need adaptation):
- `useApproval` → `useSigningApproval` (split per gomes' R3 #6)
- `useApproval`/`requestConsent` semantics changed: signing path uses `useSigningApproval.requestApproval(...)`, schedule path uses `useConfirmation.requestConfirmation(...)`. Multi-tx swap is two signing claims in sequence.
- `historicalToolIds: Set<string>` → branded `replayGuardIds: ReadonlySet<ReplayGuardId>` consumed via `isReplayed(replayGuardIds, toolCallId)` helper from `historicalToolGuard.ts`
- `awaiting_consent` reducer state DROPPED in #229 — if v-pxuw's hook references it, remove and route directly to `awaiting_approval`
- `ApprovalRequest` is now a discriminated `ApprovalSlotEntry` (`kind: 'signing' | 'confirmation'`); only `'signing'` carries vaultId
- ApprovalProvider now mounts both `SigningApprovalCtx` + `ConfirmationCtx` from a single provider in `contexts/ApprovalContext.tsx`

**Multi-tx slot lifecycle (highest-risk integration)**: the slot is single-occupancy. For ERC-20 approve+swap, the bar must claim → sign approval → release → re-claim → sign swap. Verify `useSigningApproval`'s `releaseApprovalSlot`-ownership pattern + `key={toolCallId}` bar-remount strategy supports back-to-back claims for the same tool call. May need a small lifecycle tweak in `ApprovalContext.tsx` to support two distinct phases under one toolCallId, or a v-pxuw pattern where each phase has its own ephemeral phase-id.

**Rejected approaches:**
- *Strip the stepper entirely; rely on the thinking block.* Rejected: thinking block stops updating after `execute_swap` returns — has no signal for the post-tap signing/broadcast progression.
- *One-line stopgap: flip `broadcastIndex` / `signIndex` preference + delete TimelineEntry blocks (~20 LOC).* Rejected: doesn't help ERC-20 approve+swap (two-tx flows still collapse to one signing blob), and ERC-20 swaps will be common in production.
- *Defer the port to a v-ujuc-style follow-up.* Rejected: ERC-20 swap jankiness blocks the v-llqd ship being usable for the most-common swap shape; ~2 days of port work earns truthful UX for what users will actually do.

## Files

**NEW (port from `refactor/v-pxuw-tool-consolidation`):**
- `vultiagent-app/src/features/agent/hooks/useTransactionFlow.ts`
- `vultiagent-app/src/features/agent/lib/txFlowReducer.ts`
- `vultiagent-app/src/features/agent/lib/__tests__/txFlowReducer.test.ts`

**REWRITE** (4 ExecuteCards — reference v-pxuw shape, lighter than current):
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/SendFlowCard.tsx`
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/SwapFlowCard.tsx` (the ERC-20 approve+swap case)
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/LpFlowCard.tsx`
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/ContractCallCard.tsx`

**MODIFY:**
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/shared.tsx` — `StepperRow` replaces the broadcastIndex/signIndex preference hack with reducer-driven `kind` per dot; drop comment at lines 5-8 about the deferred port (this ticket IS the deferred port)
- `vultiagent-app/src/features/agent/contexts/ApprovalContext.tsx` — only if back-to-back signing claims need a lifecycle tweak (verify first; may not need any change)

**VERIFY (unchanged unless integration requires):**
- `vultiagent-app/src/features/agent/components/AgentApprovalBar.tsx` — `key={toolCallId}` remount behavior across the two phases of a multi-tx flow
- `vultiagent-app/src/features/agent/components/tools/ExecutionHistoricalGuard.tsx` — already adapted to `replayGuardIds` per v-llqd fixup; should not need further change

**REFERENCE COMMITS in v-pxuw history:**
- `7fba772 feat(agent): useTransactionFlow + migrate SendFlowCard canary (PR 3)`
- `b321124`, `0238652`, `f7c4817`, `5b9fb98`, `130ac0b` (review-loop polish)

## Acceptance Criteria

- [ ] `useTransactionFlow.ts` and `txFlowReducer.ts` ported, adapted to `useSigningApproval` / `replayGuardIds` / `isReplayed` / discriminated `ApprovalSlotEntry` / removed `awaiting_consent`
- [ ] `txFlowReducer.test.ts` ported and passing under the post-#229 mock surfaces
- [ ] 4 ExecuteCards rewritten to delegate state-machine logic to `useTransactionFlow`; net LOC across the four files DECREASES vs current (per v-pxuw shape)
- [ ] `<TimelineEntry text="SIGNING TRANSACTION">` blocks removed from all 4 ExecuteCards (now redundant — stepper carries the phase truthfully)
- [ ] `StepperRow` in `shared.tsx` no longer uses the broadcastIndex/signIndex preference hack; phases derive from the reducer
- [ ] tsc + biome + full jest suite (unit + components) pass
- [ ] Manual: native ETH self-send (single-tx flow) still works end-to-end with stepper showing `Quote ✓ → Sign ● → Broadcast ✓` truthfully
- [ ] Manual: **ERC-20 approve + swap** (e.g., USDC → ETH on Arbitrum) shows two distinct biometric prompts; stepper shows `Quote ✓ → Approve ● → Sign ○ → Broadcast ○` then `Approve ✓ → Sign ●`; both txs sign cleanly
- [ ] Manual: card terminal APPROVED row only appears after broadcast confirms (or after `waitForEvmReceipt` resolves), not on broadcast return
- [ ] PR commit message references v-pxuw commit `7fba772` + the closed jumbo PR #164 as the source of truth
- [ ] Comment at `shared.tsx:5-8` (referencing v-ujuc as the deferred port) updated or removed — this ticket completes that work

## Gotchas

- v-pxuw hook predates #229; expect tsc errors on first cherry-pick from the renamed APIs — adapt before integrating with cards.
- `ApprovalContext` slot is single-occupancy; back-to-back signings need release-between-claims. Verify v-pxuw's pattern; may need a lifecycle tweak in `useSigningApproval` to support two `requestApproval` calls in sequence for the same `toolCallId`.
- `AgentApprovalBar` uses `key={toolCallId}` to remount transient state between txs — two-tx flow shares one toolCallId, so consider keying on `toolCallId + phase` or accept that draft state persists across phases.
- `_base.ts` in mcp-ts emits `approvalTxArgs` alongside `txArgs` for ERC-20 swaps; the prep parser already preserves both (`parsePrep` in `executePrep.ts`). Thread the optional approval through `useTransactionFlow` cleanly.
- Don't delete `useToolExecution` — `BuildTxCard` and `SignTypedDataTool` still use it. The port is purely for the 4 ExecuteCards.
- Reducer tests assume specific approval/signing service mocks — verify mocks match post-#229 service shapes (signAndBroadcast, ApprovalContext).
- Validator log `categories=[fabricated_fee]` and "max fee per gas less than block base fee" is a SEPARATE bug (mcp-ts execute_swap ships `max_fee_per_gas: "0"`); track in mcp-ts cleanup, NOT this ticket.
- Confirmation polling: `waitForEvmReceipt` already exists in `signAndBroadcast.ts`; verify v-pxuw's hook integrates it in the `confirming` phase, or note as a tight follow-up.
- Don't migrate `BuildTxCard` to `useTransactionFlow` in this ticket — legacy build_* tools may persist in agent-backend rollout; keep `BuildTxCard` on `useToolExecution` until those retire.

## Notes

**2026-04-28T02:25:04Z**

Explore complete. Plan: T1 reducer+tests (port mechanically), T2 hook port+adapt to post-#229 (HIGH risk, full review), T3 shared.tsx StepperRow refactor, T4 cards rewrite x4 (parallel).

**2026-04-28T02:28:23Z**

T1 done: txFlowReducer + tests ported byte-identical from v-pxuw 7fba772; 38/38 tests pass; tsc+biome clean. Commit 20a92f0.

**2026-04-28T02:40:55Z**

T2 ported (224e67c) + T3 shared.tsx refactored (8921c0f). T2 spec+code-quality review: approved (faithful to v-pxuw 7fba772; multi-tx single-claim preserved). T3: StepperRow now phase-driven, broadcastIndex/signIndex hack gone, mapPhase removed. Dispatching T4a-d in parallel.

**2026-04-28T02:59:30Z**

Build complete. 8 commits on surgical/v-llqd-execute-cards (20a92f0 → 65a8a8a). All 4 cards rewritten on useTransactionFlow; net -63 LOC across cards (-89 with shared.tsx). tsc clean, biome clean (175 files), reducer suite 38/38, jest 656/656 (2 pre-existing suite-load failures unrelated to v-ugzl: receiveCardSchemas + toolUIRegistry fail at expo-local-authentication ESM transform — also fails on baseline). Two T2 follow-ups deferred to separate tickets: (1) PASSWORD_INVALID rewind dead-end (consentRegisteredRef blocks re-registration after slot externally clears); (2) lost-consent on contended requestApproval returning false. Both inherit from v-pxuw 7fba772 by design.
