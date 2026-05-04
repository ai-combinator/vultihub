---
id: v-uwaj
status: open
deps: []
links: []
created: 2026-04-28T10:49:19Z
type: feature
priority: 2
assignee: Jibles
---
# mcp-ts execute_send: wire non-EVM balance lookups for max/percent amounts

## Objective

Make max / percent amount inputs ("max", "50%", "0.5x") work on every chain family supported by execute_send, not just EVM. Currently the resolver text-rejects max/percent on UTXO / Cosmos / Solana / Tron / TON / Sui / Cardano because there's no BalanceLookup wired for those families — the user has to specify a numeric amount.

## Context

execute_send pre-rejects max/percent amount inputs on non-EVM chains in `src/tools/execute/execute_send.ts:466-481` (the F7 finding from PR #49's review loop). The reason is the EVM path has a viem-based on-chain balance lookup that resolves max/percent to base units; other families don't have an equivalent wired into `_amountResolver`.

Today's behavior: user says "send max BTC to bc1...", model calls execute_send, the prep tool returns a text error pointing the user to specify a numeric amount. UX is fine but limiting.

## Scope

For each non-EVM chain family supported by execute_send, wire a BalanceLookup adapter:
- UTXO: `get_btc_balance` / equivalent — already exposed elsewhere in mcp-ts.
- Cosmos family: REST `/cosmos/bank/v1beta1/balances/{address}` per chain.
- Solana: SPL balance lookup (native SOL only — token sends are blocked separately on v-srjz).
- Tron: TronGrid balances endpoint.
- TON / Sui / Cardano: native balance via existing tooling.

`_amountResolver` already handles the percent/max math once a `currentBalance` value is available; this ticket just plugs in the per-family lookup.

## Acceptance criteria

- [ ] "send max ATOM on Cosmos to cosmos1..." resolves to a numeric amount in execute_send's prep output (less reserved gas).
- [ ] Same for Bitcoin / Solana / Tron / TON / Sui / Cardano.
- [ ] The text-rejection branch at execute_send.ts:466-481 is removed; the resolver does the work.
- [ ] Per-family integration tests cover one max + one percent case each.

## Blockers

None. Just plumbing.

## Out of scope

- Token-layer (SPL, TRC-20, Sui token, Cardano asset, TON jetton) — blocked on v-srjz.
- Cosmos CW20 — `execute_send`'s cosmos branch only handles native bank sends.
