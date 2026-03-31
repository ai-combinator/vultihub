---
id: v-hfrl
status: closed
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

**Fork target:** `premiumjibles` GitHub (forking `vultisig/vultisig-sdk`)

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

**Two separate DKLS implementations — different languages, same protocol:**
- **Rust** (`vultisig/dkls23-rs`, `vultisig/multi-party-schnorr`) → WASM via wasm-pack → SDK (Node/browser)
- **Go** (`vultisig/mobile-tss-lib`) → native via gomobile → vultiagent-app (iOS/Android)
- Both implement the same MPC protocol (`bnb-chain/tss-lib/v2` underneath)
- **vultiserver already runs Rust** — `go-wrappers` calls pre-compiled Rust `.so`/`.dylib` via CGO. Server crypto is Rust; Go is just FFI glue.
- vultiagent-app's mobile path has an unnecessary Go layer: Rust → Go CGO → gomobile → .xcframework/.aar. Direct Rust → native would be smaller and simpler.

**Three separate build/sync pipelines:**
- `sync-and-copy.ts` mirrors core packages from vultisig-windows → vultisig-sdk
- `mobile-tss-lib` produces gomobile binaries manually copied into vultiagent-app (git-ignored, no script)
- `go-wrappers` links against pre-compiled Rust `.so` for vultiserver

### Available Tools & Patterns

**SDK platform abstraction (already exists):**
- `packages/sdk/src/platforms/` — node, browser, react-native, electron-main, chrome-extension
- Each platform registers: crypto (`configureCrypto()`), storage, polyfills, WASM init (`configureWasm()`)
- Rollup builds separate bundles: `index.node.esm.js`, `index.browser.js`, `index.react-native.js`, etc.
- DI pattern via global context (`context/wasmRuntime.ts`) — callback-based registration

**wallet-core DI is complete — zero leaks:**
- Audited all 65 files importing `@trustwallet/wallet-core`. Every one receives walletCore as a function parameter.
- DI chain: `configureWasm()` → `wasmProvider.getWalletCore()` → passed to every function as parameter
- Providing an alternative implementation (native instead of WASM) requires zero changes to business logic

**Protobuf types are pure JS — no WASM dependency:**
- All `TW.*.Proto.*` types (signing inputs, outputs) come from `core_proto.js` — pure protobufjs-generated code
- These work identically regardless of WASM vs native wallet-core runtime
- This was the biggest perceived risk and it's a non-issue

**TrustWallet publishes native wallet-core binaries:**
- iOS: `.xcframework` via SPM/CocoaPods (~32 MB). Android: `.aar` via Maven (~54 MB)
- MIT licensed, actively maintained, used by Trust Wallet's own mobile app
- Native API is a superset of WASM API — same C++ core, same classes, same methods
- Expo Modules API supports synchronous JSI functions — preserves exact call pattern SDK uses
- ~15-20 wallet-core calls per transaction, sub-microsecond via JSI. Negligible.

**MPC provider pluggability proven (PoC):**
- Added `MpcProvider` interface + `configureMpc()`. Only 3 core files modified.
- All 336 SDK tests + 172 CLI tests pass. Keysign orchestration required zero changes.
- expo-dkls and SDK WASM expose the same conceptual API (setup, outputMessage, inputMessage, finish)

**vultiagent-app chain coverage:**
- 27 of 31 chains have full tx building in pure JS (@noble, @scure, manual protobuf)
- Only UTXO (Bitcoin, Litecoin, Dogecoin, Bitcoin-Cash) missing
- With native wallet-core, all 36+ SDK chains work including UTXO

**Rust source is Vultisig-owned:**
- `dkls23-rs` and `multi-party-schnorr` — Vultisig proprietary (SLL license)
- `multi-party-schnorr` has HashCloak security audit
- No third-party license concerns with bringing in-repo

### Build Environment (verified on dev machine)

| Tool | Status |
|---|---|
| Android SDK + emulator | `Medium_Phone_API_36.0` AVD ready |
| adb | Installed |
| Rust (stable, x86_64-linux-gnu) | Installed |
| wasm-pack | **Not installed** (need `cargo install wasm-pack`) |
| cbindgen | **Not installed** (need `cargo install cbindgen`) |
| cargo-ndk | **Not installed** (need `cargo install cargo-ndk`) |
| Android Rust targets | **Not installed** (need `rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android`) |
| agent-device | **Not installed** (need `npx agent-device`) — mobile simulator automation CLI for AI agents |
| iOS / macOS | **Not available** — Linux machine, iOS testing impossible locally |
| Test vault (.vult + password) | Available in SDK .env |
| Production backends | Available (agent.vultisig.com, verifier, relay, api) |

