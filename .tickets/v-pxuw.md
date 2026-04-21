---
id: v-pxuw
status: closed
deps: []
links: []
created: 2026-04-16T03:35:50Z
type: task
priority: 1
assignee: Jibles
external-ref: vultihub#17
---
# Tool Consolidation — Collapse 136 tools to ~25 for better selection accuracy

## Problem Statement

The agent exposes ~136 tools to the LLM, which causes tool selection failures (especially on Gemini Flash), burns ~68K tokens per turn on schema overhead, and forces multi-step LLM recipes (4+ sequential calls for a send) that compound error rates. "Solved" means a ~25-tool primary surface where common operations (send, swap, portfolio, contract call, LP, query) are single end-to-end tools, and semantically-similar CRUD groups collapse to single tools with action discriminators. Matches the shapeshift-agentic pattern at `/home/sean/Repos/shapeshift-agentic`.

## Architecture Reality Check

This is **primarily MCP-layer work** (mcp-ts) with supporting changes in agent-backend (compatibility + backend-owned tools) and vultiagent-app (stepped UI). The SDK is already mature.

Verified state (2026-04-16):

- **`vultisig-sdk`** already has unified entry points: `VaultBase.send()` (src/vault/VaultBase.ts:1633), `VaultBase.swap()` (:1662), `VaultBase.contractCall()` (:1722). Chain dispatch is hidden in `packages/core/mpc/keysign/send/build.ts:31`. ~90% of what we need exists. Minor gaps: `fiatToAmount()` helper, case-insensitive chain normalization.
- **`mcp-ts`** already has 153 tools (ahead of Go `mcp/`'s ~103). Partially wraps SDK (e.g. `build-swap-tx.ts` uses SDK's `findSwapQuote` + `abiEncode`; `abi-encode.ts` is a one-liner over `@vultisig/sdk`). Some per-chain code is custom (UTXO sends, balance queries via direct JSON-RPC). Compat changes for `refactor/ai-sdk-migration` are already in mcp-ts.
- **`agent-backend`** is already thin. No chain logic lives here. Current handlers dispatch to MCP tools via `s.mcpProvider.CallTool()`. The 60-line send recipe in `prompt.go:196-275` teaches the LLM to orchestrate multiple MCP calls — that orchestration moves into single MCP tools.
- **Go `mcp/`** is being replaced by `mcp-ts`. No new work lands there.

## Research Findings

### Current State
- `mcp-ts/src/tools/` — 153 tools organized by domain (send/, swap/, balance/, defi/, evm/, rujira/, polymarket/, tornado/, etc.). Tool registry in `src/tools/index.ts`.
- `mcp-ts/src/tools/swap/build-swap-tx.ts` — already wraps SDK `findSwapQuote` + manually encodes EVM calldata for THORChain deposit. Thin to extend into `execute_swap`.
- `mcp-ts/src/tools/evm/abi-encode.ts` — one-liner SDK wrapper. Example of the pattern we want.
- `mcp-ts/src/tools/send/build-utxo-send.ts` — custom logic returning PSBT preview args. Example of where mcp-ts re-implements rather than wraps. Alignment to SDK is a separate follow-up.
- `vultisig-sdk/packages/sdk/src/vault/VaultBase.ts` — unified send (line 1633), swap (line 1662), contractCall (line 1722). `SwapService.prepareSwapTx()` returns `{ keysignPayload, approvalPayload?, quote }` — the prep-only shape mcp-ts's `execute_swap` consumes.
- `agent-backend/internal/service/agent/tools.go` — 13 orchestration + 12 client-side + 5 server data tools. Backend-owned (schedule, protocol_state, credits) and client-side-dispatched (modify_vault, address_book, plugins, signing) tools stay here.
- `agent-backend/internal/service/agent/prompt.go:110-275` — 60-line send recipe + 50-line swap recipe become unnecessary once MCP exposes end-to-end tools. Delete and replace with routing table.
- `agent-backend/internal/service/agent/tool_filter.go:18-87` — category-based keyword filter already gates MCP tools via `_meta.categories`. Infrastructure ready. Building blocks get narrower categories so they're hidden unless keywords match.
- `agent-backend/internal/service/agent/data_tools.go` — `execGetBalances`/`execGetPortfolio` today run against pre-fetched balances from SSE context. Needs extension to call mcp-ts internally for non-pre-fetched chains (absorbs v-uvyo scope). Stays in agent-backend because it consumes SSE-provided balance context.
- `agent-backend/internal/service/agent/executor.go:98-166` — `normalizeMCPArgs` + `decimalToBaseUnits` are LLM-hallucination workarounds (field alias rewrites, amount auto-conversion). Once end-to-end tools replace multi-step recipes, these workarounds largely become dead code. Delete what's no longer needed.
- `vultiagent-app/src/features/agent/components/AssistantMessageView.tsx` — `coalesceParts` (lines 7-26) + `findLastBuildToolIndex` (lines 68-77) are frontend workarounds for the multi-tool-call problem. Delete both.
- `vultiagent-app/src/features/agent/lib/toolUIRegistry.ts` — maps tool names to React components. Wildcard `build_*` resolver catches all per-chain build tools today via `BuildTxCard`. Needs updates for new tool names + new card components.

### Available Tools & Patterns
- `shapeshift-agentic/apps/agentic-server/src/tools/send.ts` — reference for single end-to-end `send(asset, recipient, amount)` with internal chain routing. Structurally similar to what `execute_send` in mcp-ts becomes.
- `shapeshift-agentic/apps/agentic-chat/src/components/tools/InitiateSwapUI.tsx` — reference for Execution.Stepper pattern (QUOTE → NETWORK → APPROVE → SWAP).
- `shapeshift-agentic/apps/agentic-chat/src/lib/executionState.ts` — `ToolExecutionState` shape for client-driven step progression. Port this to vultiagent-app.
- `mcp-ts/src/tools/swap/build-swap-tx.ts` — existing SDK-wrapping pattern. Extend for `execute_swap`.
- `@vultisig/sdk` exports `findSwapQuote`, `abiEncode`, `evmCheckAllowance` today. Verify `prepareSwapTx`, `buildSendKeysignPayload`, `contractCall` prep-only variants are also exposed (or expose them if not).

