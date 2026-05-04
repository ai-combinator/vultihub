---
id: v-vtmi
status: closed
deps: []
links: []
created: 2026-04-16T23:57:20Z
type: bug
priority: 1
assignee: Jibles
---
# Self-send EVM fails with 'Failed to derive Ethereum address' on confirm

## Symptoms

On the vultiagent-app agent chat, requesting a self-send of ETH on Arbitrum ("Can we self send 0.0001 ETH on arb?") builds the tx and renders the confirmation card correctly. On password-confirm, signing fails with a red error "Failed to derive Ethereum address" and the ExecutionStepper "Sign transaction" step is marked failed with "Tap to retry".

The send flow is the core primitive — EVM address derivation should not fail. This is blocking any EVM self-send through the agent.

## Reproduction

1. Open vultiagent-app with a configured Fast/Secure vault that has Arbitrum enabled.
2. In the agent chat: "Can we self send 0.0001 ETH on arb?"
3. Agent calls `execute_send` (mcp-ts) → card renders → user taps Confirm → enters password.
4. Observed: red "Failed to derive Ethereum address" toast; stepper "Sign transaction" fails with "Tap to retry".

## Error Source

The literal string comes from `vultiagent-app/src/services/addressDerivation.ts:112`:

```ts
export function getVaultEthAddress(vault: VaultData): string {
  const addr = deriveAddress('Ethereum' as Chain, vault)
  if (!addr) throw new Error('Failed to derive Ethereum address')
  return addr
}
```

`deriveAddress` (same file, line 75-104) swallows all errors and returns `null`, logging via `console.warn`. So we lose the underlying cause — the warn line is the first thing to check in device logs.

## Suspected Area

Two plausible root causes:

1. **WalletCore not initialized by the time the EVM sign path runs.** `getWC()` at `addressDerivation.ts:48-53` throws "WalletCore not initialized. Call initWalletCore() first." if `walletCoreInstance` is null. `initWalletCore()` is called at app startup; a cold-start race or a reload that bypasses startup init could leave it unset. `deriveAddress` catches the throw and returns null, which then bubbles up as "Failed to derive Ethereum address" with no underlying cause visible.

2. **`getPublicKey` rejecting the vault's public-key shape.** `deriveAddress` calls `getPublicKey({ chain, walletCore, hexChainCode, publicKeys: { ecdsa, eddsa }, chainPublicKeys })`. If the active vault has a malformed `rootKeys.hexChainCode` / `publicKeyEcdsa` (e.g. empty, not hex, wrong length) the SDK throws and we swallow it.

Note the recent vault-free prep refactor (ticket v-pyit, closed) added `prepareSendTxFromKeys` etc. in the SDK that take a `VaultIdentity`. mcp-ts consumes those — but the failing path here is app-local (`getVaultEthAddress`) called from `src/services/evmTx.ts:62 / 171 / 321` inside `signBroadcastServerEvmTx`. The MCP never sees vault material, so the mcp-ts execute_send path itself is not the source of the derivation failure. That said, confirm mcp-ts is emitting the expected `txArgs` envelope (`tx_encoding: "evm"`, `chain: "Arbitrum"`, `to/value/data/gas` all populated) — if the envelope is malformed the app's parseServerTx may route down an unexpected branch.

## Files / Lines to Check

- `vultiagent-app/src/services/addressDerivation.ts:75-112` — swap the silent `console.warn` for a real propagated error (or at minimum stringify and re-throw from `getVaultEthAddress`) so the underlying failure is visible.
- `vultiagent-app/src/services/evmTx.ts:62, 171, 321` — all three call `getVaultEthAddress(vault)` synchronously; confirm the caller path reaches them only after `initWalletCore()` has resolved.
- `vultiagent-app/src/features/agent/lib/signAndBroadcast.ts:125-149` — `dispatchParsedTx` → EVM branch → `signBroadcastServerEvmTx`. Wrap in try/catch that surfaces the underlying cause rather than the generic "Failed to derive Ethereum address".
- `vultisig-sdk/packages/sdk/src/vault/services/AddressService.ts:56-62` — SDK-side `VaultError(AddressDerivationFailed)` with `error as Error` cause. App isn't going through this path (app uses `getPublicKey` + `sdkDeriveAddress` directly), but useful for parity when we rewrite the app to preserve cause.
- `mcp-ts/src/tools/execute/execute_send.ts:269-370` — EVM branch. Verify the envelope mcp-ts emits for a self-send on Arbitrum matches what vultiagent-app's `parseServerTx` expects (chain casing, `tx_encoding`, gas field keys).
- `vultiagent-app/src/features/agent/lib/txProposal.ts` — `parseServerTx` EVM branch; confirm it picks up the Arbitrum envelope cleanly.

## Triage Steps (not scope — for the fix ticket)

1. Reproduce locally with device logs attached; capture the `[deriveAddress] Failed for Ethereum:` warn line — that holds the real error message.
2. Either (a) ensure `initWalletCore()` has resolved before any signing entrypoint, or (b) make `deriveAddress` propagate the error with `cause` so the toast surfaces the actual problem.

## Out of Scope

- UI rendering issues around the action card / stepper are tracked separately (unified-card UI work; see v-pxuw).
- The LLM-repeats-action-card-info bug is a separate ticket (cosmetic/UX).
- Any changes to mcp-ts `execute_send` envelope beyond verifying correctness for this reproduction.

## Priority Rationale

P1 (high): blocks the core send flow for EVM chains via the agent, which is the headline user-facing capability.

## Notes

**2026-04-23T23:44:10Z**

v-hipx draft PR opened with probe + error-propagation fix: https://github.com/vultisig/vultiagent-app/pull/240 — getVaultEthAddress now propagates underlying cause; probe logs CoinType derivationPath + chainId at init. Next repro should surface the real failure in device logs.
