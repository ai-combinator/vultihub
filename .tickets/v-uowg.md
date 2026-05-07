---
id: v-uowg
status: closed
deps: [v-whzf, v-hiry, v-zhop, v-llqd]
links: []
created: 2026-04-23T23:06:23Z
type: feature
priority: 2
assignee: Jibles
---
# vultiagent-app: Query cards for consolidated DeFi (5) + Uniswap V3 (3) + Polymarket v2 (3) reads

## Objective

Add 11 read-only Query cards rendering outputs from the consolidated/ported read tools (ticket 3 DeFi, ticket 2 Uniswap, ticket 4 Polymarket). All render-only, low-risk, but with defensive per-row validation to avoid the class of crash Gomes flagged on #164.

## Context & Findings

### Reviewer findings this PR must fix

**Gomes B8 — representative of 8 cards:**
> "preferably-blocking: render crash on malformed rows (representative). Same class of issue across 8 cards. Container validated, rows not. One malformed row from any backend = render crash."

**Fix**: every row component in every Query card validates its inputs defensively. If a row is missing required fields or they're the wrong type, skip the row with a fallback or `null`, do NOT throw. Per-card container validation is not enough.

**Gomes Suggestion 2+3 on `toolUIRegistry.ts:66`:**
> "No flag gate: the entire execute/query/CRUD surface is registered unconditionally. If one backend tool shape is wrong in prod, rollback is an app-store release."

Each Query card should be behind a small feature flag (e.g., per-protocol: `query_defi_enabled`, `query_uniswap_enabled`, `query_polymarket_enabled`) OR behind the existing launch-surface gating if the app already has one. Avoids app-store rollback for a single card regression.

### What this ticket covers

**Cherry-pick from `refactor/v-pxuw-tool-consolidation` (vultiagent-app):**
- `ec1807c` — feat(agent): add QueryCards DeFi cards + shared primitives (v-pxuw Task 11)
- `4bb06d2` — feat(agent): add QueryCards Uniswap V3 cards (v-pxuw Task 11)
- `44aa5db` — feat(agent): add QueryCards Polymarket cards + toolUIRegistry wiring (v-pxuw Task 11)
- Related tests under `__tests__/DefiPricesCard.test.ts`, `QueryCards.formatters.test.ts`, `QueryCards.uniswap.test.ts`

**Adjust during cherry-pick:**
- Add defensive per-row validation in all 11 cards per Gomes B8
- Add per-protocol feature flags for registry entries
- Keep separate from ticket 10 — Query cards don't need the ExecutePrepResult / stepper / approval flow

### Dependency chain

**Must depend on tickets 2/3/4 being deployed** before opening — backend tools must exist or cards have nothing to render.

**Cannot merge before ticket 10** — shares `toolUIRegistry.ts` edits; easier if ticket 10's registry changes land first to avoid merge conflicts.

### Card inventory

**DeFi (5):**
- `DefiPricesCard.tsx` — consumes `defi_prices`
- `DefiYieldsCard.tsx` — consumes `defi_yields`
- `DefiProtocolsCard.tsx` — consumes `defi_protocols`
- `DefiStablecoinsCard.tsx` — consumes `defi_stablecoins`
- `DefiDexVolumesCard.tsx` — consumes `defi_dex_volumes`

**Uniswap V3 (3):**
- `UniswapPoolInfoCard.tsx` — consumes `uniswap_pool_info`
- `UniswapPositionInfoCard.tsx` — consumes `uniswap_position_info`
- `UniswapTickMathCard.tsx` — consumes `uniswap_tick_math`

**Polymarket (3):**
- `PolymarketSearchCard.tsx` — consumes `polymarket_search_v2` (new name per ticket 4)
- `PolymarketMarketCard.tsx` — consumes `polymarket_market_v2` (or `polymarket_market` if kept as-is)
- `PolymarketMyCard.tsx` — consumes `polymarket_my_v2` (or `polymarket_my`)

### Splittable

If reviewer bandwidth is tight, this ticket can be split into three sub-PRs: DeFi (5 cards), Uniswap (3), Polymarket (3). They're mutually independent and can merge in any order once their respective mcp-ts tickets deploy.

## Files

**New:**
- `src/features/agent/components/tools/QueryCards/` directory
- 11 card components + `shared.tsx`, `formatters.ts`, `defiPrices.ts`, `uniswap.ts`, `index.ts`
- Tests under `src/features/agent/components/tools/__tests__/` (existing) + `QueryCards.*.test.ts` files

**Modified:**
- `src/features/agent/lib/toolUIRegistry.ts` — add 11 new entries, gated behind feature flags

## Acceptance Criteria

- [ ] PR opened as **draft** against vultiagent-app main (rebased post-ticket 10), title `feat(agent): query cards for consolidated DeFi / Uniswap / Polymarket reads`
- [ ] All 11 cards defensively validate per-row inputs; no throws from malformed backend data
- [ ] All 11 cards registered in toolUIRegistry behind per-protocol feature flags
- [ ] `pnpm test`, `pnpm lint`, `pnpm tsc --noEmit` clean
- [ ] PR body has **"Blockers" section**: "Do NOT merge before tickets 2 (Uniswap), 3 (DeFi), 4 (Polymarket) are deployed to MCP server; ticket 10 merged"
- [ ] Consumer shape of `polymarket_search_v2` matches mcp-ts ticket 4's rename (not `polymarket_search`)

## Gotchas

- Match the `polymarket_search_v2` tool name from ticket 4; do not use `polymarket_search`
- Per-row defensive validation needs tests that pass malformed rows in and assert the card still renders (skipping the bad row)
- Feature flags: check the existing pattern in the app. If there's no per-tool flag infrastructure, use a single umbrella flag `query_cards_enabled` to at least enable app-store-free rollback for the whole set
- Do NOT touch `AgentApprovalBar.tsx`, `useToolExecution.ts`, or execute_* cards — those are tickets 10 and PR 229
- If ticket 4 keeps the old `polymarket_search` name registered alongside, register the OLD-named card ONLY if the old tool still serves the LLM — otherwise the old card sits unused. Skip the old-name card in this PR; the legacy polymarket tools will render through existing in-prod cards (if any) until retired in a future PR

## Notes

**2026-04-24T00:26:37Z**

Draft PR opened: https://github.com/vultisig/vultiagent-app/pull/244

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/vultiagent-app#244 (DRAFT)
