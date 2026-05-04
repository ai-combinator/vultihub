---
id: v-goft
status: closed
deps: [v-llqd, v-qjvi]
links: []
created: 2026-04-28T09:17:03Z
type: bug
priority: 1
assignee: Jibles
---
# vultiagent-app: consume approvalTxArgs array (USDT-style approve(0)+approve(N))

## Objective

Wire the vultiagent-app consumer side of the USDT-style allowance reset that mcp-ts#49 now emits server-side. After this lands, swaps of USDT / KNCv1 / OMG / BNT / similar non-zero-allowance-resetting ERC-20s with a pre-existing partial allowance succeed end-to-end instead of reverting at broadcast.

## Context

mcp-ts PR #49 (v-qjvi) added Finding #4: emit `approvalTxArgs: ExecuteTxArgs | ExecuteTxArgs[]` from `execute_swap` — array form is `[approve(0), approve(N)]` when `0 < existing_allowance < amount`. Required because USDT (and a handful of other ERC-20s) revert when `approve(N)` runs over a non-zero existing allowance.

vultiagent-app PR #242 (v-llqd) `executePrepAdapter.ts` only handles the single-object case. Array branch is silently dropped, so a USDT-style swap currently signs the main tx with the stale existing allowance and reverts.

**Not a regression in #242** — main also handles only single-object (`signAndBroadcast.ts:206`). PR #242 mirrors legacy behavior; this ticket completes a brand-new capability, not a fix for something #242 broke.

## Design

Three files in vultiagent-app, ~80-120 LoC + tests:

1. src/features/agent/lib/executePrepAdapter.ts — detect Array.isArray(approvalTxArgs); surface as approval_tx_array (or rename to approvalTxs). 8-15 LoC.

2. src/features/agent/lib/txProposal.ts — extend the kind: 'evm' variant of ParsedServerTx with approvalTxs?: ServerEvmTx[] parallel to existing approvalTx. Validate each element. Pattern already used for data.transactions (Array.isArray defensive checks at txProposal.ts:166 and :278). 12-20 LoC.

3. src/features/agent/lib/signAndBroadcast.ts — after the existing single-object if (parsed.approvalTx) branch (line 206), add a parallel if (parsed.approvalTxs) loop that signs + waits for receipt between each leg. waitForEvmReceipt (90s timeout, exponential backoff) and signBroadcastServerEvmTx (single-tx) already exist in txSigning.ts. 12-18 LoC.

Out of scope: useTransactionFlow port; non-EVM approval flows; allowance-reset detection client-side (server is source of truth).

## Acceptance Criteria

- [ ] executePrepAdapter rejects malformed array elements (same shape gate as single-object).
- [ ] executePrepAdapter.test.ts covers: array of 2 (USDT pattern), array of 1 (edge), array with malformed element rejected.
- [ ] signAndBroadcast signs each approval leg sequentially with await waitForEvmReceipt between each; aborts remaining legs + main tx if any approval fails.
- [ ] Single-object call sites unchanged (no breaking change on the common path).
- [ ] Manual: dogfood a USDT->ETH swap on a wallet with a small existing USDT allowance — verify approve(0) + approve(N) + swap all fire and the swap doesn't revert.
