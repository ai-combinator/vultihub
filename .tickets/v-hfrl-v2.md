---
id: v-hfrl-v2
status: closed
deps: []
links: [v-hfrl, v-mehr]
created: 2026-03-31T13:00:00Z
type: task
priority: 1
assignee: unassigned
---
# Unified Vultisig SDK — Production Implementation (Post-Spike)

## Context

This ticket replaces v-hfrl. The original ticket's 7 tasks were implemented as a spike on `premiumjibles/vultisig-sdk` branch `feat/unified-sdk-restructure`. The spike proved the architecture works — same Rust MPC source compiles to WASM (CLI) and native Android .so (React Native). This ticket captures everything learned and provides implementation-ready instructions for building the production version.

**What's proven (spike results, 2026-03-31):**

| Component | Evidence | Status |
|---|---|---|
| pnpm monorepo | 831 SDK + 343 rujira + 44 Rust tests pass | Done |
| Pluggable MPC provider | `MpcProvider` interface + `configureMpc()` DI | Done |
| Rust → WASM (CLI) | CLI vault import + balance fetch against production | Done |
| Rust → Android .so | cargo-ndk x86_64 + arm64-v8a builds | Done |
| JNI C glue layer | 700 lines, 42 functions, tested on emulator | Done |
| Expo module pattern | `requireNativeModule('ExpoDkls')` loads + executes | Done |
| Real vault keyshare loading | ECDSA keyshare from encrypted .vult loads via JNI | Done |
| Keygen session lifecycle | setup → session → outbound message → free (all pass) | Done |
| NativeMpcProvider adapter | TypeScript ↔ Kotlin ↔ JNI ↔ Rust C FFI chain works | Done |
| expo-wallet-core module | Kotlin + Swift + TS adapter written, untested on device | Written, unvalidated |

**Spike repo:** `premiumjibles/vultisig-sdk` branch `feat/unified-sdk-restructure`
**Test app:** `/home/sean/Repos/vultisig/sdk-rn-test/` (Expo 54, tested on `test_api34_noplay` emulator)

---

## Architecture (Validated)

### Two native dependencies, both pluggable via existing DI

| Platform | wallet-core | MPC (DKLS/Schnorr) |
|---|---|---|
| **Node.js (CLI)** | WASM (`@trustwallet/wallet-core`) | WASM (`@lib/dkls`, `@lib/schnorr`) |
| **Browser / Electron** | WASM | WASM |
| **React Native** | Native via `expo-wallet-core` | Native via `expo-dkls` (Rust → .so → JNI C glue → Kotlin) |
| **vultiserver** | N/A | Rust `.so` via Go CGO (existing) |

### DI mechanism (two independent singletons)

**wallet-core:** `configureWasm(getter)` → `getWalletCore()` → passed as parameter to every function
- All 65 files using wallet-core receive it as a function parameter — zero leaks
- React Native: `createNativeWalletCore()` returns a `WalletCoreLike` duck-type adapter

**MPC:** `configureMpc(provider)` → `getMpcProvider()` → global singleton
- Each platform entry point calls `configureMpc()` synchronously on import
- React Native: `new NativeMpcProvider()` wraps expo-dkls handle API

### Native pipeline (Android)

```
TypeScript (React Native)
    ↓ requireNativeModule('ExpoDkls')
Kotlin (ExpoDklsModule.kt) — Expo module, Base64/hex encoding, handle management
    ↓ external fun (JNI)
C glue layer (jni_dkls_bridge.c / jni_schnorr_bridge.c) — type marshalling
    ↓ plain C function call
Rust .so (libgodkls.so / libgoschnorr.so) — MPC cryptography
```

**Library load order (mandatory):**
```kotlin
System.loadLibrary("godkls")      // Rust DKLS C FFI
System.loadLibrary("goschnorr")   // Rust Schnorr C FFI
System.loadLibrary("dkls_jni")    // JNI bridge (depends on both above)
```

### Native pipeline (iOS — expected, not yet tested)

