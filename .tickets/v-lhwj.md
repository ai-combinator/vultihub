---
id: v-lhwj
status: open
deps: []
links: []
created: 2026-04-27T21:11:26Z
type: task
priority: 2
assignee: Jibles
---
# RN preamble: add tests + migrate vultiagent-app off local polyfills

Follow-ups from vultisig-sdk PR #310 review (gomesalexandre R1+R2). Both flagged non-blocking; bundling here so they ship together once the SDK with `@vultisig/sdk/rn-preamble` is published.

Context: PR #310 (https://github.com/vultisig/vultisig-sdk/pull/310) landed `@vultisig/sdk/rn-preamble` — a side-effect entry that installs `globalThis.Buffer` and patches `Buffer.prototype.subarray` so RN apps don't have to hand-roll it. PR is approved + merged. Two follow-ups remained open at merge time.

## 1. vultisig-sdk: tests for the preamble

Two test gaps R2 explicitly left as follow-ups:

- **Strict pnpm install + resolution test for `@vultisig/sdk/rn-preamble`.** Catches the `buffer` dep-declaration regression (the R1 PB) automatically. Set up a sandbox consumer with `pnpm install --frozen-lockfile`, `import '@vultisig/sdk/rn-preamble'`, assert it resolves cleanly. Mirrors the failure mode under pnpm isolated / Yarn PnP / strict linked installs.
- **Preamble ordering unit test.** Assert that after `import '@vultisig/sdk/rn-preamble'`:
  - `globalThis.Buffer` is set
  - `Buffer.prototype.subarray(...)` returns something with `.copy()` / Buffer methods (i.e. the patched implementation, not a plain Uint8Array)
  Mirror the shape of `polyfillsHoisted.test.ts` shipped in vultiagent-app#243.

Lives in vultisig-sdk repo. Suggested location: `packages/sdk/tests/unit/rn-preamble.test.ts` (Vitest) for the ordering test; the strict-install test is more naturally a CI workflow step or a small Node script under `packages/sdk/tests/integration/`.

## 2. vultiagent-app: swap local `./polyfills.ts` for `@vultisig/sdk/rn-preamble`

Once a vultisig-sdk version with `/rn-preamble` is published and vultiagent-app bumps to it, replace the locally-shipped `polyfills.ts` (vultiagent-app#243) with a single `import '@vultisig/sdk/rn-preamble'` as the first statement of the app entry. With the R2 patching strategy (patches whichever Buffer instance is in `globalThis` first, falling back to its own), the SDK preamble is safe to land alongside the existing app-side preamble during the migration window — second-loaded one wins on `globalThis.Buffer` but both get the subarray repair. Migration is low-risk.

Done when:
- vultiagent-app entry imports `@vultisig/sdk/rn-preamble` first, before any other import
- local `polyfills.ts` (and its import) is removed
- on-device boot still works on iOS + Android (Buffer-using libs — @ton/core, bs58check, @bufbuild/protobuf, xrpl — don't crash at module-init)

## Acceptance criteria

- [ ] vultisig-sdk: pnpm strict-install resolution test for `@vultisig/sdk/rn-preamble` exists and runs in CI
- [ ] vultisig-sdk: preamble ordering test asserts `globalThis.Buffer` set + `subarray` returns a Buffer-shaped result
- [ ] vultiagent-app: bumped to the published SDK version that contains `/rn-preamble`
- [ ] vultiagent-app: local `polyfills.ts` deleted; entry imports `@vultisig/sdk/rn-preamble` as first statement
- [ ] iOS + Android boot to home screen on both platforms
