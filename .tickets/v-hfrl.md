---
id: v-hfrl
status: open
deps: []
links: [v-mehr]
created: 2026-03-30T02:26:53Z
type: task
priority: 2
assignee: Jibles
---
# Unify vultiagent-app onto @vultisig/sdk — pluggable MPC provider + migration

## Problem Statement

vultiagent-app (React Native/Expo) reimplements ~10k lines of chain registry, transaction building, MPC signing, and relay orchestration logic that already exists in @vultisig/sdk. vultiagent-cli correctly consumes the SDK as a thin wrapper. The goal was for both apps to share a common SDK, but vultiagent-app went native instead. "Solved" means both apps import @vultisig/sdk, with the only app-specific code being UI and platform bindings (expo-dkls for native MPC on mobile, WASM for Node/browser).

## Research Findings

### Current State

**Three repos, two implementations of the same logic:**

| | vultiagent-cli | vultiagent-app |
|---|---|---|
| Uses @vultisig/sdk? | Yes (`file:../vultisig-sdk/packages/sdk`) | No — full reimplementation |
| Package manager | npm | npm |
| DKLS/MPC | SDK's Rust WASM (10.8 MB) via `@lib/dkls` | Go native via `expo-dkls` (gomobile xcframework/aar) |
| Chain logic | SDK's chain adapters | Own `src/lib/chains.ts` (31 chains), `src/lib/coins.ts` |
| TX building | SDK's `vault.prepareSendTx()` | Own per-chain builders: `evmTx.ts`, `cosmosTx.ts`, `solanaTx.ts`, `suiTx.ts`, `tonTx.ts`, `tronTx.ts` |
| Signing orchestration | SDK's `vault.sign()` | Own relay coordination in `src/services/auth/fastVaultSign.ts`, `src/services/relay.ts` |
| Domain logic LOC | ~2.5k (thin wrapper) | ~10k (full reimplementation) |

**vultiagent-app code layout:**
- `src/lib/` (10 files, ~2.6k LOC) — pure domain logic (chains, coins, CAIP, schedules). No RN/Expo deps. Clean.
- `src/services/` (38 files, ~7.2k LOC) — TX builders, MPC signing, vault creation, auth, relay. 22 of ~30 files import Expo things (`expo-crypto`, `expo-secure-store`, `expo-dkls`, `react-native-sse`).
- `modules/expo-dkls/` — Expo native module wrapping Go gomobile binaries for DKLS (ECDSA) and Schnorr (EdDSA).
- Code comments acknowledge duplication: "Mirrors vultisig-sdk DKLS.processOutbound", "Reference: vultisig-sdk/packages/..."

**Two separate DKLS implementations:**
- **Rust** (`vultisig-windows/lib/dkls/`) → WASM via wasm-pack → used by SDK in Node/browser
- **Go** (`vultisig/mobile-tss-lib`) → native via gomobile → used by vultiagent-app on iOS/Android
- Both implement the same MPC protocol (`bnb-chain/tss-lib/v2` underneath), different source languages

### Available Tools & Patterns

**SDK platform abstraction (already exists):**
- `packages/sdk/src/platforms/` — node, browser, react-native, electron-main, chrome-extension
- Each platform registers: crypto (`configureCrypto()`), storage, polyfills, WASM init (`configureWasm()`)
- Rollup builds separate bundles: `index.node.esm.js`, `index.browser.js`, `index.react-native.js`, etc.
- DI pattern via global context (`context/wasmRuntime.ts`) — callback-based registration

**SDK MPC layer (where the coupling is):**
- `core/mpc/lib/signSession.ts` — **directly imports** `@lib/dkls/vs_wasm` and `@lib/schnorr/vs_schnorr_wasm`
- `core/mpc/lib/initialize.ts` — directly imports WASM init functions
- `core/mpc/keysign/index.ts` — calls `makeSignSession()` which returns WASM session objects
- The `SignSession` interface is implicit (duck-typed from WASM classes): `outputMessage()`, `inputMessage()`, `finish()`

**expo-dkls native module API (`modules/expo-dkls/src/ExpoDklsModule.ts`):**
- Exposes: keygen setup/session/messages, keysign setup/session/messages, Schnorr equivalents, key import
- iOS: Swift wrapping `godkls.xcframework` + `goschnorr.xcframework` via C FFI
- Android: Kotlin wrapping `dkls-release.aar` + `goschnorr-release.aar` via JNI/SWIG
- Frameworks are pre-built binaries, not compiled from source in this repo

**vultiagent-cli as reference thin client:**
- `src/lib/sdk.ts` — initializes SDK, imports vault, overlaps WASM compilation with file I/O
- `src/lib/signing.ts` — retry wrapper around `vault.sign()`
- `src/commands/` — each command calls high-level SDK methods (`vault.balance()`, `vault.prepareSendTx()`, etc.)
- Total: ~2.5k LOC for a full-featured CLI with MCP support

### Validated Behaviors

- Confirmed: SDK's `platforms/react-native/` entry point exists and builds, but the DKLS binding is still WASM (not native)
- Confirmed: `signSession.ts` directly imports WASM classes — no abstraction point for native bindings
- Confirmed: expo-dkls and SDK WASM expose the same conceptual API (setup, outputMessage, inputMessage, finish) — just different function signatures
- Confirmed: vultiagent-app's `src/lib/` (chains, coins) has zero RN/Expo imports and could be dropped entirely in favor of SDK equivalents
- Confirmed: SDK already has `MpcProvider`-like precedent in `WasmProvider` pattern and `configureCrypto()` DI
- Confirmed: both repos use npm (package-lock.json), not pnpm/yarn

