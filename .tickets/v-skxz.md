---
id: v-skxz
status: open
deps: []
links: []
created: 2026-05-07T05:40:56Z
type: task
priority: 0
assignee: Jibles
---
# Absorb transfer escape-hatch build tools into execute surface

## Problem
The agent still has a cluster of chain/protocol-specific transfer builders (`build_spl_transfer_tx`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build_xrp_send`, `build_cw20_transfer`, plus possibly adjacent native/Polkadot builders) that survive as gated escape hatches because `execute_send` does not cover their full signing payloads. This keeps legacy tool names, prompt exceptions, schema plumbing, and loop-prone routing alive. Solved means we have a concrete plan to absorb each viable transfer case into the high-level execute surface (`execute_send`, or a deliberately new `execute_transfer`/other `execute_*` if that is cleaner), then retire the redundant `build_*` tools without losing chain coverage.

## Background
### Findings
- Overarching context from the token-cost investigation: the agent should be made cheaper and more reliable by shaping the LLM's environment, not by piling on more prompt warnings. The model should see one coherent high-level action surface wherever possible, not a mix of primary tools and legacy escape hatches.
- Tool-contract philosophy: the LLM should choose the user's transfer intent; backend/tool code should resolve wallet addresses, token metadata, decimals, contract/mint identifiers, base-unit conversion, fees, and signing payload details.
- Prompt-shrink philosophy: every surviving escape hatch tends to preserve prompt exceptions, routing rules, validator branches, and schema cost. Absorption should delete those exceptions only after coverage is proven.
- `agent-backend/internal/service/agent/prompt.go` still globally explains the escape-hatch model:
  - line ~22: prefer high-level `execute_*` tools; per-chain `build_*` tools survive only as narrow escape hatches.
  - line ~95: `build_dot_send` is still called out because `execute_send` does not route Polkadot.
  - lines ~590-592: `build_cw20_transfer` has special guidance to avoid hallucinating USTC/LUNC CW20 contracts.
  - lines ~1035-1047: `tokenTransferEscapeHatchTools` lists `build_spl_transfer_tx`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build_xrp_send`, `build_cw20_transfer` as surviving token-transfer escape hatches.
- `agent-backend/internal/service/agent/turn_state.go` still intentionally allows these through send-intent narrowing:
  - `allowCalldataToolForIntent` keeps `build_spl_transfer_tx`, `build_sui_token_transfer`, `build_ton_jetton_transfer`, `build_trc20_transfer`, and `build_cw20_transfer` for `IntentSend`.
  - tests in `turn_state_test.go` assert those tools are kept for prompts like `send 1 usdc on solana`, `send 1 usdt on tron`, `send 1 usdt on ton`, and `send some kuji to friend`.
- `agent-backend/internal/service/agent/send_intent.go` maps token kinds to preferred build tools:
  - SPL -> `build_spl_transfer_tx`
  - Sui token -> `build_sui_token_transfer`
  - Jetton -> `build_ton_jetton_transfer`
  - TRC20 -> `build_trc20_transfer`
- `mcp-ts/src/tools/index.ts` documents the remaining token-transfer escape hatches as gated `custom_send` because `execute_send` does not yet support them.
- `mcp-ts/src/tools/execute/execute_send.ts` explicitly rejects several cases and points the model toward per-chain builders:
  - SPL token sends on Solana.
  - TRC-20 token sends on Tron.
  - TON jetton sends.
  - Sui token sends.
  - Ripple/XRP sends.
  - Cardano native-asset sends are also explicitly not supported, even though they are not in the original five-tool ticket.
- `mcp-ts/src/tools/send/build-other-send.ts` contains the implementation of several surviving builders:
  - `build_spl_transfer_tx`: SPL token transfer; accepts base `amount` or `amount_usd`; requires `decimals` for fiat path.
  - `build_xrp_send`: XRP Ledger tx args; accepts drops or USD; emits fee/sequence/ledger fields and signing mode.
  - `build_trc20_transfer`: TRC-20 calldata/wire shape; requires contract address and decimals for fiat path.
  - `build_ton_jetton_transfer`: derives jetton wallet via remote lookup; throws if wallet cannot be derived.
  - `build_sui_token_transfer`: Sui token transfer; requires `coin_type`, `decimals`, `symbol` for some paths.
- `mcp-ts/src/tools/send/build-cw20-transfer.ts` handles CW20 transfers on Cosmos/Terra-family chains and includes guardrails against using CW20 for native denoms.
- Tests already exist that can seed parity work:
  - `mcp-ts/tests/token-tools.test.ts` covers canonical payloads for SPL/TRC20/TON/Sui token builders.
  - `mcp-ts/tests/cw20-transfer.test.ts` and `mcp-ts/src/tools/send/__tests__/build-cw20-transfer.test.ts` cover CW20 behavior.
  - `mcp-ts/tests/build-tools-amount-usd.test.ts` covers USD amount handling for `build_xrp_send`, `build_trc20_transfer`, `build_spl_transfer_tx`, `build_ton_jetton_transfer`, `build_sui_token_transfer`.
  - `mcp-ts/tests/scan-request-emission.test.ts` covers scan request shape/unsupported behavior for these builders.
  - `mcp-ts/tests/execute-send.test.ts` already pins the current `execute_send` rejection behavior for unsupported token sends and Ripple.
- Existing ticket `v-ujuc` previously targeted this idea: retire five chain-specific token-layer builders by extending `execute_send`. It was closed/killed because it depended on killed ticket `v-srjz` (App->SDK migration), not because the idea was judged low-value.
- Existing ticket `v-giou` now covers the broader audit lens: identify tool fields/results that expose plumbing to the LLM. The transfer escape-hatches are one of its priority areas, but `v-giou` is discovery; this ticket should produce a concrete absorption plan.
- Follow-up explorer pass confirmed all six scoped tools are still registered in `mcp-ts/src/tools/index.ts` and treated as calldata/signing-card producers via `produces_calldata`.
- Explorer pass also found adjacent candidates:
  - `build_dot_send` is still registered and category-visible for Polkadot when the chain flag is on, but forced `IntentSend` does not allow it.
  - `build_range_transfer` is registered but should not be absorbed into generic send; it belongs to a future Rujira/CCL execute surface.
- Visibility findings:
  - `build_spl_transfer_tx`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, and `build_cw20_transfer` are visible in forced `IntentSend` paths.
  - `build_xrp_send` is category-visible for XRP turns but mostly excluded by forced send narrowing, while `execute_send` currently tells the model to use the XRP builder. This mismatch can cause retry/thrash and should be fixed even before full absorption if XRP remains supported.
  - `build_dot_send` has a similar mismatch: the prompt says to use it for DOT because `execute_send` lacks Polkadot, but forced send narrowing does not allow it.
- Explorer recommended initial implementation order:
  1. Absorb SPL/TRC20/Sui token sends into `execute_send`.
  2. Absorb `build_dot_send` into `execute_send` as a Polkadot family.
  3. Fix XRP routing mismatch immediately, either by allowlisting `build_xrp_send` in forced send as an interim patch or by prioritizing `execute_send` XRP support.
  4. Absorb TON jettons after deciding server-vs-client jetton wallet derivation behavior.
  5. Absorb CW20 with explicit `token_standard=cw20` handling and native-denom guards.
  6. Leave `build_range_transfer` out of generic send consolidation.

## Current Thinking
Where we landed in conversation: this is a real simplification opportunity, but it should be planned carefully rather than treated as a quick prompt trim. The problem is not just token cost. It is that the model sees a mixed contract:

```text
For normal sends use execute_send,
except SPL/TRC20/TON jetton/Sui token/XRP/CW20/Polkadot-like cases use special build_* tools,
but do not use retired generic build_* tools,
and each surviving build_* schema has different plumbing fields and units.
```

The target mental model should be closer to:

```text
For sends/transfers, use one high-level execute tool.
```

Planner should evaluate whether that high-level tool remains `execute_send` or whether the product needs a new `execute_transfer` boundary. Default bias from the current architecture is to extend `execute_send` because it already means end-to-end send/transfer, accepts natural-language amounts, resolves token/gas/nonce metadata, emits `ExecutePrepResult`, and is wired into existing send cards. A new `execute_transfer` should only be introduced if it removes real ambiguity or cleanly separates generic token transfer from wallet-native send semantics.

Expected value:
- Prompt shrink: medium. Removing this would delete escape-hatch tool lists and special prompt sections, probably not the largest system-prompt win but still meaningful.
- Tool-schema shrink: medium/high on send turns. Send intent currently keeps these builders visible when matching token families, and each builder carries its own schema/description.
- Reliability: high. These builders ask the model for plumbing (`from`, decimals, contract addresses, coin types, jetton masters, base units), which is exactly the class of fields we want backend/tool code to resolve.
- Loop reduction: medium/high. Current flows can involve token search, amount conversion, missing-decimal errors, bad-from validation, and final narration over legacy payloads.

Candidate absorption ranking to validate:
1. **TRC-20 -> `execute_send`**: likely high value. It is a common user flow (`send USDT on Tron`) and currently exposes contract/decimal/base-unit plumbing.
2. **SPL -> `execute_send`**: likely high value. Common user flow (`send USDC on Solana`) but may require TokenProgram/ATA payload/signing support.
3. **TON jetton -> `execute_send`**: likely medium/high. Jetton wallet derivation is already centralized in the builder but may rely on remote lookup and per-owner wallet state.
4. **Sui token -> `execute_send`**: likely medium. Needs deterministic coin selection and coin type handling.
5. **XRP native -> `execute_send`**: likely high reliability value, but may be blocked by signing_pub_key / BIP-32 derivation concerns noted in current tests/comments.
6. **CW20 -> maybe `execute_send`, maybe `execute_contract_call`/new execute path**: likely valuable but needs more care because CW20 is contract-token transfer on Cosmos/Terra-family chains and has native-denom hallucination risks.
7. **Polkadot native (`build_dot_send`) -> `execute_send`**: adjacent candidate. Prompt still calls it out because `execute_send` classifyFamily does not route Polkadot. Include in audit if still LLM-visible for send turns.
8. **Other native per-chain builders (`build_solana_tx`, `build_trx_send`, `build_ton_send`, `build_sui_send`, etc.)**: verify whether current `execute_send` fully supersedes them in actual LLM-visible paths. They may be registered/internal/historical but should not survive normal send intent if execute coverage is complete.

This ticket should not implement all absorptions in one PR. It should produce an ordered implementation plan and, if appropriate, split into per-chain tickets. The first implementation ticket should target one high-value/common chain with low signing risk, then repeat the pattern.

## Constraints
- Do not delete any `build_*` tool until the execute replacement is proven equivalent or deliberately better for every supported payload shape users can currently reach.
- Preserve mobile signing/card payload compatibility. If `execute_send` emits a new tx shape, `vultiagent-app` parsers/signers must already handle it or be updated in the same rollout.
- Preserve `scan_request`/Blockaid behavior and unsupported-scan semantics.
- Preserve amount semantics: users give decimal/fiat/max/percent where supported; the LLM should not have to convert to base units or call `convert_amount` for normal transfers.
- Preserve server-side sender/from injection. The model should not be asked to copy active vault addresses into the tool input.
- If the tool needs token metadata, prefer token registry/search/balance context/server resolution over model-supplied contract/decimals.
- If a chain requires data that genuinely cannot be derived server-side, the plan must identify the missing resolver rather than pushing the field back onto the LLM.
- Historical conversations and old tool parts may still exist. Deletion/retirement plan must account for rendering/replay behavior.

## Assumptions
- The current `custom_send` gating reduces always-on exposure, so this is not the biggest normal-path token-cost lever compared with system prompt reduction and deterministic final responses.
- The killed `v-srjz` dependency means prior app/SDK migration assumptions may no longer hold. The planner must re-check current app/signing support rather than inheriting `v-ujuc`'s dependency graph blindly.
- `execute_send` is still the preferred absorption target unless evidence shows a new `execute_transfer` would be cleaner.
- The existing tests provide enough parity scaffolding to compare legacy builder output against the execute replacement.

## Non-goals
- Do not absorb Rujira, Pumpfun, Polymarket, THORChain LP, or other protocol-specific DeFi builders in this ticket. Those are separate protocol surfaces.
- Do not solve general progressive prompt disclosure; that belongs to `v-vpeb`.
- Do not solve all tool-contract cleanup; that belongs to `v-giou`.
- Do not implement deterministic final responses; that belongs to `v-glle`.
- Do not make Polkadot or Cardano asset support release-critical unless product scope says so.

## Dead Ends
- Retiring the builders before execute coverage exists was rejected in prior ticket `v-ujuc`: it would remove working coverage or require throwaway app-side plumbing.
- Treating this as only a prompt deletion was rejected. The prompt text exists because real tool/schema gaps remain; deleting the text without changing the tool surface would push ambiguity into model behavior.
- Keeping these forever as harmless gated tools is not ideal. Even if gated, they preserve legacy schemas, prompt exceptions, validator branches, card fallback paths, and model routing complexity.

## Open Questions
- Should the destination execute surface be `execute_send` or a new `execute_transfer`? What exact distinction would justify a new tool?
- Which candidates are currently visible to the LLM in real send turns under `tool_filter.go`, `turn_state.go`, and launch-surface flags?
- Which builders are still functionally required by the current app signing path, and which are now only historical/dead weight?
- Is `build_cw20_transfer` best modeled as a token send, a contract call, or a Cosmos-specific execute transfer?
- Should XRP native send be absorbed into `execute_send` before token-layer tools, or is `signing_pub_key` still a blocker?
- Should Polkadot native send and Cardano native assets be folded into the same roadmap or separate tickets?
- What is the safest first implementation slice: TRC-20, SPL, TON jetton, XRP, Sui token, CW20, or Polkadot?
- What telemetry do we have on usage frequency of these tools in production/dev exports, and should that drive prioritization?
