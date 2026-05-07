---
id: v-hiry
status: closed
deps: []
links: []
created: 2026-04-23T23:02:05Z
type: task
priority: 2
assignee: Jibles
---
# mcp-ts: DeFi read consolidation (9→5) with legacy tools kept alongside

## Objective

Ship the 5 new consolidated DeFiLlama read tools (`defi_prices`, `defi_yields`, `defi_protocols`, `defi_stablecoins`, `defi_dex_volumes`) from #30 as an additive PR — **keep the 9 legacy DeFiLlama tools registered alongside** (under a `legacy_defi` gated category) instead of deleting them. Fixes the historical-render problem reviewers implied via pattern 4a on #30 and #164.

## Context & Findings

### The core consolidation is real and good

Unlike the "~136 → ~30" marketing line that conflates prompt-visibility with registry depth, the DeFi consolidation is a genuine tool-count reduction: 9 → 5 consolidated shape-clustered reads. That IS a win for weak-LLM selection accuracy.

### What broke in the original #30

#30 deleted `src/tools/defi/defi-tools.ts` and `src/tools/defi/defillama-native.ts` outright. Per Gomes on #164 Pattern 4a: *"Hard cuts without alias / migration layers. PR128 drops old tool names with one hand-rolled shim (schedule_task); PR164 drops findLastBuildToolIndex suppression and row validators. Same fault: new shape plumbed, old callers unmigrated."*

Applied to DeFi: any persisted conversation row carrying `tool-defi_get_protocol` or `tool-defillama_get_token_prices` parts will fail to render against a registry that only has `tool-defi_protocols`/`tool-defi_prices`. Unlike the agent-backend's `schedule_task → schedule_create` alias, there's no defi migration shim.

### Fix: keep legacy registered

This ticket keeps all 9 legacy tools registered in a `legacy_defi` gated category (or the existing category, but with a sunset comment). They won't surface in the always-on LLM prompt (filter works), but:
- Historical tool-call parts still render (old name → old tool → existing card)
- LLM can still call them if it somehow picks them up from a legacy prompt
- Tool retirement happens in a future separate PR once telemetry shows zero calls (~1 release cycle post-ticket 7)

### What this ticket covers

**Cherry-pick from `refactor/v-pxuw-tool-consolidation`:**
- `1e744ec` — mcp-ts: consolidated DeFi query tools (9 → 5)
- Corresponding tests under `src/tools/defi/__tests__/consolidated.test.ts`

**Explicitly adjust from the original:**
- Do NOT apply the legacy-deletion portion of `f804203` (remove superseded defi + polymarket read tools). The polymarket half belongs to ticket 4; the defi deletion is deferred.
- Keep `src/tools/defi/defi-tools.ts` and `src/tools/defi/defillama-native.ts` intact on main
- Keep their registrations in `src/tools/index.ts` — just add the 5 new consolidated ones nearby

## Files

**New:**
- `src/tools/defi/consolidated.ts` — 5 new tools
- `src/tools/defi/__tests__/consolidated.test.ts`

**Modified:**
- `src/tools/index.ts` — add imports + registrations for 5 new tools; do NOT remove imports or registrations for the 9 legacy tools

**Do NOT delete:**
- `src/tools/defi/defi-tools.ts`
- `src/tools/defi/defillama-native.ts`

## Acceptance Criteria

- [ ] PR opened as **draft** against mcp-ts main, title `refactor(tools): consolidate DeFi reads 9→5 (legacy kept alongside)`
- [ ] PR body explicitly states: "9 legacy tools remain registered in a gated category and will be retired in a later PR after telemetry shows zero calls"
- [ ] 5 new tools pass tests; all 9 legacy tools still pass their existing tests
- [ ] `src/tools/index.ts` shows both sets registered
- [ ] `pnpm test`, `pnpm lint`, `pnpm build` clean
- [ ] PR notes this is **independent and mergeable off main**

## Gotchas

- Do NOT apply `acc99b3` (apply _meta.categories from Categories.* constants) or `f5c163e` (reorganise tool registry by gated category) in this PR — those are holistic registry rewrites
- If `Categories.*` constants don't exist on main yet, new tools can use plain string `"defi"` for `_meta.categories`
- Do not bundle Uniswap V3 (ticket 2) or Polymarket (ticket 4) changes here

## Notes

**2026-04-23T23:43:02Z**

Draft PR opened: https://github.com/vultisig/mcp-ts/pull/46

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/mcp-ts#46 (open, in review)
