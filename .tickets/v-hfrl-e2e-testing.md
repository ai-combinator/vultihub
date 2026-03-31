# E2E Native Testing Plan — Prove Unified SDK Works on React Native

## Goal

Prove that the unified SDK fork (`premiumjibles/vultisig-sdk` branch `feat/unified-sdk-restructure`) works end-to-end on React Native with **native providers** (not WASM). This means:

1. expo-wallet-core provides native wallet-core (address derivation, TX building) via TrustWallet AAR
2. expo-dkls provides native MPC signing via Rust `.so` files (DKLS + Schnorr)
3. A React Native app imports the SDK, loads a vault, fetches balances, and executes MPC signing — all through native code paths

## Current State

### What's been built (all in `/home/sean/Repos/vultisig/vultisig-sdk-fork`, branch `feat/unified-sdk-restructure`)

- **SDK monorepo** migrated to pnpm, all 831 unit tests + 343 rujira tests + 195/207 E2E tests pass
- **Pluggable MPC provider** — `MpcProvider` async interface + `configureMpc()` DI. `wasmMpcProvider` registered for Node/Browser/Electron. All callers use `await`.
- **expo-wallet-core** (`packages/expo-wallet-core/`) — Expo module with TypeScript interface, Kotlin skeleton wrapping TrustWallet AAR, Swift skeleton. **Never tested on device.**
- **expo-dkls** (`packages/expo-dkls/`) — Expo module with `NativeMpcProvider` adapter, Kotlin skeleton with JNI `external` declarations for Rust C FFI. **Never tested on device.**
- **Rust source** (`rust/dkls/`, `rust/schnorr/`) — 44 tests pass. WASM, Android `.so`, and server `.so` all build. Android `.so` files at `rust/dkls/target/aarch64-linux-android/release/` (and other archs).
- **React Native platform** (`packages/sdk/src/platforms/react-native/`) — Entry point that registers expo-wallet-core via `configureWasm()` and NativeMpcProvider via `configureMpc()`. Builds as `dist/index.react-native.js`.
- **CLI proven working** — vault import + balance fetch against production backends on the new SDK

### What's been proven on the emulator

- Android emulator boots (API 34, `test_api34_noplay` AVD, `google_apis` image)
- **SELinux fix required**: `sudo setsebool -P selinuxuser_execheap 1` (already applied)
- vultiagent-app (the OLD app, not using our SDK) builds, installs, imports vault, agent responds with portfolio
- `agent-device` works for screenshots, clicks, snapshots, typing

### Emulator details
- AVD: `test_api34_noplay` (API 34, google_apis, x86_64)
- Boot command: `/home/sean/Android/Sdk/emulator/emulator -avd test_api34_noplay -no-window -no-audio -gpu swiftshader_indirect -no-boot-anim -no-snapshot`
- ADB reverse: `adb reverse tcp:8081 tcp:8081`
- agent-device: `npx agent-device open` then `npx agent-device screenshot /tmp/foo.png`, `npx agent-device click @eN`, `npx agent-device snapshot`

### Test vault credentials
- Path: `/home/sean/Downloads/Test Vault-36f2-share1of2.vult`
- Password: `g3fXW8usFsp8Iu`
- Already pushed to emulator at `/sdcard/Download/`
- Known balances: SOL ~$3.76, RUNE ~$3.05, ETH ~$0.16, BTC ~$2.50, BNB ~$0.57 (total ~$6.97)

## What Needs To Be Done

### Level 1: SDK imports on RN (~30 min)

Create a minimal Expo test app at `/home/sean/Repos/vultisig/sdk-rn-test/`:

```bash
npx create-expo-app sdk-rn-test --template blank-typescript
cd sdk-rn-test
# Add SDK as file dependency
npm install ../vultisig-sdk-fork/packages/sdk
```

Simple App.tsx that imports `{ Chain }` from `@vultisig/sdk` and displays chain names. Build and run on emulator. This proves import resolution works.

### Level 2: expo-wallet-core on device (~2-4 hours)

The expo-wallet-core Kotlin module (`packages/expo-wallet-core/android/`) wraps TrustWallet's AAR (`com.trustwallet:wallet-core:4.2.6` from Maven). The JNI method signatures were GUESSED — they need to be validated and fixed against the actual AAR API.

**Key risk**: The Kotlin code calls methods like `wallet.core.jni.CoinType.BITCOIN` — the actual TrustWallet JNI class names and method signatures may differ. Debug via `adb logcat` for `NoSuchMethodError` / `ClassNotFoundException`.

Steps:
1. Add expo-wallet-core as a dependency in the test app
2. Import `createNativeWalletCore()` and call `CoinType` / `AnyAddress.isValid()`
3. Build, install, check `adb logcat` for JNI errors
4. Fix Kotlin method signatures iteratively until it works
5. Cross-validate: same address derivation via CLI (WASM) vs test app (native) must match

**Reference**: TrustWallet wallet-core JNI source at `https://github.com/nickatnight/trustwallet-wallet-core` or inspect the AAR directly after Gradle downloads it (at `~/.gradle/caches/`).

### Level 3: expo-dkls with Rust native MPC (~1-2 days)

