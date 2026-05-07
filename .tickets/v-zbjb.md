---
id: v-zbjb
status: open
deps: []
links: []
created: 2026-05-05T05:03:31Z
type: task
priority: 2
assignee: Jibles
---
# Skip envelope: collapse parsePrep duplicate validation, drop hybrid-envelope kludge

## Problem

vultiagent-app has two transaction-envelope validators in series with different contracts on what makes an envelope routable. `parsePrep` (executePrep.ts) validates "canonical-per-tx_encoding" shape via a 115-line hand-rolled gate. `parseServerTx` (txProposal/index.ts) dispatches by tx_encoding, falling back to `build.parseServerTx` for tx_type-discriminated envelopes (skip_swap, ibc_transfer, evm-sequence). The mismatch forces mcp-ts to emit **hybrid envelopes** for Skip routes — `tx_encoding: 'evm'` + canonical `tx: { to, value, data, chain_id }` + Skip metadata (`tx_type: 'skip_swap'`, `unsigned_msgs[]`, etc.) spread alongside — to satisfy parsePrep while smuggling Skip-routability info for parseServerTx. The kludge bridge is the early-return at `txProposal/index.ts:60-62` undoing the costume.

"Solved" means: parsePrep's `validateTxArgsShape` gate is deleted, parsePrep delegates to parseServerTx for shape validation (returns null on parse failure), the hybrid envelope is unrepresentable on the wire, and the early-return is unnecessary.

## Background

### Findings

**Two parsers in series:**
- `parsePrep` at `vultiagent-app/src/features/agent/lib/executePrep.ts:69` — hand-rolled type-guard chain.
  - `validateTxArgsShape` at `:234-348` (~115 LOC) — switch over `tx_encoding`. Strict per-encoding checks for `'evm'` (`:237-287`, regex on `to`, chain_id cross-check), `'utxo-psbt'`, `'cosmos-msg'`. Returns `true` unconditionally for `'solana-tx'`/`'ripple-tx'`/`'tron-tx'`/`'ton-tx'`/`'sui-tx'`/`'cardano-tx'` (escape-hatch pattern, defers to parseServerTx). Default returns `false` (fails closed at `:341`).
  - `executePrep.ts:117` cross-validates `approvalTxArgs.tx_encoding === txArgs.tx_encoding`. Approval legs are EVM today; if main tx flips encoding, this equality silently breaks every CCTP/USDC Skip route.
  - `executePrep.ts:163,175` (`skipEnvelopeNeedsApprovalButHasNone`) is the **actual** fail-closed guard for missing approvals — NOT `validateSkipEvmDispatch:204`. The early-return's comment at `txProposal/index.ts:59` is misleading on which validator is load-bearing.
- `parseServerTx` at `vultiagent-app/src/features/agent/lib/txProposal/index.ts:42-100` — dispatches by `tx_encoding` (`:64-97`), with the Skip kludge early-return at `:60-62`, falling back to `build.parseServerTx` for envelopes without `tx_encoding` (`:99`).
- `build.parseServerTx` at `vultiagent-app/src/features/agent/lib/txProposal/build.ts` (~700 LOC) — handles tx_type-discriminated envelopes: `skip_swap` (`:190`), `ibc_transfer` (promoted internally `:381-409`), `wasm_execute` / `wasm_execute_multi` / `cosmwasm` (`:892-1129`), `staking_*` (5 variants `:1166-1172`), `astroport_swap`, `evm-sequence` (Morpho prepare_*), `thorchain-deposit`, `cw20`.
- **The dispatcher is fine.** `signAndBroadcast.ts:524` already routes off `parsed.kind` correctly across ~28 envelope kinds (`txProposal/types.ts:112-308`). The architectural smell is the duplicate-validation gate upstream, not the dispatcher downstream.

**The hybrid envelope:**
- Emitted at `mcp-ts/src/tools/execute/execute_swap.ts:826-849` (and `:889-913` for the wrapping). Two branches today (EVM first leg vs Cosmos first leg) both set the costume.
- TxEncoding union: `mcp-ts/src/tools/execute/_base.ts:26-35`.
- Narrowing checks that read tx_encoding as a routing key: `mcp-ts/src/tools/execute/execute_swap.ts:1503,1608,1618,1621`.
- EvmTxArgs narrow type at `_base.ts:38-81` — ~12 mcp-ts call sites depend on `tx_encoding: 'evm'` as a literal type narrower.