## Constraints

- Expo-specific APIs (secure storage, biometrics, camera) stay in the app, not the SDK
- vultiserver is Go — consumes MPC via CGO. Rust build must produce compatible `.so` + C headers
- No macOS available — iOS native modules can only be written and structurally tested; actual iOS verification requires a Mac. Android is fully testable via emulator.
- wallet-core native binary version must match WASM npm package version for behavioral parity

## Dead Ends

- **Go → WASM**: Bundles entire Go runtime (~15-20 MB), no threads, poor performance. Not viable.
- **Pure TypeScript MPC**: Orders of magnitude slower, security risk. Must stay in systems language.
- **Second SDK from vultiagent-app code**: Coupled to Expo, would need the same platform abstraction the existing SDK already has.
- **Pure JS wallet-core shim for RN**: 27/36 chains covered but UTXO missing, permanent maintenance tax. Native wallet-core gets all 36+ chains for free.

## Goal Architecture

### Two native dependencies, both pluggable via existing DI

| Platform | wallet-core | MPC (DKLS/Schnorr) |
|---|---|---|
| **Node.js (CLI)** | WASM (`@trustwallet/wallet-core`) | WASM (`@lib/dkls`, `@lib/schnorr`) |
| **Browser / Electron** | WASM | WASM |
| **React Native** | Native via `expo-wallet-core` | Native via `expo-dkls` (Rust → native, no Go) |
| **vultiserver** | N/A | Rust `.so` via Go CGO |

### Target monorepo structure

```
@vultisig/sdk (pnpm monorepo — premiumjibles fork)
├── rust/
│   ├── dkls/              # Rust DKLS source (from vultisig/dkls23-rs)
│   ├── schnorr/           # Rust Schnorr source (from vultisig/multi-party-schnorr)
│   └── Cargo.toml         # Workspace — WASM, Android, server targets (iOS needs Mac CI)
├── packages/
│   ├── core-chain/        # Chain implementations (owned, no longer synced)
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
│   └── expo-dkls/         # Expo module wrapping Rust MPC native binaries
├── apps/
│   └── cli/               # vultiagent-cli (thin wrapper)
└── pnpm-workspace.yaml
```

### MPC Rust build pipeline — one source, four targets

```bash
# WASM (Node/browser/Electron)
wasm-pack build -t web --out-dir packages/lib-dkls/wasm

# Android native (.so for expo-dkls)
cargo ndk -t arm64-v8a -t armeabi-v7a -t x86_64 build --release

# Server native (.so + C headers for vultiserver CGO)
cargo build --release --target x86_64-unknown-linux-gnu
cbindgen --config cbindgen.toml --output include/dkls.h

# iOS native (.xcframework — requires Mac CI, not buildable locally)
# cargo build --release --target aarch64-apple-ios
# cargo build --release --target aarch64-apple-ios-sim
# xcodebuild -create-xcframework ...
```

### What gets eliminated

- `mobile-tss-lib` Go repo — replaced by Rust → native directly
- `go-wrappers` build chain — consumes our `.so` + headers instead of owning Rust compilation
- `sync-and-copy.ts` — we own the core packages
- Manual binary copying into expo-dkls — build pipeline produces them
- ~10k LOC of reimplemented logic in vultiagent-app
- Go runtime overhead in mobile binaries (~9.6 MB per platform)
- Two independent implementations of the same MPC protocol

## Implementation Plan

### 1. Foundation — fork, restructure, verify (2-3 days)

**Work:**
- Fork `vultisig/vultisig-sdk` to `premiumjibles`
- Convert from yarn to pnpm (see v-mehr)
- Delete `scripts/sync-and-copy.ts`, promote synced packages to first-class workspace packages
- Rename workspace packages (`@core/chain` → `core-chain`, etc.) with proper `package.json` names
- Update all import paths and TypeScript path aliases
- Update Rollup build config for pnpm workspace resolution

**Verification:**
- `pnpm install` succeeds
- `tsc --noEmit` passes across all packages
- All 336 SDK unit tests pass
- All 172 CLI tests pass
- All 5 platform bundles build (node, browser, electron, chrome-extension, react-native)
- CLI can import vault from `.vult` file and run `vault.address()` against live backend

### 2. Pluggable MPC provider — production-ready (1-2 days)