This is the hardest part. The Rust `.so` files export C FFI functions, but Android JNI needs `JNIEXPORT` wrapper functions that match the Java package/class/method naming convention.

**The gap**: Our Kotlin declares `external fun dkls_keygen_setup(...)` but the Rust `.so` exports `dkls_keygen_setup` as a plain C symbol. JNI expects symbols named like `Java_expo_modules_dkls_ExpoDklsModule_dkls_1keygen_1setup`. A thin C/JNI glue layer is needed.

Steps:
1. **Write JNI glue** (`android/src/main/jni/dkls_jni.c`):
   - Include `<jni.h>` and the Rust C headers (`go-dkls.h`, `go-schnorr.h`)
   - For each `external fun` in Kotlin, write a `JNIEXPORT` function that calls the corresponding Rust C function
   - Compile with CMake/ndk-build and link against the Rust `.so`

2. **Package the `.so` files** into the Expo module:
   - Copy `rust/dkls/target/{arch}/release/libgodkls.so` into `packages/expo-dkls/android/src/main/jniLibs/{arch}/`
   - Same for `libgoschnorr.so`
   - Architectures needed: `arm64-v8a` (real devices), `x86_64` (emulator)

3. **Update build.gradle** to compile the JNI C code and link against the `.so` files

4. **Test MPC keysign on emulator**:
   - Import test vault in the test app
   - Trigger a signing operation (sign arbitrary bytes)
   - Verify signature matches what CLI produces for the same payload
   - This requires the relay server (production at `api.vultisig.com`) and VultiServer for FastVault 2-of-2 signing

5. **Cross-validate**: Same vault, same message, sign via CLI (WASM MPC) and via test app (native MPC) — signatures must match

### Alternative approach for Level 3

Instead of writing raw JNI glue, consider using **Rust UniFFI** which auto-generates JNI bindings:
- Add `uniffi` to the Rust crates
- Define the FFI interface in a `.udl` file
- UniFFI generates Kotlin bindings automatically
- This is cleaner but requires modifying the Rust workspace

## Verification Checklist

After all levels complete, run these checks:

```bash
# === TypeScript / SDK ===
cd /home/sean/Repos/vultisig/vultisig-sdk-fork
pnpm -w run typecheck                    # Must pass
pnpm --filter @vultisig/sdk test         # 831 tests
pnpm --filter @vultisig/rujira test      # 343 tests
pnpm --filter @vultisig/sdk build        # All 6 platform bundles
pnpm --filter @vultisig/cli build        # CLI binary

# === CLI E2E ===
node clients/cli/dist/index.js balance --output json  # Real balances from production

# === Rust ===
cd rust/dkls && cargo test               # 23 tests
cd rust/schnorr && cargo test            # 21 tests

# === Android emulator ===
# Boot emulator (if not running)
/home/sean/Android/Sdk/emulator/emulator -avd test_api34_noplay -no-window -no-audio -gpu swiftshader_indirect -no-boot-anim -no-snapshot &
# Wait for boot
adb wait-for-device shell getprop sys.boot_completed

# Build and install test app
cd /home/sean/Repos/vultisig/sdk-rn-test
npx expo run:android

# Verify with agent-device
npx agent-device open
npx agent-device screenshot /tmp/test.png
# Import vault, check balances, verify they match CLI output
# Sign bytes, verify signature matches CLI output
```

## Key Files

| File | Purpose |
|---|---|
| `packages/core/mpc/lib/mpcProvider.ts` | MpcProvider async interface |
| `packages/core/mpc/lib/wasmMpcProvider.ts` | WASM implementation |
| `packages/expo-dkls/src/nativeMpcProvider.ts` | Native implementation (adapter) |
| `packages/expo-dkls/android/.../ExpoDklsModule.kt` | Android Kotlin with JNI externals |
| `packages/expo-wallet-core/android/.../ExpoWalletCoreModule.kt` | Android Kotlin wrapping TrustWallet |
| `packages/sdk/src/platforms/react-native/index.ts` | RN platform entry point (wires both providers) |
| `rust/dkls/wrapper/go-dkls/include/go-dkls.h` | Rust C FFI header for DKLS |
| `rust/dkls/wrapper/go-schnorr/include/go-schnorr.h` | Rust C FFI header for Schnorr |
| `rust/dkls/target/x86_64-linux-android/release/libgodkls.so` | Android x86_64 .so (for emulator) |

## Known Issues

1. **Source-built Schnorr WASM breaks EdDSA signing** — The Rust source produces Schnorr WASM that fails `signBytes` for Solana/Sui E2E tests. Committed binaries work. Don't swap WASM yet.
2. **`KeyImportSession` → `KeyImporterSession` rename** in Rust source vs committed WASM. SDK imports `KeyImportSession`. Would break keygen if WASM swapped.
3. **`extractSetupMessageHash`** throws on native provider (not implemented). Will need a native FFI wrapper or the hash passed through from the caller.
4. **expo-wallet-core JNI signatures are guessed** — need validation against actual TrustWallet AAR.
5. **expo-dkls needs JNI C glue layer** — Rust `.so` exports plain C symbols, JNI needs `JNIEXPORT` wrappers.