**Receipts/replay are safe.** `PendingTx['data']` is memory-only at sign-time. No SQLite/AsyncStorage column pins `tx_encoding`; receipts persist `txHash` and resolved labels, not raw txArgs. `persistWithRetry.ts` doesn't touch this shape.

**LLM contract drift risk.** `agent-backend/internal/service/agent/prompt.go:76-85` enumerates `tx_type` values to the LLM for `build_custom_tx`. The on-wire `tx_type` field is itself a public LLM contract (`'deposit'`, `'evm_contract'`, `'wasm_execute'` are the three values today).

**Test fixture surface:** ~175 `tx_encoding` references across `__tests__/`. Codemod handles ~80%. The exception: `txProposal.skip.test.ts:406` asserts the kludge directly ("routes to skip_swap when both tx_type and tx_encoding are set") — must be redesigned, not migrated.

**Security guard to preserve:** `assertChainMatchesEncoding` at `txProposal/index.ts:33` defends against "EVM tx claiming Solana chain". Must stay on the leaf-format axis (per-leg `tx_encoding` describing wire format), not the routing axis.

**Blockaid coupling:** scan-request builder keys off `tx_encoding === 'evm'` upstream. Skip-EVM routes today flow through Blockaid's EVM scan branch because of the costume. Tests don't cover Skip-vs-Blockaid interaction.

**mcp-ts deploy lag:** mcp-ts deploys are not atomic with vultiagent-app deploys. Deployed mcp-ts servers will keep emitting the hybrid envelope for ~3 weeks after the app ships the new path. The early-return shim must stay during that window.

### External Context

