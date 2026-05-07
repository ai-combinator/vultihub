---
id: v-bnsg
status: closed
deps: []
links: []
created: 2026-05-05T02:13:46Z
type: feature
priority: 2
assignee: Jibles
---
# Skip Go: Tier 1+2 — expand chain coverage + wire USDC approve flow

## Problem

The v-rjfc Skip Go stack only ships native ETH on Ethereum (chain `1`) → LUNA today. Real Vultisig users want USDC routes (highest-volume, all CCTP chains) and other native gas tokens (MATIC/AVAX). USDC routes blocked at the validator (`required_erc20_approvals` fail-closed); other natives blocked at the EVM router allowlist. "Solved" means: native MATIC + AVAX → LUNA, OSMO → LUNA, LUNA→LUNC, and USDC.{eth,base,arb,op,polygon,avax} → LUNA all sign + broadcast end-to-end.

## Background

### Findings

**Allowlist surface today** (vultiagent-app):
- `SKIP_GO_EVM_ROUTERS` at `src/features/agent/lib/skipDispatchValidators.ts:61-69` only contains chain `'1'` → `0xb773bcc5b325ad9ac6b36e1a046ad4466833a16e` (Ethereum Axelar gateway).
- `SKIP_COSMOS_ENTRY_CONTRACTS` at `:130-132` only contains `phoenix-1` → `terra13jfx06k...spee8lp` (LUNA Skip entry).
- `validateSkipEvmDispatch` at `:204` fail-closes on `evmTx.required_erc20_approvals.length > 0`.

**Existing approve+swap rails** (used today by THORChain / 1inch / Morpho):
- `signAndBroadcast.ts:357-365` — `kind: 'evm'` case signs `parsed.approvalTx` first, waits for receipt, then signs the swap.
- `txProposal/build.ts:724,738` — synthesizes `approvalTx` from upstream `data.approval_tx` (ServerEvmTx).
- `txFlowReducer.ts:17` — `SigningStep = 'approve' | 'sign' | 'broadcast'`.
- `useTransactionFlow.ts:202` — single-claim multi-tx: ONE user tap fires both approve + swap signatures.

**Skip dispatch path is separate**:
- `signAndBroadcast.ts:530+` `case 'skip_swap'` calls `signBroadcastServerEvmTx` directly with `evm_tx` from the Skip envelope.
- `validateSkipEvmDispatch` runs at line 552; rejects `required_erc20_approvals.length > 0` at `skipDispatchValidators.ts:204`.

**ERC-20 approve calldata encoder lives in mcp-ts**, not vultiagent-app:
- `mcp-ts/src/tools/execute/execute_swap.ts:165` uses `abiEncode('function approve(address,uint256)', [spender, amount])` from `@vultisig/sdk`.
- Approval amount convention: **exact-amount, NOT MaxUint256** (`execute_swap.ts:175-184`), with USDT-style `approve(0)` reset path when current allowance > 0 and < amount.
- mcp-ts emits one `approval_tx` per turn — multi-step reset is serialized across turns, not arrayed.

**Skip API live probes (2026-05-05)** — concrete shapes:

| Probe | Result |
| --- | --- |
| MATIC(137) → uluna, 5e18 | OK 1-tx, `evm_tx.to = 0xD87E98798b1441c4b7fB7ABd918F90e42ae3D735`, no approvals, osmosis-poolmanager |
| AVAX(43114) → uluna, 1e17 | OK 1-tx, `evm_tx.to = 0xBeB12d8861765c850E8426Cc5c6a5207222f2477`, no approvals, osmosis-poolmanager |
| USDC.base(8453, 0x833589fcd6edb6e08f4c7c32d4f71b54bda02913) → uluna, 5e6 | OK 1-tx, `evm_tx.to = 0x569594F181ac9AbdE3eBf89954F3D3Af40d63600`, **single approval at `evm_tx.required_erc20_approvals[0]`** (NOT top-level — that's null), `spender == evm_tx.to`, `amount = "5000000"` (decimal exact), terra-astroport venue |
| USDC.arb(42161, 0xaf88d065e77c8cc2239327c5edb3a432268e5831) → uluna, 5e6 | OK 1-tx, `evm_tx.to = 0xEF1cE4489962E6d6D6BE8066E160B2799610cB85` (different per chain), single approval, `spender == evm_tx.to` |
| ATOM(cosmoshub-4) → uluna | OK 1-tx, `MsgTransfer` (NOT `MsgExecuteContract`). Sender is user; receiver is phoenix-1 entry contract on dest side via PFM memo. **Bypasses `SKIP_COSMOS_ENTRY_CONTRACTS` entirely** — promotes to `kind: 'ibc_transfer'` per `txProposal/build.ts:381-409`. |
| INJ(injective-1) → uluna | OK 1-tx, same `MsgTransfer` shape as ATOM |
| OSMO(osmosis-1) → uluna | OK 1-tx, `MsgExecuteContract` against `osmo10a3k4hvk37cc4hnxctw4p95fhscd2z6h2rmx0aukc6rm8u9qqx9smfsh7u`. **The only cosmos source that needs an entry-contract allowlist entry.** |
| NTRN(neutron-1) → uluna | "no routes found" |
| KUJI(kaiyo-1) → uluna | bech32 fail across all generated addrs — Skip doesn't accept kuji-prefix bech32 for this dest today |
| phoenix-1 → columbus-5 (LUNA→LUNC) | OK 1-tx, `MsgTransfer` on phoenix-1 (PFM via osmosis channel-1, embedded receiver `osmo10a3k4hvk...`) |
| **columbus-5 → phoenix-1 (LUNC→LUNA)** | **`/route` returns a quote BUT `/msgs_direct` returns "no routes found"** — even with osmosis hop addr supplied. **Effectively unavailable today.** |

**Working broadcast receipts on the v-rjfc stack today:**
- ETH-on-Ethereum → LUNA: `0x9696...c567` (confirmed)
- ETH → LUNC: `0xdcc9...693e` (confirmed, via osmosis-poolmanager)

Both prove the existing rails (single-tx Skip envelope, native EVM source, Ethereum router allowlist entry) work end-to-end.

### External Context

- Skip Go API: `https://api.skip.build/v2/fungible/{route, msgs_direct}`. Catalog evolves silently (the cross-research agent confirmed native ETH→cosmos was once available globally and got dropped without doc updates). Don't trust docs — probe.
- Repo's pr-receipts skill at `vultiagent-app/.agents/skills/pr-receipts/SKILL.md`. Upload helper at `scripts/upload-receipt.sh` — ephemeral default (litterbox 72h).
- Each contributor brings their own test vault per `AGENTS.md:118-130` and `.env.example:40-41` (`DETOX_IMPORT_VAULT_FILE` + `VULTISIG_FAST_VAULT_PASSWORD`). Funding USDC across CCTP chains + LUNA on phoenix-1 is on the implementer.

## Current Thinking

### Tier 1 — pure allowlist additions (low risk, signing path untouched)

1. **`SKIP_GO_EVM_ROUTERS` adds**:
   ```ts
   '137':   Object.freeze(['0xd87e98798b1441c4b7fb7abd918f90e42ae3d735']),  // Polygon Skip Go router, probed 2026-05-05
   '43114': Object.freeze(['0xbeb12d8861765c850e8426cc5c6a5207222f2477']),  // Avalanche Skip Go router, probed 2026-05-05
   ```

2. **`SKIP_COSMOS_ENTRY_CONTRACTS` adds**:
   ```ts
   'osmosis-1': Object.freeze(['osmo10a3k4hvk37cc4hnxctw4p95fhscd2z6h2rmx0aukc6rm8u9qqx9smfsh7u']),
   ```
   Don't add cosmoshub-4 / injective-1 / neutron-1 / kaiyo-1 — see Dead Ends.
   Don't add columbus-5 — see Dead Ends.

3. **Existing skipDispatchValidators tests** at `src/features/agent/lib/__tests__/skipDispatchValidators.test.ts` need updates (the "rejects unknown chain" test currently asserts only `'1'` is allowed; flip to "accepts new chains" for 137 + 43114 + osmosis-1).

### Tier 2 — wire ERC-20 approval into the existing approve-rails

**Where the approval is synthesized**: in **mcp-ts** (not vultiagent-app), mirroring `execute_swap.ts:165`. When Skip's `evm_tx.required_erc20_approvals[0]` is present, mcp-ts emits a `data.approval_tx: ServerEvmTx` field alongside the swap envelope — same shape THORChain / 1inch already use. The shape:
```ts
data.approval_tx = {
  to: required.token_contract,
  value: '0',
  data: encodeApprove(required.spender, required.amount),  // 0x095ea7b3 + spender + amount
  chain_id: evm_tx.chain_id,
}
```
Use **exact amount** (matching repo convention at execute_swap.ts:175-184). No MaxUint256.

**Why mcp-ts and not vultiagent-app**: the encoder + amount-policy is already in mcp-ts via `@vultisig/sdk`'s `abiEncode`. Putting it in vultiagent-app would duplicate the encoder, drift from the repo convention, and add ethers/viem to the app for nothing.

**Where it's signed** (vultiagent-app):
In `signAndBroadcast.ts` `case 'skip_swap'` (around line 530), BEFORE signing the swap leg: when `parsed.approvalTx` is present, sign + broadcast the approval, `await waitForReceipt(...)`, THEN sign the swap. Mirror the existing `case 'evm'` block at lines 357-365.

**Don't reroute Skip envelopes through `kind: 'evm'`** (Dead End below) — that loses Skip-specific router allowlist enforcement.

**Drop `validateSkipEvmDispatch:204` fail-closed**: relax to "fail-closed only if approval wasn't synthesized upstream" (i.e., still reject `required_erc20_approvals` when `parsed.approvalTx` is missing — defense in depth).

### LLM prompt: chain-support gating

In `agent-backend/internal/service/agent/prompt.go` near the existing Base→LUNA worked example at `prompt.go:298`, add a negative steer:
- Native ETH on Arbitrum (42161) / Base (8453) / Optimism (10) → Cosmos: NO route. Suggest "use Ethereum mainnet or USDC instead."
- Native BNB on BSC (56) → Cosmos: NO route.
- LUNC → LUNA: NO route via Skip msgs_direct today. Refuse cleanly.

Cleaner UX than fail-closed dispatch errors.

### Receipts plan

Each contributor brings a test vault funded across the chains they're testing. Required scenarios (broadcast, not just card-render):
- ✓ ETH-on-Ethereum → LUNA (`0x9696...c567`, banked on PR #387)
- ✓ ETH → LUNC (`0xdcc9...693e`, banked on PR #387)
- MATIC.polygon → LUNA (Tier 1)
- AVAX.avax → LUNA (Tier 1)
- OSMO → LUNA (Tier 1, exercises new `osmosis-1` cosmos entry contract)
- ATOM → LUNA (Tier 1, exercises IBC-promotion path — *not* a Skip cosmos validator path)
- USDC.base → LUNA (Tier 2, primary)
- USDC.arb → LUNA (Tier 2, second-chain confirmation)
- LUNA → USDC.eth (CCTP-out, source-side phoenix-1 entry contract)
- LUNA → LUNC (one-way only — LUNC → LUNA dead per probes)

## Constraints

- **MPC ceremonies are slow.** Use `useTransactionFlow.ts:202`'s single-claim-multi-tx pattern so ONE user tap covers both approve + swap signatures (no double-prompt).
- **Validator strictness must hold on the swap leg.** Tier 2 must keep `validateSkipEvmDispatch` engaged: router allowlist + chain_id match. Don't bypass by rerouting to `kind: 'evm'`.
- **Approval amount = exact, not MaxUint256.** Repo convention at `execute_swap.ts:175-184`.
- **No new MCP tool.** v-rjfc retired `build_skip_swap` intentionally. Tier 2 wiring lives in `execute_swap`'s Terra branch + the existing `approval_tx` contract — not a separate Skip-approval tool.
- **No in-app calldata encoder.** Reuse mcp-ts's existing `abiEncode` path; don't add ethers/viem to vultiagent-app.

## Assumptions

- Skip's catalog stays stable enough to keep the probed router addresses valid through merge. Mitigation: scripts/qa canary or CI re-probe (see Open Question #4).
- Single approval per route (`required_erc20_approvals.length === 1`) — true for all probed CCTP sources today. If a route ever returns >1, the existing single-`approvalTx` field shape would need extension to an array.
- LUNA → LUNC keeps working at `/msgs_direct` time (probe shows OK today, not yet broadcast-tested on the rebased stack).
- The IBC-dispatch path (`buildSignBroadcastCosmosIbcTransfer` for ATOM/INJ → LUNA) doesn't need new validation for THIS scope. But triage flagged: it doesn't have a per-chain receiver-allowlist, so a malicious mcp-ts could redirect funds via the `receiver` field on MsgTransfer without hitting any Skip validator. See Open Question #1.
- Test vaults can be funded with USDC + LUNA + cosmos sources by the implementer; no shared committed vault per AGENTS.md:118-130.

## Non-goals

- Native ETH on Arbitrum / Base / Optimism → Cosmos. Skip has no route. Block at the prompt level instead.
- Native BNB on BSC. Skip rejects with "Source token not found" — denom-name issue, not officially supported.
- Solana ↔ Terra. CCTP path is unstable in probes; defer.
- LUNC ↔ native EVM. No path at all today.
- CW20 swaps on columbus-5. Skip's router never picks them.
- Permit2 / off-chain permit signatures. Probes show Skip emits standard ERC-20 approval txs today.
- Multi-tx Skip envelopes. All probed routes today are `tx_count: 1`. The `MultiTxNotSupportedError` guard at `signAndBroadcast.ts:535` stays in place; lifting it is a separate scope.
- A second MCP tool for Skip approvals. Reuse `data.approval_tx` rail.

## Dead Ends

- **Native ETH on Arbitrum → LUNA.** Live-probed today, returns "no routes found". The Axelar gas-token bridge path Skip uses for Eth/Polygon/Avalanche → LUNA doesn't exist for L2 native gas. Don't try to allowlist Arbitrum's router as a workaround — there's no router because there's no route.
- **LUNC → LUNA via Skip msgs_direct.** `/route` advertises a quote, but `/msgs_direct` returns "no routes found" even with osmosis hop supplied. Don't add columbus-5 to `SKIP_COSMOS_ENTRY_CONTRACTS` as a source — dead code today.
- **Adding cosmos-source allowlist for ATOM / INJ / NTRN / KUJI.** Triage confirmed these promote to `kind: 'ibc_transfer'` (per `txProposal/build.ts:381-409`) and bypass `validateSkipCosmosWasmDispatch` entirely. Adding entries to `SKIP_COSMOS_ENTRY_CONTRACTS` for these chains is dead code. Only `osmosis-1` actually exercises that allowlist.
- **Rerouting Skip envelopes through `kind: 'evm'` to use existing approve+swap rails.** Loses Skip-specific router allowlist enforcement on the swap leg. Pick the inline-approve approach in `case 'skip_swap'` instead.
- **Synthesizing the approve calldata in vultiagent-app.** Duplicates mcp-ts's encoder, risks drifting from the repo's exact-amount + USDT-reset convention, and adds ethers/viem dependency for one shape.
- **MaxUint256 approvals.** Repo convention is exact-amount with reset-on-decrease; don't break it.

## Open Questions

1. **IBC-dispatch receiver allowlist for Skip-routed transfers.** Triage flagged this as the unguarded surface that lets ATOM/INJ → LUNA bypass Skip validation entirely. Defending against malicious mcp-ts requires checking the MsgTransfer's `receiver` against a known set of Skip cosmos entry contracts per dest chain. **Should we add a `SKIP_IBC_RECEIVERS` allowlist in this scope, or open a follow-up?**
2. **Where in mcp-ts to synthesize `approval_tx`** — directly in `execute_swap.ts`'s Terra branch (which is already busy), or factor a small helper that the Terra branch calls? Both viable; planner picks.
3. **Stepper rendering for the approve+swap flow** — does the existing single-claim-multi-tx pattern at `useTransactionFlow.ts:202` already render two stepper entries (Approve / Sign), or does it collapse to one? If it's currently single-step rendering, do we want to surface the approval as a visible step for transparency or keep collapsed for brevity?
4. **Re-probe cadence** — how often does CI re-probe Skip's `/route` to catch silent catalog drops (the way native-ETH → cosmos broke globally per the earlier research)? Add a scheduled GH Actions canary, or skip and rely on user error reports?
5. **LUNA → LUNC live-broadcast confirmation** before merging — `/msgs_direct` advertises the route; we should capture a real broadcast receipt to confirm the route Skip's `/route` advertises actually executes, before relying on it in tests.
