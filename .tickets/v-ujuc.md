---
id: v-ujuc
status: open
deps: [v-srjz]
links: []
created: 2026-04-20T06:56:39Z
type: task
priority: 2
assignee: Jibles
---
# Retire token-layer build_* tools via execute_send extension

## Objective

Retire five chain-specific token-layer MCP tools (`build_spl_transfer_tx`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build_xrp_send`) by extending `execute_send` to cover their capabilities. Gated on the App→SDK migration completing (ticket `v-srjz`), so this work is mcp-ts + parity tests only, not app-side plumbing.

## Context & Findings

These five tools each handle a chain+token-layer combination that `execute_send` currently rejects with a pointer message because the wallet's signing path doesn't handle token-specific payload fields yet:
- **SPL**: TokenProgram instructions
- **TRC-20**: `function_selector`, `parameter`, `fee_limit_sun`
- **TON jettons**: `jetton_wallet` per-owner derivation
- **Sui tokens**: `coin_type` + Move-call payload
- **XRP**: `signing_pub_key` via BIP-32 derivation

SDK partial state from recon:
- **TON jettons**: likely already done at resolver level (`ton/index.ts:60-79` emits `jettonAddress`); just need app/SDK wiring verification
- **Tron TRC-20**: fee computation exists (`tron/fee.ts:34-65`, builds `function_selector` + parameter); protobuf needs extension
- **Solana SPL**: ATA fetching exists (`solana/index.ts:35-51`); TokenProgram instruction builder missing
- **Sui tokens**: all-coins fetch exists (`sui/index.ts:22`); no coin-type filtering
- **XRP**: BIP-32 child-key derivation not threaded through prep; IOU logic absent

After migration, the retirement pattern per chain is: extend `execute_send` handler → parity test (Layer 1 + 2 from ticket `va-tixe`'s harness) → remove `build_*_transfer` tool. Estimated ~1 day per chain × 5 = ~1 week once migration is done. Sequence easiest-first: TON jettons → TRC-20 → SPL → Sui → XRP.

**Rejected approaches:**
- Retiring before migration — would require per-tool app-side plumbing that gets ripped out during migration (~12-16 days of partially-rotting work vs ~5 days after migration).
- Folding into `va-tixe` — different timeline (gated on migration), different touch points (mcp-ts-heavy vs backend+app), cleaner to track separately.

**Out of scope:** Rujira family, PumpFun, `sign_typed_data`, `build_custom_tx`, `build_evm_tx` — these are genuine low-level primitives or protocol-specific tools, kept regardless.

## Files

- `mcp-ts/src/tools/execute/execute_send.ts` (or equivalent handler) — extend chain switch for SPL/TRC20/jetton/Sui-token/XRP transfers
- `mcp-ts/src/tools/send/build-spl-transfer-tx.ts`, `build-trc20-transfer.ts`, `build-ton-jetton-transfer.ts`, `build-sui-token-transfer.ts`, `build-xrp-send.ts` — tools to retire
- `mcp-ts` test harness — reuse Layer 1 parity infra from `va-tixe`
- `agent-backend` allowlist — remove retired tool registrations
- `vultiagent-app/src/features/agent/lib/toolUIRegistry.ts` — verify `build_*` wildcard still catches anything transitional, trim if needed
- `vultisig-sdk/packages/core/mpc/keysign/chainSpecific/resolvers/{solana,tron,ton,sui,ripple}/*` — resolvers may already be ready post-migration; extend where gaps remain

## Acceptance Criteria

- [ ] Depends on `v-srjz` (app→SDK migration) marked complete before starting
- [ ] `execute_send` handles SPL, TRC-20, jetton, Sui-token, XRP token transfers
- [ ] Layer 1 parity tests pass for each: `execute_send` output == `build_*_transfer` output for equivalent intent (byte-equivalent or justified drift documented)
- [ ] Layer 2 client-side test confirms `parseServerTx` handles both outputs identically
- [ ] All 5 `build_*_transfer` tools removed from mcp-ts registration + agent-backend allowlist
- [ ] No functional regression: SPL send, TRC-20 send, TON jetton send, Sui token send, XRP send all work via `execute_send`
- [ ] PR description enumerates each retired tool and its `execute_send` replacement path

## Gotchas

- TON jetton wallet derivation is per-owner — don't cache across users.
- TRC-20 `fee_limit_sun` depends on contract + current network energy pricing; fetch per call, don't hardcode.
- SPL Token-2022 is distinct from classic SPL — at minimum support classic; Token-2022 can be a follow-up.
- Sui coin selection must be deterministic for the same intent or Layer 1 parity tests flap.
- XRP IOU sends are a separate surface from native XRP — this ticket is XRP sends only; IOU is a follow-up.
- Client `validateTxArgsShape` is permissive for solana/ripple/tron/ton/sui families — don't tighten in this ticket.
