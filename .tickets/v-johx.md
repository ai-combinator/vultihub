---
id: v-johx
status: closed
deps: []
links: []
created: 2026-04-22T19:38:31Z
type: task
priority: 2
assignee: Jibles
---
# Revert SDK v-pxuw tool-consolidation (PR #284 / 0.17.0), preserve standalone fixes

## Objective

Back out the vault-free tool-consolidation work that shipped in `@vultisig/sdk@0.17.0` (SDK PR #284, branch `refactor/v-pxuw-tool-consolidation`) because the downstream consumer PRs (mcp-ts #30, vultiagent-app #164) are not going to merge. Preserve the two behavioural improvements inside that PR that stand on their own merits. Cut a new SDK version (0.18.0) with the removal + preserved fixes as a single PR.

## Context & Findings

**Why revert.** 0.17.0 was shaped around mcp-ts's v-pxuw tool-consolidation branch. On the main/production consumer trees, the newly-exported surfaces (vault-free `prepare*FromKeys` helpers, `fiatToAmount`, `normalizeChain`, token-resolution re-exports) have zero call sites. Without the consumer PRs landing, this is dormant public API that cements on npm; cleaner to remove than to leave.

**Safe to revert — nothing actively consumes 0.17.0.** Audited main branches of mcp-ts, vultiagent-app, agent-backend. vultiagent-app's SDK bump lives only on branch #164 (not merged). No production consumer pulls 0.17.x. Revert is invisible to deployed systems.

**What's inside #284 and must go:**
- `packages/sdk/src/tools/prep/*` — new directory with six `prepare*FromKeys` helpers + shared `types.ts`/`maxSend.ts`
- `packages/sdk/src/utils/fiatToAmount.ts`, `packages/sdk/src/utils/normalizeChain.ts`
- Delegation refactor inside `packages/sdk/src/vault/VaultBase.ts`, `packages/sdk/src/vault/services/TransactionBuilder.ts`, `packages/sdk/src/vault/services/SwapService.ts` — restore the pre-0.17 inline implementations using `buildSendKeysignPayload` / `buildSwapKeysignPayload` / `buildSignAminoKeysignPayload` / `buildSignDirectKeysignPayload` directly
- Re-exports in `packages/sdk/src/index.ts` + `packages/sdk/src/tools/index.ts` of: `fiatToAmount`, `FiatToAmountError`, `normalizeChain`, `UnknownChainError`, `chainFeeCoin`, `knownTokens`, `knownTokensIndex`, `getTokenMetadata`, `getNativeSwapDecimals`, `getCoinBalance`, `getPublicKey`, `prepare*FromKeys`, `getMaxSendAmountFromKeys`, and types `Coin`/`CoinKey`/`CoinMetadata`/`KnownCoin`/`KnownCoinMetadata`/`TokenMetadataResolver`/`VaultIdentity`/`PrepareSendTxFromKeysParams`/`PrepareSwapTxFromKeysParams`/`GetMaxSendAmountFromKeysParams`
- Associated tests under `packages/sdk/tests/unit/tools/prep/`, `packages/sdk/tests/unit/utils/fiatToAmount.test.ts`, `packages/sdk/tests/unit/utils/normalizeChain.test.ts`, `packages/sdk/tests/unit/utils/publicExports.test.ts`
- `chainPublicKeys` threading into `TransactionBuilder.estimateSendFee` — design-risk (stale-key possibility), remove with the rest

**What's inside #284 but must be preserved (cherry-pick / reimplement):**

| Change | Commit | Strategy | Rationale |
|---|---|---|---|
| `getPublicKey` throws on empty WalletCore derivation path | `02738c15` (clean, single-file) | Cherry-pick | Strictly better error signal; no behaviour that worked before breaks. |
| `getErc20Prices` lowercases contract-address keys | bundled inside `7035e4dd` (not clean) | **Reimplement manually** (~6 lines + the matching test in `getErc20Prices.test.ts`) | Ethereum addresses are case-insensitive; upstream CoinGecko casing is unstable. Consumers already normalize with `.toLowerCase()`. |

**What's NOT part of #284 and stays untouched:** `getCosmosAccountInfo` → 0/0-on-unknown-address fix. That landed via separate PR #303 (commit `b86792ad`). Independent of this revert.

**Borderline items to discard along with the revert:**
- `VaultBase.getMaxSendAmount` up-front receiver validation (cosmetic — saves one balance fetch on bad input)
- Cosmos hardcoded-chain-list → `isChainOfKind('cosmos')` switch (equivalent if set parity holds, unverified)
- `chainPublicKeys` threading into prep helpers (speculative optimisation; staleness risk not assessed)

All three are low-value on their own; not worth the surgical effort to preserve.

**Easiest mechanical path.** `git revert 0fa10b96` (the #284 merge commit) on a fresh branch, then cherry-pick `02738c15` on top, then hand-apply the `getErc20Prices` lowercase normalization + its test. Add a changeset, open PR, tag as 0.18.0.

## Files

**Revert (from #284 merge `0fa10b96`):**
- `packages/sdk/src/tools/prep/` (entire directory — delete)
- `packages/sdk/src/utils/fiatToAmount.ts` (delete)
- `packages/sdk/src/utils/normalizeChain.ts` (delete)
- `packages/sdk/src/vault/VaultBase.ts` (restore pre-0.17 `getMaxSendAmount`)
- `packages/sdk/src/vault/services/TransactionBuilder.ts` (restore inline `buildSendKeysignPayload` / `buildSignAminoKeysignPayload` / `buildSignDirectKeysignPayload` / contract-call calldata encoding)
- `packages/sdk/src/vault/services/SwapService.ts` (restore inline `buildSwapKeysignPayload`)
- `packages/sdk/src/index.ts` (strip the new exports block)
- `packages/sdk/src/tools/index.ts` (strip prep + token re-exports + atomic `getCoinBalance`/`getPublicKey` re-exports)
- `packages/sdk/tests/unit/tools/prep/` (delete)
- `packages/sdk/tests/unit/utils/fiatToAmount.test.ts` (delete)
- `packages/sdk/tests/unit/utils/normalizeChain.test.ts` (delete)
- `packages/sdk/tests/unit/utils/publicExports.test.ts` (delete)
- `packages/sdk/README.md` (drop the vault-free sections; the pre-0.17 README is the reference)

**Preserve / reimplement:**
- `packages/core/chain/publicKey/getPublicKey.ts` — cherry-pick `02738c15`
- `packages/core/chain/coin/price/evm/getErc20Prices.tsx` — hand-apply the 6-line lowercase normalization: wrap `queryCoingeickoPrices(...)` return in `Object.fromEntries(Object.entries(prices).map(([k, v]) => [k.toLowerCase(), v]))`
- `packages/core/chain/coin/price/evm/getErc20Prices.test.ts` — reimplement the casing test that was added in #284 (32 lines)

**Changeset + versioning:**
- New `.changeset/revert-tool-consolidation.md` — major-bump wording (removes exports); will land as 0.18.0 via the usual version-packages flow

## Acceptance Criteria

- [ ] All `prepare*FromKeys` helpers, `fiatToAmount`, `FiatToAmountError`, `normalizeChain`, `UnknownChainError` removed from `@vultisig/sdk` public surface
- [ ] Token-resolution re-exports (`chainFeeCoin`, `knownTokens`, `knownTokensIndex`, `getTokenMetadata`, `getNativeSwapDecimals`) and atomic `getCoinBalance` / `getPublicKey` removed from `@vultisig/sdk` public surface
- [ ] Types `Coin`, `CoinKey`, `CoinMetadata`, `KnownCoin`, `KnownCoinMetadata`, `TokenMetadataResolver`, `VaultIdentity`, `PrepareSendTxFromKeysParams`, `PrepareSwapTxFromKeysParams`, `GetMaxSendAmountFromKeysParams` removed from public exports
- [ ] `VaultBase.getMaxSendAmount`, `TransactionBuilder.prepareSendTx` / `prepareContractCallTx` / `prepareSignAminoTx` / `prepareSignDirectTx`, `SwapService.executeSwap` restored to pre-0.17 inline implementations (no `tools/prep/` imports)
- [ ] `getPublicKey` in core-chain keeps the explicit throw on empty WalletCore derivation path (from `02738c15`)
- [ ] `getErc20Prices` lowercases contract-address keys in the returned object; unit test covers mixed-case input → lowercase output
- [ ] `getCosmosAccountInfo` zero-on-unknown-address behaviour (from separate PR #303) remains untouched
- [ ] New changeset file drafted; version bump cuts as 0.18.0 after `chore: version packages` merges
- [ ] `yarn build`, `yarn test`, `yarn lint` green in SDK repo
- [ ] mcp-ts #30 and vultiagent-app #164 closed (or their SDK pins reverted to `^0.16.x`) so no stale `^0.17.0` branches linger

## Gotchas

- Commit `7035e4dd` (`refactor(sdk): thread chainPublicKeys through prep helpers + tighten validation`) bundles the `getErc20Prices` lowercase fix with unrelated CLI changes and the `chainPublicKeys` threading — cannot cherry-pick; reimplement the 6-line fix manually.
- `b86792ad` (Cosmos `getCosmosAccountInfo` fix) is from PR #303, not #284 — do NOT revert it, it's independent.
- Reverting the `0fa10b96` merge commit requires `-m 1` (merge-commit parent). Branch out from the current `main` tip, then `git revert -m 1 0fa10b96` before cherry-picking.
- `getPublicKey.ts` lives in `@vultisig/core-chain` (not `@vultisig/sdk`); the preserved fix needs its own core-chain changeset entry if the revert's changeset only covers `@vultisig/sdk`.
- The `prep/` helpers currently imported by `VaultBase` / `TransactionBuilder` / `SwapService` use `vaultDataToIdentity(this.vaultData)` + `this.wasmProvider.getWalletCore()`; the pre-0.17 inline versions use `this.vaultData.publicKeys` directly and call `getPublicKey({...})` inline. Don't half-restore.
- `chainPublicKeys` threading also touches `TransactionBuilder.estimateSendFee` (line ~140 in the 0.17 file) — make sure the revert removes that `chainPublicKeys: this.vaultData.chainPublicKeys` argument from the `getPublicKey` call.
- Once 0.18.0 publishes, the `file:` override grep still applies to any dev branches: `grep -E 'file:\.\./|workspace:' package.json pnpm-lock.yaml` before pushing anything.

## Notes

**2026-05-05T00:19:03Z**

killed: premise overtaken — replacement PRs (mcp-ts#49, app#242) merged consuming the SDK 0.17.0 helpers; revert is moot
