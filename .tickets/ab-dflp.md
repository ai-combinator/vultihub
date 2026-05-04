---
id: ab-dflp
status: closed
deps: []
links: []
created: 2026-04-27T04:11:53Z
type: task
priority: 2
assignee: Jibles
---
# Improve mcp-ts#49: drop SDK duplication, hard-cut legacy build_*, fix self-review findings

## Objective

Improve mcp-ts PR #49 (`surgical/v-qjvi-execute-foundation`) along three converging axes: (1) delete the SDK duplication that was inlined to dodge a now-stable v-johx revert, (2) hard-cut the legacy `build_*_send` / `build_swap_tx` / `build_thorchain_lp_*` tools that `execute_*` already covers — pre-launch policy, no users, no soak window required — and (3) fix the 9 outstanding findings from the AI-assisted self-review including 1 BLOCKER and 5 MAJOR. Net effect: smaller diff, closed contract, no deferred follow-ups (v-ujuc folds in here), several real signing-adjacent bugs closed.

## Why hard-cut legacy now (not later)

**Vultisig has no real users yet.** The legacy `build_*` tools and the new `execute_*` tools were both registered in PR #49 with the framing "additive — retire legacy in v-ujuc once telemetry confirms zero calls." That framing comes from the standard launched-product playbook: dual-registration, on-read migration, telemetry-gated retirement. None of that machinery is doing useful work pre-launch:

- **No live conversations to maintain.** Old `build_*` calls in past chats wouldn't be replayed by anyone. There's no soak window to honor and no risk of breaking a user mid-flow.
- **Dual-registration costs prompt budget.** Every legacy tool that stays registered consumes LLM tool-description tokens and tempts the model into the wrong tool. `agent-backend/prompt.go` carries ~200 LoC of mitigation against `build_*` fumbles (decimals, nonce, base-units, exact field names). That's deletable here, but only after the legacy tools themselves are gone.
- **Pre-launch policy is hard-cut.** Skipping dual-registration / on-read migration / soak windows is the team's stated default until first user. Saved memory note `feedback_prelaunch_persisted_state.md` codifies it.

