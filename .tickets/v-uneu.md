---
id: v-uneu
status: in_progress
deps: []
links: []
created: 2026-04-30T02:07:51Z
type: bug
priority: 2
assignee: Jibles
---
# SOL→PENGU swap (li.fi) fails at Sign with 'no structured tx data'

## Objective
SOL→PENGU swaps via the li.fi route fail at the Sign step with "Cannot sign: no structured tx data and no fallback text available." Quote succeeds but the proposal payload reaches signing without being recognized by `parseServerTx`. Blocks any Solana SPL-output swap routed through li.fi.

## Context & Findings
- Repro: in vultiagent-app, request a 0.008 SOL → PENGU swap. Card progresses Quote → Sign and fails immediately with the message above (route shown as `li.fi`, source chain Solana).
- Confirmed NOT caused by branch `va-tbdw-await-main-tx-receipt`. The only diff to `signAndBroadcast.ts` is a rename `waitForEvmReceipt` → `waitForReceipt`, and `txProposal.ts` is untouched on the branch (`git log main..HEAD -- src/features/agent/lib/txProposal.ts` returns nothing). Same failure should reproduce on `main`.
- Error origin: `vultiagent-app/src/features/agent/lib/signAndBroadcast.ts:56` in `shouldDispatchFallback` — fires when `parseServerTx()` returns `null` AND `fallbackChainText` is `null`. The caller hardcodes `fallbackChainText: null` (`useTransactionFlow.ts:255`, identical on `main`), so the fallback gate is effectively a no-op and any null parse surfaces as this exact opaque message.
- Why the parse returns null: the Solana prebuilt-tx branch in `parseServerTx` (`txProposal.ts:248`) only matches when `data.swap_tx.data` is a string. MCP's `build_swap_tx` for li.fi must be emitting the prebuilt bytes under a different field or envelope shape (e.g. nested `transaction`, `tx_data`, top-level `serialized_tx`, or shape divergence between EVM-side and Solana-side outputs).
- UNCONFIRMED: actual wire shape of the failing proposal. First step in any fix is to log `pendingTx.data` at the gate and capture the real payload — fix layer (mcp-ts vs vultiagent-app) depends on what's found.
- `signBroadcastPrebuiltSolanaTx` already handles both legacy and `VersionedTransaction` (`vultiagent-app/src/services/solanaTx.ts:89`), so once the parser hands off a base64 blob the downstream signing path is fine.

## Files
- `vultiagent-app/src/features/agent/lib/txProposal.ts:241-260` — `parseServerTx` Solana branch; extend to recognize whatever field li.fi/build_swap_tx actually emits, OR
- `mcp-ts/...build_swap_tx` — normalize Solana prebuilt output to `swap_tx.data: <base64>` matching the existing contract
- `vultiagent-app/src/features/agent/lib/signAndBroadcast.ts:56,129` — gate / suggested log point for diagnosis
- `vultiagent-app/src/services/solanaTx.ts:89` — reference for what the prebuilt signer expects (base64 of either `VersionedTransaction` or legacy `Transaction`)

## Acceptance Criteria
- [ ] Capture actual `pendingTx.data` shape for a failing SOL→PENGU li.fi swap (one log at the gate)
- [ ] Decide fix layer: parser-side (vultiagent-app) or normalizer-side (mcp-ts)
- [ ] SOL→PENGU swap reaches and completes the Sign step end-to-end
- [ ] At least one EVM→Solana and one Solana→native swap manually regression-tested (no parser regressions)
- [ ] Add a `txProposal.test.ts` fixture for the recovered shape to prevent regression
- [ ] Lint and type-check pass

## Gotchas
- Hardcoded `fallbackChainText: null` means any future parser miss surfaces as this same opaque message — consider including `Object.keys(pendingTx.data)` in the error to surface which fields came in.
- `shouldDispatchFallback` has Solana-specific bech32 carve-outs (`signAndBroadcast.ts:72-83`); unrelated, but easy to misread when scanning.
- Don't conflate with the SPL-transfer path (`txProposal.ts:379`) — that branch is `mint`-set, native-style send. Li.fi swap is the prebuilt envelope, a distinct branch.

## Notes

**2026-04-30T04:19:11Z**

Added diagnostic log at signAndBroadcast.ts:129 — captures topLevelKeys, swapTxKeys, swapTxDataType, and presence of common alt envelopes (transaction/tx_data/serialized_tx) when parseServerTx returns null. Next: repro SOL→PENGU swap on-device and capture the log to determine fix layer.
