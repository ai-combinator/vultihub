---
id: v-hfrl-v3
status: open
deps: []
links: [v-hfrl, v-hfrl-v2, v-mehr]
created: 2026-03-31T14:00:00Z
type: task
priority: 1
assignee: unassigned
---
# Unified Vultisig SDK — Production Implementation

## Problem Statement

The Vultisig ecosystem has fragmented implementations across clients: vultiagent-app reimplements ~10k lines of chain/signing logic that exists in @vultisig/sdk, the MPC protocol has parallel Rust (WASM) and Go (native) implementations, and core packages are owned by vultisig-windows with a sync-and-copy pipeline to the SDK. "Solved" means one SDK monorepo that owns all business logic (TypeScript) and MPC cryptography (Rust source), compiling to every target. CLI, mobile app, and server are thin wrappers.

**Fork target:** `premiumjibles` GitHub (forking `vultisig/vultisig-sdk`)
**Spike repo:** `github.com/premiumjibles/vultisig-sdk` branch `feat/unified-sdk-restructure` (7 commits, `65af681a`)
**Spike test app:** `/home/sean/Repos/vultisig/sdk-rn-test/` (local only, not pushed)

## Prior Art — Spike Results (v-hfrl, v-hfrl-v2)

A spike on `premiumjibles/vultisig-sdk` branch `feat/unified-sdk-restructure` proved the architecture works. **The spike code is reference material, not production code** — it has memory leaks, broken key import, missing error handling, and zero tests. This ticket is a clean-slate implementation informed by everything the spike discovered.

**What the spike proved works:**

| Component | Evidence |
|---|---|
| pnpm monorepo | 831 SDK + 343 rujira + 44 Rust tests pass |
| MpcProvider DI interface | `configureMpc()` singleton, WasmMpcProvider + NativeMpcProvider swap cleanly |
| Rust → WASM (CLI) | CLI vault import + balance fetch against production |
| Rust → Android .so | `cargo ndk` produces working x86_64 + arm64-v8a binaries |
| JNI C glue pattern | `jbyteArray` → `go_slice` → C FFI call → `tss_buffer` → `jbyteArray` works |
| Expo module loading | `requireNativeModule('ExpoDkls')` resolves and executes |
| Real vault keyshare | ECDSA keyshare from encrypted .vult loads and deserializes through full JNI chain |
| Keygen lifecycle | setup → session → outbound message → free — all pass on emulator |
| Binary size | ~2.5 MB total (Rust .so + JNI bridge) vs ~22 MB (Go gomobile AAR) |

**What the spike found broken or missing (informing this ticket):**

| # | Finding | Impact |
|---|---|---|
| 1 | Rust .so exports plain C FFI, not JNI symbols — need ~700 line C glue layer | Adds a CMake build step, but the glue is mechanical boilerplate |
| 2 | `Option<&T>` in Rust FFI = pass `NULL` for None, not pointer to zero struct | One-line fix per optional param — but causes `LIB_UNKNOWN_ERROR` if wrong |
| 3 | CMake `IMPORTED_LOCATION` embeds absolute paths — need `IMPORTED_NO_SONAME TRUE` | Runtime `dlopen` failure without this |
| 4 | `go-dkls.h` and `go-schnorr.h` define identical types — compile as separate .c files | Linker error if combined in one translation unit |
| 5 | `schnorr_keyshare_free` missing from C header | Memory leak for Schnorr keyshares — needs cbindgen investigation |
| 6 | `System.loadLibrary` order: godkls → goschnorr → dkls_jni | Symbol resolution failure if reversed |
| 7 | iOS needs no JNI glue — Swift calls C directly (~200 lines vs ~700) | Positive finding — iOS is simpler |
| 8 | TrustWallet AAR on GitHub Packages, requires PAT with `read:packages` | Build config, not architecture issue |
| 9 | EdDSA keyshare (132 bytes) fails `schnorr_keyshare_from_bytes` with `SERIALIZATION_ERROR` | Needs format investigation before Schnorr signing works natively |
| 10 | Sign session validates setup message cryptographically — can't test without relay | Expected — full test needs VultiServer as second party |
| 11 | `extractSetupMessageHash()` not exposed through JNI — blocks signing flow | Must wrap `dkls_decode_message` / `schnorr_decode_message` |
| 12 | expo-wallet-core missing `PrivateKey.create()` and `Bech32.encode()` | 2 methods needed for 100% coverage (BlockAid + Cardano) |
| 13 | Spike key import JNI returns raw pointer instead of bytes — completely broken | Must be redesigned from scratch |