### External Context — LLM Tool Selection Research
- GitHub Copilot reduced 40→13 tools: 2-5% accuracy gain on SWE-Lancer/SWEbench-Verified, 400ms latency drop, 94.5% tool coverage vs 69% static (https://github.blog/ai-and-ml/github-copilot/how-were-making-github-copilot-smarter-with-fewer-tools/)
- Microsoft Research "tool-space interference" (2025) — semantically similar tools degrade selection accuracy; per-chain send tools are textbook example.
- NESTFUL benchmark (EMNLP 2025) — GPT-4o hits only 28% accuracy on sequential multi-step tool chains; smaller models worse.
- Practical thresholds: 5-10 tools safe, 10+ degradation starts, 30+ semantic confusion, 100+ "virtually guaranteed to fail".
- Token overhead: 136 tools ≈ 68K tokens schema/request, 25 tools ≈ 15K tokens → 53K recovered.

### Validated Behaviors
- `vault.send()`, `vault.swap()`, `vault.contractCall()` are unified SDK entry points. Chain dispatch hidden from callers. Validated by reading VaultBase.ts and core-chain build.ts.
- `SwapService.prepareSwapTx()` returns prep-only keysign payload including optional `approvalPayload` — the shape `execute_swap` MCP tool needs.
- mcp-ts already imports from `@vultisig/sdk` for swap + ABI operations. Extending that pattern to send + contractCall is consistent with existing code.
- mcp-ts is at 153 tools, ahead of Go mcp's ~103. Already includes compat changes for `refactor/ai-sdk-migration`.
- agent-backend has no chain logic — all chain work dispatches to MCP. "Routing" for per-chain build tools is done today by the LLM via prompt instructions.
- `tool_filter.go` category-based gating works today via `_meta.categories` set on MCP-side tool definitions.
- Per-chain send tools require LLM to select correct `build_*_send` from 15+ options — textbook tool-space interference.
- EVM ERC-20 send currently requires 4 LLM tool calls: `convert_amount` → `abi_encode` → `evm_tx_info` → `build_evm_tx`. Each is a failure point.

## Constraints

- **Branch:** `refactor/ai-sdk-migration` across `vultisig-sdk` (if needed), `mcp-ts`, `agent-backend`, `vultiagent-app`. One PR per repo. mcp-ts + agent-backend + vultiagent-app already have in-flight compat work on this branch.
- **mcp-ts is the target MCP server.** agent-backend points at mcp-ts. Go mcp/ is being retired.
- **Architecture discipline:**
  - New business logic (chain routing, amount conversion, ABI encoding, tx building) lands in `vultisig-sdk` first, exposed to mcp-ts.
  - MCP tools in mcp-ts are thin wrappers — they call SDK, return structured output. No chain-specific re-implementation for new tools.
  - agent-backend stays thin. It dispatches MCP calls, manages conversations, runs prompts, handles backend-owned state (schedule/credits/protocol_state) and client-side action dispatch (vault mutations, address book, plugins).
- **Schema rules for Gemini Flash compatibility:**
  - Flat schemas only — no nested objects
  - Enum values in JSON schema `enum` field, not description text
  - Self-documenting enum values (`"add_coin"` not `"add"`)
  - Enum cardinality <10 per parameter
  - No `oneOf`/`anyOf` unions — use flat optional params
  - Tool descriptions under 100 words
- **Keep escape hatches:** all existing `build_*_send` tools, `build_evm_tx`, `abi_encode`, `evm_tx_info`, `convert_amount`, per-chain balance tools stay registered in mcp-ts but get narrow `_meta.categories` so they're hidden from default tool list.
- **Existing work absorbed:** `.tickets/v-uvyo.md` (balance fan-out), `.tickets/v-gmvc.md` (swap stepper), `.tickets/v-bauf.md` (AssistantMessageView simplification) — delete on design. `tic-6c08` partially absorbed (stepper replaces tool execution hooks; `pendingTx` removal still needed separately).

## Scope — End to End

### Layer 1: `vultisig-sdk` (minimal — gap-filling only)

- **Verify exports:** `prepareSwapTx`, `buildSendKeysignPayload`-equivalent, `contractCall` prep variant are exposed from `@vultisig/sdk` package for mcp-ts to consume. If not, expose them.
- **New:** `fiatToAmount(fiatValue, chain, tokenId?, decimals, fiatCurrency?)` utility. ~2 hours.
- **New:** `normalizeChain(input: string): Chain` case-insensitive resolution + aliases (BTC → Bitcoin, ETH → Ethereum, etc.). ~3 hours.
- **Out of scope:** migrating mcp-ts's custom UTXO send / balance implementations into SDK primitives. That's SDK-alignment hygiene, separate ticket.

### Layer 2: `mcp-ts` (bulk of the work — new end-to-end tools + query consolidations)

**End-to-end action tools (new):**
- `execute_send(chain, token, amount, amount_kind, recipient, memo?)` — thin wrapper over SDK's send prep primitive. Routes chain internally. Returns stepper state + unsigned tx args for client signing. Replaces the need for LLM to pick among 15+ per-chain `build_*_send` tools.
- `execute_swap(sell_token, buy_token, amount, amount_kind)` — extends existing `build-swap-tx.ts` pattern. Wraps SDK `SwapService.prepareSwapTx()`. Emits stepper state (QUOTE → APPROVE? → SIGN → BROADCAST).
- `execute_contract_call(chain, contract, function_sig, args, value?)` — wraps SDK `contractCall` prep. Consolidates `abi_encode` + `evm_tx_info` + `build_evm_tx` for custom EVM calls.
- `execute_lp_add(pool, amount_rune, mode?)` — consolidates the forward-looking 4-step THORChain LP add recipe into a single tool. Internal: query pool → check halts → quote → build.
- `execute_lp_remove(pool, basis_points, withdraw_to_asset?)` — consolidates 4-step LP remove. Internal: check position → check lockup → check halts → build.

**Query consolidations (new):**
- `query_defi(type: protocol|yields|tvl|prices|price_change|price_chart|stablecoins|yield_history|dex_volumes, ...)` — consolidates 9 DeFiLlama tools.
- `query_uniswap(action: pool_info|position_info|tick_math, ...)` — consolidates 3 Uniswap V3 tools.
- `polymarket_query(action: search|market_info|orderbook|price|positions|trades|open_orders, ...)` — consolidates 7 Polymarket read tools.
- `tornado_deposit(chain, denomination)` and `tornado_withdraw(chain, pool, note, recipient)` — consolidate 6 of 8 Tornado tools. Keep `tornado_check_deposit` + `tornado_check_spent` as monitoring reads.

**Category metadata (visibility gating support):**
- New end-to-end tools: `_meta.categories: ["utility"]` (always-on).
- Existing building blocks get narrower categories:
  - Per-chain `build_*_send` tools → `"custom_send"` category (gated behind "raw", "custom", specific chain keywords)
  - `build_evm_tx`, `abi_encode`, `evm_tx_info` → `"contract"` category (gated behind "contract", "abi", "custom tx")
  - Per-chain balance tools → `"chain_balance"` category (gated behind specific chain names)
  - `convert_amount` → `"contract"` category (same gating)
- Query tools that consolidate (e.g. old `defillama_*`) remain registered but with narrower categories; consolidated versions are always-on.

**Keep as-is (already clean patterns):**
- `get_address`, `get_tx_status`, `search_token` (single-purpose utilities)
- `set_vault_info` (required bootstrap tool)

### Layer 3: `agent-backend` (compatibility + backend-owned consolidations)

**Compatibility work:**
- Update `tool_classify.go` to classify new MCP tools (`execute_*`, `query_*`) correctly.
- Update `tool_filter.go` category keyword mappings if new categories are introduced by mcp-ts metadata.
- Remove or prune `normalizeMCPArgs` field-alias rewrites (executor.go:98-166) — most become dead code once end-to-end tools replace multi-step recipes. Audit what's still needed.
- Remove `decimalToBaseUnits` auto-conversion — end-to-end tools take human-readable amounts; no pre-conversion needed.

**CRUD consolidations (backend-owned state tools):**
- `manage_schedule(action: create|list|update|cancel, ...)` — replaces `schedule_task`, `list_scheduled_tasks`, `update_scheduled_task`, `cancel_scheduled_task`. New handler in executor.go.
- `protocol_state(action: store|get|list|update, ...)` — replaces `store_protocol_state`, `get_protocol_state`, `list_protocol_states`, `update_protocol_state`. New handler.
- `credits(detail?: boolean)` — replaces `check_credits`, `get_spending_history`. New handler.

**Client-side action tool consolidations (dispatched to app):**
- `modify_vault(action: add_coin|remove_coin|add_chain|remove_chain, ...)` — replaces 4 vault mutation tools. Update `buildActionToolPrep()` and `clientSideToolNames` registry.
- `address_book(action: add|remove, ...)` — replaces `address_book_add`, `address_book_remove`.
- `manage_plugin(action: install|create_policy|delete_policy, ...)` — replaces `plugin_install`, `create_policy`, `delete_policy`.

**Data tool extension:**
- `get_portfolio` / `get_balances` in `data_tools.go` — extend server-side fan-out to cover non-pre-fetched chains via mcp-ts balance tools internally (absorbs v-uvyo). Stays in agent-backend because it consumes SSE-provided balance context.

**Prompt rewrite (`prompt.go`):**
- Delete 60-line send recipe (lines 196-275).
- Delete 50-line swap recipe (lines 110-194).
- Add concise tool routing table:
  - Send → `execute_send`
  - Swap → `execute_swap`
  - Portfolio/balances → `get_portfolio` / `get_balances`
  - Custom EVM contract → `execute_contract_call`
  - THORChain LP → `execute_lp_add` / `execute_lp_remove`
  - DeFi analytics → `query_defi`
  - Uniswap V3 → `query_uniswap`
  - Polymarket → `polymarket_query` (read) / `polymarket_place_bet` (action)
- Simplify or remove the prompt's THORChain LP procedural section (replaced by `execute_lp_*`).
- Keep Rujira secured-asset guidance intact (not covered by end-to-end tools; protocol-specific context still needed).
- Keep address-book / vault-to-vault / fiat-amounts sections (still useful context, just reference new tool names).

### Layer 4: `vultiagent-app` (frontend)

**Execution.Stepper infrastructure (new):**
- Port `ToolExecutionState` shape from shapeshift-agentic `apps/agentic-chat/src/lib/executionState.ts`
- Build `Execution.Root` / `Execution.Step` / `Execution.Stepper` / `Execution.ErrorFooter` / `Execution.HistoricalGuard` components
- State management via `useToolExecution()` hook (client-driven step progression)

**New tool UI cards:**
- `SendFlowCard.tsx` — PREPARE → CONFIRM → SIGN → BROADCAST
- `SwapFlowCard.tsx` — QUOTE → APPROVE? → SIGN → BROADCAST
- `ContractCallCard.tsx` — PREPARE → CONFIRM → SIGN → BROADCAST
- `LpFlowCard.tsx` — QUOTE → SIGN → BROADCAST
- `DefiQueryCard.tsx` / `UniswapQueryCard.tsx` / `PolymarketQueryCard.tsx` — render consolidated query results

**CRUD tool UI consolidation:**
- `ModifyVaultTool.tsx` — replaces `AddCoinTool`, `RemoveCoinTool`, `AddChainTool`, `RemoveChainTool`
- `AddressBookTool.tsx` — covers add/remove actions
- `ManagePluginTool.tsx` — covers install/create_policy/delete_policy
- `ScheduleTool.tsx` — covers all schedule CRUD actions

**Deletions:**
- `src/features/agent/lib/coalesceParts.ts` + `__tests__/coalesceParts.test.ts`
- `src/features/agent/components/AggregateToolIndicator.tsx`
- `findLastBuildToolIndex` function in `AssistantMessageView.tsx`
- Collapsing branch in `AssistantMessageView.tsx` (currently lines 117-125)

**Registry updates:**
- `toolUIRegistry.ts` — map new tool names (`execute_send`, `execute_swap`, etc.) to new card components.
- Keep `BuildTxCard` as the fallback for gated escape-hatch building blocks (when keyword filter surfaces them).
- Update component imports and exports.

### Kept as-is (not consolidated)
- `set_vault`, `get_skill`, `create_vault` — single-purpose singletons
- `propose_plan` + `update_plan` — different schemas, only 2 tools
- `sign_typed_data` + `polymarket_sign_bet` — different schemas, different flows
- `search_token`, `get_market_price`, `get_tx_status`, `get_address_book`, `list_vaults` — data queries, semantically distinct

### Tickets superseded (delete on design approval)
- `v-uvyo` — balance fan-out (absorbed into Layer 3 data tool extension)
- `v-gmvc` — swap stepper (absorbed into Layer 2 `execute_swap` + Layer 4 SwapFlowCard)
- `v-bauf` — AssistantMessageView simplification (absorbed into Layer 4 deletions)
- `tic-6c08` — partially absorbed (stepper replaces tool execution hooks; `pendingTx` removal still needed separately)

## Dead Ends

- **Splitting by repo into multiple tickets** — rejected. Frontend depends on backend + MCP tool shapes. Splitting creates merge conflicts and half-working states. One agent session across all repos.
- **Splitting MCP query consolidation out** — rejected. User wants everything in one ticket. Same architectural pattern (semantic-similarity groups → action-discriminator tool), same research justification. Same risk profile.
- **Putting chain/amount/ABI logic in agent-backend** — rejected. Violates architecture. agent-backend has no chain logic today; adding it would be a regression. Business logic lives in SDK, MCP wraps it.
- **Aligning mcp-ts custom implementations with SDK primitives** (e.g. migrating `build-utxo-send.ts` to use SDK `VaultBase.send()` prep) — rejected for this ticket. That's hygiene, not blocking consolidation. Follow-up ticket.
- **Soft steering (keep all tools visible, prompt steers)** — rejected for sends. Research shows LLMs ignore steering under prompt load. Hard gating via `_meta.categories` + `tool_filter.go` is safer.
- **Removing per-chain build tools entirely** — rejected. Kept as gated escape hatches for custom transactions (raw calldata, custom gas, specific provider override).
- **Consolidating `sign_typed_data` + `polymarket_sign_bet`** — rejected. Different schemas, different flows. Only 2 tools.
- **Consolidating `propose_plan` + `update_plan`** — rejected. Schemas too different.
- **Aggressive consolidation of distinct data queries (`get_balances` + `get_portfolio` + `get_market_price`)** — rejected. Semantically distinct, already small, separation helps LLM select correctly.

## Open Questions

1. **`execute_send` amount schema** — single `amount` field + `amount_kind: "token"|"usd"` discriminator, or two exclusive fields (`amount` / `amount_usd`)? Flat-schema rule favors single field; current mcp-ts tools are inconsistent (some use `amount_usd`, others don't).

2. **`execute_contract_call` args encoding** — array of strings, array of typed values, or pre-encoded calldata? Array-of-strings is simplest but loses type info. Pre-encoded forces the LLM to still call `abi_encode` (defeating the purpose).

3. **Hard gating vs soft steering per tool** — which building blocks get fully gated vs visible-but-deprioritized? Proposed: gate per-chain send tools + per-chain balance tools hard; gate `build_evm_tx` / `abi_encode` behind "contract"/"abi" keywords.

4. **Stepper state ownership** — client-driven (shapeshift pattern) or server-driven (backend streams step transitions via tool result updates)? Client-driven is simpler and matches shapeshift reference.

5. **`modify_vault` consolidation level** — the 4 actions have different parameter shapes (token arrays vs chain arrays). Flat schema possible but verbose. Research favors consolidation; UX of error messages may favor separation.

6. **`execute_lp` consolidation shape** — one tool with `action: add|remove|query_position` or separate `execute_lp_add` / `execute_lp_remove` / `query_lp_position`? LP position queries overlap with portfolio queries.

7. **Tool deprecation strategy** — keep old tool names as aliases that dispatch to new tools for one release cycle, or hard cut? Hard cut is cleaner but risks breaking active conversations. mcp-ts pattern makes aliases cheap (just re-register old name pointing to new handler).

8. **`normalizeMCPArgs` cleanup depth** — once end-to-end tools land, which of the alias rewrites (`from1_*` → `from_*`, `sender` → `from`, etc.) are still needed? Audit during implementation.

9. **SDK gap filling timing** — build `fiatToAmount()` and `normalizeChain()` utilities first (blocking mcp-ts tools that need them), or stub them in mcp-ts and migrate later? Prefer: build in SDK first, ~5 hours total.

10. **Prompt rewrite depth** — simplify Rujira/THORChain prompt sections too, or only send/swap recipes? Proposed: LP section gets simplified (replaced by `execute_lp_*`); Rujira stays (secured assets still need contextual guidance).

## Notes

**2026-04-16T05:20:27Z**

# Design (2026-04-16)

## Decisions locked this session

1. **Stepper state ownership — client-driven.** MCP returns full prep payload in one response (`{ keysignPayload, approvalPayload?, quote, stepperConfig }`). App's `useToolExecution()` advances QUOTE/APPROVE/SIGN/BROADCAST locally. No long-lived MCP streams, no per-step server emissions. Matches shapeshift-agentic `apps/agentic-chat/src/lib/executionState.ts`.

2. **LLM-facing schemas are natural-language strings; Zod `.transform()` parses server-side.** Principle: weak models echo user phrasing reliably, fail at multi-field coordination. Applied:
   - `amount: z.string()` accepts `"10"` / `"$50"` / `"10 USDC"` / `"max"` / `"50%"`. Transform parses to typed `{ kind, value }`. Validation failures come back as LLM-readable error strings ("Could not parse amount 'dollars'. Use '10', '$10', or 'max'.") enabling next-turn self-correction.
   - `execute_contract_call` takes `function_sig: string, args: string[]` (e.g. `"transfer(address,uint256)", ["0x...", "1000000"]`). SDK parses sig with viem. Matches how users read contract calls on Etherscan.
   - Tool results **echo interpretation** — `resolved_amount: "0.4721 ETH"`, `normalized_chain: "Ethereum"` — so the LLM reasons over concrete values in subsequent turns.

3. **Naming — keep `execute_*` prefix** for end-to-end action tools. `tool_filter.go` and `toolUIRegistry` pattern-match on the prefix. `build_*` reserved for gated escape-hatch building blocks. `get_*` / `search_*` for reads.

4. **CRUD consolidation — split aggressively.** Action-discriminator polymorphic tools are a Gemini footgun: Gemini's API does not support `oneOf`/`anyOf` ([Firebase AI Logic docs](https://firebase.google.com/docs/ai-logic/function-calling)), so `modify_vault(action, coins?, chains?)` collapses to a flat all-optional schema where conditional requirements live in prose only — weak models fill these wrong. Research: MSR flattening yielded up to 47% tool-call improvement; Copilot 40→13 gained 2-5pp but mixed pruning with routing. Consolidation starts paying off around 20-25 tools, but **only** where param shapes are genuinely symmetric.

   Revised table vs. original ticket proposal:

   | Group | Decision | Tools |
   |---|---|---|
   | `execute_*` action | Keep consolidated | 5 (send, swap, contract_call, lp_add, lp_remove) |
   | Schedule | **Split** | `schedule_create`, `schedule_list`, `schedule_update`, `schedule_cancel` |
   | Protocol state | **Split** | `protocol_state_{store,get,list,update}` |
   | Credits | Keep | `credits(detail?: bool)` — single optional bool, not discriminator |
   | Vault coin/chain | **Split group, collapse within** | `vault_coin(action: add\|remove, coins)`, `vault_chain(action: add\|remove, chains)` (add/remove truly symmetric within each; coin vs chain unrelated) |
   | Address book | Collapse | `address_book(action: add\|remove, entry)` — symmetric |
   | Plugin | **Split** | `plugin_install`, `plugin_create_policy`, `plugin_delete_policy` |
   | DeFi queries | **Cluster by shape (5, not 9)** | `defi_prices` (prices+change+chart), `defi_yields` (yields+history), `defi_protocols` (protocols+tvl), `defi_stablecoins`, `defi_dex_volumes` |
   | Uniswap | **Split** | `uniswap_pool_info`, `uniswap_position_info`, `uniswap_tick_math` |
   | Polymarket read | **Cluster by shape (3, not 7)** | `polymarket_search`, `polymarket_market` (info+price+orderbook), `polymarket_my` (positions+orders+trades) |
   | Tornado | Keep 4 | `tornado_deposit`, `tornado_withdraw`, `tornado_check_deposit`, `tornado_check_spent` |

5. **Deprecation — hard cut.** Old tool names removed from default registration at ship time. Building blocks (per-chain `build_*_send`, `abi_encode`, `evm_tx_info`, `build_evm_tx`, `convert_amount`, per-chain balance tools) stay registered but move to narrow `_meta.categories` — only surfaced when `tool_filter.go` matches keywords like "raw", "custom", "contract", specific chain names. No alias dispatch layer.

## Tool surface — always-on vs gated

**Always-on (~30 tools, default visible):**
- Action: `execute_send`, `execute_swap`, `execute_contract_call`, `execute_lp_add`, `execute_lp_remove`
- Reads: `get_portfolio`, `get_balances`, `get_address`, `get_market_price`, `search_token`, `get_tx_status`, `get_address_book`, `list_vaults`, `get_skill`
- State: `set_vault`, `set_vault_info`, `create_vault`
- Planning: `propose_plan`, `update_plan`
- CRUD: `vault_coin`, `vault_chain`, `address_book`, `schedule_{create,list,update,cancel}`, `protocol_state_{store,get,list,update}`, `credits`
- Signing (distinct flows): `sign_typed_data`

**Gated by keyword category:**
- `defi` category → `defi_prices`, `defi_yields`, `defi_protocols`, `defi_stablecoins`, `defi_dex_volumes`
- `uniswap` category → `uniswap_pool_info`, `uniswap_position_info`, `uniswap_tick_math`
- `polymarket` category → `polymarket_search`, `polymarket_market`, `polymarket_my`, `polymarket_sign_bet`
- `tornado` category → `tornado_{deposit,withdraw,check_deposit,check_spent}`
- `plugin` category → `plugin_install`, `plugin_create_policy`, `plugin_delete_policy`
- `contract` / `raw` category → `build_evm_tx`, `abi_encode`, `evm_tx_info`, `convert_amount`
- `custom_send` category → all `build_*_send` per-chain tools
- `chain_balance` category → per-chain balance tools

Final surface: ~30 always-on, ~40 gated escape hatches. Weak-model selection burden cut from 136 to ~30 in the default case.

## Architecture invariants

- Chain dispatch lives in SDK (`VaultBase.send/swap/contractCall`). mcp-ts tools wrap SDK prep functions — no per-chain re-implementation for new `execute_*` tools.
- Zod is the schema source of truth (mcp-ts `ToolDef<T extends ZodShape>` already). `.transform()` pushes natural-string parsing server-side; `.refine()` emits LLM-readable validation errors.
- agent-backend stays thin: MCP dispatch + backend-owned state (schedule, protocol_state, credits) + client-side action prep (vault/address_book/plugin). No chain logic.
- Client-driven stepper: MCP returns one payload; app advances steps. No MCP progress notifications.

## Tasks

Each task is a coherent PR-sized change set. Dependencies noted.

1. **SDK gap-fill** (vultisig-sdk) — Add `fiatToAmount(fiatValue, chain, tokenId?, decimals, fiatCurrency?)` and `normalizeChain(input: string): Chain` (case-insensitive + aliases). Verify `prepareSwapTx`, send-prep, and contractCall-prep variants are exported from `@vultisig/sdk` package root; expose any missing. **Independent; blocks #2.**

2. **mcp-ts shared helpers** — `parseAmount` (handles `"10"` / `"$10"` / `"10 USDC"` / `"max"` / `"50%"` → typed result), Zod transform helpers with LLM-readable error messages, shared `_meta.categories` constants, `execute_*` tool factory. Files: `mcp-ts/src/lib/parseAmount.ts`, `mcp-ts/src/lib/zodHelpers.ts`, `mcp-ts/src/tools/execute/_base.ts`. **Depends on #1.**

3. **mcp-ts `execute_*` action tools** — `execute_send`, `execute_swap` (refactor existing `build-swap-tx.ts`), `execute_contract_call`, `execute_lp_add`, `execute_lp_remove`. Each wraps SDK prep. Returns full stepper payload (`keysignPayload`, optional `approvalPayload`, `quote`, `stepperConfig`). **Depends on #2. Can parallelize internally by tool.**

4. **mcp-ts query consolidations** — DeFi 9→5, Uniswap → 3, Polymarket read → 3. Shape-clustered tools under `mcp-ts/src/tools/defi/`, `uniswap/`, `polymarket/`. **Can parallelize with #3.**

5. **mcp-ts category metadata + registration cleanup** — Apply `_meta.categories` to all tools per gating table above. Remove old per-chain `build_*_send`, `build_evm_tx`, `abi_encode`, `evm_tx_info`, `convert_amount`, per-chain balance tools from default registration; keep as gated. Update `src/tools/index.ts`. **Depends on #3, #4.**

6. **agent-backend CRUD + client-side action handlers** — New handlers in `executor.go`: `schedule_{create,list,update,cancel}`, `protocol_state_{store,get,list,update}`, `credits`. Client-side action tool entries: `vault_coin`, `vault_chain`, `address_book`, `plugin_{install,create_policy,delete_policy}`. Update `buildActionToolPrep()` + `clientSideToolNames`. **Can parallelize with mcp-ts work.**

7. **agent-backend data tool fan-out** — Extend `execGetBalances` / `execGetPortfolio` in `data_tools.go` to call mcp-ts balance tools for chains not in SSE-prefetched set (absorbs v-uvyo). **Can parallelize with #6.**

8. **agent-backend prompt + filter + executor cleanup** — Delete send recipe (`prompt.go:196-275`) + swap recipe (`prompt.go:110-194`); add concise tool routing table. Simplify THORChain LP section (replaced by `execute_lp_*`). Update `tool_classify.go` for new tool names. Update `tool_filter.go` category keyword mappings. Audit + delete dead `normalizeMCPArgs` alias rewrites (`executor.go:98-166`); delete `decimalToBaseUnits`. **Depends on #3, #4 tool names being stable.**

9. **app Execution.Stepper infrastructure** — Port `ToolExecutionState` type + `useToolExecution()` hook from `shapeshift-agentic/apps/agentic-chat/src/lib/executionState.ts`. Build `Execution.Root` / `Execution.Step` / `Execution.Stepper` / `Execution.ErrorFooter` / `Execution.HistoricalGuard` components. Client-driven state transitions. **Independent; blocks #10-12.**

10. **app action flow cards** — `SendFlowCard`, `SwapFlowCard`, `ContractCallCard`, `LpFlowCard`. Each consumes stepper payload from MCP, drives QUOTE/APPROVE?/SIGN/BROADCAST. Register in `toolUIRegistry.ts`. **Depends on #9.**

11. **app query cards** — `DefiQueryCard` variants (one per shape cluster), `UniswapQueryCard`, `PolymarketQueryCard`. Render consolidated query results. **Can parallelize with #10.**

12. **app CRUD cards + legacy cleanup** — `VaultCoinTool`, `VaultChainTool`, `AddressBookTool`, `ManagePluginTool`, `ScheduleTool`. Delete `coalesceParts.ts` + tests, `AggregateToolIndicator.tsx`, `findLastBuildToolIndex`. Remove collapsing branch in `AssistantMessageView.tsx` (lines 117-125). Keep `BuildTxCard` as fallback for gated escape-hatch tools. **Depends on #9 for stepper types.**

### Parallelism graph

```
#1 SDK ──▶ #2 mcp helpers ──▶ #3 execute_* ──┐
                           └▶ #4 queries ────┼─▶ #5 categories
                                             │
#6 agent-backend CRUD ───────────────────────┤
#7 agent-backend data ───────────────────────┤
                                             ▼
                                         #8 prompt+filter cleanup
                                             │
#9 stepper infra ──┬─▶ #10 action cards ─────┤
                   ├─▶ #11 query cards ──────┤
                   └─▶ #12 CRUD + cleanup ───┘
```

Critical path: #1 → #2 → #3 → #5 → #8. Everything else parallelizes.

## Superseded tickets (delete on design approval)

- `v-uvyo` — absorbed into task #7
- `v-gmvc` — absorbed into tasks #3 (`execute_swap`) + #10 (`SwapFlowCard`)
- `v-bauf` — absorbed into task #12 deletions
- `tic-6c08` — partially absorbed (stepper replaces tool execution hooks via task #9; `pendingTx` removal still separate)

## Open questions retired

- Q#1 amount schema → single `amount: string` + Zod transform
- Q#2 contract call args → `function_sig + args: string[]`
- Q#3 gating → hard gating via `_meta.categories` per surface table above
- Q#4 stepper ownership → client-driven
- Q#5 modify_vault → split into `vault_coin` / `vault_chain` (symmetric within each; coin/chain unrelated)
- Q#6 LP shape → separate `execute_lp_add` / `execute_lp_remove`
- Q#7 deprecation → hard cut with gating, no aliases
- Q#8 normalizeMCPArgs cleanup → audit + delete dead entries in task #8
- Q#9 SDK gap timing → first (task #1)
- Q#10 prompt rewrite depth → LP section simplified; Rujira stays

## Research-backed rationale

- GitHub Copilot 40→13 tools: +2-5pp SWE-Lancer/SWEbench-Verified, -400ms latency (mixed pruning with routing). [github.blog](https://github.blog/ai-and-ml/github-copilot/how-were-making-github-copilot-smarter-with-fewer-tools/)
- MSR "tool-space interference": up to 85% degradation on large tool spaces; flattening nested params alone → up to 47% improvement. [microsoft.com](https://www.microsoft.com/en-us/research/blog/tool-space-interference-in-the-mcp-era-designing-for-agent-compatibility-at-scale/)
- NESTFUL: top models at 28% full-sequence match on nested tool chains. [arxiv.org/abs/2409.03797](https://arxiv.org/abs/2409.03797)
- Gemini function-calling limit on `oneOf`/`anyOf`: structural API constraint, not model capability. [firebase.google.com](https://firebase.google.com/docs/ai-logic/function-calling)
- Gemini 3 Flash: 78% SWE-bench Verified; no published BFCL numbers; schema polymorphism limit still applies. [blog.google](https://blog.google/products-and-platforms/products/gemini/gemini-3-flash/)
- Anthropic define-tools: "consolidate related operations" advice; still flags Haiku-class as weaker at inferring missing params. [platform.claude.com](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)

Selection-accuracy win comes primarily from **tool-count reduction (~30 default-visible vs. 136)** + **schema flattening**, not from aggressive polymorphic consolidation.

**2026-04-16T08:47:44Z**

Task 1 complete (vultisig-sdk @ 8f4b48b on refactor/ai-sdk-migration): Added fiatToAmount() + normalizeChain() utilities with strict TDD (141 new tests, all passing). Exported from @vultisig/sdk package root. Prep primitives already public as VaultBase instance methods (prepareSendTx/prepareSwapTx/prepareContractCallTx) — no extraction needed. mcp-ts consumers call vault.prepareSendTx() etc.

**2026-04-16T08:56:50Z**

Task 2 complete (mcp-ts @ db4e9da + 5cfb396 on refactor/ai-sdk-migration): parseAmount, zodHelpers (amountString/chainString/addressString/llmError), toolCategories constants, execute/_base scaffolding. SDK linked via pnpm override (file:../vultisig-sdk/packages/sdk). 68 new tests, all 136 mcp-ts tests passing. Conventions for downstream tasks: use Categories.* constants (no string literals); llmError marker is params.code='llm_readable' for agent-backend filtering; parseAmount result is pre-conversion — execution layer resolves fiat/max/percent against balance/price.

**2026-04-16T09:18:21Z**

Tasks 6, 7, 9 complete. Task 3 surfaced architectural blocker: mcp-ts has no vault instance (by design — security boundary: public-key metadata only via set_vault_info). SDK prep methods require full VaultBase. Resolution: mcp-ts execute_* tools return tx-args JSON (matching existing build-swap-tx/build-evm-tx pattern); client builds KeysignPayload via existing parseServerTx → signAndBroadcast path. _base.ts keysignPayload type updated from sdk.KeysignPayload to ExecuteTxArgs (Record<string, unknown> with chain-family discriminator). Task 3 resumed with this direction.

**2026-04-16T09:41:14Z**

Task 3 complete (mcp-ts 9 commits, 193fa16..0fa1f26 on refactor/ai-sdk-migration): Five execute_* tools (send/swap/contract_call/lp_add/lp_remove) with shape parity to existing build_* tools. Key architectural decision: mcp-ts tools return tx-args JSON (not KeysignPayload). _base.ts retyped: txArgs/approvalTxArgs with tx_encoding discriminator (evm|utxo-psbt|cosmos-msg|solana-tx|ripple-tx|tron-tx|ton-tx|sui-tx|cardano-tx). Registered in src/tools/index.ts top block. Tests: 194 passing. Known gap: max/percent balance lookup only wired for EVM; other families throw AmountResolutionError — ok for now.

**2026-04-16T10:26:25Z**

All 12 tasks complete across 4 repos. Cumulative commit summary on refactor/ai-sdk-migration: vultisig-sdk (1 commit, 8f4b48b), mcp-ts (~15 commits, db4e9da..de9fb7c), agent-backend (~5 commits, f929270..8a342b5), vultiagent-app (~12 commits, 5921628..c7388e1). Tool surface: 136 → ~30 always-on (mcp-ts side), with ~110 gated escape hatches still registered. Dispatching final integration review.

**2026-04-16T10:51:32Z**

All criticals fixed and verified. Final state on refactor/ai-sdk-migration across 4 repos:
- vultisig-sdk: 1 commit (8f4b48b), 1067/1067 tests
- mcp-ts: 19 commits (db4e9da..d87483a), 252/252 tests, includes C1-C4 fixes (LP kind discriminator, token-send rejection for SPL/TRC20/TON-jetton/SUI, Ripple rejection, UTXO swap fee_rate)
- agent-backend: 7 commits (f929270..2d724ee), tests pass, includes C5-C6 fixes (address_book entry.name alignment, full historical part-type migration in convert.go for 13 renamed tools)
- vultiagent-app: 12 commits (5921628..c7388e1), 507/507 tests
Tool surface: 136 → ~30 always-on (mcp-ts) + ~110 gated escape hatches.
Major findings tracked for follow-up (not blocking): M1 cosmos denom default, M2 redundant tx_ready emission for execute_*, M3 actions.go legacy names (analytics-only), M4 unmapped gated tools render nothing, M5 stale mcp-ts docs.

**2026-04-17T00:03:52Z**

Executed .tickets/v-pxuw-test-plan.md end-to-end without the emulator.

## Tier A — automated suites (post-rebase)

| Repo | Count | Result |
|---|---|---|
| vultisig-sdk | 1108 + 343 + 210 + 47 (4 skipped) | ✅ |
| mcp-ts | 263 / 263 | ✅ (exceeds ticket baseline of 252) |
| agent-backend | all packages ok | ✅ |
| vultiagent-app | 554 / 554 | ✅ (exceeds ticket baseline of 507) |

## §1a execCredits coverage added

Refactored `creditRepo` field from `*postgres.CreditRepository` to a minimal `creditStore` interface (`GetBalance` + `ListTransactions`) so tests can inject a fake without spinning up Postgres. Added 4 new cases in `credits_flag_test.go`:

- `TestExecCredits_CreditsEnabled_Detail_NoCreditRepo` (path 4)
- `TestExecCredits_CreditsEnabled_Summary` (path 5)
- `TestExecCredits_CreditsEnabled_Detail_WithRepo` (path 6, incl. ListTransactions paging args)
- `TestExecCredits_CreditsEnabled_BalanceError` (guard)

All 6 execCredits subtests green. Build + vet clean.

## Tier B — mcp-ts HTTP smoke

| Step | Result | Notes | Priority |
|---|---|---|---|
| B.1 tools/list | ✅ | 162 tools exposed; execute_send/swap/contract_call/lp_add/lp_remove present; defi_* 5 tools; uniswap_* 3; polymarket_* 8; build_*_send still in raw list but tagged `_meta.categories:["custom_send"]` — backend filter gates these, not mcp-ts | — |
| B.2 execute_send EVM | ✅ | nonce/gas_limit/max_fee/max_priority_fee all server-resolved via viem; ticket arg-name typo (`sender` → `from`) | — |
| B.3 SPL reject | ✅ | rejects with "SPL token sends … not yet supported. Use `build_spl_transfer_tx`". `token_decimals required` surfaces before supportedness check | P3 |
| B.4 Ripple reject | ✅ | rejects cleanly pointing at `build_xrp_send` | — |
| B.5 UTXO swap fee_rate | ✅ | `fee_rate: 3` present on `txArgs` for BTC→ETH via THORChain; ticket arg-name typo (`sender` → swap uses `sender` + optional `from_address` for token contract) | — |
| B.6 LP `kind` discriminator | ⚠️ | live THORChain mainnet has `PAUSELPDEPOSIT-*=1` for all pools (network state, not code); `execute_lp_remove` blocked by no-position. **Unit coverage** in mcp-ts `execute-lp.test.ts` (9/9) already asserts the discriminator | — |
| B.7 parseAmount | ✅ | `"$50"`, `"max"`, `"50%"` all resolve correctly against a funded EVM address; `"10 USDC"` on a native-ETH send resolves to an ETH amount that then hits a gas-arithmetic overflow — probably wants a "cross-token ambiguous amount" rejection instead | P3 |

## Tier C — agent-backend integration

| Step | Result | Notes |
|---|---|---|
| C.1 health | ✅ | /healthz 200 |
| C.2 auth | ✅ | Crafted local HS256 JWT with valid claims; `DEV_SKIP_AUTH=true` accepts unsigned-sig tokens |
| C.3 no-tools-field SendMessage | ✅ | Conversation proceeds without `tools` field in request (§1c Step 2 regression check) |
| C.3 mcp-ts fan-out | ⚠️ | LLM chose `tool-vault_chain` rather than fanning out to mcp-ts for a Solana balance lookup; the plumbing works (tool-call parts appear) but the specific fan-out path wasn't exercised by this prompt |
| C.4 vault_coin client-side bridge | ✅ | `tool-vault_coin` part with `action=add`, `coins=[{chain,contract_address,decimals,ticker}]` returned in non-stream JSON response |
| C.5 convert.go replay | ✅ | `TestMessageFromDB_RenamedToolPartsMigration` exercises all 13 entries in `renamedToolMap` + `TestMessageFromDB_LegacyPartTypesMigrateToCurrent` covers `data-confirmation` + `tool-schedule_task` rebuilds |
| C.6 credits live | ✅ | DEV-mode short-circuit → summary balance returned ($4.75 → $4.71) |

## Tier D — full stack via CLI

**P0 fix applied before Tier D could proceed.** Out of the box, every `vsig agent ask` returned an empty response with no tool calls — session id printed but nothing else. Root cause: agent-backend now emits AI SDK v5 streaming (`data:` lines only, event type in JSON `type` field — `text-delta`, `data-message`, `data-title`, `finish`, etc.), but the CLI's SSE parser was still keying off legacy SSE `event:` headers (`text_delta`, `message`, `done`). Every real event fell through the CLI's `message` default case and was discarded. This file-pair was introduced on the `refactor/ai-sdk-migration` branch that merged into v-pxuw (protocol_v1.go does not exist on `main`), so the drift is in v-pxuw's tree.

**Fix:** `clients/cli/src/agent/client.ts`
1. In `handleSSEEvent`, prefer `parsed.type` over the SSE `event:` header when present; map new type strings to the existing switch buckets (`mapV1EventType`: `text-delta→text_delta`, `data-message→message`, `tool-input-start|tool-output-available→tool_progress`, `data-title→title`, `data-actions→actions`, `data-suggestions→suggestions`, `data-tx_ready→tx_ready`, `error→error`, `finish→done`, default `ignore`).
2. Each `case` normalises between legacy-inline (`parsed.message`) and v1-wrapped (`parsed.data.message`) payload shapes.
3. `error` case accepts both `parsed.error` (legacy) and `parsed.errorText` (v1).
4. Added 2 new client.test.ts cases locking in the v1 routing + v1 error shape. CLI test suite now 49/49 (was 47).

Post-fix results:

| Step | Prompt | Result |
|---|---|---|
| D.1 balance | "whats my ETH balance" | ✅ "0.000075744423902647 ETH" — matches live chain |
| D.2 multi-chain | "list my BTC and ETH balances" | ✅ BTC 0.0000205 + ETH 0.000075…, both real |
| D.3 send prep | "prepare to send 0.0001 ETH to 0x…dEaD — do not sign" | ✅ tool fired; correctly surfaces insufficient-funds (vault is near-empty) |
| D.4 swap prep | "quote me a swap of 0.0001 ETH to USDC on Arbitrum" | ✅ `execute_swap` + `build_swap_tx` in tool_calls, all success |
| D.5 credits | "whats my credit balance" | ✅ $4.71 |
| D.6 vault_coin add | "add BONK (Solana) to my vault" | ✅ search_token then client-side vault_coin dispatch, "BONK has been added to your vault" |

## Tier E — tool-selection eval

Not run. `agent-backend/scripts/perf/` is a latency/cache benchmark, not a tool-selection accuracy harness. Ticket's §Tier E is predicated on a harness that doesn't exist today. Tier D qualitative signal was strong: LLM picked `execute_swap` for the swap prompt, `execute_send`-family for the send prompt, `credits` for the credit prompt, `vault_coin` for the add prompt.

## P0 / P1 count

- **1 P0 fixed in-branch** — CLI ↔ backend SSE protocol drift (AI SDK v5). Fix in `vultisig-sdk/clients/cli/src/agent/client.ts` with 2 new tests; rebuilt CLI v0.15.2 now parses v5 events.
- **1 P0 remediated in-branch** — §1a paths 4+6 had no test coverage; refactored to interface + added 4 new tests.
- **0 P1.**

## P2 / P3 punch list

- **P3 (B.3)**: `execute_send` on Solana with a token but no `token_decimals` surfaces "decimals required" before the "SPL not supported" message. Minor UX inversion — the LLM will read "decimals required" and try to look them up rather than immediately switching to `build_spl_transfer_tx`.
- **P3 (B.7)**: `"10 USDC"` as the `amount` on an Ethereum native-ETH send is accepted and converted to ETH, leading to a gas-arithmetic error rather than a "cross-token amount on native send — specify token" rejection.
- **P3 (ticket)**: `.tickets/v-pxuw-test-plan.md` §Tier B uses `sender` for `execute_send` input (actual schema is `from`) and for `execute_swap`'s wallet address (actual schema is `sender` and `from_address` is the token contract). Not blocking — just trip-hazards for the next runner.

## Ship-readiness

Branch is **ship-ready on this surface** subject to the remaining emulator work. The consolidated `execute_*` wire contract behaves correctly, backend orchestration and client-side bridges are intact, historical conversations replay cleanly, credits gating is right under both flag states. The only blocker found was the CLI SSE drift, which is now fixed and test-locked.

## Emulator-only gaps (deferred)

- Stepper UI transitions (swap/send confirmation card)
- Historical conversation re-render (visual verification; data-layer already covered by convert_test.go)
- Device MPC keysign round-trip from the app
- On-device vault_coin / vault_chain / address_book card rendering with the renamed tool surface
- Stepper event wiring for `tx_ready` payload from execute_send / execute_swap on mobile

## Files touched

- `agent-backend/internal/service/agent/agent.go` — introduced `creditStore` interface; field type change; nil-safe `SetCreditRepo`.
- `agent-backend/internal/service/agent/credits_flag_test.go` — +4 tests, fake creditStore, imports updated.
- `vultisig-sdk/clients/cli/src/agent/client.ts` — SSE event routing now supports AI SDK v5 payload-embedded types in addition to legacy `event:` headers.
- `vultisig-sdk/clients/cli/src/agent/__tests__/client.test.ts` — +2 tests pinning v1 routing.
- `vultisig-sdk/clients/cli/dist/index.js` — rebuilt.

Handoff doc: `.tickets/v-pxuw-test-plan.md`.
