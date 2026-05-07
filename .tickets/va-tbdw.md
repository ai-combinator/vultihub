---
id: va-tbdw
status: closed
deps: []
links: []
created: 2026-04-23T02:54:11Z
type: task
priority: 2
assignee: Jibles
---
# Swaps broadcast successfully then revert on-chain (out-of-gas); UI shows APPROVED

## Problem Statement

Users report that swaps appear to succeed in the UI — card latches green `APPROVED` with a clickable tx hash — but the on-chain tx reverts, most recently observed as out-of-gas. "Solved" means: (a) we don't underestimate gas for the provider/route combinations that are failing, and (b) on-chain revert surfaces in the card instead of a false `APPROVED` status.

## Research Findings

### Current State

Two independent issues converge into a single confusing UX:

**1. Gas limit sourcing (root cause candidates)**

- `mcp-ts/src/tools/swap/build-swap-tx.ts:128,141` — **hardcoded `gas_limit: 160000`** for THORChain / MayaChain native EVM router calls (`depositWithExpiry`). No simulation, no buffer, no chain-specific tuning.
- `mcp-ts/src/tools/swap/build-swap-tx.ts:159` — hardcoded `gas_limit: 60000` for ERC-20 approvals that precede native swaps.
- `mcp-ts/src/tools/swap/build-swap-tx.ts:374` — general branch (Kyber / 1inch / LiFi / Uniswap) uses `tx.evm.gasLimit` **directly from the SDK's `GeneralSwapQuote`**, which is whatever the provider API returned at quote time. No client-side buffer.
- `vultisig-sdk/packages/core/chain/swap/general/GeneralSwapQuote.ts:5-15` — the SDK's `GeneralSwapTx.evm` type exposes only `{ from, to, data, value, gasLimit, affiliateFee }`. No `gasPrice` / `maxFeePerGas` / `estimatedGas` — the SDK isn't forwarding fee data from underlying providers.

In both branches the gas limit is chosen at quote/build time and **never re-simulated at broadcast time**. State drift between quote and inclusion (mempool delay, router-state changes) can push actual gas use above the limit.

**2. Client-side receipt-wait gap**

`vultiagent-app/src/features/agent/lib/signAndBroadcast.ts:197-223`:

```ts
case 'evm': {
  if (parsed.approvalTx?.to) {
    const approvalResult = await signBroadcastServerEvmTx(...)
    await waitForEvmReceipt(parsed.chain, approvalResult.txHash, { signal })  // ← only here
  }
  const result = await signBroadcastServerEvmTx(...)  // main swap tx
  return result.txHash                                 // returns on broadcast-accept, no receipt wait
}
```

`waitForEvmReceipt` is called for the ERC-20 approval leg (when present) but **not for the main swap tx**. The main tx returns as soon as the RPC accepts the broadcast. `useToolExecution` therefore transitions to `{step: 'success', result}` on broadcast-accept, and `BuildTxCard` renders `ActionCard.Status kind="approved"` with the tx hash — before the chain has actually settled.

If the tx reverts (OOG, slippage overflow, router revert), the UI never re-checks and users see a false `APPROVED`. This is a **pre-existing gap**, not introduced by the ActionCard PR, but the new prominent in-card `APPROVED` row exposes it more visibly than the older `TxResultView` pill.

### Available Tools & Patterns

- `vultiagent-app/src/features/agent/lib/txSigning.ts:27-63` — `waitForEvmReceipt(chain, txHash, { timeoutMs?, signal? })` already polls `eth_getTransactionReceipt`, honors abort signals, and throws `Transaction reverted on-chain` when `receipt.status === '0x0'` or times out after 90s. Usable for the main tx with one extra call.
- `mcp-ts/src/tools/evm/evm-tx-info.ts:97-100` — has access to `estimatedGas` via `evmTxInfo()` (SDK call). Already used for EVM sends' gas/nonce pre-flight. Swap build could consume the same estimate + apply a buffer.
- `mcp-ts/src/tools/evm/build-evm-tx.ts` — existing pattern of accepting `gas_limit` as input (LLM-supplied from a prior `evm_tx_info` call). A similar pipeline could route swap gas through an estimate step.
- No existing gas-buffer multiplier anywhere in the stack. Everything is verbatim pass-through from provider / SDK.