## Constraints

- Expo-specific APIs (secure storage, biometrics, camera) stay in the app, not the SDK
- vultiserver is Go — consumes MPC via CGO. Rust build must produce compatible `.so` + C headers
- No macOS available — iOS native modules can only be written and structurally tested; actual iOS verification requires a Mac. Android is fully testable via emulator.
- wallet-core native binary version must match WASM npm package version for behavioral parity
- TrustWallet AAR hosted on GitHub Packages — requires authentication (see Task 3a)

## Goal Architecture

### Two native dependencies, both pluggable via existing DI

| Platform | wallet-core | MPC (DKLS/Schnorr) |
|---|---|---|
| **Node.js (CLI)** | WASM (`@trustwallet/wallet-core`) | WASM (`@lib/dkls`, `@lib/schnorr`) |
| **Browser / Electron** | WASM | WASM |
| **React Native** | Native via `expo-wallet-core` | Native via `expo-dkls` (Rust → .so → JNI C glue → Kotlin) |
| **vultiserver** | N/A | Rust `.so` via Go CGO (existing) |

### DI mechanism (two independent singletons, both proven in spike)

**wallet-core:** `configureWasm(getter)` in `context/wasmRuntime.ts` — lazy-initialized, passed as function parameter to all 65 consuming files. Zero leaks.

**MPC:** `configureMpc(provider)` in `core/mpc/lib/mpcProvider.ts` — module-level singleton, configured synchronously at platform import.

```typescript
// MpcProvider interface (from spike — this is correct and should be kept)
interface MpcProvider {
  initialize(algo: SignatureAlgorithm): Promise<void>
  deserializeKeyshare(keyShareBase64: string, algo: SignatureAlgorithm): Promise<MpcKeyshareHandle>
  createSetupMessage(keyId, chainPath, messageHash, parties, algo): Promise<Uint8Array>
  extractSetupMessageHash(setupMsg, algo): Promise<Uint8Array | undefined>
  createSignSession(setupMessage, localPartyId, keyShare, algo): Promise<MpcSignSession>
}

interface MpcKeyshareHandle {
  publicKey(): Uint8Array    // Must be populated — spike returned empty
  keyId(): Uint8Array
  rootChainCode(): Uint8Array  // Must be populated — spike returned empty
  toBytes(): Uint8Array
}

interface MpcSignSession {
  outputMessage(): { body: Uint8Array; receivers: string[] } | undefined
  inputMessage(msg: Uint8Array): boolean
  finish(): Uint8Array
}
```

### Native pipeline — Android (proven architecture)

```
TypeScript (React Native)
    ↓ Expo module bridge
Kotlin (ExpoDklsModule.kt) — encoding, handle registry, error types
    ↓ external fun (JNI)
C glue layer (jni_bridge.c) — type marshalling jbyteArray ↔ go_slice/tss_buffer
    ↓ plain C function call
Rust .so (libgodkls.so / libgoschnorr.so) — MPC cryptography
```

