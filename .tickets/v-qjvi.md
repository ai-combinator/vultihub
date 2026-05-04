---
id: v-qjvi
status: in_progress
deps: []
links: []
created: 2026-04-23T22:59:56Z
type: feature
priority: 1
assignee: Jibles
---
# mcp-ts: execute_* foundation — 5 action tools + shared scaffolding (additive, leave legacy registered)

## Objective

Land the v-pxuw execute_* tool family (execute_send, execute_swap, execute_contract_call, execute_lp_add, execute_lp_remove) plus the shared scaffolding (`_base`, `_amountResolver`, `_thorchain`, `resolveToken`, `parseAmount`, `zodHelpers`, `toolCategories`, `chain-family`, `cosmos-chains`) as a **draft PR against mcp-ts main**. This is the biggest real win from the abandoned PR #30 — ~4k LoC impl + ~4k LoC tests covering unified send/swap/contract/LP flows with a typed ExecutePrepResult envelope.

**Critical adjustment vs #30:** Keep every legacy `build_*_send` / `build_swap_tx` / `build_thorchain_lp_*` tool registered. Move them to gated categories (`custom_send`, `defi`, `raw`) but **do not delete**. Deletion stays on a separate ticket (`v-ujuc`) for a later release cycle, AFTER telemetry shows zero calls to the legacy names.

## Context & Findings

### Why the split

Per memory and v-johx: the original #30 is parked — bundled too much and depended on SDK 0.17 which is being reverted (v-johx). The execute_* family is the single strongest piece of the bundle by reviewer signal. 0xApotheosis on PR #128's sibling fan-out: *"Solid consolidation. Test coverage on the fan-out paths is exemplary, and the security fixes are correct and tested."* Gomes on `executePrep.ts:170`: *"Agree the chain_id/chainRegistry cross-check is valuable — and this is where the boundary should explicitly be widened."*

The problem with #30's version was bundling: (a) same PR hard-deletes 34 legacy tools without alias, (b) same PR ports Uniswap V3 as a new protocol surface, (c) same PR consolidates DeFi + Polymarket reads. Reviewers hit multiple preferably-blocking findings mostly about those bundled-in pieces, not the execute_* tools themselves.

### What this ticket covers

**Cherry-pick from branch `refactor/v-pxuw-tool-consolidation` on mcp-ts:**

- `c1aecae` — mcp-ts: add parseAmount, zodHelpers, toolCategories, execute/_base scaffolding
- `516adef` — mcp-ts: retype ExecutePrepResult for client-side payload assembly
- `7f2d986` — mcp-ts: add direct viem dep for execute_* tool family
- `9a01e05` — mcp-ts: shared amount resolver for execute_* tool family
- `d17a542` — mcp-ts: execute_send unified send tool
- `fdc5ce4` — mcp-ts: execute_swap unified swap tool
- `54a6aa7` — mcp-ts: execute_contract_call generic EVM call tool
- `d0b3fac` — mcp-ts: execute_lp_add + execute_lp_remove THORChain LP tools
- `0c24dc9` — mcp-ts: register the five execute_* tools
- `82deafc` — mcp-ts: execute_swap dispatch non-EVM native swaps by chain family
- `deff0ec` — v-pxuw C1: add `kind` discriminator + camelCase fields to LP execute tools
- `5b7ea66` — v-pxuw C2: reject token sends in execute_send for Solana/Tron/TON/Sui
- `f9a219a` — v-pxuw C3: reject Ripple sends in execute_send (missing signing_pub_key)
- `0e54fff` — v-pxuw C4: emit fee_rate on UTXO native swaps in execute_swap
- `c05340b` — mcp-ts: consolidate chain-family + cosmos + gas helpers across execute_* tools
- `a07af21` + `71702b5` — execute stepper quote/prepare steps (drop + restore pair; net = keep)
- `b4f4be4` — refactor(execute): typed label/raw echoes + centralized resolveToken (v-wldr)
- `98df42f` — fix(execute): drop raw SDK quote from ExecutePrepResult; keep jsonResult strict
- `d497df3` — refactor(mcp-ts): harden category typing + simplify EvmGasInput
- `9d6b134` — feat(tools): textError helper + swap execute_* validation to isError
- `e202ea2` — fix(cosmos-balance): emit formatted balance_<ticker> for multi-denom chains

**Do NOT cherry-pick (stays out of this PR):**
- `ea38d96` — Uniswap V3 port → ticket 2
- `1e744ec`, `f804203`, `acc99b3`, `f5c163e` — DeFi/Polymarket consolidation + registry reorg → tickets 3, 4
- `b207da5`, `5fbc4b8`, `7f25d97`, `9231a24` — legacy build_* retirement (va-tixe) → separate ticket `v-ujuc` (already exists, open)
- `57df865`, `af7792a` — SDK link/bump (0.17 surface) → see v-johx (SDK revert)

### Reviewer feedback to address

