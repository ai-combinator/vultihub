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
# Unified Vultisig SDK — single source of truth for all clients

## Problem Statement

The Vultisig ecosystem has fragmented implementations across clients: vultiagent-app reimplements ~10k lines of chain/signing logic that exists in @vultisig/sdk, the MPC protocol has parallel Rust (WASM) and Go (native) implementations, and core packages are owned by vultisig-windows with a sync-and-copy pipeline to the SDK. "Solved" means one SDK monorepo that owns all business logic (TypeScript) and MPC cryptography (Rust source), compiling to every target. CLI, mobile app, and server are thin wrappers.

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

- Expo-specific APIs (secure storage, biometrics, camera) are app-level concerns and stay in the app, not the SDK
- SDK's `@core/mpc` and `@lib/dkls` packages are currently synced from vultisig-windows — forking the SDK means owning these directly
- vultiserver is Go — it consumes MPC via CGO and must continue to. Changes to the Rust build must produce compatible `.so` + C headers

## Dead Ends

- **Go → WASM for unified binary**: Go's WASM output bundles the entire Go runtime (~15-20 MB), no threads, poor performance vs Rust WASM. Not viable.
- **Pure TypeScript MPC**: Elliptic curve MPC in JS would be orders of magnitude slower and a security risk. Must stay in a systems language.
- **Extracting vultiagent-app into a second SDK**: Its service code is coupled to Expo. The existing SDK already has the platform abstraction it would need. No point building a second one.
- **Pure JS shim for wallet-core on RN**: vultiagent-app covers 27/36 chains with @noble/@scure, but UTXO is missing and maintaining parity with wallet-core's behavior across 36+ chains is a permanent tax. Native wallet-core is the better path.

## Validated Research

### wallet-core DI is complete (2026-03-30)

Audited all 65 files that import `@trustwallet/wallet-core`. Every single one receives walletCore as a function parameter — zero direct runtime imports in business logic. The DI chain:

```
Platform entry point: configureWasm(async () => initWalletCore())
    → SdkContextBuilder: wasmProvider = { getWalletCore }
        → Service classes: this.wasmProvider.getWalletCore()
            → Pure functions: walletCore passed as parameter
                → Chain resolvers, address derivation, TX compilation
```

This means providing an alternative wallet-core implementation (native instead of WASM) requires zero changes to business logic — just a different `configureWasm()` registration.

### TrustWallet native wallet-core is viable for React Native (2026-03-30)

**Key findings:**
- wallet-core is a C++ library. WASM is the secondary target; native iOS/Android is primary. Trust Wallet's own mobile app uses the native version.
- iOS: `.xcframework` via SPM/CocoaPods (~32 MB). Android: `.aar` via Maven (~54 MB). Both actively maintained, MIT licensed.
- **Protobuf types (`TW.*.Proto.*`) are pure protobufjs-generated JavaScript** — zero WASM dependency. They work as-is regardless of WASM vs native runtime. This was the biggest risk and it's a non-issue.
- Expo Modules API supports **synchronous JSI functions** — native wallet-core methods can be called synchronously from JS, preserving the exact call pattern the SDK uses (no async conversion).
- A typical transaction involves ~15-20 wallet-core calls. With JSI, that's sub-microsecond per call. Negligible vs MPC ceremony and network RTTs.
- Native wallet-core API is a **superset** of the WASM API — same C++ core, same classes, same methods.
- No maintained RN bridge exists, but wrapping in an Expo module (like expo-dkls) is straightforward — ~15 classes/namespaces to expose.

**What `expo-wallet-core` needs to expose:**
- Static utilities: `CoinType`, `CoinTypeExt`, `HexCoding`, `TransactionCompiler`, `AnySigner`, `TransactionDecoder`, `Bech32`, `Curve`
- Factory/instance classes (as JSI HostObjects): `HDWallet`, `PublicKey`, `AnyAddress`, `DataVector`, `BitcoinScript`, `EthereumAbi`/`EthereumAbiFunction`, `SolanaAddress`, `TONAddressConverter`, `PrivateKey`
- NOT needed in native bridge: All `TW.*` protobuf types (pure JS, already works)

### Pluggable MPC Provider proven (2026-03-30)

Added `MpcProvider` interface + `configureMpc()` DI pattern. Only 3 core files modified. All 336 SDK tests + 172 CLI tests pass. Keysign orchestration (`keysign/index.ts`) required zero changes. See PoC details below.

