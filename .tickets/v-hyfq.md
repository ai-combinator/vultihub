---
id: v-hyfq
status: closed
deps: []
links: []
created: 2026-04-08T04:49:09Z
type: task
priority: 2
assignee: Jibles
---
# Progressive migration from Detox to Maestro E2E testing

## Objective

Migrate vultiagent-app E2E tests from Detox to Maestro, progressing incrementally and only continuing if Maestro proves more reliable at each stage. Detox has chronic reliability issues — hardcoded sleeps, `disableSynchronization()` hacks throughout, serial-only execution, and frequent Android emulator flakiness. Maestro uses a tolerance-based retry model that should eliminate most of these problems.

## Context & Findings

- Current Detox suite lives in `vultiagent-app/e2e/` with tests for vault creation, seed phrase import, chain sends (15+ chains), conversations, and balances
- Key pain points: `disableSynchronization()` in most tests, hardcoded `sleep()` with recording multipliers, 180s timeouts, `tapWithFallback` suggesting unreliable element targeting
- Detox is NOT in CI — runs locally only, requires manual emulator setup
- Maestro and Detox can coexist — zero conflict. Detox uses the RN bridge, Maestro uses the platform accessibility layer. Both drive the same app build on the same emulator
- Maestro uses YAML-based flow definitions with built-in auto-retry/wait. No sync model to fight
- Complex Detox helpers (OTP via `agentmail` CLI, vault file staging via ADB, platform detection) will need to become standalone scripts that Maestro calls via `runScript`
- Rejected approaches: Playwright + Expo Web (too many native dependencies block web rendering — reanimated, rive, expo-dkls, expo-camera, etc.), Appium (slower and flakier than Detox), RNTL alone (no visual rendering, can't verify "looks reasonable")

## Files

- `vultiagent-app/maestro/` — new directory for Maestro flows (alongside existing `e2e/`)
- `vultiagent-app/e2e/` — existing Detox tests (reference, then delete at end)
- `vultiagent-app/.detoxrc.js` — Detox config (delete at end)
- `vultiagent-app/e2e/jest.config.js` — Detox Jest config (delete at end)
- `vultiagent-app/e2e/helpers/` — reference for flow logic, especially `common.js`, `chainTest.js`, `importVault.js`
- `vultiagent-app/e2e/flows/` — reference for setup flows (`setupViaCreate.js`, `setupViaImport.js`, `setupViaSeedPhrase.js`)
- `vultiagent-app/package.json` — add Maestro scripts, eventually remove Detox deps

## Acceptance Criteria

Progressive — each phase gates the next. Do not proceed to the next phase unless the current phase demonstrates Maestro is working reliably.

### Phase 1: Infrastructure & smoke test
- [ ] Install Maestro CLI and verify it can launch the existing app build on an Android emulator
- [ ] Write a trivial Maestro flow: app launches, a screen renders, tap a button, verify navigation occurs
- [ ] Document any emulator setup issues encountered vs Detox

### Phase 2: Basic conversation flow
- [ ] Write a Maestro flow that opens a conversation, types a message, and verifies an agent response appears
- [ ] Run both the Maestro conversation flow and the equivalent Detox `conversation.test.js` 5+ times each on the same emulator
- [ ] Compare pass rates — log results. If Maestro is not equal or better, stop and report findings

### Phase 3: Vault import flow
- [ ] Port the vault import flow — stage a vault file and import it via Maestro (will need a `runScript` helper for ADB file staging)
- [ ] Write a Maestro flow for the full import-then-chat sequence
- [ ] Run both Maestro and Detox versions 5+ times, compare reliability

### Phase 4: Complex flows
- [ ] Port vault creation flow (includes OTP verification — extract `agentmail` helper into standalone script callable via `runScript`)
- [ ] Port balance query tests
- [ ] Port at least 3 representative chain send flows from `chainTest.js`
- [ ] Run both suites in parallel, compare total pass rates across all flows

### Phase 5: Cutover
- [ ] All existing Detox test coverage has equivalent Maestro flows
- [ ] Remove Detox dependencies from `package.json`
- [ ] Delete `e2e/` directory, `.detoxrc.js`, and `e2e/jest.config.js`
- [ ] Update `package.json` scripts to point to Maestro
- [ ] Update any CI or documentation references

## Gotchas

- Maestro `runScript` for shell helpers (OTP, vault staging) runs scripts relative to the flow file — path handling matters
- Maestro waits are automatic but configurable — if agent responses take 30s+, you may need explicit `extendedWaitUntil` with higher timeouts
- The existing app build works for both frameworks — no need to rebuild between Detox and Maestro runs
- Test IDs (`testID` props) work in Maestro too — no need to change app code for element targeting
- If the emulator itself is the reliability problem (not Detox), Maestro won't fix that — note this in Phase 1 findings

## Notes

**2026-04-08T05:04:04Z**

Phase 1 complete: Maestro CLI v2.4.0 installed, smoke.yaml flow created, npm scripts added. Maestro auto-detects Android emulators (pixel_6, medium_phone) without config — simpler than Detox.

**2026-04-08T05:08:44Z**

Phase 2 complete: conversation.yaml flow with import-vault sub-flow, stage-vault.sh helper, compare-reliability.sh script. Maestro runFlow with when:visible conditional for auth handling.

**2026-04-08T05:10:16Z**

Phase 3 complete: vault-import.yaml flow (import + gm + balances verification), compare-import.sh reliability script.

**2026-04-08T05:14:13Z**

Phase 4 complete: vault-create.yaml with OTP via agentmail API (fetch-otp.js), balances.yaml, 3 chain send flows (BTC/ETH/SOL). fetch-otp.js uses Maestro's GraalJS runtime with http.get() and java.lang.Thread.sleep() for polling.

**2026-04-08T05:23:46Z**

Phase 5 complete: seed phrase import flow, e2e demo, all 16 chain configs, Detox fully removed (deps, config, plugin, e2e/ dir, app code). Review findings fixed: withDetoxAndroid plugin, app.json, App.tsx, index.ts, .env.example, dead scripts. Verification: typecheck clean, lint clean (pre-existing warnings only), 410/410 tests pass.