```
TypeScript (React Native)
    ↓ requireNativeModule('ExpoDkls')
Swift (ExpoDklsModule.swift) — Expo module, bridging header
    ↓ direct C function call (no JNI glue needed)
Rust static library (.a) — MPC cryptography
```

iOS is architecturally simpler (~200 lines Swift vs ~700 lines C for Android). Requires macOS to build/test.

---

## Monorepo Structure (Implemented)

```
@vultisig/sdk (pnpm monorepo)
├── rust/dkls/                    # Rust MPC source (DKLS + Schnorr)
│   ├── wrapper/go-dkls/          # C FFI wrapper → libgodkls.so
│   ├── wrapper/go-schnorr/       # C FFI wrapper → libgoschnorr.so
│   ├── wrapper/vs-wasm/          # WASM target → lib-dkls
│   ├── wrapper/vs-schnorr-wasm/  # WASM target → lib-schnorr
│   └── ci/                       # Build scripts (Android, Darwin)
├── packages/
│   ├── core/chain/               # Chain implementations (owned)
│   ├── core/mpc/                 # MPC orchestration + provider interface
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

---

## Implementation Tasks

### Task 1: Foundation (DONE — spike complete)

Fork, pnpm migration, restructure, pluggable MPC provider. All verified: 831 SDK + 343 rujira + 44 Rust tests pass, all platform bundles build.

**No remaining work.** Spike branch has all of this.

---

### Task 2: expo-dkls — Rust native MPC for Android (SPIKE DONE, needs production hardening)

**What the spike proved:**
- 42 JNI functions compile, load, and execute on Android emulator
- Real vault ECDSA keyshare loads and deserializes
- Keygen session lifecycle works end-to-end
- 9 of 11 emulator tests pass

**What needs production hardening:**

#### 2a. Fix key import memory leak

The `nativeDklsKeyImportInitiatorNew` and `nativeSchnorrKeyImportInitiatorNew` JNI functions have a critical bug. They return `setup_msg.ptr` cast to `jlong` — a raw pointer value. The Kotlin side wraps this in `ByteBuffer.putLong()` which creates an 8-byte buffer of the pointer address, NOT the actual setup message bytes. The Rust-allocated buffer is never freed.

**Fix:** Change the JNI function to return the setup message bytes as a `jbyteArray` alongside the session handle. Return a `jobjectArray` or use two separate output parameters. The Kotlin side should Base64-encode the actual bytes.

**Workaround from spike:** These functions are not used in the current codebase. The signing flow uses `createSignSetupMessage` (which works correctly) not `keyImportInitiatorNew`.

#### 2b. Add missing JNI wrappers for production features

The spike wraps 42 of 85 available C FFI functions (49% coverage). The core keygen/sign flow works, but production needs:

| Function group | Why needed | Priority |
|---|---|---|
| `dkls_decode_message` / `schnorr_decode_message` | Required by `extractSetupMessageHash()` in NativeMpcProvider — **currently throws at runtime** | **P0 — blocks signing flow** |
| `dkls_keyshare_derive_child_public_key` | HD key derivation for multi-account support | P1 |
| `schnorr_keyshare_chaincode` | EdDSA chaincode access (exists in header, not wrapped) | P1 |
| `dkls_key_import_initiator_new` (fix) | Seed phrase → DKLS import (exists but broken, see 2a) | P1 |
| `dkls_key_refresh_*` / `schnorr_key_refresh_*` | Key resharing for adding/removing devices | P2 |
| `dkls_qc_*` / `schnorr_qc_*` (7+7 functions) | Quality check protocol | P2 |
| `dkls_key_export_*` / `schnorr_key_export_*` | Key export for backup/migration | P2 |
| `dkls_presign_*` | Pre-signing for latency optimization | P3 |

**Pattern for adding new JNI wrappers** (mechanical, ~15 lines each):
```c
// 1. In jni_dkls_bridge.c:
JNIEXPORT jbyteArray JNICALL
Java_expo_modules_dkls_ExpoDklsModule_nativeNewFunctionName(
    JNIEnv *env, jobject thiz, jbyteArray input) {
    jbyte *pin;
    struct go_slice s = pin_to_goslice(env, input, &pin);
    struct tss_buffer out = {0};
    enum lib_error err = dkls_new_function(&s, &out);
    unpin(env, input, pin);
    if (err != LIB_OK) { throw_dkls_error(env, "dkls_new_function", err); return NULL; }
    return buf_to_jbytearray(env, &out);
}