So hard-cut is the cheaper, cleaner choice. The 5 v-srjz-blocked token-layer build tools (`build_spl_transfer_tx`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build_xrp_send`) plus `build_cw20_transfer` stay registered — they cover real coverage gaps (`execute_send` doesn't yet handle SPL / TRC-20 / jettons / Sui tokens / XRP / Cosmos CW-20 contract calls). Re-scope v-ujuc to delete those 5 token-layer tools once v-srjz lands.

## Context & Findings

### Why this exists

PR #49 was scoped as "additive only — leave legacy registered, retire later in v-ujuc once telemetry confirms zero calls." That framing was inherited from the parked v-pxuw plan and was written for a launched product. Vultisig has no real users yet; the team's stated pre-launch policy is hard-cut, no dual-registration / on-read migration / soak windows. So the legacy retention is doing no work and adding maintenance fat.

A second framing change: at PR-open time, inlining of SDK-0.17-only helpers (`resolveToken`, plus `chain-family` / `cosmos-chains` re-implementations) was hedging against the v-johx SDK revert. v-johx is NOT reverting — SDK 0.17 surface is stable. The duplication is now pure debt.

### Fat to cut (load-bearing items NOT to trim)

Load-bearing — keep: `_base.ts`, `_amountResolver.ts`, `_thorchain.ts`, `parseAmount.ts`, `zodHelpers.ts`, the test bulk, the typed `ExecutePrepResult` envelope, the 5 `execute_*` tools landing together (the cohort is what stress-tests the envelope across chain families).

Fat — cut:
- `src/lib/resolveToken.ts` (271 LoC) + `resolveToken.test.ts` (318 LoC) — SDK exports `resolveToken`, was inlined only to dodge v-johx
- `src/lib/chain-family.ts` (27 LoC) — duplicates SDK's `EvmChain` / `UtxoChain` enums and `getCosmosChainKind()`
- `src/lib/cosmos-chains.ts` (132 LoC) — substantial overlap with SDK's `packages/core/chain/chains/cosmos/` (`chainInfo.ts`, `getDenom.ts`, `cosmosGasLimitRecord.ts`); legacy-only fields (`toolName`, `bech32Prefix`, `hasDenomParam`, `feeBase`, `feeHuman`) disappear when legacy tools are deleted
- `Category = string & {}` widening in `src/lib/toolCategories.ts` — only widened to keep loose legacy literals (`'gaia'`, `'thor'`, `'osmosis'`) typechecking
- `asBuilderConfig()` runtime narrowing in `build-cosmos-send.ts` — pure interop glue between legacy + new
- Legacy-tool-name pointer text in `execute_send.ts` rejection messages (e.g. "use `build_spl_transfer_tx`")
- Type-only touches in `build-utxo-send.ts`, `utxo-fees.ts` (Category widening only)
- "These supersede the per-chain build_*..." comment block in `index.ts`

### Atomic-vs-unified decision

Legacy `build_*_send` / `build_swap_tx` / `build_thorchain_lp_*` are write-primitives the LLM has historically fumbled (decimals, base-unit conversion, nonce, exact field names — `agent-backend/prompt.go` has ~200 lines of mitigation for this). Read/encoding primitives that ARE composable (`abi_encode`, `abi_decode`, `evm_call`, `evm_tx_info`, `evm_check_allowance`, `resolve_selector`, `resolve_ens`, `search_token`, `convert_amount`, balance / fee tools) all stay registered. The LLM keeps every primitive it actually composes with.

Genuine flexibility losses we accept: custom gas overrides, custom nonce overrides, batched tx shapes — none reliably accessible via the current LLM today, no users to demand them. Re-expose as advanced fields if/when needed.

One real escape-hatch concern: `execute_contract_call` must accept raw `data: 0x...` calldata (not just `function_signature + args`) to preserve the `abi_encode + raw-tx` arbitrary-contract-call escape hatch. Verify before deleting `build-evm-tx.ts`; add the field if missing.

### Tools that MUST stay (real coverage gaps, not legacy fat)

- `build_spl_transfer_tx`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build_xrp_send` — blocked on v-srjz (app→SDK migration); SDK signing path doesn't yet handle token-layer payload fields. `execute_send` rejects these chains with explicit text errors (rejection logic stays; only legacy-tool-name pointers strip out).
- Audit before cutting: `build-cw20-transfer.ts` (CW20 cosmos contract calls — likely NOT covered by `execute_send`'s native cosmos path) and `build-other-send.ts` (chain coverage unclear; inspect first).

### Sequence dependency (critical)

Self-review finding #7 (Zcash ZIP-317): `build-utxo-send.ts:104` is the canonical impl for Zcash's client-side fee computation. `execute_send` already mirrors it (`execute_send.ts:464-466`); `execute_swap` does not — currently treats `UTXO_FEE_FALLBACK['Zcash'] = 100` as sat/vB and feeds it into the `utxo-psbt` envelope. **Port the ZIP-317 `fee_note` pattern into `execute_swap` BEFORE deleting `build-utxo-send.ts`**, otherwise hard-cut silently regresses Zcash swap fees.

### Self-review findings (10 total; #1 already fixed in commit 2e19e8b)

1. ~~`assertSafeDestination` regression on cosmos + missing on execute_send~~ — **fixed in commit 2e19e8b**.
2. **[BLOCKER]** `tolerance_bps` cosmetic — `execute_swap.ts:475`: schema accepts the field, UI echo computes `minOut` from it, but it's never threaded into `findSwapQuote` and never lands in the THORChain deposit memo. User requesting 0.1% slippage sees that on the card while L1 contract executes against provider default. **Decision:** drop from schema (simple, restores parity with `build_swap_tx`'s implicit default) — only thread into THOR memo if a real use-case demands it.
3. **[MAJOR]** `max` / `25%` advertised but throws — `execute_lp_add.ts:74` and `execute_swap.ts:394`: schema descriptions advertise both forms; `resolveExecuteAmount` is called without `balanceLookup`, throws cryptic dev-facing error. **Decision:** strip the advertised forms from descriptions until a per-family lookup builder lands (cheap correct fix); full wiring is a follow-up.
4. **[MAJOR]** USDT-pattern approval revert — `execute_swap.ts:117`: USDT/KNCv1/OMG/BNT et al. revert when `approve(N)` is called with non-zero current allowance. **Decision:** always emit two-step `approve(0)` → `approve(N)` when current allowance > 0 (don't bother with token-allowlist heuristics — always-reset is safe across all ERC-20s).
5. **[MAJOR]** Cosmos vesting accounts — `execute_send.ts:159`: REST parser reads `acct.account?.account_number` directly; vesting accounts (gaia/osmosis/kujira) nest those under `account.base_vesting_account.base_account`. Today falls back to `'0'` and broadcasts wrong-sequence tx. Walk vesting nesting first, fall back to direct.
6. **[MAJOR]** `evmTxInfo` error masking — `execute_send.ts:416`: generic "failed to fetch gas/nonce" hides "insufficient native balance for gas" (common on "send max ETH" turns). Probe `error.message` for viem `InsufficientFundsError` / `EstimateGasExecutionError` shape, surface specific text.
7. **[MINOR]** Zcash fee-fallback unit error — `execute_swap.ts:152`: see "Sequence dependency" above.
8. **[MINOR]** Silent allowance check failure — `execute_swap.ts:114`: `catch {}` swallows `evmCheckAllowance` errors; failing-closed is correct but transient RPC silently inflates user cost. Add `console.error` log.
9. **[MINOR]** Empty `approvalTarget` — `execute_swap.ts:461`: when both `native.router` and `native.inbound_address` are empty, control reaches `abiEncode('approve(...)', ['', amount])` which throws inside viem; outer catch then emits an approval with `spender = ''` that would revert on broadcast. Reject the quote at line 460 if both empty.
10. **[QUESTION]** Tron ref-block fields — `execute_send.ts:518`: emits `tx_encoding: 'tron-tx'` without `ref_block_bytes` / `ref_block_hash` / `expiration`. TON correctly fetches `seqno` server-side. Verify `vultiagent-app` `parseServerTx` Tron branch fetches them client-side; either document the contract with a code comment or add server-side fetch.

### Downstream (tracked under v-rfqt / v-llqd, NOT in scope here)

The biggest prize from this consolidation isn't the tool-count reduction — it's the prompt simplification it unblocks. `agent-backend/prompt.go` has ~200 lines of mitigation for LLM fumbles on the build_* tools (`evm_tx_info → build_evm_tx` recipe, ~50-line `build_swap_tx` field-name table, the auto-generated `Chains:` line that reads from `build_*_send` registrations, decimals/nonce/base-unit cautions). Most becomes deletable. Also: `agent-backend/executor.go:98-166` (`normalizeMCPArgs` + `decimalToBaseUnits` LLM-hallucination workarounds) becomes dead code. `vultiagent-app/AssistantMessageView.tsx` `coalesceParts` + `findLastBuildToolIndex` (multi-tool-call workarounds) also deletable. Call out in this PR's body so the implementing agent doesn't lose them, but don't touch in this ticket.

### Decided NOT to do (don't re-investigate)

- Do not split into 2 PRs. Reviewer ergonomics, not value reduction. The 5-tool cohort stress-tests the envelope contract across chain families.
- Do not trim `_amountResolver` / `parseAmount` / `zodHelpers` / `_thorchain` / test bulk — load-bearing.
- Do not pre-emptively add `gas_override` / `nonce_override` advanced fields to `execute_*`. No users → no demand → premature surface area.
- Do not implement v-ujuc as a separate later ticket — fold the deletion of replaceable legacy tools (the ~30, not the 5 v-srjz-blocked) into this work. Re-scope v-ujuc to "delete the 5 token-layer build tools once v-srjz lands."

## Files

**Delete (legacy hard-cut):**
- `mcp-ts/src/tools/send/build-cosmos-send.ts`
- `mcp-ts/src/tools/send/build-utxo-send.ts` *(after porting Zcash ZIP-317 logic to execute_swap)*
- `mcp-ts/src/tools/swap/build-swap-tx.ts`
- `mcp-ts/src/tools/evm/build-evm-tx.ts` *(after verifying execute_contract_call accepts raw calldata)*
- The thorchain LP build/quote tools in `mcp-ts/src/tools/defi/thorchain-lp.ts` (`buildThorchainLpAdd`, `buildThorchainLpRemove`, `quoteThorchainLpAdd`) — keep query/inbound/halts/lockup/pools tools
- `mcp-ts/tests/cosmos-send.test.ts` and any other legacy-tool test files
- `mcp-ts/src/lib/resolveToken.ts` + `mcp-ts/src/lib/resolveToken.test.ts`
- `mcp-ts/src/lib/chain-family.ts`

**Audit, then decide:**
- `mcp-ts/src/tools/send/build-cw20-transfer.ts` — does `execute_send` cover Cosmos CW20 contract calls? If not, keep.
- `mcp-ts/src/tools/send/build-other-send.ts` — what chain/token coverage does it provide? Keep if `execute_send` doesn't cover.

**Keep (v-srjz-blocked):**
- `mcp-ts/src/tools/send/` SPL/TRC-20/jetton/Sui-token/XRP build tools (verify exact paths)

**Modify:**
- `mcp-ts/src/lib/cosmos-chains.ts` — strip legacy-only fields (`toolName`, `bech32Prefix`, `hasDenomParam`, `feeBase`, `feeHuman`); replace remaining content with SDK imports from `@vultisig/sdk` / `@vultisig/core-chain/chains/cosmos/` where possible. May reduce to a thin re-export or be deletable entirely.
- `mcp-ts/src/lib/toolCategories.ts` — remove `| (string & {})` from `Category` type; closed literal union only.
- `mcp-ts/src/tools/index.ts` — drop legacy imports + registrations; delete "supersede" comment block.
- `mcp-ts/src/tools/types.ts` — keep the `Category[]` type tightening + `textError` / `jsonError` additions; revisit only if anything else flips.
- `mcp-ts/src/tools/execute/execute_send.ts` — review-finding fixes #5, #6, #10; strip legacy-tool-name pointer text from rejection messages (keep chain-level rejection).
- `mcp-ts/src/tools/execute/execute_swap.ts` — review-finding fixes #2, #3, #4, #7, #8, #9; port Zcash ZIP-317 logic from `build-utxo-send.ts:104` BEFORE that file is deleted.
- `mcp-ts/src/tools/execute/execute_lp_add.ts` — review-finding fix #3.
- `mcp-ts/src/tools/execute/execute_contract_call.ts` — verify accepts raw `data: 0x...` calldata; add the field if missing.
- `mcp-ts/src/tools/balance/cosmos-balance.ts` — leave the Rujira fallback fix as-is (independent drive-by); only revisit if Category type tightening breaks anything.

**Reference (out of scope, tracked under v-rfqt / v-llqd):**
- `agent-backend/internal/service/agent/prompt.go`
- `agent-backend/internal/service/agent/executor.go:98-166`
- `agent-backend/internal/service/agent/tool_filter.go`
- `vultiagent-app/src/features/agent/components/AssistantMessageView.tsx`
- `vultiagent-app/src/features/agent/lib/toolUIRegistry.ts`

## Acceptance Criteria

- [ ] All SDK-duplicated helpers deleted (`resolveToken.ts` + test, `chain-family.ts`); imports point to `@vultisig/sdk` exports.
- [ ] `cosmos-chains.ts` reduced to non-duplicated content only (or deleted entirely if all replaceable from SDK); legacy-only fields removed.
- [ ] `Category` is a closed literal union — `(string & {})` widening removed; all tools across the registry typecheck against it without expanding the union.
- [ ] Legacy tool files deleted: `build-cosmos-send.ts`, `build-utxo-send.ts`, `build-swap-tx.ts`, `build-evm-tx.ts`, the three thorchain LP build/quote tools, plus their test files.
- [ ] v-srjz-blocked 5 build tools remain registered (SPL/TRC-20/jetton/Sui-token/XRP); `execute_send` chain-level rejection messages no longer name them by tool name.
- [ ] `build-cw20-transfer.ts` and `build-other-send.ts` audited; PR description documents whether each was kept (with reason) or deleted.
- [ ] Zcash ZIP-317 `fee_note` pattern ported from `build-utxo-send.ts` into `execute_swap.ts` before legacy file deletion; `execute_swap` Zcash branch matches `execute_send.ts:464-466` semantics.
- [ ] `execute_contract_call` accepts raw `data: 0x...` calldata (verified or added).
- [ ] BLOCKER finding #2 resolved: `tolerance_bps` either removed from `execute_swap` schema OR correctly threaded into the THORChain deposit memo and `findSwapQuote` parameters.
- [ ] All MAJOR findings resolved: #3 (lp/swap max-percent description), #4 (USDT-pattern always-reset approve flow), #5 (vesting-account-aware cosmos REST parser), #6 (viem `InsufficientFundsError` probe in `evmTxInfo` catch).
- [ ] All MINOR findings resolved: #7 (Zcash unit fix is part of the port above), #8 (allowance-check error log), #9 (reject swap quote when both `router` and `inbound_address` empty).
- [ ] QUESTION #10 closed: Tron `parseServerTx` ref-block contract verified — either documented with a code comment near `execute_send.ts:518` or server-side fetch added.
- [ ] `pnpm typecheck`, `pnpm test`, `pnpm lint`, `pnpm build` all clean; total test count adjusted for deleted legacy tests, but `execute_*` coverage of cosmos-send / utxo-send / swap paths is at least equivalent to what the deleted tests exercised.
- [ ] PR body updated to reflect "removes legacy" framing; `v-ujuc` reference removed (re-scoped — see Decided NOT to do).
- [ ] PR title gets `[sdk/signing]` tag if any change ends up touching SDK-side code or signing/derivation; PR body lists affected chain IDs for any signing-adjacent fix (#2, #4, #5).

## Gotchas

- **Sequence:** Zcash ZIP-317 logic must port to `execute_swap` BEFORE `build-utxo-send.ts` is deleted, or hard-cut silently regresses Zcash swap fees.
- `convertAmount` (existing read-only utility) must stay registered — it's not a `build_*` tool, it's a helper still useful for "what's $50 in BTC" educational queries. Don't sweep during registry edits.
- `cosmos-balance.ts` Rujira fallback fix is drive-by behavior improvement, NOT legacy-tied — leave bundled, just note in PR body so reviewers don't expect it to relate to the consolidation thesis.
- The `isError` / `textError` / `jsonError` additions in `types.ts` are MCP-spec correctness, not legacy work. Don't touch.
- Do NOT extend the `Category` union when tightening — if any tool fails to typecheck against the closed 14-literal set, that's a signal the tool's category needs fixing, not that the union should grow.
- The PR is signing-adjacent (per repo CLAUDE.md safety policy). Findings #2 (tolerance_bps), #4 (USDT approval), #5 (vesting accounts) all touch tx construction. Add tests; list affected chain IDs in PR body; tag PR title appropriately.
- v-srjz-blocked tools' rejection text loses the legacy-tool-name pointers but KEEPS the per-chain rejection — `execute_send` still text-errors on SPL/TRC-20/jetton/Sui-token/Cardano-asset/XRP token sends, just with a chain-level message instead of a "use `build_*_transfer`" pointer.
- The 10 review findings cite line numbers from PR head commit `c5457dc5`. After hard-cut + bug-fix work the line numbers will shift; locate by symbol/function name, not line number.
- Don't delete `evm_tx_info` — it's a read-only primitive the LLM still needs even after `build_evm_tx` is gone.
- Re-scope (don't delete) v-ujuc to cover only the v-srjz-blocked 5 tools once v-srjz lands. The "delete ~30 replaceable legacy tools" half folds into this ticket.

## Notes

**2026-04-27T04:21:21Z**

Precondition: implementing agent must `git checkout surgical/v-qjvi-execute-foundation` in `/home/sean/Repos/vultisig/mcp-ts` before starting. Repo is currently on `fix/cosmos-balance-raw-shape-consistency` (unrelated). All file paths in this spec are relative to `mcp-ts/`; SDK paths under `vultisig-sdk/`.

**2026-04-27T06:04:08Z**

Task 1 complete (commits 587738d, 70cc33c, 9e1686c, 2057543): execute_contract_call accepts raw 0x data with hex format validation; execute_swap emits Zcash ZIP-317 fee_note; execute_send Tron ref-block contract documented. 465/465 tests pass; typecheck/lint/build clean.

**2026-04-27T06:22:23Z**

Tasks 2/3/4 complete (commits d1163bc bd4e7ec 8e93f64 f7a1aea 92c1c49 b607fbe 021adee 8278ce6 2ab3a83). All 9 review findings closed. 481/481 tests pass.

CRITICAL DISCOVERY during Task 3 fix: vultiagent-app currently consumes the OLD build_swap_tx wire shape (approval_tx + swap_tx + needs_approval), NOT the new execute_* envelope (txArgs + approvalTxArgs + stepperConfig). The new execute_* shape has zero references in vultiagent-app/src/. Hard-cut of build_swap_tx in Task 6 will break swap UX until v-rfqt/v-llqd lands the app→execute_* integration. Spec is aware of v-srjz (app→SDK migration) but doesn't explicitly address this integration gap. User decision needed before Task 6.

**2026-04-27T06:59:00Z**

Task 5 complete (audit-only; no code change). User direction: skip SDK changes. resolveToken stays (LLM-input adapter, not duplication). chain-family stays (needs SDK value-export to swap). cosmos-chains legacy-field strip folded into Task 6. Finding A (vultiagent-app integration) resolved per user — separate open PR landing shortly. Proceeding with Task 6 hard-cut.

**2026-04-27T07:17:34Z**

Tasks 6-8 complete. Final state: 18 commits on surgical/v-qjvi-execute-foundation. typecheck/test/lint/build all clean. 421 tests pass (60 legacy tests removed; execute_* coverage intact at 171 tests across 6 files).

Task 6 (hard-cut) shipped: deleted build-cosmos-send.ts, build-utxo-send.ts, build-swap-tx.ts, build-evm-tx.ts, 3 thorchain LP build/quote tools, 5 native-send legacy exports from build-other-send.ts. Stripped legacy-tool-name pointers from execute_send rejection text. Stripped 6 legacy-only fields from cosmos-chains.ts. Kept: 5 v-srjz-blocked token-layer tools + build-cw20-transfer.

Task 7 DEFERRED — Category type tightening's premise was wrong. The (string & {}) widening isn't dead legacy; it's load-bearing for agent-backend tool_filter.go keyword routing (~90 of 95 typecheck errors are chain/domain tags actively consumed by tool_filter.go for LLM tool selection). Fix requires coordinated agent-backend change → out of scope here. Tracked for v-rfqt/v-llqd.

Tasks 1-6 covered all 9 review findings + raw-data calldata escape hatch + Zcash ZIP-317 port + Tron ref-block doc + USDT-pattern reset flow with proper wire-shape widening in _base.ts.

**2026-04-27T07:28:21Z**

E2E testing complete (commits/state on surgical/v-qjvi-execute-foundation @ HEAD).

Spun up local mcp-ts (amcp/pnpm dev:http on :9091) + agent-backend (aback/go run ./cmd/server on :8084), 150 tools wired through. Ran a 14-test matrix:
- A1-A2: tools/list — all 11 deleted legacy tools absent; all 11 kept tools (5 execute_* + 6 escape hatches) registered ✓
- A3-A5: schemas — execute_swap has no tolerance_bps, no max/25%; execute_lp_add no max/25%; execute_contract_call has data field ✓
- B1: raw data validation (notHex / 0x123 / data+function_sig ambiguous all rejected; 0xa9059cbb accepted) ✓
- C2: agent-backend EVM swap → execute_swap picked, 4968-byte prep payload ✓
- C3: agent-backend Cosmos send → execute_send picked, 663-byte envelope ✓
- C4'/C5': direct MCP — Bitcoin emits utxo-psbt+fee_rate; Zcash emits utxo-psbt+fee_note (no fee_rate) ✓
- C6/C6': agent-backend Solana SPL → model picks build_spl_transfer_tx (kept escape hatch); execute_send rejection text doesn't name legacy tool ✓
- TRC-20 rejection: text doesn't name build_trc20_transfer ✓
- C7: hello world sanity ✓

PR #49 body updated with full test summary, WHY-hard-cut rationale, wire-shape coordination notes for vultiagent-app, and affected chain IDs. Ticket body has prominent 'Why hard-cut legacy now (not later)' section.

Ready for review.

**2026-04-27T08:10:49Z**

Final state — PR #49 ready for review.

Rebased onto origin/main (resolved delete-vs-modify conflicts from PR #45 by accepting our deletions of build_swap_tx + build_evm_tx, kept new src/tools/quest-metadata.ts module + tests).

Ported quest_metadata emission to execute_swap (both native THOR/Maya and general 1inch/LiFi/Kyber branches) + execute_contract_call (function_sig and raw data paths). ExecutePrepResult widened with quest_metadata?: QuestMetadata. Top-level placement (sibling of txArgs) verified end-to-end via direct MCP calls — matches consumer's txData['quest_metadata'] read.

E2E re-verified post-rebase: native THOR ETH→BTC emits {token_in_symbol:'ETH', token_out_symbol:'BTC', source_chain_id:1, router_address:'0xd37bbe…lowercase'}; general LiFi ETH→USDC emits both chain_ids + lowercased router; execute_contract_call on Ethereum + Base emits target_contract lowercased + source_chain_id.

Final commit count: 61 ahead of base. Force-pushed cleanly. PR mergeable: MERGEABLE.

Test counts: 447 tests across 26 files (typecheck/lint/build clean).

Follow-up tracked: ticket v-mjfl — agent-backend extend knownCalldataToolPrefixes to cover execute_*. mcp-ts PR #50 (amount_usd fallback) commented to coordinate rebase.

Confidence to ship: ~90% — code clean, downstream contract preserved, integration coordination explicit and tracked.
