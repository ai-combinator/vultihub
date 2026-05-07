---
id: v-xgqo
status: closed
deps: []
links: []
created: 2026-04-30T07:10:38Z
type: task
priority: 2
assignee: Jibles
---
# mcp-ts: revert execute_* inner txArgs to legacy shapes (keep outer ExecutePrepResult wrapper); drop PR #320

## Objective

Revert the `tx_encoding` inner-shape contract that the execute_* family introduced. Keep the outer `ExecutePrepResult` wrapper (stepperConfig, resolved echoes, approvalTxArgs, quest_metadata) — those bring real product wins. Replace the inner `txArgs` with the legacy shapes that the still-unmigrated `build_*` tools already emit. Eliminates the parser dual-shape problem at its root: the vultiagent-app parser stops needing two execution branches, and v-uneu becomes a one-line mcp-ts fix as a side effect.

## Context & Findings

- The execute_* migration (v-pxuw) bundled two independent changes: (1) a new outer wrapper `ExecutePrepResult` carrying `stepperConfig` / `resolved` / `approvalTxArgs` / `quest_metadata`, and (2) a new inner-shape contract via `tx_encoding` discriminator. Only (1) was strictly required for the wrapper's user-facing wins; (2) was an elective inner-shape cleanup.
- (2) created the dual-shape situation: ~7 tools emit canonical, ~9 (`build_spl_transfer_tx`, `build_xrp_send`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build-cw20-transfer`, Rujira swap/staking, Morpho prepare_*, etc.) still emit legacy. The parser must handle both indefinitely — and getting that wrong is what produced v-uneu.
- vultiagent-app PR #320 added a per-chain canonical dispatcher (`txProposal/` folder, 12 files, 1815 lines vs original 1191) to bridge the gap. PR is open, not yet merged.
- Reverting (2) while keeping (1) costs: TypeScript narrowing on `EvmTxArgs` inside mcp-ts (real but small), forward-extensibility of the `TxEncoding` union, conceptual cleanliness of having a discriminator. Gains: drop 1815 lines of parser refactor, drop the per-tool migration trigger, no half-migrated state to reason about.
- No user-visible loss. agent-backend doesn't introspect inner `txArgs` shape (verified: `quest_metadata` extraction reads sibling fields, not `tx_encoding`). vultiagent-app's `useToolExecution`/`BuildTxCard`/`Execution.*` consume the outer wrapper, not the inner shape.
- The legacy parser (`vultiagent-app/src/features/agent/lib/txProposal.ts` pre-refactor, equivalently `txProposal/legacy.ts` in PR #320) is the authoritative source for what each legacy inner shape looks like. Every emit site below maps onto a parser branch already present.
- Rejected: full unmigrate (Option A in conversation). Would lose stepper UI, Stage 3 quest metadata, LLM resolved-echoes, UI label echoes (v-wldr regression class), unify-outer-shape, and would block six in-flight tickets (v-bbds, v-rzkp, v-uwaj, v-mjfl, v-rfqt, etc.). Never the right call.
- Rejected: completing the migration forward (audit + retire each `build_*` tool, then delete legacy). Tractable but multi-quarter; net code stays ~the same; structural debt during the long tail.

## Files

### mcp-ts (primary scope)
- `src/tools/execute/_base.ts` — remove `TxEncoding` union, `EvmTxArgs` narrow type, `ExecuteTxArgs` discriminated union; rewrite `buildEvmTxArgs` to emit legacy `{swap_tx: {...}}` or `{send_tx: {...}}` shape (whichever the legacy parser already handles via `getServerTx`).
- `src/tools/execute/execute_send.ts` — every `tx_encoding: '...'` emission site (lines ~340 cosmos, ~666 utxo, ~716 solana, ~749 tron, ~776 ton, ~791 sui, ~804 cardano). Each must emit a shape the legacy parser branches recognise.
- `src/tools/execute/execute_swap.ts` — every emission site (lines ~393/411 utxo-psbt, ~439 cosmos-msg, ~467-487 native-non-EVM fallback, ~759 EVM general via `buildSwapEvmTx`, ~779 solana general).
- `src/tools/execute/_thorchain.ts` — `buildThorchainLpTxArgs` should emit the legacy `kind: 'thorchain_lp_add' | 'thorchain_lp_remove'` discriminated shapes (the parser already knows them; lines ~678 / ~701 in legacy.ts).
- `src/tools/execute/execute_contract_call.ts` — uses `buildEvmTxArgs` indirectly; verify rewrite covers it.
- `src/tools/execute/execute_lp_add.ts` / `execute_lp_remove.ts` — emit via reverted `buildThorchainLpTxArgs`.
- `src/tools/execute/__tests__/*` — fixture shapes update to match the reverted contract.

### vultiagent-app (coordinated revert)
- Either close PR #320 before merge, or open a revert PR. Net result: `src/features/agent/lib/txProposal.ts` returns to its pre-refactor state; `src/features/agent/lib/txProposal/` folder removed; `src/features/agent/lib/__tests__/txProposal.{dispatch,canonical,evm-canonical,solana-canonical,utxo-canonical,cosmos-canonical,misc-canonical}.test.ts` removed; existing test files (`txProposal.test.ts`, `txProposal.lpContract.test.ts`, `txProposal.tokensMvp.test.ts`, `txProposal.xrp.test.ts`) untouched and become the regression net again.
- `signAndBroadcast.ts` — undo the `thorchain-deposit` dispatch case (line ~408-410); legacy `thorchain_lp_add` / `thorchain_lp_remove` cases remain and are sufficient.

### Reference patterns
- `vultiagent-app/src/features/agent/lib/txProposal.ts` (pre-refactor) — every parser branch shows exactly what shape to emit. Use this as the contract source of truth.
- `getServerTx` (line 88) — defines the legacy EVM lookup chain (`swap_tx ?? send_tx ?? tx`). Pick one consistently.
- `getServerSolanaTx` (line 96) — defines legacy Solana native shape (top-level `chain: Solana, to, amount`).

## Acceptance Criteria

- [ ] mcp-ts execute_swap general EVM emits a shape `getServerTx` recognises (e.g. `{swap_tx: {to, value, data, gasLimit, ...}}`)
- [ ] mcp-ts execute_swap general Solana emits `{swap_tx: {data: <base64>}}` (legacy.ts:344 Solana-prebuilt branch)
- [ ] mcp-ts execute_swap THORChain UTXO source emits the legacy UTXO shape `getServerTx` falls into (top-level `from/to/amount/fee_rate/memo`)
- [ ] mcp-ts execute_swap THORChain cosmos source emits the legacy cosmos shape (the existing parser cosmos branch handles it without changes)
- [ ] mcp-ts execute_swap native non-EVM fallback (Tron/TON/Sui/Cardano/Ripple) emits shapes the parser's generic / chain-specific branches already accept
- [ ] mcp-ts execute_send for every chain family emits a shape the legacy parser handles (audit per chain by reading the corresponding legacy parser branch)
- [ ] mcp-ts execute_lp_add / execute_lp_remove emit `kind: 'thorchain_lp_add'` / `kind: 'thorchain_lp_remove'` (already in the parser's union)
- [ ] mcp-ts execute_contract_call emits via the same EVM shape used by execute_send / execute_swap
- [ ] `TxEncoding`, `EvmTxArgs`, `ExecuteTxArgs`, `tx_encoding` field removed from `_base.ts` and from every emit site
- [ ] Outer wrapper preserved: every execute_* tool still emits `ExecutePrepResult` with `stepperConfig`, `resolved`, `approvalTxArgs?`, `quest_metadata?`
- [ ] mcp-ts unit tests pass; fixtures updated to legacy shapes
- [ ] vultiagent-app: PR #320 reverted (or closed); `txProposal.ts` and 4 existing test files restored; no `txProposal/` folder; `signAndBroadcast.ts` reverted on the thorchain-deposit case
- [ ] vultiagent-app full test suite passes against the reverted state
- [ ] On-device smoke: SOL→PENGU swap (original v-uneu repro), SOL native send, ETH send, BTC send, THOR LP add — all reach Sign and broadcast
- [ ] No agent-backend changes needed (verify by running curl-replay tier 1 fixtures unchanged against the reverted mcp-ts)

## Gotchas

- Cardano / Sui / TON / Tron may not have a 1:1 legacy native-send branch in the parser — they likely fall through the `generic` chain-kind branch (legacy.ts ~730: `if (to && amount && info.kind !== 'utxo' && info.kind !== 'evm')`). Verify the generic branch produces the right `ParsedServerTx.kind` for each before assuming it works; add a parser branch only if necessary (and prefer keeping the parser pre-refactor verbatim — if a new branch is needed, the case for option B weakens).
- TON's `seqno` is fetched in mcp-ts and required on the wire. Pre-refactor parser may not extract `seqno` from a top-level shape; check if there's a `build_ton_send` style legacy shape it already understands.
- Cardano's `decimals` is required for display; check if the legacy `generic` branch surfaces it.
- v-uneu repro must work end-to-end after the mcp-ts change without ANY app change (zero `txProposal.ts` edits). If it doesn't, the spec is wrong about which legacy shape to emit.
- `quest_metadata` lives on `ExecutePrepResult` (sibling to `txArgs`), not inside it. Outer wrapper changes nothing here.
- agent-backend's curl-replay fixtures may match against `tx_encoding` strings. Grep `scripts/qa/curl-replay/` for `tx_encoding` and update fixtures if any reference it.
- Six in-flight tickets (v-bbds, v-rzkp, v-uwaj, v-mjfl, v-rfqt, v-ujuc) are built on the execute_* contract — outer wrapper untouched, but verify none of them introspect `tx_encoding` directly.
- The existing PR #320 has 1813 passing tests including new canonical tests. The revert removes those test files; lose the regression coverage they added against canonical shapes (mooted because canonical shapes won't exist, but if the migration ever resurfaces those tests are gone).
- Coordinate landing order: mcp-ts revert must merge BEFORE the vultiagent-app revert lands in production, or one mcp-ts deploy will break unmigrated app builds. Reverse the sequence and the app revert breaks the in-flight canonical mcp-ts deploy.

## Notes

**2026-05-05T00:19:03Z**

killed: forward path chosen — app#320 merged; reverting tx_encoding inner shapes no longer an option
