---
id: v-ljlg
status: open
deps: []
links: []
created: 2026-04-27T20:51:41Z
type: task
priority: 3
assignee: Jibles
external-ref: vultisig/mcp-ts#52
---
# mcp-ts cosmos balance: enrich known non-native denoms (THOR/MAYA tickers + decimals registry)

## Objective

Improve the raw-fallback non-native denom shape in `src/tools/balance/cosmos-balance.ts:119-123` so that *known* multi-denom chain natives (MayaChain `maya`, THORChain `tcy`/`ruji`, etc.) come back with proper ticker casing — and ideally `formatted`/`decimals` — instead of the current `{denom, symbol: denom (lowercase), amount}` passthrough.

Follow-up surfaced by Codex 5.4 adversarial review on vultisig/mcp-ts#52 (now merged as squash commit vultisig/mcp-ts@7004de19). Tagged "follow-up" by reviewer NeOMakinG and verified out-of-scope for the merged PR.

## Context & Findings

### What PR #52 already did

PR #52 unified the raw-fallback shape so every entry exposes at least `{denom, symbol, amount}`, matching the rujira-enriched shape. Native denoms get full `{symbol, formatted, decimals}` from chain config; non-native denoms (rujira tokens, secured assets, IBC) pass through with `symbol: denom`.

### What this ticket addresses

`src/tools/balance/cosmos-balance.ts:119-123` (the non-native passthrough) emits:

```ts
return {
  denom: b.denom,
  symbol: b.denom,
  amount: b.amount,
}
```

For known multi-denom chain natives we have enough chain knowledge to do strictly better than `symbol: <lowercase denom>`.

### Cheap win (already actionable)

A `tickerByDenom` map exists in `src/tools/send/build-cosmos-send.ts:45`:

```ts
{ name: 'MayaChain', ..., tickerByDenom: { cacao: 'CACAO', maya: 'MAYA' }, ... }
```

Lifting/sharing this with the balance tool would at minimum fix the symbol case for `maya` (`'maya'` -> `'MAYA'`) on MayaChain and unblock the same pattern for THORChain `tcy` / `ruji` if/when added.

Estimated size: ~10 lines + a unit test asserting MayaChain `maya` denom comes back as `symbol: 'MAYA'`.

### Harder part (separate work, may want to split)

Adding `formatted` + `decimals` for non-native known denoms requires real per-denom decimals data:

- MayaChain `maya` -> 4 decimals
- THORChain `tcy` -> 8 decimals
- THORChain `ruji` -> ? (unverified)
- Secured assets -> vary per asset

This data is **NOT** currently in the codebase. Codex's claim that "MAYA 4 decimals already in build-cosmos-send.ts:45" was a hallucination — only ticker mapping is there. So this part requires:

1. Sourcing decimals from chain docs or a Rujira lookup
2. A place to put the data — probably a new `knownDenoms` constants module shared across `send/` and `balance/` tools

### Scope decision

Ticker uppercasing is a quick win (~10 lines + test). Decimals/formatted is a separate, larger piece requiring a known-denoms registry. **Consider splitting into two PRs** if the registry work blocks the symbol fix:

- **Phase 1 (quick)**: extract shared `tickerByDenom` map (or similar), use it in `cosmos-balance.ts` raw-fallback non-native branch. Just fixes ticker casing.
- **Phase 2 (registry)**: introduce `src/lib/known-denoms.ts` (or similar) with `{ denom -> { ticker, decimals } }` entries for MAYA, TCY, RUJI, common secured assets. Wire into both `cosmos-balance.ts` (formatted/decimals on non-native) and `build-cosmos-send.ts` (replace local `tickerByDenom`).

## Files

**Modified (Phase 1):**
- `src/tools/balance/cosmos-balance.ts` — replace lines 119-123 passthrough with a known-denom lookup before falling back to `{denom, symbol: denom, amount}`
- `src/tools/send/build-cosmos-send.ts` — extract or share `tickerByDenom` map

**New (Phase 2):**
- `src/lib/known-denoms.ts` (or `src/tools/balance/known-denoms.ts`) — registry of `{ denom -> { ticker, decimals } }` for known non-native multi-denom chain assets
- Tests under `src/tools/balance/__tests__/` covering MayaChain `maya` denom, THORChain `tcy` denom (if registry covers it)

## Acceptance Criteria

- [ ] MayaChain raw-fallback returns `symbol: 'MAYA'` (uppercase) for `maya` denom — not `'maya'`
- [ ] Shared registry / map used (no duplicated `tickerByDenom` between balance and send tools)
- [ ] Non-native known denoms in registry come back with `formatted` + `decimals` matching native-denom shape (Phase 2 only)
- [ ] Unknown denoms still pass through as `{denom, symbol: denom, amount}` (graceful fallback)
- [ ] `pnpm test`, `pnpm typecheck`, `pnpm lint` clean
- [ ] PR description references vultisig/mcp-ts#52 and merged commit vultisig/mcp-ts@7004de19

## Gotchas

- Don't claim decimals data the codebase doesn't have. MAYA = 4 and TCY = 8 need to be verified against chain docs / Rujira before encoding.
- The shared map should live in a place importable by both `tools/balance/` and `tools/send/` without circular imports — `src/lib/` is the safe spot.
- Don't break the existing native-denom path (lines 110-117). The known-denom lookup belongs strictly in the non-native branch (line 119+).
- If this ticket gets split, link the Phase 2 registry ticket here with `tk dep`.

## References

- Merged PR: https://github.com/vultisig/mcp-ts/pull/52
- Squash commit: vultisig/mcp-ts@7004de19
- Reviewer: NeOMakinG (Codex 5.4 adversarial pass)

## Notes

**2026-05-05T00:19:15Z**

demoted p2 → p3 (meh bucket: internal refactor / polish, no user-facing payoff pre-launch)