**Work:**
- Productionize the PoC `MpcProvider` interface + `configureMpc()` pattern
- Add `MpcKeygenProvider` interface for keygen flows (same pattern)
- Move provider abstraction to `packages/sdk/src/mpc/` (not `packages/core/` since core was synced)
- Register `wasmMpcProvider` in all platform entry points
- Write adapter: `NativeMpcProvider` implementing `MpcProvider` using expo-dkls handle API

**Verification:**
- All existing SDK + CLI tests still pass (regression)
- New unit test: `MpcProvider` interface with mock provider → keysign flow runs, returns expected signature shape
- TypeCheck passes

### 3. expo-wallet-core — native wallet-core for React Native (5-8 days)

**Work:**
- Create Expo module `packages/expo-wallet-core/`
- Kotlin implementation (Android): wrap TrustWallet `.aar` via JNI, expose ~15 classes as JSI synchronous functions
- Swift implementation (iOS): wrap TrustWallet `.xcframework` via native calls, expose same ~15 classes
- TypeScript interface (`ExpoWalletCoreModule.ts`) matching the duck-typed interface the SDK expects from `initWasm()`
- Classes to expose: `CoinType`, `CoinTypeExt`, `HDWallet`, `PublicKey`, `PublicKeyType`, `AnyAddress`, `TransactionCompiler`, `AnySigner`, `DataVector`, `HexCoding`, `BitcoinScript`, `EthereumAbi`, `EthereumAbiFunction`, `SolanaAddress`, `TONAddressConverter`, `PrivateKey`, `Curve`, `Bech32`, `TransactionDecoder`
- JSI HostObjects for stateful classes (HDWallet, DataVector, PublicKey, AnyAddress, BitcoinScript) with proper lifecycle (`delete()` methods)

**Verification — Node.js cross-validation (no simulator needed):**
- Build a `MockWalletCore` using @noble/curves + @scure/bip32 that implements the same interface
- Run SDK unit tests through mock provider via `configureWasm()`
- Cross-validate: run same operations through WASM and mock, compare outputs byte-for-byte
- This proves the DI works and the interface shape is correct

**Verification — Android emulator (closes the native loop):**
- Boot `Medium_Phone_API_36.0` emulator headlessly (`emulator -no-window -avd Medium_Phone_API_36.0`)
- Build minimal Expo test app that imports expo-wallet-core and runs: address derivation, tx building, signing input encoding
- `npx expo run:android` to build + install
- Use `agent-device` to verify app launches and displays correct addresses
- `adb logcat` to catch native crashes, iterate until clean
- Cross-validate: compare addresses/signing inputs from emulator vs Node.js WASM — must match

### 4. Bring Rust source + build WASM and native (3-5 days)

**Work:**
- Clone `vultisig/dkls23-rs` and `vultisig/multi-party-schnorr` into `rust/`
- Set up Cargo workspace
- Install missing tools: `cargo install wasm-pack cbindgen cargo-ndk`, add Android Rust targets
- WASM build: `wasm-pack build -t web`, diff output against existing committed binaries
- Android native build: `cargo ndk` for arm64, armv7, x86_64
- Server native build: `cargo build --release` + `cbindgen` for C headers
- iOS native build scripts (written but not runnable locally — needs Mac CI)

**Verification:**
- `cargo test` passes on all Rust crates
- WASM output byte-identical to existing committed binaries (or functionally equivalent — verify by running SDK tests with new WASM)
- Server `.so` + C headers ABI-compatible with go-wrappers (diff generated headers against existing)
- Android `.so` files produced for all 3 architectures

### 5. expo-dkls — Rust native MPC for React Native (3-5 days)

**Work:**
- Move `vultiagent-app/modules/expo-dkls/` into `packages/expo-dkls/`
- Update Kotlin wrapper: replace Go JNI calls (`godkls.*`) with Rust JNI calls (link against `.so` from step 4)
- Update Swift wrapper: replace Go C FFI calls with Rust C FFI calls (link against Rust output)
- Update podspec and build.gradle to reference new binary sources
- Wire `NativeMpcProvider` adapter (from step 2) to use updated expo-dkls

**Verification — Android emulator:**
- Build test app with expo-dkls using Rust native binaries
- Run on emulator, execute MPC keysign against production relay/VultiServer using test vault
- Verify signature matches what CLI produces for same payload (cross-platform parity)
- `agent-device` to automate the flow, `adb logcat` for crash debugging

### 6. Wire platforms/react-native + migrate app (3-5 days)

