---
id: v-srjz
status: closed
deps: []
links: []
created: 2026-04-20T06:56:03Z
type: task
priority: 2
assignee: Jibles
---
# App to SDK chain-logic migration

## Objective

Move chain-specific signing, encoding, and derivation logic from `vultiagent-app/src/services/` into `vultisig-sdk`, making the app a thin consumer that calls an SDK-level `signAndBroadcast` entrypoint. The monorepo's architecture doctrine is "business logic lives in the SDK, not in app/CLI/backend layers" — this pays down accumulated drift and unblocks simpler tool/chain work downstream (including token-layer tool retirement).

## Context & Findings

The app currently holds 7 chain-logic modules that should live in the SDK:
- `vultiagent-app/src/services/evmTx.ts` — RLP encoding, gas integration, EVM receipt polling
- `vultiagent-app/src/services/cosmosTx.ts` — amino/proto sign_doc construction
- `vultiagent-app/src/services/tronTx.ts` — Tron serialization + BIP-32 derivation
- `vultiagent-app/src/services/xrpTx.ts` — XRP serialization, DER encoding
- `vultiagent-app/src/services/suiTx.ts` — Sui coin selection + Move-call construction
- `vultiagent-app/src/services/addressDerivation.ts` — BIP-32 from root keys
- `vultiagent-app/src/services/fastSign.ts` — single-sig / fast-sign orchestration

The SDK already owns: the resolver layer (`packages/core/mpc/keysign/chainSpecific/resolvers/*`), MPC handshake (`packages/core/mpc/keysign/`), and prep helpers (`packages/sdk/src/tools/prep/*`). Recent branch `refactor/v-pxuw-tool-consolidation` introduced a `signAndBroadcast` wrapper in the app but per-chain specifics still live in the app.

**Goal shape:** app calls `sdk.signAndBroadcast(pendingTx, vault, password, opts)` and renders UI around the ceremony. Biometric prompt + MPC participation stay in the app because they need device state; payload construction, per-chain signing assembly, and broadcast RPC dispatch move to the SDK.

**Rejected approaches:**
- "Just improve the SDK facade without moving logic." Violates architecture doctrine and forces every future chain/token addition to touch both repos.

## Files

- `vultiagent-app/src/services/{evmTx,cosmosTx,tronTx,xrpTx,suiTx,fastSign,addressDerivation}.ts` — source modules to port
- `vultiagent-app/src/features/agent/lib/signAndBroadcast.ts` — current orchestrator; becomes a thin SDK wrapper
- `vultisig-sdk/packages/core/chain/chains/*` — destination per-chain logic
- `vultisig-sdk/packages/sdk/src/` — SDK-level public API surface
- `vultisig-sdk/packages/core/mpc/keysign/chainSpecific/resolvers/*` — existing resolver layer to extend
- `CLAUDE.md` (monorepo root) — architecture doctrine

## Acceptance Criteria

- [ ] SDK exposes a unified `signAndBroadcast(pendingTx, vault, password, opts)` entrypoint that dispatches per-chain
- [ ] Each of the 7 app-side modules is moved into the SDK with feature parity
- [ ] `vultiagent-app/src/services/` no longer contains chain-specific encoding/serialization logic
- [ ] Existing user flows unchanged end-to-end: EVM send, UTXO send, cosmos send/swap, tron/xrp/sui sends, EVM contract calls, LP flows
- [ ] Unit tests for migrated modules live in vultisig-sdk
- [ ] App typecheck + tests + lint pass with the thin wrappers
- [ ] SDK typecheck + tests + lint pass with the migrated modules
- [ ] PR description enumerates each migrated module and its SDK destination

## Gotchas

- WalletCore is initialized in the app; SDK needs it via dependency injection or initialize inside SDK — don't duplicate init.
- `waitForEvmReceipt` / `txVerifier` use AbortSignal — preserve this pattern in SDK.
- `fastSign.ts` handles a single-sig path distinct from MPC — verify the SDK has a parallel path and isn't assuming MPC.
- `probeDerivationPaths` (gated by `__DEV__`) — move to SDK as a diagnostic, not app-level.
- Don't move React-specific code. The SDK function should be framework-agnostic; React hooks (`useTransactionFlow`, `useToolExecution`) call into it.
- App still needs to render stepper progress during signing — design the SDK API to emit progress events (callbacks / async iterator) rather than forcing polling.
- This ticket is vultiagent-app + vultisig-sdk only. mcp-ts and agent-backend updates only if their runtime depends on SDK shape.

## Notes

**2026-05-05T00:19:03Z**

killed: pre-launch defer — App→SDK chain-logic migration is huge-scope architectural doctrine work; finish in-flight first
