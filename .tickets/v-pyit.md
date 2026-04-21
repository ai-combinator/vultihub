---
id: v-pyit
status: closed
deps: []
links: []
created: 2026-04-16T19:29:45Z
type: task
priority: 2
assignee: Jibles
---
# SDK: expose vault-free atomic prep functions

## Objective

Expose the existing vault-free building blocks (`buildSendKeysignPayload`, `buildSwapKeysignPayload`, cosmos prep, `getCoinBalance`, etc.) as a public SDK surface under `packages/sdk/src/tools/prep/`, and refactor `VaultBase` prep methods into thin wrappers. This lets MCP servers and other consumers without a vault instance build unsigned transactions directly, eliminating ~2,000 LOC of duplicated per-chain logic in `mcp-ts/` without changing the security boundary.

## Context & Findings

The SDK already has a three-layer architecture in the code; only the top layer is public:

- **Atoms** (exist in `@vultisig/core-*`, not re-exported from SDK index): `getPublicKey` (`packages/core/chain/publicKey/getPublicKey.ts`), `deriveAddress`, `getCoinBalance` (`packages/core/chain/coin/balance/index.ts`), `getChainSpecific`, `refineKeysignAmount`, `refineKeysignUtxo`.
- **Compositions** (exist, not public): `buildSendKeysignPayload` (`packages/core/mpc/keysign/send/build.ts:31`), `buildSwapKeysignPayload`, `getSendFeeEstimate`. Already take `{publicKeys, hexChainCode, vaultId, localPartyId, walletCore, tx params}` — no vault instance needed.
- **Vault layer** (public): `VaultBase` + `vault/services/*` (`TransactionBuilder`, `SwapService`, `BalanceService`). Holds `vaultData`, `keyShares`, storage, cache.

Today only `deriveAddressFromKeys` (`packages/sdk/src/tools/address/deriveAddressFromKeys.ts:35`) is a public vault-free composition. Every other prep method is an instance method on `VaultBase`.

**Key discovery:** The `vaultId` parameter threaded through `buildSendKeysignPayload` is literally `vaultData.publicKeys.ecdsa` (see `packages/sdk/src/vault/services/TransactionBuilder.ts:126`). mcp-ts already has this via `set_vault_info`. `localPartyId` is device metadata stamped into the signed payload but is opaque at prep time — clients can override before signing.

**Why now:** v-pxuw shipped `execute_*` tools in mcp-ts that re-implement chain dispatch (~3,800 LOC across `tools/execute/`, `tools/send/`, `tools/balance/`, `tools/evm/`) because the SDK forced it. Every future `execute_*` flow inherits the same problem.

**Decisions:**
- **Comprehensive surface, not minimal.** Covers send, swap, contractCall, cosmos, maxSend, balance, public-key. Core functions are already pure so the lift is re-exports and thin wrappers.
- **`VaultBase` public API unchanged.** Methods become one-line wrappers that build a `VaultIdentity` from `this.vaultData` and delegate. No breaking change for Windows/mobile/CLI/browser consumers.
- **SDK returns raw `KeysignPayload`; JSON-envelope serialization is a consumer concern (mcp-ts).**
- **Security boundary unchanged.** mcp-ts still cannot sign — no `keyShares` or signing helpers exposed.

**Rejected:**
- New class hierarchy / strategy pattern — pure functions are cheaper.
- Minimal migration (send/swap/contractCall only) — leaves balance/cosmos/LP/fee duplication; next `execute_*` re-inherits drift.
- Incremental opportunistic migration — same total effort, drift lives for months.

## Files

**New SDK surface** (`packages/sdk/src/tools/prep/`):
- `types.ts` — `VaultIdentity` type: `{ ecdsaPublicKey, eddsaPublicKey, hexChainCode, localPartyId?, publicKeyMldsa?, chainPublicKeys?, libType? }`
- `send.ts` — `prepareSendTxFromKeys`
- `swap.ts` — `prepareSwapTxFromKeys`
- `contractCall.ts` — `prepareContractCallTxFromKeys`
- `cosmos.ts` — `prepareSignAminoTxFromKeys`, `prepareSignDirectTxFromKeys`
- `maxSend.ts` — `getMaxSendAmountFromKeys`

**Re-exports** (`packages/sdk/src/tools/index.ts` + `packages/sdk/src/index.ts`):
- `getCoinBalance` from `@vultisig/core-chain/coin/balance`
- `getPublicKey` from `@vultisig/core-chain/publicKey/getPublicKey`

**Refactor to thin wrappers:**
- `packages/sdk/src/vault/services/TransactionBuilder.ts` — delegate to `prepare*FromKeys`
- `packages/sdk/src/vault/services/SwapService.ts` — same pattern
- `packages/sdk/src/vault/VaultBase.ts` — unchanged public API