- **Keep legacy registered**: Gomes' Pattern 4a (hard cuts without alias) and Pattern 4b (lost launch-flag guardrails) apply. This PR keeps legacy tools in gated categories, does not change `executor.go:1345` swap-backend allowlist enforcement (that lives in agent-backend and is out of scope here).
- **chain_id cross-check** lives on the app side (ticket 10) per original split but the mcp-ts side needs to emit `chain_id` in the ExecutePrepResult for the app to validate against.
- **Stepper config null-fallback**: Gomes B7a on #164 `executePrep.ts:103`: *"PR body claims `stepperConfig` allowed to be null with default fallback. This validator rejects missing/null `stepperConfig`."* mcp-ts side must emit `stepperConfig` on every ExecutePrepResult — no null — so the app-side fallback never triggers on the normal path. Verify during cherry-pick.

### SDK dependency

The v-pxuw branch on mcp-ts uses SDK exports that are being reverted by v-johx (SDK → 0.18.0). **Before raising this PR**, confirm the SDK version it pins against. If the branch uses 0.17-only exports (`prepare*FromKeys`, `fiatToAmount`, `normalizeChain`, `resolveToken` helpers), those need either reimplementation inline in mcp-ts or coordination with the v-johx revert to keep what's needed. Specifically:
- `fiatToAmount` — reimplement in `mcp-ts/src/lib/` if still used by `_amountResolver.ts`
- `normalizeChain` — already has an mcp-ts copy (widened in commit per PR 229 body); use that
- `resolveToken` — inline in `mcp-ts/src/lib/resolveToken.ts` per commit `b4f4be4`
- `prepare*FromKeys` — NOT used by mcp-ts (those live in clients); should be fine

## Files

**New files (from cherry-picks):**
- `src/tools/execute/_base.ts`, `_amountResolver.ts`, `_thorchain.ts`
- `src/tools/execute/execute_send.ts`, `execute_swap.ts`, `execute_contract_call.ts`, `execute_lp_add.ts`, `execute_lp_remove.ts`
- `src/lib/resolveToken.ts`, `resolveToken.test.ts`
- `src/lib/parseAmount.ts`, `zodHelpers.ts`, `toolCategories.ts`, `chain-family.ts`, `cosmos-chains.ts`

**Modified:**
- `src/tools/index.ts` — register 5 new tools AT THE TOP of `allTools[]`. Keep all other registrations intact. Do NOT move anything into "gated category" blocks in this PR (that's part of ticket 7's filter work).
- `src/tools/types.ts` — add categories field if needed
- `package.json` — add `viem` dep if not already present

**Do NOT modify (intentional):**
- Any legacy `build_*_send`, `build_swap_tx`, `build_thorchain_lp_*` files — leave as-is
- `src/tools/defi/`, `src/tools/polymarket/`, `src/tools/uniswap/` — separate tickets

## Acceptance Criteria

- [ ] PR opened as **draft** against `vultisig/mcp-ts` main, title `feat(tools): execute_* action tool family`
- [ ] PR body follows structure: Summary, What's in, What's deliberately left out (points to tickets 2/3/4 and v-ujuc), Dependencies (standalone, mergeable off main), Test plan
- [ ] All 5 execute_* tools registered in `src/tools/index.ts`
- [ ] All legacy `build_*_send`, `build_swap_tx`, `build_thorchain_lp_*` tools STILL REGISTERED (no deletions)
- [ ] `pnpm test` passes; includes all execute_* unit tests (~4k LoC)
- [ ] `pnpm lint` clean
- [ ] `pnpm build` clean
- [ ] PR description notes this is **independent and mergeable off main**
- [ ] No dependency on SDK 0.17-only exports (either inline or coordinated with v-johx)
- [ ] ExecutePrepResult emits `chain_id`, `labels`, `raw?`, `stepperConfig` on every success path
- [ ] Token sends for SPL / TRC-20 / TON-jetton / SUI-token / Cardano-asset / XRP return explicit rejection with pointers to escape-hatch `build_*` tools

## Gotchas

- This PR is **additive only** — reviewers should see "we're adding new capabilities, nothing existing is removed or broken." Lead the PR body with that framing.
- Gomes' review on #30 did not include line-level comments (reviewed only at issue level). Don't assume "no findings = approval"; the concerns he raised on #128 / #164 (hard cuts, lost guardrails) applied to #30 implicitly. Anticipate and address preemptively.
- The branch's `package.json` bumps `@vultisig/sdk` to `^0.17.0` — this will change to `^0.18.0` once v-johx lands. Pin to whatever SDK version is live on main at PR-open time; don't bump as part of this PR.
- Verify `convertAmount` stays registered (existing util). Do not accidentally remove it during the registry edits.
- `refactor/v-pxuw-tool-consolidation` is also on monorepo-poc PR #41; the same commits exist there. Cherry-pick from v-pxuw branch for consistency.
- The adopted pattern is `_meta.categories` on each tool via `Categories.*` constants from `toolCategories.ts`. Preserve this so ticket 7 (agent-backend filter work) has a stable contract.

## Notes

**2026-04-23T23:50:17Z**

Draft PR opened: https://github.com/vultisig/mcp-ts/pull/49