### vultiagent-app chain coverage (2026-03-30)

The app has full tx building (address derivation + tx construction + signing) for **27 of 31 registry chains** using pure JS:
- 13 EVM, 10 Cosmos, Solana, Sui, TON, Tron
- Only UTXO missing (Bitcoin, Litecoin, Dogecoin, Bitcoin-Cash — explicitly stubbed)
- Libraries: @noble/curves, @noble/hashes, @scure/bip32, @scure/bip39, @solana/web3.js, @ton/core, manual protobuf

This validates that the SDK's chain logic runs fine without WASM wallet-core for most chains. But with native wallet-core, all 36+ chains work out of the box — including UTXO.

## Final Goal Architecture

### Two native dependencies, both pluggable via DI

The SDK has exactly two binary dependencies that can't be pure TypeScript:

1. **wallet-core** — address derivation, tx building, compilation (36+ chains)
2. **DKLS/Schnorr** — MPC threshold signing

Both are already injected via DI (`configureWasm()` / `configureMpc()`). Each platform provides the right implementation:

| Platform | wallet-core | MPC (DKLS/Schnorr) |
|---|---|---|
| **Node.js (CLI)** | WASM (`@trustwallet/wallet-core`) | WASM (`@lib/dkls`, `@lib/schnorr`) |
| **Browser** | WASM | WASM |
| **Electron** | WASM | WASM |
| **React Native** | Native via `expo-wallet-core` | Native via `expo-dkls` |
| **vultiserver** | N/A (server doesn't build txs) | Rust `.so` via Go CGO |

### Target monorepo structure

```
@vultisig/sdk (pnpm monorepo — source of truth for all clients)
├── rust/
│   ├── dkls/              # Rust DKLS source (from dkls23-rs)
│   ├── schnorr/           # Rust Schnorr source (from multi-party-schnorr)
│   └── Cargo.toml         # Workspace — builds WASM, iOS, Android, server targets
├── packages/
│   ├── core-chain/        # Chain implementations (from vultisig-windows, now owned)
│   ├── core-mpc/          # MPC orchestration (TS — keysign, keygen, relay, messaging)
│   ├── core-config/       # Chain metadata
│   ├── lib-dkls/          # DKLS WASM output + TS types
│   ├── lib-schnorr/       # Schnorr WASM output + TS types
│   ├── lib-utils/         # Crypto helpers
│   ├── sdk/               # Main SDK — platform abstraction + vault API
│   │   └── platforms/
│   │       ├── node/          # WASM wallet-core + WASM MPC
│   │       ├── browser/       # WASM wallet-core + WASM MPC
│   │       ├── react-native/  # expo-wallet-core + expo-dkls
│   │       └── electron/      # WASM wallet-core + WASM MPC
│   ├── expo-wallet-core/  # Expo module wrapping TrustWallet native binaries
│   └── expo-dkls/         # Expo module wrapping Rust MPC native binaries (moved from app)
├── apps/
│   └── cli/               # vultiagent-cli (thin wrapper)
└── pnpm-workspace.yaml
```

vultiagent-app stays as a separate repo but becomes a thin React Native shell importing `@vultisig/sdk/react-native`.

### MPC Rust build pipeline — one source, four targets

```bash
# 1. WASM (Node/browser/Electron)
wasm-pack build -t web --out-dir packages/lib-dkls/wasm

# 2. iOS native (.xcframework for expo-dkls)
cargo build --release --target aarch64-apple-ios
cargo build --release --target aarch64-apple-ios-sim
xcodebuild -create-xcframework ...

# 3. Android native (.so → .aar for expo-dkls)
cargo ndk -t arm64-v8a -t armeabi-v7a -t x86_64 build --release

# 4. Server native (.so + C headers for vultiserver CGO)
cargo build --release --target x86_64-unknown-linux-gnu
cbindgen --config cbindgen.toml --output include/dkls.h
```

### Current binary landscape (what we're cleaning up)

| Source | Language | Output | Consumer | Status after unification |
|---|---|---|---|---|
| `vultisig/dkls23-rs` | Rust | WASM | SDK (Node/browser) | Moved into our repo |
| `vultisig/multi-party-schnorr` | Rust | WASM | SDK (Node/browser) | Moved into our repo |
| `vultisig/mobile-tss-lib` | Go | .xcframework/.aar | vultiagent-app | **Eliminated** — Rust compiles to native directly |
| `vultisig/go-wrappers` | Go (CGO) | Go package | vultiserver | Consumes our `.so` + headers instead |
| `@trustwallet/wallet-core` | C++ | WASM npm package | SDK (all platforms) | Stays for WASM platforms |
| TrustWallet native | C++ | .xcframework/.aar | Trust Wallet app | Wrapped in `expo-wallet-core` for RN |
| vultisig-windows `sync-and-copy` | — | — | SDK packages | **Eliminated** — we own the source |

### What each consumer looks like after unification

**vultiagent-cli** (already close to this):
```typescript
import { Vultisig } from '@vultisig/sdk/node'
const sdk = new Vultisig()
const vault = await sdk.importVault(vaultFile, password)
await vault.sign(payload)
```

**vultiagent-app** (currently reimplements everything, becomes thin):
```typescript
// platforms/react-native/index.ts registers:
// - expo-wallet-core as wallet-core provider
// - expo-dkls as MPC provider
import { Vultisig } from '@vultisig/sdk/react-native'
const sdk = new Vultisig()
// Same API as CLI — address derivation, tx building, signing all work
```

**vultiserver** (minimal change):
- Continues using Go + CGO
- `mpc_wrapper.go` unchanged — same C FFI function signatures
- `.so` + C headers come from our Rust build instead of go-wrappers

## Phased Implementation

### Phase 1: SDK pluggability + expo-wallet-core (unblocks RN adoption)

1. **Fork SDK, own core packages** — remove sync-and-copy dependency on vultisig-windows
2. **`expo-wallet-core` Expo module** — wrap TrustWallet native iOS/Android binaries, expose ~15 classes via JSI synchronous calls. Register via `configureWasm()` in `platforms/react-native/`
3. **`MpcProvider` abstraction** (PoC complete) — production-ready the pluggable MPC interface. Register expo-dkls as the native provider in `platforms/react-native/`
4. **Move `expo-dkls` into SDK monorepo** — becomes `packages/expo-dkls/`
5. **Migrate vultiagent-app** — replace reimplemented services with `@vultisig/sdk/react-native` imports. App becomes UI + Expo-specific features only.

**Result:** CLI and mobile app both consume the same SDK. Two native Expo modules provide platform-specific bindings. All business logic shared.

### Phase 2: Unified Rust MPC build (eliminates Go dependency)

1. **Bring Rust source into repo** — `dkls23-rs` and `multi-party-schnorr` into `rust/`
2. **Verify WASM builds match** — ensure `wasm-pack` output is identical to existing binaries
3. **Add server `.so` + C header build** — verify go-wrappers/vultiserver compatibility
4. **Add iOS/Android native builds** — `cargo-ndk` + `uniffi-rs`/`cbindgen`
5. **Update expo-dkls wrappers** — Swift/Kotlin call Rust C FFI directly instead of Go FFI
6. **Deprecate mobile-tss-lib and go-wrappers**

**Result:** One Rust codebase → WASM + iOS + Android + server. No Go MPC dependencies. Smaller mobile binaries (no Go runtime overhead).

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

### Key findings
1. **Clean abstraction**: Only 3 files needed modification in core MPC. Keysign orchestration required zero changes.
2. **Schnorr type difference**: DKLS `setup()` takes nullable messageHash, Schnorr takes non-nullable. Provider interface uses nullable, WASM provider asserts non-null for Schnorr.
3. **Keygen not covered**: `dkls/dkls.ts` and `schnorr/schnorrKeygen.ts` still import WASM directly. Would need `MpcKeygenProvider`. Fine for now — vultiagent-app creates vaults via server.

## Open Questions

- **expo-wallet-core scope**: Should it expose the full wallet-core API or only the ~15 classes the SDK actually uses? Minimal surface = less bridge code, but limits future SDK features.
- **Migration strategy for vultiagent-app**: Incremental (chain registry first, then TX building, then MPC) vs big bang? The DI architecture suggests incremental is safe — each layer can be swapped independently.
- **Monorepo structure**: Should vultiagent-app live inside the SDK monorepo (as `apps/mobile/`) or stay as a separate repo consuming published SDK packages?
- **Testing parity**: How do we verify native wallet-core + native MPC produces identical results to WASM + WASM? Need cross-platform integration tests.
- **wallet-core version pinning**: The native iOS/Android binaries must match the WASM npm package version exactly. How do we enforce this?