**Reference patterns:**
- `packages/sdk/src/tools/address/deriveAddressFromKeys.ts` — pattern for WASM init + input validation
- `packages/core/mpc/keysign/send/build.ts:31` — the composition being wrapped
- `packages/sdk/src/vault/services/TransactionBuilder.ts:86-146` — logic to extract into pure form

## Acceptance Criteria

- [ ] `packages/sdk/src/tools/prep/` directory exists with: `prepareSendTxFromKeys`, `prepareSwapTxFromKeys`, `prepareContractCallTxFromKeys`, `prepareSignAminoTxFromKeys`, `prepareSignDirectTxFromKeys`, `getMaxSendAmountFromKeys`
- [ ] `VaultIdentity` type exported from SDK public index
- [ ] `getCoinBalance` and `getPublicKey` re-exported from `packages/sdk/src/tools/index.ts`
- [ ] `TransactionBuilder` and `SwapService` delegate to the new functional entry points — no logic duplication between the two paths
- [ ] `VaultBase` public API unchanged; existing Windows/mobile/CLI/browser consumers compile and pass without changes
- [ ] New unit tests cover: (a) ECDSA chain (Ethereum), (b) EdDSA chain (Solana), (c) MLDSA chain (QBTC with `publicKeyMldsa`), (d) UTXO with refinement (Bitcoin), (e) Cosmos with chain-specific fetch
- [ ] `yarn check` and `yarn test` pass
- [ ] Changeset added for `@vultisig/sdk`
- [ ] SDK README / `vultisig-sdk/CLAUDE.md` mentions the new vault-free prep surface

## Gotchas

- `localPartyId` is not cosmetic — it is written to `keysignPayload.vaultLocalPartyId` and downstream MPC participants may match strictly. Require callers to pass it; warn when missing.
- `buildSendKeysignPayload` is async because `getChainSpecific` makes RPC calls — prep is not free local compute.
- `wasmRuntime.getWalletCore()` is a memoized singleton — concurrent-safe, already in production via `deriveAddressFromKeys`.
- QBTC path uses `publicKeyMldsa` instead of ECDSA/EdDSA (`hexPublicKeyOverride` in `build.ts:24`); `VaultIdentity` must carry this; mirror `TransactionBuilder.ts:110-131`.
- Swap prep returns the SDK-native embedded shape (`keysignPayload.erc20ApprovePayload` inside main payload). Do not pre-split in SDK — leave that to the mcp-ts wrapper if it needs a two-tx envelope.
- mcp-ts migration is out of scope for this ticket — this ships SDK surface only; mcp-ts deletions tracked separately.

## Notes

**2026-04-16T20:42:35Z**

Implementation complete on branch refactor/v-pxuw-tool-consolidation. 12 commits from 9dedf309 to HEAD.

Verification:
- yarn check (typecheck + lint + knip + format): all green
- yarn workspace @vultisig/sdk test:unit: 1108/1108 passing across 65 files (+30 new prep tests)
- VaultBase public API unchanged (publicExports.test.ts pins prepareSendTx/prepareSwapTx/prepareContractCallTx on VaultBase.prototype)

What landed:
- packages/sdk/src/tools/prep/{types,send,contractCall,swap,cosmos,maxSend,index}.ts
- VaultIdentity type + vaultDataToIdentity helper
- Re-exports of getCoinBalance + getPublicKey from tools/index.ts and src/index.ts
- TransactionBuilder, SwapService, VaultBase.getMaxSendAmount refactored to thin wrappers (delegate to prep functions, preserve VaultError contract)
- prep functions take optional walletCore override (3rd positional) — wrappers pass through their injected WasmProvider; MCP/vault-free callers omit and let global getWalletCore() resolve
- .changeset/vault-free-prep-surface.md, README §12 "Vault-Free Transaction Prep", CLAUDE.md Key Entry Points bullet

Optional follow-ups (from final integration review, not blockers):
1. Extract resolveSendPublicKey helper to dedupe QBTC branch in send.ts + maxSend.ts (and a future estimateSendFeeFromKeys)
2. Add estimateSendFeeFromKeys + thin TransactionBuilder.estimateSendFee — closes the last duplication gap
3. (DONE) Comment why TransactionBuilder/SwapService bypass the prep barrel
4. Dedupe COSMOS_CHAINS list (cosmos.ts has it; TransactionBuilder.isCosmosChain has its own)
5. Move buildCosmosPayload.ts from vault/services/cosmos/ to tools/prep/cosmos/ — fixes tools→vault directional smell
6. Document walletCoreOverride in README §12 (one sentence)
7. Re-export ContractCallTxParams from tools/index.ts for export symmetry
8. Unify cosmos prep signature to (identity, params, walletCore?) for positional consistency