**Library load order (mandatory, spike finding #6):**
```kotlin
System.loadLibrary("godkls")      // Rust DKLS C FFI
System.loadLibrary("goschnorr")   // Rust Schnorr C FFI
System.loadLibrary("dkls_jni")    // JNI bridge (depends on both above)
```

### Target monorepo structure

```
@vultisig/sdk (pnpm monorepo — premiumjibles fork)
├── rust/dkls/                    # Rust MPC source (vendored from dkls23-rs + multi-party-schnorr)
│   ├── wrapper/go-dkls/          # C FFI wrapper → libgodkls.so (cbindgen → go-dkls.h)
│   ├── wrapper/go-schnorr/       # C FFI wrapper → libgoschnorr.so (cbindgen → go-schnorr.h)
│   ├── wrapper/vs-wasm/          # WASM target → lib-dkls
│   ├── wrapper/vs-schnorr-wasm/  # WASM target → lib-schnorr
│   └── ci/                       # Build scripts: build-android-*.sh, build-darwin-*.sh
├── packages/
│   ├── core/chain/               # Chain implementations (owned, not synced)
│   ├── core/mpc/                 # MPC orchestration + MpcProvider interface
│   ├── core/config/              # Chain metadata
│   ├── lib/dkls/                 # DKLS WASM output
│   ├── lib/schnorr/              # Schnorr WASM output
│   ├── lib/utils/                # Crypto helpers
│   ├── sdk/                      # Main SDK + platform entry points
│   ├── rujira/                   # DEX integration
│   ├── expo-wallet-core/         # Native TrustWallet for React Native
│   └── expo-dkls/                # Native Rust MPC for React Native
└── clients/cli/                  # CLI (thin wrapper)
```

### What gets eliminated

- `mobile-tss-lib` Go repo — replaced by Rust → native directly
- `go-wrappers` build chain — consumes our `.so` + headers instead of owning Rust compilation
- `sync-and-copy.ts` — SDK owns core packages
- ~10k LOC reimplemented logic in vultiagent-app
- Go runtime overhead in mobile binaries (~9.6 MB → ~2.5 MB)
- Two independent implementations of the same MPC protocol

---

## Implementation Plan

### 1. Foundation — fork, restructure, verify

**Work:**
- Fork `vultisig/vultisig-sdk` to `premiumjibles`
- Convert from yarn to pnpm (see v-mehr)
- Delete `scripts/sync-and-copy.ts`, promote synced packages to first-class workspace packages
- Update all import paths and TypeScript path aliases
- Update Rollup build config for pnpm workspace resolution

**Verification:**
- `pnpm install` succeeds
- `tsc --noEmit` passes across all packages
- All 831 SDK unit tests + 343 rujira tests pass
- All platform bundles build (node, browser, electron, chrome-extension, react-native)
- CLI can import vault from `.vult` file and fetch balances against live backend

**Spike reference:** `premiumjibles/vultisig-sdk@315c493f` — clean, can be cherry-picked or reimplemented.

---

### 2. Pluggable MPC provider

**Work:**
- Add `MpcProvider`, `MpcKeyshareHandle`, `MpcSignSession` interfaces to `packages/core/mpc/lib/mpcProvider.ts`
- Add `configureMpc()` / `getMpcProvider()` singleton
- Create `WasmMpcProvider` in `packages/core/mpc/lib/wasmMpcProvider.ts` wrapping existing WASM DKLS/Schnorr
- Refactor `signSession.ts`, `keyshare.ts`, `initialize.ts` in core/mpc to use `getMpcProvider()` instead of direct WASM imports
- Register `WasmMpcProvider` in all existing platform entry points (node, browser, electron, chrome-extension)
- Write unit test with mock MpcProvider

**Verification:**
- All existing SDK + CLI tests still pass (regression)
- New unit test: mock provider returns expected shapes, keysign flow runs
- TypeCheck passes

**Spike reference:** `premiumjibles/vultisig-sdk@9702d170` — interface design is correct. The WasmMpcProvider is clean. Copy the interface and WasmMpcProvider directly.

---

### 3. expo-wallet-core — native wallet-core for React Native

**Work:**
- Create Expo module `packages/expo-wallet-core/`
- Kotlin (Android): wrap TrustWallet AAR via JNI. 36 functions covering: `CoinType`, `CoinTypeExt`, `PublicKey`, `PublicKeyType`, `HDWallet`, `PrivateKey`, `AnyAddress`, `TransactionCompiler`, `DataVector`, `HexCoding`, `Curve`, `Purpose`, `HDVersion`
- Swift (iOS): same 36 functions wrapping TrustWalletCore pod
- TypeScript `WalletCoreLike` adapter that matches the duck-type the SDK expects from `configureWasm()`
- Handle management for `HDWallet` (integer ID map, explicit `delete()`)
- All data as hex strings at the bridge boundary

**Spike reference:** The existing expo-wallet-core code in the spike is well-written. Reuse it, but add these fixes:

**Must add (spike finding #12):**
- `PrivateKey.create()` — used by BlockAid validation and Polkadot fee estimation
- `Bech32.encode(hrp, data)` — used by Cardano address derivation

**Must configure (spike finding #8):**
- TrustWallet AAR is on GitHub Packages, NOT Maven Central
- **Production approach:** Add to `android/build.gradle`:
  ```gradle
  maven {
      url = uri("https://maven.pkg.github.com/trustwallet/wallet-core")
      credentials {
          username = findProperty('gpr.user') ?: System.getenv('GITHUB_USER')
          password = findProperty('gpr.key') ?: System.getenv('GITHUB_TOKEN')
      }
  }
  ```
- Local dev: `~/.gradle/gradle.properties` with `gpr.user` + `gpr.key` (GitHub PAT with `read:packages`)
- CI: GitHub Actions secret
- **Fallback if auth is problematic:** Download AAR once, vendor as `android/libs/wallet-core.aar`, reference via `implementation files('libs/wallet-core.aar')`. Switch to Maven later.

**Must verify:** The JNI class names `wallet.core.jni.*` are from TrustWallet's documented Java API. They need device validation:
- Build, check `adb logcat` for `ClassNotFoundException` / `NoSuchMethodError`
- Fix iteratively — error messages tell you exactly what's wrong

**Verification:**
- Emulator: derive ETH/BTC/SOL addresses from test vault ECDSA pubkey `023eaa7d0a8170437db7e072db1a49b4b970ddbcaaa13e86d41f088412b81536f2`
- Cross-validate: addresses must match CLI output (`node clients/cli/dist/index.js balance --output json`)
- `anyAddressIsValid()` correctly validates known addresses

---

### 4. Rust source + multi-target build

**Work:**
- Vendor `vultisig/dkls23-rs` and `vultisig/multi-party-schnorr` into `rust/`
- Set up Cargo workspace
- Install tools: `cargo install wasm-pack cbindgen cargo-ndk`, add Android Rust targets
- WASM build: `wasm-pack build -t web` for vs-wasm and vs-schnorr-wasm
- Android native build: use existing `rust/dkls/ci/build-android-godkls.sh` and `build-android-goschnorr.sh`
- Copy built .so to `packages/expo-dkls/android/src/main/jniLibs/{arch}/`
- Copy generated headers to `packages/expo-dkls/android/src/main/cpp/`

**Build commands (from existing CI scripts):**
```bash
# Android .so (arm64 + x86_64)
cd rust/dkls
cargo ndk -p 21 -t aarch64-linux-android -t x86_64-linux-android \
  build -p go-dkls --release
cargo ndk -p 21 -t aarch64-linux-android -t x86_64-linux-android \
  build -p go-schnorr --release

# Headers are auto-generated by build.rs via cbindgen
# Output: rust/dkls/wrapper/go-dkls/include/go-dkls.h
#         rust/dkls/wrapper/go-schnorr/include/go-schnorr.h
```

**Verification:**
- `cd rust/dkls && cargo test` passes (44 tests)
- WASM builds produce functional output (run SDK tests with new WASM)
- Android .so files produced for arm64-v8a and x86_64
- Headers generated and match expected function signatures

**Spike reference:** `premiumjibles/vultisig-sdk@9702d170` vendored the Rust source (611 files, ~87K lines). This is correct — copy it.

---

### 5. expo-dkls — Rust native MPC for React Native (FRESH IMPLEMENTATION)

This is where the spike code should NOT be reused. Write from scratch using the spike as a reference for patterns and pitfalls.

**Architecture:**

```
packages/expo-dkls/
├── android/
│   ├── build.gradle                          # Expo module + CMake config
│   └── src/main/
│       ├── java/expo/modules/dkls/
│       │   └── ExpoDklsModule.kt             # Kotlin Expo module
│       ├── cpp/
│       │   ├── CMakeLists.txt                # Build JNI bridge, link Rust .so
│       │   ├── jni_bridge.c                  # UNIFIED JNI bridge (both DKLS + Schnorr)
│       │   ├── go-dkls.h                     # Copied from rust build
│       │   └── go-schnorr.h                  # Copied from rust build
│       └── jniLibs/{arm64-v8a,x86_64}/       # Pre-built Rust .so files
├── ios/
│   └── ExpoDklsModule.swift                  # Swift module (C direct via bridging header)
├── src/
│   ├── ExpoDklsModule.ts                     # TypeScript type declarations
│   ├── NativeMpcProvider.ts                  # MpcProvider implementation
│   └── index.ts
├── expo-module.config.json
└── package.json
```

#### 5a. JNI C bridge — write clean, unified implementation

**Key design decisions (informed by spike):**

1. **Single .c file, not two.** The spike split into `jni_dkls_bridge.c` and `jni_schnorr_bridge.c` because headers define identical types. Instead: create a thin shared header that forward-declares the common types, and use prefixed wrapper functions. Or keep two files but extract shared helpers into a `jni_common.h`.

2. **Every function follows the same pattern:**
   ```c
   JNIEXPORT jbyteArray JNICALL
   Java_expo_modules_dkls_ExpoDklsModule_nativeFunctionName(
       JNIEnv *env, jobject thiz, jbyteArray input1, jint input2) {
       // 1. Pin: jbyteArray → go_slice
       jbyte *pin1;
       struct go_slice s1 = pin_to_goslice(env, input1, &pin1);
       // 2. Call C FFI
       struct tss_buffer out = {0};
       enum lib_error err = dkls_function_name(&s1, (uint32_t)input2, &out);
       // 3. Unpin
       unpin(env, input1, pin1);
       // 4. Check error
       if (err != LIB_OK) { throw_error(env, "dkls_function_name", err); return NULL; }
       // 5. Marshal output: tss_buffer → jbyteArray, free tss_buffer
       return buf_to_jbytearray(env, &out);
   }
   ```

3. **Critical rules from spike:**
   - `Option<&T>` params: pass literal `NULL` for None, never `&zero_initialized_struct` (spike finding #2)
   - `#include <stdio.h>` required for `snprintf` (NDK clang treats as error without it)
   - Return `tss_buffer` data as `jbyteArray`, not as pointer-cast-to-jlong (spike finding #13)

4. **Key import functions — redesign the return:**
   The spike's `keyImportInitiatorNew` returned `[pointer_as_long, session_handle]` which is broken. Correct design: return a `jobjectArray` containing a `jbyteArray` (setup message bytes) and a `jint` (session handle). Or split into two calls: one that creates and returns the session handle, another that reads the setup message from the session.

5. **Functions to wrap (prioritized):**

   **P0 — Required for signing flow:**
   - Keygen: setupmsg_new, session_from_setup, session_output_message, session_message_receiver, session_input_message, session_finish, session_free (7 DKLS + 7 Schnorr)
   - Keyshare: from_bytes, to_bytes, public_key, key_id, chaincode, free (6 DKLS + 5 Schnorr — `schnorr_keyshare_free` may not exist)
   - Sign: setupmsg_new, session_from_setup, session_output_message, session_message_receiver, session_input_message, session_finish, session_free (7 DKLS + 7 Schnorr)
   - Decode: decode_message (1 DKLS + 1 Schnorr) — **required by `extractSetupMessageHash()` which spike left unimplemented**
   - Key import: key_import_initiator_new (1 DKLS + 1 Schnorr) — **with correct byte return, not pointer**
   - Total P0: ~50 functions

   **P1 — Production features:**
   - `dkls_keyshare_derive_child_public_key` — HD derivation
   - `decode_key_id`, `decode_session_id`, `decode_party_name` — message introspection

   **P2 — Advanced (defer):**
   - Key refresh, key migration, QC protocol, presign, key export

#### 5b. CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.22.1)
project("dkls_jni" C)

# Import pre-built Rust .so from jniLibs
# CRITICAL: IMPORTED_NO_SONAME TRUE prevents embedding absolute build paths (spike finding #3)
add_library(godkls SHARED IMPORTED)
set_target_properties(godkls PROPERTIES
    IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}/libgodkls.so
    IMPORTED_NO_SONAME TRUE)

add_library(goschnorr SHARED IMPORTED)
set_target_properties(goschnorr PROPERTIES
    IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}/libgoschnorr.so
    IMPORTED_NO_SONAME TRUE)

# JNI bridge
add_library(dkls_jni SHARED jni_bridge.c)
target_include_directories(dkls_jni PRIVATE ${CMAKE_SOURCE_DIR})
target_link_libraries(dkls_jni godkls goschnorr log)
```

#### 5c. Kotlin module — clean architecture

Design principles (fixing spike issues):
- **Sealed error types**: `DklsError(code: Int, function: String)` instead of raw RuntimeException
- **Resource management**: Session/Keyshare wrappers with cleanup tracking
- **No code duplication**: Parameterized helpers for DKLS vs Schnorr dispatch
- **Consistent encoding**: Document which functions return Base64 vs hex in the ModuleDefinition
- **Party ID encoding**: `partyIdsToBytes()` — null-separated UTF-8, handle edge cases (empty list, empty IDs)
- **Handle registry**: Single `mutableMapOf<Int, Any>` with type checking on retrieval

#### 5d. NativeMpcProvider — complete implementation

The spike's NativeMpcProvider had these gaps. The production version must:
- **Implement `extractSetupMessageHash()`** by wrapping `dkls_decode_message` / `schnorr_decode_message`
- **Populate `MpcKeyshareHandle.publicKey()` and `rootChainCode()`** by calling through to native `keysharePublicKey` and `keyshareChaincode`
- **Handle sync/async gap**: Either make `MpcSignSession` methods async in the interface, or implement sync wrappers using blocking native calls

#### 5e. Investigate EdDSA keyshare format (spike finding #9)

`schnorr_keyshare_from_bytes()` fails with `SERIALIZATION_ERROR` on the 132-byte EdDSA keyshare from the test vault. Before implementing Schnorr JNI wrappers:
1. Check how the WASM SDK deserializes EdDSA keyshares — does it pre-process?
2. Check how vultiagent-app's Go expo-dkls loads EdDSA keyshares
3. Compare the raw bytes at each stage

**Verification:**
- Emulator test app with comprehensive test suite (not the spike's App.tsx — write proper tests)
- All P0 JNI functions exercised
- Load real ECDSA keyshare from test vault → extract key ID → create sign setup message → create sign session (will get `SETUP_MESSAGE_VALIDATION` without relay — that's OK, it proves the JNI chain works)
- Load real EdDSA keyshare (if format investigation succeeds)
- Free all handles without crash
- Cross-validate: `createKeygenSetupMessage()` output should be parseable by CLI

---

### 6. Wire platforms/react-native + E2E validation

**Work:**
- Update `platforms/react-native/index.ts`:
  - `configureWasm(async () => createNativeWalletCore())` from expo-wallet-core
  - `configureMpc(new NativeMpcProvider())` from expo-dkls
  - RN-specific polyfills (crypto.getRandomValues, fetch)
- Build test app that imports `@vultisig/sdk` with react-native platform
- Run: vault import → address derivation → balance fetch

**Verification:**
- Same vault, same operations via CLI vs mobile — addresses must match
- MPC signing attempt starts (may not complete without relay — proves the provider chain works)

---

### 7. CI pipeline + cleanup

**Work:**
- GitHub Actions: typecheck, lint, unit tests, SDK build (all platforms)
- Rust CI: `cargo test`, `wasm-pack build`, `cargo-ndk build`
- Expo-wallet-core CI: needs GitHub PAT secret for TrustWallet AAR
- Document: adding chains, updating wallet-core version, rebuilding Rust binaries

**Verification:**
- CI green on all checks
- CLI tests pass in CI
- WASM builds reproducible

---

## Task Dependencies

```
Task 1 (foundation)
  ├── Task 2 (MPC provider)
  │     └── Task 5 (expo-dkls) ──┐
  ├── Task 3 (expo-wallet-core)  ├── Task 6 (wire RN + E2E) → Task 7 (CI)
  └── Task 4 (Rust build) ───────┘
```

Tasks 2, 3, 4 are independent after Task 1 — all three can run concurrently.

---

## Testing Strategy

| Layer | Method | Automated? |
|---|---|---|
| TypeScript correctness | `tsc --noEmit` across all packages | Yes — CI |
| SDK unit tests (831) | Vitest with WASM providers | Yes — CI |
| Rujira tests (343) | Vitest | Yes — CI |
| MPC provider DI | Mock provider unit test | Yes — CI |
| Rust crates | `cargo test` (44 tests) | Yes — CI |
| expo-dkls JNI bridge | Emulator test app exercising all P0 functions | Yes — local |
| expo-wallet-core | Emulator: address derivation cross-validated against CLI | Yes — local |
| Cross-platform parity | Same vault → same addresses from CLI and emulator | Yes — local |
| Full signing | Test vault + production relay/VultiServer (if relay integration done) | Manual |
| iOS | Swift structurally correct, untestable without Mac | **No — needs Mac** |

## Emulator Testing Guide

### Setup

```bash
# Start emulator
/home/sean/Android/Sdk/emulator/emulator -avd test_api34_noplay \
  -no-window -no-audio -gpu swiftshader_indirect -no-boot-anim -no-snapshot &
adb wait-for-device shell getprop sys.boot_completed  # wait for "1"
adb reverse tcp:8081 tcp:8081
```

### Test vault

- **Path:** `/home/sean/Downloads/Test Vault-36f2-share1of2.vult`
- **Password:** In `/home/sean/Repos/vultisig/vultisig-sdk-fork/packages/sdk/tests/e2e/.env` (TEST_VAULT_PASSWORD)
- **Type:** FastVault (2-of-2, encrypted, DKLS)
- **ECDSA pubkey:** `023eaa7d0a8170437db7e072db1a49b4b970ddbcaaa13e86d41f088412b81536f2`
- **EdDSA pubkey:** `380e7962878c39d84d5dd93bb4e10a5ea75d5c363ad338f5e2177c18b40aa9d0`
- **Signers:** `['2510ERA8BG-c18', 'Server-253']`
- **Known balances:** SOL ~$3.76, RUNE ~$3.05, ETH ~$0.16, BTC ~$2.50

### Build and test

```bash
cd /home/sean/Repos/vultisig/sdk-rn-test
npm install ../vultisig-sdk-fork/packages/expo-dkls
npx expo prebuild --platform android --clean
npx expo run:android
```

### Debug native crashes

```bash
adb logcat | grep -iE "(UnsatisfiedLink|NoSuchMethod|ClassNotFound|dkls_jni|lib_error|FATAL)"
unzip -l android/app/build/outputs/apk/debug/app-debug.apk | grep "\.so"
```

## Go-Based expo-dkls Reference (Production Comparison)

The existing Go-based module in `vultiagent-app/modules/expo-dkls/` is the production reference:

| Aspect | Go (production) | Rust (target) |
|---|---|---|
| Binary format | 2 AARs via gomobile (dkls 9.3MB + schnorr 1.8MB) | 2 .so via cargo-ndk (~1.3MB each) + C glue |
| JNI layer | SWIG-generated (`com.silencelaboratories.godkls.*`) | Manual C bridge (`Java_expo_modules_dkls_ExpoDklsModule_*`) |
| Data encoding | Base64 + hex at boundary, null-separated party IDs | Same |
| Handle management | Kotlin `mutableMapOf<Int, Handle>` | Same pattern |
| TypeScript API | 40 async/sync functions | Same signatures |
| Total size | ~22 MB (includes Go runtime) | ~2.5 MB |

The Go module's `fastVaultSign.ts` in vultiagent-app shows the complete signing flow through expo-dkls — use as reference for what the MPC message loop looks like from the app side.

## Open Questions

1. **EdDSA keyshare format** — Why does `schnorr_keyshare_from_bytes` fail on the vault's EdDSA keyshare? Must investigate before Schnorr signing works natively.
2. **`extractSetupMessageHash` in signing flow** — Is this called in FastVault (2-of-2) or only SecureVault? Determines P0 vs P1 priority.
3. **iOS timeline** — Swift code can be written structurally. When do we need Mac CI for actual validation?
4. **wallet-core version** — Target 4.2.6 (spike) or 4.6.0 (latest)? Must match WASM npm package.
5. **Key import usage** — Is `dkls_key_import_initiator_new` used in the current app's seed phrase import? If not, can be P2.
