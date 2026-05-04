---
id: va-flpx
status: closed
deps: []
links: []
created: 2026-04-30T00:17:34Z
type: feature
priority: 2
assignee: Jibles
---
# Multi-chain receipt-polling: abstract dispatcher + 5 new probes

## Objective

Replace the EVM-only `waitForEvmReceipt` with a chain-agnostic `waitForReceipt(chain, txHash)` dispatcher and ship probes for 5 additional chain kinds (Solana, Cosmos, XRP, Sui, Tron). Lights up the va-tbdw two-status lifecycle (Submitted → Confirmed/Failed) for ~27 of our 35-ish supported chains. Deferred families (UTXO, TON, Cardano, Polkadot) pass through transparently — same observable behavior as today, no Confirm dot in the stepper.

## Context & Findings

PR #296 (branch `va-tbdw-await-main-tx-receipt`) introduced the Submitted/Confirmed/Failed/Cancelled lifecycle but only EVM has a real receipt-poll today. Non-EVM transitions submitted → confirmed in one render with no honest "is it on-chain yet?" check.

Polling machinery (loop, backoff, timeout, abort, error normalization) is genuinely shared across chains. Per-chain logic is only "given a tx hash, return pending/success/failure right now" — fits a clean `ReceiptProbe` interface + a generic `pollReceipt` loop.

**Validated APIs (all probes are ~10-20 LOC):**
- **EVM**: existing — `eth_getTransactionReceipt` → `result.status: 0x0/0x1`.
- **Solana**: `getSignatureStatuses` → commitment level + `meta.err`. Use `confirmed` commitment (matches Phantom, ~1-2s); `finalized` adds ~13s without meaningful safety gain.
- **Cosmos**: REST `/cosmos/tx/v1beta1/txs/{hash}` → `tx_response.code` (0 = ok). Same shape across all 10 cosmos chains in our registry. Tendermint instant-finality on inclusion.
- **XRP**: JSON-RPC `tx` method → `meta.TransactionResult` (`tesSUCCESS`, `tecPATH_DRY`, …). ~3s validation.
- **Sui**: `sui_getTransactionBlock` with `showEffects: true` → `effects.status.status` (`success`|`failure`).
- **Tron**: validated by sub-agent probe against a real Tron tx — `tron-rpc.publicnode.com` serves eth-compat JSON-RPC at `POST /jsonrpc` (NOT root `/`), returning the same shape as EVM. Probe should share parser with EVM via a small factory.

**Decisions made (these are settled, do not re-litigate):**
- Solana commitment level: **confirmed**, not finalized.
- `waitForEvmReceipt`: **migrate everything to `waitForReceipt`** and delete the old name. Don't keep as a thin re-export.
- Test strategy: **one probe test per added chain** (6 total), mocked `fetch`. Plus dispatcher routing test.
- THORChain outbound observation: **out of scope, separate follow-up**. The cosmos probe verifies inbound success only.

**Deferred families (each gets a one-liner rationale in the PR body):**
- **UTXO** (BTC, LTC, DOGE, BCH, DASH, ZEC): 10-min block times — foreground spinner is wrong UX; wants async/push pattern.
- **TON**: workchain-sharded async settle by `msg_hash`; multi-step probe needs careful design.
- **Cardano**: indexer (Koios) dependency, lower-priority chain in this app.
- **Polkadot**: subscription-based event model, doesn't fit request/response probe interface.

**Rejected approaches:**
- Native Tron `gettransactioninfobyid`: more code than eth-compat for marginal benefit. Failure-reason granularity is the only delta — can be added later as a fallback if needed.
- Per-chain URL config strings on the chain registry: pushes complexity into the registry that lives better in the probes.
- A two-tier `Subscription`/`Probe` abstraction: not justified by current scope (no subscription chains added in this PR).

## Files

- `src/features/agent/lib/receiptPolling.ts` — new module: `ReceiptStatus`, `ReceiptProbe`, `pollReceipt`, `RECEIPT_PROBES` map, public `waitForReceipt`, `hasReceiptPoll`.
- `src/features/agent/lib/probes/` — new directory:
  - `evmReceipt.ts` (lifted/refactored from current `waitForEvmReceipt` body)
  - `tronReceipt.ts` (eth-compat probe via `${getRpcUrl(chain)}/jsonrpc` — share parser with EVM)
  - `solanaReceipt.ts`, `cosmosReceipt.ts`, `xrpReceipt.ts`, `suiReceipt.ts`
