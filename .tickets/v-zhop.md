---
id: v-zhop
status: closed
deps: []
links: []
created: 2026-04-23T23:02:15Z
type: task
priority: 2
assignee: Jibles
---
# mcp-ts: Polymarket read consolidation (7→3) with distinct v2 names and legacy kept

## Objective

Ship the consolidated Polymarket read tools (3 new, replacing 7 legacy reads) as an additive PR — but with **distinct `_v2` tool names** instead of in-place renaming `polymarket_search`, and with the 7 legacy tools **kept registered alongside**. Fixes the schema-match hazard reviewers would hit on the original #30's in-place rename.

## Context & Findings

### The hazard with the original PR

#30's `f96cab7` consolidation reuses the name `polymarket_search` for the new tool while changing its schema. Any persisted conversation or LLM prompt history referencing the old `polymarket_search` argument shape will silently mismatch against the new one — Zod validation will throw, renderer will fail. There's no way to tell "this was the old polymarket_search or the new one" because they share a name.

This is a subtler version of Pattern 4a (hard cuts without alias) applied to a schema change rather than a tool removal.

### Fix: distinct names

- Rename the new consolidated tool to `polymarket_search_v2` (and optionally prefix the other two: `polymarket_market_v2`, `polymarket_my_v2`, though these are genuinely new names so no hazard)
- Keep all 7 legacy Polymarket read tools registered in the `polymarket_legacy` gated category (or existing `polymarket` category with sunset comment)
- Future retirement PR once telemetry shows zero calls to old names

### What this ticket covers

**Cherry-pick from `refactor/v-pxuw-tool-consolidation`:**
- `f96cab7` — mcp-ts: consolidated Polymarket read tools (7 → 3) — **adjust during cherry-pick to rename the search tool from `polymarket_search` to `polymarket_search_v2`**
- Tests under `src/tools/polymarket/__tests__/consolidated.test.ts`

**Explicitly adjust from original:**
- Rename `polymarket_search` → `polymarket_search_v2` in tool name, description, and all test assertions
- Do NOT apply the polymarket-deletion portion of `f804203`; keep legacy 7 tools registered

## Files

**New:**
- `src/tools/polymarket/consolidated.ts` — 3 new tools (with `_v2` suffix on the search one)
- `src/tools/polymarket/__tests__/consolidated.test.ts`

**Modified:**
- `src/tools/index.ts` — add imports + registrations for 3 new tools. Do NOT remove imports or registrations for the 7 legacy tools

**Do NOT delete:**
- Anything in `src/tools/polymarket/polymarket-tools.ts` beyond what's needed for the new tools' implementation

## Acceptance Criteria

- [ ] PR opened as **draft** against mcp-ts main, title `refactor(tools): add polymarket consolidated v2 reads (legacy kept alongside)`
- [ ] New search tool is named `polymarket_search_v2` (not `polymarket_search`) — no name collision with legacy
- [ ] PR body explicitly calls out the naming decision: "avoids schema-mismatch hazard on conversation replay"
- [ ] All 7 legacy polymarket_* tools still registered and pass their existing tests
- [ ] 3 new tools pass their tests
- [ ] `pnpm test`, `pnpm lint`, `pnpm build` clean
- [ ] PR notes this is **independent and mergeable off main**

## Gotchas

- The `polymarket_sign_bet` client-side action tool (in agent-backend) is unrelated — do not touch it
- The other polymarket action tools (`polymarketBuildOrder`, `polymarketSubmitOrder`, `polymarketCheckApprovals`, `polymarketPlaceBet`, `polymarketCancelOrder`) are also separate from this consolidation — do not touch them
- Check the agent-backend `tool_classify.go` to see if it references `polymarket_search` by name; if so, an analytics adjustment is needed in ticket 7

## Notes

**2026-04-23T23:44:07Z**

Draft PR opened: https://github.com/vultisig/mcp-ts/pull/47

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/mcp-ts#47 (open, in review)