## Constraints

- expo-dkls native module must remain for React Native — WASM performance on Hermes (RN's JS engine) is poor, which is likely why they went native
- The Go (mobile-tss-lib) and Rust (vultisig-windows/lib/dkls) are separate codebases implementing the same protocol — unifying them into one source language is a separate effort (Phase 2)
- SDK's `@core/mpc` and `@lib/dkls` packages are synced from vultisig-windows (read-only upstream) — changes to core MPC must flow through the SDK's own abstraction, not by modifying synced files
- vultiagent-app uses Expo-specific APIs for secure storage (`expo-secure-store`), biometrics (`expo-local-authentication`), camera, etc. — these are app-level concerns, not SDK concerns, and should stay in the app

## Dead Ends

- **Go → WASM for unified binary**: Go's WASM output bundles the entire Go runtime (~15-20 MB), has no threading, and performs poorly vs Rust WASM. Not viable for web/Node.
- **Pure TypeScript MPC implementation**: MPC protocol involves intensive elliptic curve operations — would be orders of magnitude slower and a security risk. The heavy crypto must stay in a systems language.
- **Extracting vultiagent-app's code into a new SDK**: The app's service code is coupled to Expo (`expo-crypto` for random bytes, `expo-dkls` for signing). Would need the same platform abstraction the existing SDK already has. Building a second SDK makes no sense.

## PoC Results: Pluggable MPC Provider (2026-03-30)

### What was done
Added an `MpcProvider` interface + `configureMpc()` DI pattern to the SDK, wrapping all WASM imports behind a pluggable provider. The existing WASM path works identically through the new abstraction.

### Files changed (SDK)
- **NEW** `packages/core/mpc/lib/mpcProvider.ts` — interface + registry (55 LOC)
- **NEW** `packages/core/mpc/lib/wasmMpcProvider.ts` — WASM implementation (63 LOC)
- **MODIFIED** `packages/core/mpc/lib/initialize.ts` — delegates to provider (was 18 LOC, now 13)
- **MODIFIED** `packages/core/mpc/lib/keyshare.ts` — delegates to provider (was 22 LOC, now 15)
- **MODIFIED** `packages/core/mpc/lib/signSession.ts` — delegates to provider (was 35 LOC, now 55)
- **MODIFIED** 4 platform entry points + 2 test setups — 2-line addition each (`configureMpc(wasmMpcProvider)`)
- **UNCHANGED** `keysign/index.ts`, `keysign/setupMessage/make.ts` — zero modifications needed

### Test results
- TypeScript typecheck: PASS
- SDK unit tests: 336/336 passed (21 files)
- CLI tests: 172/172 passed (22 files)
- SDK build (node): PASS
- E2E tests: setup loads correctly, skips at vault loading (no test vault configured)

### Key findings
1. **Clean abstraction**: Only 3 files needed modification in core MPC. The keysign orchestration (`keysign/index.ts`) required zero changes — it already imported through the lib files we shimmed.
2. **Schnorr type difference**: DKLS `setup()` takes `messageHash: Uint8Array | null | undefined`, Schnorr takes `message: Uint8Array` (non-nullable). Provider interface uses nullable, WASM provider asserts non-null for Schnorr with `!`.
3. **Keygen not covered**: `dkls/dkls.ts` and `schnorr/schnorrKeygen.ts` still import WASM directly. These would need a separate `MpcKeygenProvider` interface. Fine for now since vultiagent-app creates vaults via server, not local keygen.
4. **Upstream sync risk**: Modified files in `packages/core/` are synced from vultisig-windows. Production approach: either upstream the abstraction or move the provider layer to `packages/sdk/src/` with import interception.

### Confidence assessment
- **MPC keysign pluggability**: HIGH — proven working, clean interface
- **Full vultiagent-app migration to SDK**: MEDIUM — chain registry + TX building are trivial (zero RN deps in `src/lib/`), but MPC keygen + the expo-dkls handle-based API vs WASM class-based API needs careful adapter work
- **Native MpcProvider for expo-dkls**: MEDIUM — the conceptual API maps well (setup/output/input/finish), but expo-dkls uses integer handles vs WASM objects, and all messages are base64 strings vs Uint8Array. Adapter is straightforward but needs testing with real native module.

## Open Questions

- **MpcProvider interface shape**: The SDK's WASM classes use `new SignSession(setup, id, keyshare)` with `outputMessage()/inputMessage()/finish()`. expo-dkls uses flat functions (`dklsKeysignSetup()`, `dklsKeysignSession()`, `dklsKeysignOutbound()`, etc.). How thin should the adapter be? Should it mirror the WASM class API or define a higher-level interface?
- **Where does the MpcProvider get registered?** Current pattern is `configureWasm()` in platform entry points. Should MPC follow the same pattern (`configureMpc()`) or be part of the existing `configureWasm()` call?
- **Migration strategy for vultiagent-app**: Big bang (replace all services at once) vs incremental (start with chain registry/coins, then TX building, then MPC signing)? The chain registry and coins have zero RN deps and could be swapped trivially. MPC is the hard part.
- **expo-dkls ownership**: Should it stay in vultiagent-app's `modules/` or move into the SDK monorepo as a platform-specific package (e.g., `@vultisig/dkls-native`)?
- **Testing parity**: How do we verify the native MPC provider produces identical signatures to the WASM provider? Is there an existing test harness for this?
- **SDK React Native build target**: The SDK already has `platforms/react-native/` — is it currently tested/used by anything, or is it a skeleton? Does it need work before vultiagent-app can consume it?