- `src/features/agent/lib/txSigning.ts` — delete `waitForEvmReceipt`. Delete `getEvmRpcUrl` if it has no other consumers post-migration; otherwise keep, marked deprecated.
- `src/features/agent/lib/signAndBroadcast.ts` — migrate `waitForEvmReceipt` call sites (in `evm` and `evm-sequence` branches) to `waitForReceipt`.
- `src/features/agent/hooks/useTransactionFlow.ts` — replace `waitForEvmReceipt` import with `waitForReceipt`. Drop `prep.txArgs.tx_encoding === 'evm'` and `prep.approvalTxArgs.tx_encoding === 'evm'` gates around receipt-wait calls (dispatcher no-ops for unsupported chains, gate becomes redundant). Replace `isEvm(submittedChainName)` in resume-on-hydrate effect with `hasReceiptPoll(submittedChainName)`.
- `src/features/agent/components/tools/ExecuteCards/shared.tsx` — `getEffectiveSteps` uses `hasReceiptPoll(chain)` instead of `tx_encoding === 'evm'` to decide whether to append the Confirm dot. Derive `Chain` from prep via existing `resolvePendingTxChain(prep.txArgs)`.
- `src/features/agent/lib/__tests__/receiptPolling.test.ts` — pin dispatcher: routes by `chainKind`, no-ops for unsupported, `hasReceiptPoll` boolean.
- `src/features/agent/lib/__tests__/probes/{evm,solana,cosmos,xrp,sui,tron}Receipt.test.ts` — one file per probe, mocked `fetch`, covering pending + success + (representative) failure shapes.
- **PR #296 body** — update via `gh pr edit 296 --repo vultisig/vultiagent-app --body-file <tmpfile>`. Add a "Receipt-poll coverage" section listing the 6 added chain kinds AND the 4 deferred families with one-liner rationale each. Preserve the existing summary, test plan, and receipts blocks above the auto-generated trailer.

## Acceptance Criteria

- [ ] `receiptPolling.ts` exports the public API (`ReceiptStatus`, `ReceiptProbe`, `pollReceipt`, `waitForReceipt`, `hasReceiptPoll`)
- [ ] Probes implemented for: evm, solana, cosmos, xrp, sui, tron — all in `lib/probes/`
- [ ] `RECEIPT_PROBES` map covers exactly those 6 chain kinds; UTXO/TON/Cardano/Polkadot have no entry
- [ ] Solana probe treats `confirmationStatus === 'confirmed'` (or `'finalized'`) as success — not waiting for `finalized` exclusively
- [ ] Tron probe uses `\${rpcUrl}/jsonrpc` and shares the EVM parser
- [ ] `waitForEvmReceipt` is removed from the codebase; all call sites migrated to `waitForReceipt`
- [ ] `useTransactionFlow.ts` has no `tx_encoding === 'evm'` gate around receipt-wait calls
- [ ] `useTransactionFlow.ts` resume effect uses `hasReceiptPoll`, not `isEvm`
- [ ] Stepper's `getEffectiveSteps` uses `hasReceiptPoll(chain)` for the Confirm-dot gate
- [ ] One unit test file per added probe (6 total), mocked `fetch`, covering pending + success + failure shapes where representative
- [ ] Dispatcher unit test pins routing per chain kind + no-op for unsupported
- [ ] `npx tsc --noEmit` clean
- [ ] `npx jest src/features/agent` — existing 835+ tests still pass plus new tests
- [ ] PR #296 body has a "Receipt-poll coverage" section enumerating: added chains (with brief mechanism summary) and deferred families (with rationale per skip)
- [ ] Commit + push to `va-tbdw-await-main-tx-receipt` branch (non-force, fast-forward); the PR auto-syncs

## Gotchas

- Use `getRpcUrl(chain)` from `@/lib/chains` for all probes — don't reinvent URL resolution per chain.
- Cosmos hash format: uppercase hex, no `0x` prefix (cosmos-sdk REST is strict). Normalize incoming hashes accordingly.
- Solana signatures are base58, NOT hex — don't apply EVM-style hash normalization.
- Tron eth-compat lives at `POST /jsonrpc`, not at root `/`. Root returns HTTP 405.
- `waitForReceipt` no-op for unsupported chains is the explicit design — DO NOT throw "chain not supported". Pre-PR behavior had no receipt wait at all; this change is strictly additive.
- The `signAndBroadcast.ts` `evm-sequence` loop calls `waitForEvmReceipt` between consecutive txs in a multi-tx envelope. Migrating those call sites to `waitForReceipt` is correct (the branch's precondition keeps it EVM in practice).
- Public node providers (esp. Tron) rate-limit aggressively. The existing 500ms→2s exponential backoff handles this fine; no special-casing.
- THORChain swaps will surface as "failed" via the cosmos probe more often than other Cosmos sends due to inbound-leg refund logic. That is correct — the user IS seeing a real on-chain failure. Outbound observation (T+1 settle) is a separate concern, filed as follow-up.
- Existing manual receipts on the PR (no-approve happy path, with-approve happy path, OOG failure, self-send, Arbiscan corroboration) are EVM-only. They don't need to be re-captured for non-EVM coverage; mention coverage in the body text and let probe unit tests carry the proof for the non-EVM additions. Optional: capture a Cosmos or Solana real-world receipt as bonus evidence if cheaply available.

## Notes

**2026-04-30T00:28:20Z**

Task 1 complete: receiptPolling.ts dispatcher + EVM probe extracted into probes/. 23 new tests pass; full agent suite 858/858 still passes. Other 5 probes + map entries land in Task 2.

**2026-04-30T00:44:54Z**

Tasks 2-5 complete. 6 probes (evm/solana/cosmos/xrp/sui/tron) shipped, 5 call sites migrated, stepper gate switched. Final review caught hash-corruption bug in pollReceipt (forced 0x prefix would have silently broken solana/sui/xrp); fixed and covered by regression test. 910 unit + 47 component tests pass, tsc clean. Pushed 637baf5b to va-tbdw-await-main-tx-receipt; PR #296 body updated with Receipt-poll coverage section.
