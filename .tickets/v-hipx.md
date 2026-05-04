---
id: v-hipx
status: in_progress
deps: []
links: []
created: 2026-04-23T23:04:36Z
type: task
priority: 2
assignee: Jibles
parent: v-vtmi
---
# vultiagent-app: WalletCore derivation-path + chainId init probe for v-vtmi bug investigation

## Objective

Extract the WalletCore debug probe (commit `e14f503` tagged v-vtmi on `refactor/v-pxuw-tool-consolidation`) into its own tiny PR linked to the **v-vtmi bug** ("Self-send EVM fails with 'Failed to derive Ethereum address' on confirm"). The probe logs WalletCore initialization state so we can diagnose whether the failure is an init race or malformed vault public-key shape.

## Context & Findings

### Parent bug

v-vtmi (open, P1): *"On the vultiagent-app agent chat, requesting a self-send of ETH on Arbitrum builds the tx correctly, but on password-confirm signing fails with red 'Failed to derive Ethereum address'."*

v-vtmi's triage note says:
> "Two plausible root causes: (1) WalletCore not initialized by the time the EVM sign path runs… (2) getPublicKey rejecting the vault's public-key shape."

### What the probe does

Commit message from `e14f503`:
> "debug(wallet-core): probe derivation paths + chainId at init (v-vtmi)"

Logs derivation paths + chainId at WalletCore init so we can see (at instrumentation-level) what the SDK is loading. Does not fix v-vtmi — purely diagnostic.

### Decision point — probe or fix?

v-vtmi's open triage asks: "Either (a) ensure initWalletCore() has resolved before any signing entrypoint, or (b) make deriveAddress propagate the error with cause so the toast surfaces the actual problem."

Option (b) is a **real fix**, not a probe. Consider landing (b) in this PR instead of (or in addition to) the probe. The probe on its own is useful short-term but the propagated-cause change is strictly better.

**Decision for the agent**:
- If v-vtmi has fresh device logs attached with the root cause identified → land the real fix, skip the probe
- If v-vtmi is still "need more info" → land the probe AND the error-propagation improvement (so next repro surfaces cause)
- If neither is clear → just land the probe (commit `e14f503`) and update v-vtmi with any new findings

### What this ticket covers

At minimum: cherry-pick `e14f503` surgically.

Ideally: also include the `addressDerivation.ts:75-112` error-propagation fix from v-vtmi's triage (swap silent `console.warn` + `return null` for real propagated error with cause).

## Files

- `src/services/addressDerivation.ts` — probe instrumentation + optional error-propagation fix
- `src/services/evmTx.ts` — only if wrapping in try/catch for clearer surfacing (optional per v-vtmi triage step 3)

## Acceptance Criteria

- [ ] PR opened as **draft** against vultiagent-app main, title `debug(wallet-core): probe derivation paths + init state (v-vtmi diagnostic)` (or `fix(wallet-core): propagate derivation failures with cause` if including the fix)
- [ ] PR description links to v-vtmi bug and quotes the "two plausible root causes" paragraph
- [ ] If landing the fix: `deriveAddress` propagates the underlying error (with cause), so "Failed to derive Ethereum address" toast shows the real cause
- [ ] `pnpm test`, `pnpm lint`, `pnpm tsc --noEmit` clean
- [ ] PR notes this is **independent and mergeable off main**
- [ ] v-vtmi ticket updated with a `tk note` linking to the draft PR

## Gotchas

- The probe is logging, not error-handling — keep it minimal; don't hide real failures behind probe output
- If the bug is reproducible locally, prefer landing the fix over the probe. Logs without the fix still leave users seeing a cryptic toast
- Do NOT pull in any v-pxuw tool-consolidation work from the same session's commits (`ee3776b` Maven is ticket 8; everything else stays out)
- Parent ticket linkage: set parent to `v-vtmi` via `--parent v-vtmi` on creation or via `tk dep` after

## Notes

**2026-04-23T23:43:56Z**

Draft PR opened: https://github.com/vultisig/vultiagent-app/pull/240
