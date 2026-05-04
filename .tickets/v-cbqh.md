---
id: v-cbqh
status: closed
deps: []
links: []
created: 2026-04-14T04:34:16Z
type: bug
priority: 2
assignee: Jibles
---
# @vultisig/mpc-native@0.1.2 ships mismatched Kotlin + AAR bindings

## Objective
The `@vultisig/mpc-native@0.1.2` npm package cannot compile against its own bundled AARs. Every fresh install hits a wall of `Unresolved reference 'godkls'` / `Unresolved reference 'goschnorr'` Kotlin errors. This blocks `npm run android` for any consumer that doesn't already have a working local link to an unpublished SDK state, and silently breaks the signing path in vultiagent-app's recently-merged SDK migration.

## Context & Findings

### The mismatch
- **Kotlin source** in `node_modules/@vultisig/mpc-native/android/src/main/java/expo/modules/mpcnative/ExpoMpcNativeModule.kt` (and the same file in `vultisig-sdk/packages/mpc-native/` HEAD) imports:
  ```kotlin
  import godkls.Godkls
  import goschnorr.Goschnorr
  ```
  and calls ~64 methods in the shape `Godkls.dklsKeygenSetupmsgNew(...)`, `Godkls.dklsSignSessionInputMessage(...)` etc. (camelCase method names on a PascalCase class at the package root).

- **Bundled AARs** (`node_modules/@vultisig/mpc-native/android/libs/{dkls,goschnorr}-release.aar`, also the same AARs vendored at `vultisig-sdk/packages/mpc-native/android/libs/`) expose:
  ```
  com.silencelaboratories.godkls.godkls.dkls_keygen_setupmsg_new(...)
  com.silencelaboratories.goschnorr.goschnorr.schnorr_keygen_setup(...)
  ```
  (lowercase class name inside a `com.silencelaboratories` package, snake_case method names). Verified via `javap -public` on the extracted `classes.jar`.

- No version of the AARs in git history (back to `db62f09f` "feat: pluggable MpcEngine architecture for React Native support") ever had the `godkls.Godkls` flat namespace. The Kotlin source was born importing symbols that the shipped binaries never provided.

### How this made it to npm
- The AARs are SWIG-generated (per `packages/mpc-native/android/libs/README.md` — built via the private `dkls23-rs` + public `dkls-android` pipeline, SWIG + CMake).
- The Kotlin source is shaped like gomobile `bind` output — different generator, incompatible naming.
- Whoever tested locally must have had an unpublished, gomobile-style AAR set linked in. The checked-in + published pipeline only produces SWIG AARs, so the `npm publish` ships files that cannot compile together.
- Published 2026-04-13 by rcoderdev. Only 0.1.1 and 0.1.2 exist on npm; both are broken this way.

### Secondary issues uncovered on the same build
Not blockers for this ticket but worth bundling into the fix:
- `@vultisig/walletcore-native@0.1.1` `ExpoWalletCoreModule.kt` is missing `import wallet.core.java.AnySigner` (only wildcard-imports `wallet.core.jni.*`, but AnySigner lives under `wallet.core.java`).
- Same file calls `AnySigner.plan(byte[], CoinType)` — no such overload in wallet-core 4.3.22. Intended call is `AnySigner.nativePlan(byte[], int)`.
- Same file calls `TONAddressConverter.toUserFriendly(String)` — 4.3.22 signature is `toUserFriendly(String, boolean, boolean)` (bounceable, testnet).
- `anyAddressIsValidSS58` throws from a `Function` lambda without a concrete return type, hitting the "Cannot use 'Nothing' as reified type parameter" error.

### Downstream impact
- vultiagent-app `main`'s package.json pins `^0.1.2` for both `@vultisig/mpc-native` and `@vultisig/walletcore-native`. Any clean clone + `npm run android` will hit these compile errors.
- The recently-merged SDK migration PR (vultiagent-app#33) depends on both packages for keygen/keysign and address derivation. Without a fix, the chat-UI signing flow for every chain silently doesn't build from scratch.

### Rejected approaches
- **Patch the Kotlin source in node_modules via patch-package** — works to get past compile but still leaves MPC stubbed (any sign/keygen call throws at runtime). Not a real fix.
- **Rebuild the AARs with gomobile bind** — would need the private `dkls23-rs` Go source. Not available to every consumer.
- **Ignore the broken API surface, call the SWIG API directly from Kotlin** — requires rewriting the entire Expo module against a different (snake_case, go_slice/tss_buffer-based) API. Very invasive; the high-level ByteArray-returning API the rest of the SDK expects wouldn't match.

## Files
- `vultisig-sdk/packages/mpc-native/android/src/main/java/expo/modules/mpcnative/ExpoMpcNativeModule.kt`
- `vultisig-sdk/packages/mpc-native/android/libs/{dkls,goschnorr}-release.aar`
- `vultisig-sdk/packages/walletcore-native/android/src/main/java/expo/modules/walletcore/ExpoWalletCoreModule.kt`
- Ancillary: `vultisig/dkls-android` (SWIG pipeline source), whichever repo produced the gomobile-shaped Kotlin API the module expects.

## Acceptance Criteria
- [ ] A fresh `npm install @vultisig/mpc-native` + `./gradlew app:assembleDebug` on a clean Android project completes without Kotlin compile errors
- [ ] Either: Kotlin source is rewritten to call the actual SWIG-generated `com.silencelaboratories.godkls.*` API, OR new AARs shipping a `godkls.Godkls` / `goschnorr.Goschnorr` flat namespace are bundled with the package
- [ ] `@vultisig/walletcore-native` compile errors fixed: AnySigner import, `nativePlan` signature, `toUserFriendly(addr, bounceable, testnet)`, and the SS58 stub gives the Kotlin compiler a concrete return type
- [ ] A smoke test (even a minimal RN example) confirms keygen + ECDSA sign work end-to-end with the published tarball, not just the monorepo workspace link
- [ ] vultiagent-app on a clean clone can `npm install && npm run android` and sign a test EVM send without patching node_modules

## Gotchas
- The `npm pack` tarball is ~480 MB (the iOS xcframeworks dominate). Publishing iteration is slow — worth verifying locally with `npm pack && tar -xzf vultisig-mpc-native-*.tgz` and compiling against the extracted tarball, not the workspace link, to catch this class of bug pre-publish.
- CI in the SDK monorepo apparently doesn't build a downstream consumer against the packed tarball — if it did, this would have been caught. Worth adding that to release workflow.
- The AARs vendored in `vultisig-sdk/packages/mpc-native/android/libs/` are actual binaries (not LFS pointers after commit 50cbd56a). A contributor's fresh clone with git-lfs not installed WILL see LFS pointer stubs, which is a separate footgun.

## Notes

**2026-04-22T01:12:45Z**

Closing per user direction. @vultisig/mpc-native pinned at ^0.1.3 in vultiagent-app/package.json (was 0.1.2 when ticket was written). Not independently verified that 0.1.3 resolves the godkls/Godkls vs com.silencelaboratories.* SWIG mismatch — run a fresh 'npm run android' on a clean install to confirm before considering this truly done.
