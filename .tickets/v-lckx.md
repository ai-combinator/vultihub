---
id: v-lckx
status: closed
deps: []
links: []
created: 2026-04-29T03:24:10Z
type: bug
priority: 1
assignee: Jibles
---
# Fix approve+swap continuation gap and misleading green stepper on stall

## Objective

For ERC-20-input swaps, the approve leg signs and broadcasts successfully but the swap leg never fires — the flow stalls after the approve receipt and the card misleadingly renders a fully-green "APPROVED" stepper. Fix the continuation: after the approve leg confirms on-chain, the swap leg should be prepped/signed/broadcast under the same single-claim approval, and the card stepper should reflect the actual progress (only show APPROVED when both legs are done).

## Context & Findings

- Surfaced during v-gopv on-device smoke (PR #242, commit `eaefb06`, 2026-04-29). Test: LINK → ETH on Arbitrum via Kyber. Sean's wallet had no prior Kyber USDC allowance for LINK, so the prep included `approvalTxArgs`.
- Observed: tick tapped → password entered → one `/vault/sign` round trip → one `arb/` POST → approve broadcast at `0x24fb9997…8abc7e`. Then the bar disappeared. No swap leg `/vault/sign`, no swap broadcast. The card UI rendered all stepper dots green ("APPROVED") even though only the approve leg had run.
- Likely root cause: `useTransactionFlow.ts:253-310` (the approve-then-sign block in `runSigning`) gates the swap leg on `!completed.includes('sign')` and reads `prep.txArgs` directly. If the prep was generated under conditions where the swap leg's `eth_estimateGas` reverted with `TransferHelper: TRANSFER_FROM_FAILED` (expected pre-allowance), the backend may have either omitted `txArgs` or emitted a malformed envelope that the code then skipped silently. Alternatively, the runSigning closure may have completed both legs but the swap leg's `signAndBroadcast` call returned an error that wasn't surfaced.
- Either way: a swap that requires approve must end either with both legs broadcast, or with a failed state showing the user *which* leg failed and why.
- The misleading "APPROVED" label is downstream — once the continuation gap is fixed, the stepper will reflect reality. But the labelling itself is also worth a separate look: a stepper showing all dots green when only the approve broadcast happened is wrong UX even in error states.
- Verified pre-existing: the approve+swap continuation logic landed before v-gopv. v-gopv only changed the trigger source (auto-fire useEffect → tick onPress), not the runSigning sequencing. Both PRs touched the same file but the bug is in the sequencing, which v-gopv preserves verbatim.

## Files

Repo: `vultiagent-app`. Likely branch: `fix/approve-continuation` (new).

- `src/features/agent/hooks/useTransactionFlow.ts` — `runSigning` approve→sign continuation logic at lines ~253-310; verify the swap leg actually runs after the receipt-wait, and surface its errors instead of swallowing them
- `src/features/agent/lib/signAndBroadcast.ts` — confirm the EVM dispatch path correctly handles single-tx swap after a separate approve broadcast (vs the `evm-sequence` shape that bundles both)
- `src/features/agent/components/tools/ExecuteCards/SwapFlowCard.tsx` and `shared.tsx` — fix the stepper status logic so all dots are NOT painted green when the flow stalled mid-sequence; should show approve=green, sign=current/error, broadcast=pending

## Acceptance Criteria

- [ ] Reproduce: LINK → ETH on Arbitrum (no Kyber allowance for LINK), full swap flow runs both legs and broadcasts the swap leg
- [ ] When the swap leg fails (any reason: gas estimation revert, RPC error, MPC timeout), card shows the actual failure state with the failing step highlighted, not a fully-green "APPROVED" stepper
- [ ] On-chain verification: 1 approve + 1 swap tx for the wallet during the test (not just 1 approve)
- [ ] If the backend emits a malformed `txArgs` for the swap leg, surface the error to the user with a recoverable retry option (don't silently terminate)
- [ ] Reducer test pinning the failure path: approve completed, sign attempted, sign failed → card phase = `failed` with `lastStep: 'sign'`, NOT phase = `done`
- [ ] Lint + tsc + test suite green

## Gotchas

- v-gopv is unchanged in this fix. Don't reintroduce auto-fire on the swap leg as a "fix" for the continuation gap — the swap leg shares the same single-claim approval as the approve leg per the existing `runSigning` design.
- The `evm-sequence` shape (Morpho-style multi-tx) is a different code path from the approve+swap shape. Verify which path LINK→ETH actually takes — if Kyber emits `evm-sequence`, the bug is in the loop at `signAndBroadcast.ts:233-245`; if it emits separate `approvalTxArgs + txArgs`, the bug is in `useTransactionFlow.ts`.
- Receipt-polling `waitForEvmReceipt` may be hitting its 90s timeout silently if the approve receipt is slow to propagate. Check whether the abort path is firing.
- Stepper green-on-stall is a separate UX defect from the continuation gap. They land together because both surfaced in the same test run, but the fixes are independent.
