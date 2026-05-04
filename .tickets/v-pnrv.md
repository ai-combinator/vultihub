---
id: v-pnrv
status: open
deps: []
links: []
created: 2026-04-27T20:52:16Z
type: task
priority: 2
assignee: Jibles
external-ref: vultisig/mcp-ts#52
---
# mcp-ts cosmos balance: investigate Rujira enriched-path partial-response reconciliation

## Objective

Investigate whether the Rujira enriched-balance path in `src/tools/balance/cosmos-balance.ts:80-99` can silently drop denoms when Rujira returns a partial set vs. the underlying chain bank module. If real, implement a union-merge reconcile step so the response shape never regresses below what raw REST exposes.

Follow-up surfaced by Codex 5.4 adversarial review on vultisig/mcp-ts#52 (now merged as squash commit vultisig/mcp-ts@7004de19). Tagged "follow-up" by reviewer NeOMakinG. Codex flagged this as a MEDIUM correctness concern.

## Context & Findings

### Pre-existing, not introduced by #52

This is **pre-existing behavior** — the merged PR did not introduce or change lines 80-99. Pre-PR raw fallback was `{denom, amount}`; PR #52 unified the raw fallback to `{denom, symbol, amount}`. The partial-loss issue (if real) predates the unification work.

### The concern

In `src/tools/balance/cosmos-balance.ts:80-99`, when Rujira enrichment succeeds we return `client.deposit.getBalances(address)` wholesale:

```ts
if (config.rujiraEnrich) {
  try {
    const client = await getRujiraClient()
    const enriched = await client.deposit.getBalances(address)
    if (enriched.length > 0) {
      return jsonResult({
        address,
        balances: enriched.map(...)
      })
    }
  } catch {
    // Fall through to raw response
  }
}
```

The raw REST fallback only triggers on `enriched.length === 0` or thrown error — **not** on "enriched returned non-empty but missing some denoms." If Rujira returns a partial set vs. the chain bank module, denoms could be silently dropped from the response (raw REST has them, enriched does not).

### Status: unverified

Whether `client.deposit.getBalances` actually returns partial sets vs. complete sets needs a live test against thornode plus a read of the `@vultisig/rujira` SDK source. The `getRujiraClient` lives in `src/lib/rujira.ts` (uses `rcpEndpoint: https://rpc.ninerealms.com`).

## Investigation Steps

1. **Live comparison** — pick a known multi-denom THOR address (one with secured assets, RUNE, TCY, RUJI, etc.) and:
   - Hit Rujira: `client.deposit.getBalances(address)`
   - Hit raw REST: `GET https://thornode.ninerealms.com/cosmos/bank/v1beta1/balances/{addr}`
   - Diff the denom sets

2. **Decision point:**
   - If Rujira returns a strict superset or equal set across all addresses tested → close as non-issue, leave a note in the file
   - If Rujira returns a subset (e.g., omits secured assets or new denoms it doesn't know about) → proceed to step 3

3. **Reconcile implementation (if needed):**
   - Union-merge enriched + raw, keyed by denom
   - Enriched fields win for denoms present in both
   - Raw-only entries get the passthrough shape introduced by PR #52: `{denom, symbol: denom, amount}` (mirrors the merged PR's contract)
   - Order: enriched-first, raw-only appended

## Files

**Investigation only:**
- `src/tools/balance/cosmos-balance.ts:80-99` — area under review
- `src/lib/rujira.ts` — `getRujiraClient` setup
- `node_modules/@vultisig/rujira/...` — SDK source for `deposit.getBalances` to understand the denom-set semantics

**Modified (only if reconcile is warranted):**
- `src/tools/balance/cosmos-balance.ts` — add union-merge step between enriched and raw before returning
- New unit/integration test covering a mocked partial-set Rujira response + complete raw response → asserts union output

## Acceptance Criteria

- [ ] Live comparison performed against at least one multi-denom THOR address (RUNE + TCY + RUJI + secured assets if reachable)
- [ ] Findings documented in ticket notes (`tk note`) with denom-set diff
- [ ] **If non-issue**: ticket closed with a one-line code comment in `cosmos-balance.ts:80-99` documenting that Rujira's enriched response was verified to be a superset/equal set as of the investigation date
- [ ] **If real issue**: reconcile step implemented; raw-only denoms appear in the response with the PR #52 passthrough shape; unit test covers partial-set scenario
- [ ] If implementation done: `pnpm test`, `pnpm typecheck`, `pnpm lint` clean
- [ ] PR description (if opened) references vultisig/mcp-ts#52 and merged commit vultisig/mcp-ts@7004de19, calls out the issue is pre-existing

## Gotchas

- Don't assume Rujira coverage from name alone — secured assets and freshly-listed denoms are the most likely subset case. Pick a test address that has at least one of each kind.
- The reconcile path must preserve ordering expectations callers may rely on. Enriched-first is the safer default.
- The `if (enriched.length > 0)` guard at line 84 means an empty Rujira response already falls through to raw REST. Don't break that — only the "non-empty but partial" case is at issue.
- If Rujira returns a different denom-key normalization (e.g., trims a prefix) the merge key needs care. Compare denoms exactly as the upstream APIs return them before normalizing.
- This change touches the THORChain and (if `rujiraEnrich` is enabled elsewhere) MAYA balance paths — list affected chain IDs in the PR body per the Vultisig safety rule.

## References

- Merged PR: https://github.com/vultisig/mcp-ts/pull/52
- Squash commit: vultisig/mcp-ts@7004de19
- Reviewer: NeOMakinG (Codex 5.4 adversarial pass) — flagged MEDIUM correctness
- Rujira SDK: `@vultisig/rujira`, `rcpEndpoint: https://rpc.ninerealms.com`