### External Context

- Kyber API: returns `gasUsd` + estimated gas in quote responses; their own docs recommend a ≥1.2× buffer on volatile chains (Arbitrum, Base).
- 1inch API: returns `tx.gas` in /v6.0/{chain}/swap response. Also known to under-estimate by ~5-15%.
- THORChain router `depositWithExpiry`: actual gas varies 110k-200k depending on token leg + affiliate memo length. 160k is tight for token legs, safe for native.

### Validated Behaviors

- Confirmed: `git diff origin/feat/bump-sdk-0.16.1..HEAD` — **our ActionCard PR does not touch any gas field**. Only gas-related code added is inside `BuildTxCard.tsx:extractFee()` which computes a display value (`gas_limit × max_fee_per_gas`) for the "Est. fee" row. `pendingTx.data` passed into `signAndBroadcast` is byte-for-byte unchanged. Rules out the PR as cause of OOG.
- Confirmed: `useToolExecution.ts:153-156` wraps the raw MCP tool output into `pendingTx` without mutation, so gas fields pass through unchanged.
- Confirmed: `signBroadcastServerEvmTx` in `@/services/evmTx` submits tx fields as received. No client-side gas adjustment.
- Confirmed: the approval-tx receipt IS awaited (`signAndBroadcast.ts:214`), so the "approved USDC before swap" leg is observed to completion — only the main swap leg lacks receipt polling.

## Constraints

- Gas sourcing fix lives in `mcp-ts` (and possibly `vultisig-sdk` for the general branch) — requires cross-repo changes if we want buffered estimates propagated through the general flow.
- Receipt-wait fix is client-local (`signAndBroadcast.ts`), self-contained, and low-risk.
- `waitForEvmReceipt` already has a 90s timeout. Adding it to the main tx means users wait up to 90s after broadcast before the card settles to its final state — the in-card "SIGNING TRANSACTION" spinner can cover this window, but UX should decide whether to show a "waiting for confirmation" intermediate state.
- Must not regress the current behavior where signing errors (wrong password, chain error) still render red `REJECTED` immediately.

## Dead Ends

- **Blaming the ActionCard PR**: ruled out by diff inspection + signing-flow trace. Gas params are untouched.
- **Client-side gas bump**: considered adding a multiplier in `signAndBroadcast`, rejected because the gas limit is embedded in the `pendingTx.data` passed to `signBroadcastServerEvmTx`; modifying mid-sign would diverge from what the approval card displayed and would also break non-EVM paths. The fix belongs where gas is sourced (MCP / SDK).

## Open Questions

- Which provider/route(s) are actually failing? Need an Arbiscan tx hash to confirm whether this is THORChain native (160k hardcode suspect), Kyber/1inch general (SDK estimate suspect), or a non-OOG revert masquerading as OOG (slippage / router revert consuming all gas).
- `gasUsed` vs `gas_limit` ratio on the failing tx will disambiguate: if `gasUsed == gas_limit` it's true OOG; if `gasUsed < gas_limit` it's a revert that still consumed gas.
- Buffer strategy: apply a provider-specific multiplier in `mcp-ts` (e.g. 1.2× for Kyber on Arbitrum, 1.3× for THORChain token legs), or do a full `eth_estimateGas` simulation at build time? Simulation is authoritative but adds an RPC hop.
- Receipt-wait UX: should the card show a dedicated `BROADCASTING` / `CONFIRMING` intermediate state between sign-complete and chain-confirm, or keep the current spinner up until receipt arrives? Affects 517 jest tests / 3-6 UI components.
- Should we also instrument the `signAndBroadcast` path with a per-stage log (broadcast-start, broadcast-accept, receipt-polled, receipt-status) so future OOG reports have observability without needing a fresh repro?

## Notes

**2026-05-04T23:09:10Z**

auto-closed: merged via vultisig/vultiagent-app#296 + vultisig/mcp-ts#62