// 2. In ExpoDklsModule.kt — add external fun + AsyncFunction wrapper
// 3. In ExpoDklsModule.ts — add TypeScript type declaration
```

#### 2c. Fix `schnorr_keyshare_free` no-op

The `go-schnorr.h` header doesn't export `schnorr_keyshare_free()`. The JNI wrapper is currently a no-op. Investigate whether:
- The function was omitted from cbindgen (check Rust source)
- Schnorr keyshares have a different lifetime model
- It needs to be added to the Rust FFI

#### 2d. Investigate EdDSA keyshare serialization error

`schnorr_keyshare_from_bytes()` returns `LIB_SERIALIZATION_ERROR` when loading the EdDSA keyshare from the test vault. The EdDSA keyshare is 132 bytes vs 65,970 bytes for ECDSA. Investigate:
- Whether the vault stores EdDSA keyshares in a different format
- Whether the WASM SDK pre-processes the keyshare before passing to the Schnorr library
- Compare how vultiagent-app's Go-based expo-dkls loads EdDSA keyshares

#### 2e. NativeMpcProvider sync/async gap

The `MpcSignSession` interface defines sync methods (`outputMessage()`, `inputMessage()`, `finish()`). The `NativeSignSession` only implements async variants and throws on sync calls. The core-mpc keysign loop may call the sync versions.

**Fix:** Either make the keysign loop use async consistently, or implement sync wrappers in NativeSignSession.

#### 2f. NativeKeyshareHandle returns empty publicKey/chainCode

`NativeKeyshareHandle.publicKey()` and `rootChainCode()` return empty `Uint8Array(0)`. These should call through to the native `nativeDklsKeysharePublicKey` and `nativeDklsKeyshareChaincode` functions which exist and work.

---

### Task 3: expo-wallet-core — Native TrustWallet for Android

**What's written:** Complete Kotlin module (337 lines), Swift module (364 lines), TypeScript adapter (312 lines), 36 native functions.

**What's NOT validated:** Nothing has been tested on a device. The JNI class names (`wallet.core.jni.*`) are based on TrustWallet's documented API but unverified against the actual AAR.

#### 3a. Resolve TrustWallet AAR dependency

The AAR (`com.trustwallet:wallet-core`) is hosted on **GitHub Packages**, not Maven Central. Requires authentication.

**Production approach:**
```gradle
// In android/build.gradle (allprojects.repositories)
maven {
    url = uri("https://maven.pkg.github.com/trustwallet/wallet-core")
    credentials {
        username = findProperty('gpr.user') ?: System.getenv('GITHUB_USER')
        password = findProperty('gpr.key') ?: System.getenv('GITHUB_TOKEN')
    }
}
```

**Local development:** Add to `~/.gradle/gradle.properties` (global, one-time):
```
gpr.user=<github-username>
gpr.key=<github-pat-with-read:packages>
```

**CI/CD:** Store PAT as GitHub Actions secret, inject as env var.

**Quick unblock alternative:** Download the AAR once, vendor it in `android/libs/wallet-core.aar`, reference as `implementation files('libs/wallet-core.aar')`. Switch to Maven dependency later.

**Version:** The module references 4.2.6 but latest is 4.6.0. Update to latest before validating.

#### 3b. Validate JNI class names on device

Once the AAR resolves, build and check `adb logcat` for:
- `ClassNotFoundException` → wrong package name (e.g., `wallet.core.jni.CoinType` might be `com.trustwallet.core.CoinType`)
- `NoSuchMethodError` → wrong method signature
- `UnsatisfiedLinkError` → native library not loaded

Fix iteratively. The Go-based expo-dkls uses `com.silencelaboratories.godkls.*` — TrustWallet will use its own namespace.

#### 3c. Add two missing wallet-core methods

The SDK uses two wallet-core features not in the current adapter:

| Method | Used by | Impact |
|---|---|---|
| `PrivateKey.create()` | BlockAid validation (dummy signatures for fee estimation), Polkadot fee estimation | Medium — these features fail silently on RN |
| `Bech32.encode(hrp, data)` | Cardano address derivation | Low — only Cardano affected |

**Fix:** Add both to `ExpoWalletCoreModule.kt`, `ExpoWalletCoreModule.swift`, `ExpoWalletCoreModule.ts`, and `walletCore.ts`.

#### 3d. Cross-validate addresses

Once wallet-core works on device:
1. Derive ETH/BTC/SOL addresses from the test vault's ECDSA public key (`023eaa7d0a8170437db7e072db1a49b4b970ddbcaaa13e86d41f088412b81536f2`)
2. Compare against CLI output: `node clients/cli/dist/index.js balance --output json`
3. They MUST match — same vault, same public key, different wallet-core implementations

---

### Task 4: Rust build pipeline automation

**What exists:**
- `rust/dkls/ci/build-android-godkls.sh` and `build-android-goschnorr.sh` — shell scripts using `cargo ndk`
- `rust/dkls/ci/build-darwin-*.sh` — iOS/macOS build scripts
- `cbindgen.toml` + `build.rs` in each wrapper crate — auto-generates C headers
- Pre-built .so files already in `packages/expo-dkls/android/src/main/jniLibs/`

**What's missing:**
- No integration between Gradle and Rust builds (manual `cp` of .so files)
- No unified "build all Android targets" command
- No automated header copying from `rust/dkls/wrapper/*/include/` to `packages/expo-dkls/android/src/main/cpp/`
- No binary versioning or integrity checking

**Recommended approach:**

```bash
# Single script: scripts/build-android-native.sh
#!/bin/bash
set -e

# Build Rust .so for Android
cd rust/dkls
cargo ndk -p 21 -t aarch64-linux-android -t x86_64-linux-android \
  -o ../../packages/expo-dkls/android/src/main/jniLibs \
  build -p go-dkls --release
cargo ndk -p 21 -t aarch64-linux-android -t x86_64-linux-android \
  -o ../../packages/expo-dkls/android/src/main/jniLibs \
  build -p go-schnorr --release

# Copy headers
cp wrapper/go-dkls/include/go-dkls.h ../../packages/expo-dkls/android/src/main/cpp/
cp wrapper/go-schnorr/include/go-schnorr.h ../../packages/expo-dkls/android/src/main/cpp/

echo "Android native build complete"
```

Don't integrate into Gradle — pre-built binaries committed to the repo (or fetched from CI artifacts) is simpler and faster. Most developers won't modify Rust code.

---

### Task 5: Wire platforms/react-native + validate E2E

**What needs to happen:**
1. `packages/sdk/src/platforms/react-native/index.ts` calls:
   - `configureWasm(async () => createNativeWalletCore())` — from expo-wallet-core
   - `configureMpc(new NativeMpcProvider())` — from expo-dkls
2. Build a test app that imports `@vultisig/sdk` with the react-native platform
3. Run vault import → address derivation → balance fetch → signing attempt

**This is the final integration test.** If addresses match CLI and signing starts (even if it can't complete without a relay), the unified SDK is production-ready.

---

### Task 6: Migrate vultiagent-app to use SDK

Not part of this ticket. This is a separate effort that depends on Tasks 2-5 being complete.

---

### Task 7: CI pipeline

Not part of this ticket. Standard GitHub Actions setup after implementation stabilizes.

---

## Technical Deviations from Original Ticket (Complete List)

These are facts discovered during the spike that differ from the original ticket's assumptions. An implementing agent must know all of these.

### 1. JNI C glue layer is required (~700 lines)

**Original assumption:** "Replace Go JNI calls with Rust JNI calls" — implying Rust .so has JNI-compatible exports.

**Reality:** Rust .so exports plain C FFI (`dkls_keygen_setupmsg_new` taking `go_slice*`, `tss_buffer*`). Android JNI expects `JNIEXPORT` functions with JNI type signatures. A C glue layer bridges the two.

**Impact:** Add CMake build step to Gradle. The glue is mechanical boilerplate — each function follows the same pattern: extract `jbyteArray` → `go_slice`, call C FFI, convert `tss_buffer` → `jbyteArray`.

**Files:**
- `packages/expo-dkls/android/src/main/cpp/jni_dkls_bridge.c` (~400 lines, 21 DKLS functions)
- `packages/expo-dkls/android/src/main/cpp/jni_schnorr_bridge.c` (~300 lines, 21 Schnorr functions)
- `packages/expo-dkls/android/src/main/cpp/CMakeLists.txt`

### 2. Rust `Option<&T>` maps to NULL pointer, not empty struct

When the C FFI takes `Option<&go_slice>`, pass literal `NULL` for `None`. Passing a pointer to a zero-initialized struct gives `Some({ptr: null})` not `None`, causing `LIB_UNKNOWN_ERROR`.

**Rule:** Every JNI wrapper with optional parameters must use `NULL` for the absent case.

### 3. CMake IMPORTED_NO_SONAME required

Without `IMPORTED_NO_SONAME TRUE`, CMake embeds the build machine's absolute path into `DT_NEEDED` tags. At runtime on Android, `dlopen` fails because the path doesn't exist.

### 4. Dual C headers define identical types

`go-dkls.h` and `go-schnorr.h` both define `tss_buffer`, `go_slice`, `Handle`, `lib_error`. They can't be included in the same translation unit. Each bridge .c file includes only its own header.

### 5. `schnorr_keyshare_free` missing from header

Not exported from `go-schnorr.h`. JNI wrapper is a no-op. Potential memory leak.

### 6. System.loadLibrary order matters

Must load in dependency order: `godkls` → `goschnorr` → `dkls_jni`. Reverse order causes symbol resolution failure.

### 7. iOS is simpler than Android

Swift calls C directly via bridging headers — no JNI glue needed. ~200 lines Swift vs ~700 lines C. Requires macOS.

### 8. TrustWallet AAR requires GitHub Packages authentication

`com.trustwallet:wallet-core` is NOT on Maven Central. Hosted at `https://maven.pkg.github.com/trustwallet/wallet-core`. Requires GitHub PAT with `read:packages` scope.

### 9. EdDSA keyshare format may differ from ECDSA

`schnorr_keyshare_from_bytes()` returns `LIB_SERIALIZATION_ERROR` on the 132-byte EdDSA keyshare from the test vault. The ECDSA keyshare (65,970 bytes) loads fine. Needs investigation.

### 10. Sign session creation validates cryptographically

`dkls_sign_session_from_setup()` doesn't just parse the setup message — it validates it against the keyshare. A setup message with test parameters fails with `LIB_SETUP_MESSAGE_VALIDATION`. Full testing requires a real second party (VultiServer via relay).

### 11. Go-based expo-dkls uses SWIG wrapper pattern

The existing production module loads `libgodklsswig.so` (30KB SWIG wrapper) + `libgodkls.so` (11MB Go+Rust). The Rust replacement eliminates the Go layer entirely: `libgodkls.so` (1.3MB Rust) + `libdkls_jni.so` (~100KB C glue). Total: ~2.5MB vs ~22MB.

### 12. `extractSetupMessageHash()` not implemented in NativeMpcProvider

This method throws at runtime. It requires wrapping `dkls_decode_message` / `schnorr_decode_message` C FFI functions which exist but aren't exposed through JNI yet. **This blocks the signing flow on React Native.**

### 13. expo-wallet-core missing two methods

`PrivateKey.create()` (BlockAid/Polkadot fee estimation) and `Bech32.encode()` (Cardano addresses) are not in the WalletCoreLike adapter. 96% coverage — these are the remaining 4%.

---

## Comparison with Go-Based expo-dkls (Production Reference)

The existing Go-based module in `vultiagent-app/modules/expo-dkls/` is the production reference. Key differences:

| Aspect | Go (production) | Rust (spike) |
|---|---|---|
| Native libraries | 2 AARs: dkls-release.aar (9.3MB) + goschnorr-release.aar (1.8MB) | 2 .so files: libgodkls.so (1.3MB) + libgoschnorr.so (1.3MB) |
| JNI binding | SWIG-generated (`com.silencelaboratories.godkls.*`) | Manual C glue (`Java_expo_modules_dkls_ExpoDklsModule_*`) |
| Handle management | Kotlin-side `mutableMapOf<Int, Handle>` | Same pattern |
| Data encoding | Base64 + hex at boundary | Same |
| Party ID format | Null-separated UTF-8 bytes | Same |
| Total binary size | ~22 MB (includes Go runtime) | ~2.5 MB |
| TypeScript API | Identical function signatures | Identical |

The Rust version is a drop-in replacement at the TypeScript level. The Kotlin implementation is different (no SWIG, manual JNI) but exposes the same Expo module interface.

---

## Emulator Testing Guide

### Prerequisites

```bash
# Android emulator
/home/sean/Android/Sdk/emulator/emulator -avd test_api34_noplay \
  -no-window -no-audio -gpu swiftshader_indirect -no-boot-anim -no-snapshot &
adb wait-for-device shell getprop sys.boot_completed  # wait for "1"
adb reverse tcp:8081 tcp:8081

# Test vault
# Path: /home/sean/Downloads/Test Vault-36f2-share1of2.vult
# Password: (in /home/sean/Repos/vultisig/vultisig-sdk-fork/packages/sdk/tests/e2e/.env)
# Type: FastVault (2-of-2), DKLS, encrypted
# ECDSA pubkey: 023eaa7d0a8170437db7e072db1a49b4b970ddbcaaa13e86d41f088412b81536f2
# EdDSA pubkey: 380e7962878c39d84d5dd93bb4e10a5ea75d5c363ad338f5e2177c18b40aa9d0
# Signers: ['2510ERA8BG-c18', 'Server-253']
# Known balances: SOL ~$3.76, RUNE ~$3.05, ETH ~$0.16, BTC ~$2.50
```

### Build and test expo-dkls

```bash
cd /home/sean/Repos/vultisig/sdk-rn-test
npm install ../vultisig-sdk-fork/packages/expo-dkls
npx expo prebuild --platform android --clean
npx expo run:android
# Tap "Run All Tests" button
# adb exec-out screencap -p > /tmp/screen.png  # capture results
```

### Debug native crashes

```bash
adb logcat | grep -iE "(UnsatisfiedLink|NoSuchMethod|ClassNotFound|dkls_jni|lib_error|FATAL)"
adb logcat -d | grep -A20 "FATAL EXCEPTION"
unzip -l android/app/build/outputs/apk/debug/app-debug.apk | grep "\.so"
```

### Cross-validate with CLI

```bash
cd /home/sean/Repos/vultisig/vultisig-sdk-fork
pnpm --filter @vultisig/sdk build:fast
pnpm --filter @vultisig/cli build
node clients/cli/dist/index.js balance --output json
# Compare addresses with emulator output
```

---

## Open Questions

1. **EdDSA keyshare format** — Why does `schnorr_keyshare_from_bytes` fail? Is the vault format different from what the raw FFI expects? Compare WASM vs native deserialization paths.

2. **`extractSetupMessageHash` priority** — Is this called in the FastVault signing path or only SecureVault? If only SecureVault, it can be deferred.

3. **iOS timeline** — Requires macOS. The Swift module is written but untested. When do we need iOS validation?

4. **wallet-core version** — Should we target 4.2.6 (matches spike) or 4.6.0 (latest)? Version must match WASM npm package for behavioral parity.

5. **Key import flow** — Is `dkls_key_import_initiator_new` used in the current app? If not, the broken JNI wrapper can be deferred.