This work is the architectural follow-up to the v-rjfc cycle (mcp-ts#78, agent-backend#272, vultiagent-app#387) which shipped the hybrid envelope + early-return as the pragmatic patch. v-rjfc landed Tier 1+2 chain coverage for Skip Go's LUNA/LUNC routes; this ticket cleans up the architectural smell that v-rjfc papered over.

## Current Thinking

**The architectural sin is the duplicate-validation gate, not the encoding/type split.** `parsePrep`'s `validateTxArgsShape` re-implements what `parseServerTx` already does properly. The hybrid envelope exists to satisfy that duplicate gate. The cleanest fix is to delete `validateTxArgsShape` entirely and have `parsePrep` delegate to `parseServerTx` for shape validation (return null on parse failure). Doing that eliminates the gate, eliminates the costume requirement on the mcp-ts side, and eliminates the early-return — all at once.

**`tx_type` becomes the de facto routing key when present, `tx_encoding` becomes a leaf-format hint.** `signAndBroadcast.ts:524` already dispatches on `parsed.kind` across 28 envelope types — the dispatcher already lives in the right place. The fix is upstream: parsePrep + parseServerTx switch their dispatch axis from `tx_encoding` to `tx_type ?? tx_encoding`. `tx_encoding` stays on per-leg signable txs as a wire-format hint (`'evm'`, `'cosmos-msg'`, etc.) — that role is correct and stays.

**Migration is incremental, not big-bang.** Three slices:

1. **Slice 1 (~3h, deploys safely):** Make `parsePrep` accept envelopes where `tx_type` is the discriminator and `tx_encoding` is absent. Backwards-compatible — existing payloads with `tx_encoding` still pass. mcp-ts `execute_swap` stops emitting the `tx_encoding: 'evm'` costume; Skip envelope becomes pure `tx_type: 'skip_swap'`. Early-return at `txProposal/index.ts:60-62` becomes unreachable for fresh payloads but stays as deploy-lag shim.

2. **Slice 2 (~6h):** Migrate Blockaid scan-request from `tx_encoding === 'evm'` gate to per-leg `signing_method`. Refactor `validateTxArgsShape` to dispatch on `tx_type ?? tx_encoding`. Fix `executePrep.ts:117` cross-validation to compare effective discriminators, not raw `tx_encoding`. Update `EvmTxArgs` narrow type at `_base.ts:38-81` to decouple from literal `tx_encoding`.

3. **Slice 3 (~3h):** After mcp-ts old emit is fully gone from prod (~3 weeks post-slice-1), delete the precedence shim and the hybrid-envelope branch. Update tests including the `txProposal.skip.test.ts:406` redesign.

Total: ~16h ≈ 2 working days, ~6.5× the option-2 estimate of 2.5h. Slice 1 alone is the same cost as option 2 (2.5-3h) but actually unwinds the underlying duplication instead of dressing the costume up nicer.

**Trigger:** ride slice 1 on the next Skip-shaped tool's PR (KyberSwap cross-chain, LiFi, deBridge). At that point two cases stops being a snowflake and starts being a category that deserves a model. Slices 2 and 3 follow on the same momentum.

## Constraints

- **`assertChainMatchesEncoding` at `txProposal/index.ts:33` must be preserved.** Security guard against "EVM tx claiming Solana chain". Stays on the leaf-format axis (per-leg wire format), not on the routing axis.
- **`agent-backend/internal/service/agent/prompt.go:76-85` is an LLM-facing contract.** The on-wire `tx_type` field name is fixed; can't be renamed to `kind` without lockstep prompt updates that risk a 100% prod-failure path on `build_custom_tx`. Discriminator stays as `tx_type`.
- **The early-return at `txProposal/index.ts:60-62` cannot be deleted in slice 1.** Must stay as deploy-lag shim for ~3 weeks. Slice 3 is the deletion.
- **`build.parseServerTx`'s deep parsing logic (~700 LOC) is canonical.** Don't duplicate or rewrite. Slice 1's `parseSkipCanonical` (if anything is named that) is a 5-line wrapper that delegates.
- **`parseEvmCanonical` and the other canonical parsers at `txProposal/index.ts:67-96` enforce stricter shape checks than `build.parseServerTx`** (regex on `to`, mixed-chain detection, sequence-cap, inner-tx chainId cross-check at `evm.ts:46-60`). These checks must survive in some form — losing them is a real EVM regression. Slice 2's parsePrep refactor must preserve them on the leaf-format axis.

## Assumptions

- **Most `tx_encoding` consumers are routing-keyed** (parsePrep `validateTxArgsShape`, parseServerTx dispatch, `executePrep.ts:117` cross-validation, Blockaid scan) and need to switch. A smaller set are leaf-format consumers (`EvmTxArgs` narrow type, `signAndBroadcast.ts:1503,1608,1618-1624` narrowing, `contractCall.ts:5`) and can stay. If actually-routing-keyed turns out to be larger, slice 2 expands.
- **Stored receipts don't pin `tx_encoding`.** Verified by grep — `signingCards.ts`, `useReceiptPoll.ts`, the receipts tests have no `tx_encoding` references. `PendingTx['data']` is memory-only.
- **Skip-shaped tools will appear** (KyberSwap cross-chain, LiFi, deBridge). If the team deprioritizes those for ≥6 months, the trigger condition changes — slice 1 may need to ride on something else or be done standalone.
- **Slice 1 is genuinely ~3h.** Estimate from sub-agent investigation grounded in file:line citations. Real impl may surface narrowing-type cascades that bump it.
- **mcp-ts deploys aren't atomic with app deploys.** Standing assumption from prior conversation — verify the actual deploy cadence before committing the ~3 week shim window in slice 3.

## Non-goals

- **Big-bang refactor in a single PR.** Incremental path is viable and explicitly preferred.
- **Renaming `tx_type` to `kind` (or any other discriminator name change).** Locked out by `agent-backend/prompt.go:76-85` LLM contract. `tx_type` stays.
- **Refactoring the dispatcher at `signAndBroadcast.ts:524`.** Already routes correctly off `parsed.kind`; the architectural smell isn't there.
- **Adding `tx_type` to vanilla `execute_send` / `execute_swap` envelopes that don't have one today.** They don't need it. `tx_encoding` remains their natural single-leg discriminator after the refactor; only multi-leg / route-spanning envelopes (Skip, IBC-promoted, evm-sequence) need `tx_type`.
- **Receipts / replay infrastructure changes.** Already safe — receipts persist `txHash` + labels, not raw txArgs. No migration needed.
- **Touching `build.parseServerTx`'s 700-line deep parser.** Canonical implementation; reuse via thin wrapper if needed.
- **agent-backend behavioral changes.** `enrichBuildResult` at `agent.go:4554` only logs `tx_encoding` in a comment; behavior is shape-agnostic.

## Dead Ends

- **Option 2 (add `tx_encoding: 'skip-swap'` as a first-class encoding).** Half-measure that just renames the costume. Doesn't actually delete the kludge — mcp-ts deploy lag means the early-return must stay anyway, just narrowed. The `executePrep.ts:117` cross-validation gotcha (`approvalTxArgs.tx_encoding !== txArgs.tx_encoding` under the new encoding) silently breaks every CCTP/USDC route. The deeper duplicate-validation problem persists. ~2.5h cost for marginal architectural gain. Rejected.
- **Option 1 (drop the canonical fan-out at `txProposal/index.ts:64-97`, delegate everything to `build.parseServerTx`).** Wrong direction. `parseEvmCanonical` at `evm.ts:46-60` enforces strict checks `build.parseServerTx` doesn't (regex on `to`, mixed-chain detection, sequence-cap, inner-tx chainId cross-check). Collapsing loses those — real EVM regression. Rejected.
- **Option 3 (full unification of parsePrep + parseServerTx contracts into a single validator).** Too aggressive. The proposed approach (delete `validateTxArgsShape`, have parsePrep delegate to parseServerTx) achieves the same architectural win incrementally without a 2-week refactor. The single-validator end-state is a follow-up that may or may not be worth it depending on what slice 2 reveals.
- **Hybrid keep-and-document forever.** Viable today (the kludge is honestly commented and load-bearing security guards live elsewhere via `executePrep.ts:163,175`), but doesn't survive a second Skip-shaped tool. The right call is patch-now-simplify-later — this ticket is the "later".
- **Renaming `tx_type` to `kind`.** Locked out by `agent-backend/prompt.go:76-85` LLM-facing contract.
- **Doing slice 1 standalone, not riding on the next Skip-shaped tool's PR.** Conversation landed on "ride on next tool" — single-case justification is weak. Planner can revisit if there's appetite for a standalone slice.

## Open Questions

1. **Trigger condition.** Conversation landed on "ride slice 1 on the next Skip-shaped tool's PR (KyberSwap / LiFi / deBridge)." If those tools are ≥3 months out, is standalone-slice-1 worth doing now to clean up v-rjfc's debt, or wait? Plan-mode call.
2. **Slice 1 implementation detail.** When the new "envelope-type-keyed" parsePrep path lands, is it a thin `parseSkipCanonical` wrapper around `build.parseServerTx`, or does parsePrep grow a new dispatch arm directly? Affects test coverage shape.
3. **`txProposal.skip.test.ts:406` redesign.** Currently asserts the kludge: "routes to skip_swap when both tx_type and tx_encoding are set". The redesigned assertion is something like "routes to skip_swap on tx_type alone" + a separate "rejects malformed hybrid". Plan-mode picks the test shape.
4. **Deploy-lag window.** Sub-agent estimated ~3 weeks for the early-return shim to live before slice 3 deletes it. Verify actual mcp-ts deploy cadence before committing — could be shorter or longer.
5. **`EvmTxArgs` narrow type cascade in mcp-ts.** Making `tx_encoding` optional at `_base.ts:38-81` changes ~12 call sites' inferred TS types. Sub-agent flagged this as a "real TS migration, not a rename." Slice 2 effort estimate (~6h) assumes this is contained — if the cascade is larger, slice 2 grows.
6. **Slice scope.** Sub-agent proposed three slices; alternative is two slices (combining slice 2 + slice 3 once the deploy window passes). Plan-mode decides on slice granularity based on what fits a single PR cleanly.