**Work:**
- Update `platforms/react-native/index.ts`:
  - `configureWasm()` → registers expo-wallet-core
  - `configureMpc()` → registers expo-dkls native provider
  - Configure RN-specific polyfills (crypto.getRandomValues, fetch)
- Migrate vultiagent-app:
  - Replace `src/services/evmTx.ts`, `cosmosTx.ts`, `solanaTx.ts`, etc. with SDK `vault.prepareSendTx()` + `vault.sign()`
  - Replace `src/lib/chains.ts`, `src/lib/coins.ts` with SDK chain registry
  - Replace `src/services/auth/`, `src/services/relay.ts` with SDK signing services
  - Keep: all UI, Expo-specific features (biometrics, secure store, camera, speech), navigation, state, scheduling, agent chat
- Update vultiagent-app's `package.json` to depend on `@vultisig/sdk` (file reference to monorepo or published package)

**Verification — full end-to-end on Android emulator:**
- `npx expo run:android` with migrated app
- Use `agent-device` to:
  - Launch app, verify it loads
  - Import vault (using test .vult file)
  - Navigate to balances, verify chain balances display
  - Initiate a send, verify TX building succeeds
  - Execute MPC signing against production relay/VultiServer
  - Verify transaction broadcasts (or at least signing completes)
- Compare: same vault, same operation via CLI vs mobile app — results must match
- Run vultiagent-app's existing Detox E2E suite (may need Android config — currently iOS only)

### 7. CI pipeline + cleanup (1-2 days)

**Work:**
- GitHub Actions workflow: typecheck, lint, unit tests, SDK build (all platforms)
- Rust CI: `cargo test`, `wasm-pack build`, `cargo-ndk build`
- Document: how to add a new chain, how to update wallet-core version, how to rebuild Rust binaries
- Remove: all deleted reimplemented code from vultiagent-app, unused dependencies

**Verification:**
- CI green on all checks
- CLI tests pass in CI
- WASM builds reproducible in CI

## Testing Strategy Summary

| Layer | Method | Automated? |
|---|---|---|
| TypeScript correctness | `tsc --noEmit` across all packages | Yes — CI |
| SDK unit tests (336) | Jest with WASM providers | Yes — CI |
| CLI tests (172) | Jest with WASM providers | Yes — CI |
| DI verification | Mock wallet-core + mock MPC provider pass full test suite | Yes — CI |
| Cross-platform parity | Same operations through WASM vs mock, byte-for-byte diff | Yes — CI |
| Rust crates | `cargo test` | Yes — CI |
| WASM build verification | Diff against existing binaries | Yes — CI |
| Android native wallet-core | expo-wallet-core on emulator via `agent-device` | Yes — local |
| Android native MPC | expo-dkls (Rust) on emulator, real signing against prod | Yes — local |
| Full app E2E | Migrated vultiagent-app on emulator via `agent-device` | Yes — local |
| Real MPC signing | Test vault + production relay/VultiServer | Yes — local (same as CLI) |
| iOS native modules | Swift code written, structurally matches Kotlin | **No — needs Mac** |
| Server .so compatibility | Diff C headers against go-wrappers | Yes — CI |

**Key insight:** Everything except iOS can be tested autonomously. The CLI already hits production backends with the test vault — the mobile app will do the same thing through the same SDK, just with native providers instead of WASM.

## Open Questions

- **expo-wallet-core scope**: Minimal (~15 classes the SDK uses) vs full wallet-core API? Recommend minimal — less bridge code, add more later if needed.
- **Monorepo vs separate repo for vultiagent-app**: Recommend separate repo consuming published SDK — keeps the SDK focused and the app's Expo/RN concerns separate.
- **wallet-core version pinning**: Native binaries must match WASM npm version. Enforce via CI check that compares versions in expo-wallet-core podspec/gradle against `@trustwallet/wallet-core` in SDK package.json.
- **iOS CI**: Need a Mac runner (GitHub Actions has macOS runners) for iOS xcframework builds and expo-wallet-core iOS testing.

## Notes

**2026-03-30T04:24:20Z**

## Approved Design (2026-03-30)

### Architecture

One SDK monorepo (forked to premiumjibles) owns all business logic (TypeScript) and MPC cryptography (Rust source). Two native dependencies, both pluggable via existing DI:

| Platform | wallet-core | MPC |
|---|---|---|
| Node.js (CLI) | WASM | WASM |
| Browser / Electron | WASM | WASM |
| React Native | Native (expo-wallet-core) | Native (expo-dkls, Rust) |
| vultiserver | N/A | Rust .so via CGO |

### Key Decisions

1. Rust-only MPC, no Go fallback. Burn the boats — prove it fast on the fork.
2. Native wallet-core from day one via expo-wallet-core Expo module.
3. SDK is its own repo. Apps (vultiagent-app, vultiagent-cli, eventually vultiagent-windows) consume it as a dependency.
4. vultiagent-app strips all domain logic and replaces with SDK calls — becomes a thin UI/UX wrapper.

### What Gets Eliminated

- mobile-tss-lib Go repo (Rust-only MPC)
- go-wrappers build chain (consumes our .so + headers)
- sync-and-copy.ts (SDK owns core packages)
- ~10k LOC reimplemented logic in vultiagent-app
- Go runtime overhead in mobile binaries (~9.6 MB)

### Monorepo Structure

```
@vultisig/sdk (pnpm monorepo — premiumjibles fork)
├── rust/
│   ├── dkls/              # Rust DKLS source
│   ├── schnorr/           # Rust Schnorr source
│   └── Cargo.toml         # Workspace — WASM, Android, server targets
├── packages/
│   ├── core-chain/        # Chain implementations (owned)
│   ├── core-mpc/          # MPC orchestration (TS)
│   ├── core-config/       # Chain metadata
│   ├── lib-dkls/          # DKLS WASM output + TS types
│   ├── lib-schnorr/       # Schnorr WASM output + TS types
│   ├── lib-utils/         # Crypto helpers
│   ├── sdk/               # Main SDK — platform abstraction + vault API
│   ├── expo-wallet-core/  # Expo module: TrustWallet native binaries
│   └── expo-dkls/         # Expo module: Rust MPC native binaries
├── apps/
│   └── cli/               # vultiagent-cli (thin wrapper)
└── pnpm-workspace.yaml
```

### Tasks

1. Fork + pnpm migration + restructure — Fork to premiumjibles, convert to pnpm, delete sync-and-copy, promote synced packages to first-class workspace packages, update imports/builds. Verify: pnpm install, tsc, all 336+172 tests, all 5 platform bundles, CLI vault import against live backend. [no deps]

2. Pluggable MPC provider — Productionize MpcProvider + MpcKeygenProvider interfaces, move to packages/sdk/src/mpc/, register wasmMpcProvider in all platform entry points, write NativeMpcProvider adapter shape. Verify: all existing tests pass, new mock-provider unit test. [depends on 1]

3. expo-wallet-core — Create Expo module wrapping TrustWallet native binaries. Kotlin (Android): JNI → .aar, ~15 classes as JSI sync functions. Swift (iOS): same interface, structurally correct but untestable locally. TypeScript interface matching SDK's initWasm() duck type. Verify: cross-validate WASM vs native outputs on Android emulator, SDK DI works with native provider. [depends on 1]

4. Rust source + multi-target build — Bring dkls23-rs and multi-party-schnorr into rust/, set up Cargo workspace, build WASM (wasm-pack), Android native (cargo-ndk), server native (cargo build + cbindgen). Verify: cargo test, WASM functionally equivalent to existing binaries, Android .so for 3 architectures, server headers ABI-compatible. [depends on 1]

5. expo-dkls (Rust native) — Migrate expo-dkls from Go JNI/FFI to Rust JNI/FFI, link against Android .so from task 4, wire NativeMpcProvider adapter. Verify: MPC keysign on Android emulator against production relay/vultiserver using test vault, signature matches CLI output. [depends on 2, 4]

6. Wire react-native platform + migrate vultiagent-app — Update platforms/react-native to register expo-wallet-core and expo-dkls providers. In vultiagent-app: replace all chain logic, TX builders, signing orchestration, relay code with SDK imports. Keep UI, Expo-specific features, navigation, state. Verify: full E2E on Android emulator — vault import, balances, send, MPC signing. [depends on 3, 5]

7. CI + cleanup — GitHub Actions for typecheck, lint, tests, SDK builds, Rust CI. Remove deleted code from vultiagent-app. Document: adding chains, updating wallet-core, rebuilding Rust. [depends on 6]

Parallelism: Tasks 2, 3, and 4 are independent after task 1 — all three can run concurrently. Task 5 waits on 2+4. Task 6 waits on 3+5.

**2026-03-30T05:49:11Z**

All 7 tasks complete. Fork pushed to premiumjibles/vultisig-sdk branch feat/unified-sdk-restructure. Verified: 831 SDK tests, 343 rujira tests, 44 Rust tests, all 6 platform bundles, CLI build, typecheck, zero yarn refs remaining.
