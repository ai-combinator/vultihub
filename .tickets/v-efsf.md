---
id: v-efsf
status: in_progress
deps: []
links: []
created: 2026-04-28T02:07:14Z
type: task
priority: 1
assignee: Jibles
---
# agent-backend: rewrite prompt for execute_* tool family

## Objective

Rewrite `agent-backend/internal/service/agent/prompt.go` to teach the LLM the `execute_*` tool family (`execute_send`, `execute_swap`, `execute_contract_call`, `execute_lp_add`, `execute_lp_remove`) instead of the deleted `build_*` family. Currently the prompt instructs the model to call tools that no longer exist on mcp-ts `surgical/v-qjvi-execute-foundation`, so the LLM falls back to surviving `build_*` tools (e.g. `build_fin_swap`) for use cases they don't cover, thrashes on intermediate prep tools, and frequently trips the loop breaker.

## Context & Findings

- mcp-ts `surgical/v-qjvi-execute-foundation` (PR vultisig/mcp-ts#49) DELETED the generic single-flow builders: `build_swap_tx`, `build_evm_tx`, all `build_*_send` (BTC/LTC/DOGE/BCH/DASH/ZEC/THOR/MAYA/GAIA/OSMOSIS/KUJIRA/SOL/SUI/TON/TRX/CARDANO), and `build_thorchain_lp_add` / `build_thorchain_lp_remove`. Replaced by the five `execute_*` tools.
- mcp-ts INTENTIONALLY KEPT (different use cases — do not retire in this work): Rujira/THORChain DeFi (`build_fin_swap`, FIN orderbook, Rujira staking, Ghost lending, Range CCL), Pumpfun (`build_pumpfun_create`), and 6 token-transfer escape hatches (`build_spl_transfer_tx`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build_xrp_send`, `build_cw20_transfer`) — those last 6 are tracked separately under ticket `v-ujuc`.
- agent-backend's `prompt.go` currently references deleted tool names ~36 times: `build_swap_tx` 18+ mentions, `build_evm_tx` 15+ mentions, plus several `build_*_send`. Zero mentions of any `execute_*` tool.
- Closed PR vultisig/agent-backend#128 (`refactor/v-pxuw-tool-consolidation`, state=CLOSED, +3228/-5844) included the prompt rewrite as a sub-diff (`prompt.go` +110/-559, `prompt_test.go` +48/-3, `prompt_launchsurface_test.go` -167 deleted). Locally available as branch `refactor/v-pxuw-tool-consolidation` for cherry-pick reference.
- Demonstrated failure mode (Android emulator, 2026-04-28): user prompts "swap 0.0001 ETH on arb to USDC on arb" → LLM calls `search_token` → `build_fin_swap` (THORChain CosmWasm — wrong, can't do Arb↔Arb) → tool returns `textResult` error → "Transaction build failed" red banner → LLM thrashes on `convert_amount` → loop_breaker trips at iteration 5 (`tool_loop_broken: thrashing detected`, `cache_ratio=0.97`).
- After prompt fix, the same prompt should land directly on `execute_swap` with simple params; the thinking-block timeline shrinks from 5+ rows to 1.

**Rejected approaches:**
- Reopen and merge PR #128 wholesale — too large (+3228/-5844), bundled with CRUD consolidation + balance fan-out; cherry-pick the prompt sub-diff into a slim PR instead.
- Update tool descriptions in mcp-ts to bias selection — does not address the prompt's explicit `build_*` instructions; the model follows the prompt.

## Files

- `agent-backend/internal/service/agent/prompt.go` — main rewrite. Reference the equivalent file on local branch `refactor/v-pxuw-tool-consolidation` for the +110/-559 diff shape (use `git show refactor/v-pxuw-tool-consolidation:internal/service/agent/prompt.go` from inside the agent-backend repo).
- `agent-backend/internal/service/agent/prompt_test.go` — update assertions for the new prompt content (#128 diff: +48/-3).
- `agent-backend/internal/service/agent/prompt_launchsurface_test.go` — delete (#128 removed it; its assertions were tied to the legacy prompt).
- Reference: `agent-backend/internal/service/agent/tool_filter.go` + `tool_filter_test.go` — verify any `build_*` allowlist entries match the surviving tools listed in the Context section above; remove deleted ones.

## Acceptance Criteria

- [ ] `prompt.go` contains zero references to `build_swap_tx`, `build_evm_tx`, or any deleted `build_*_send` tool name (grep clean).
- [ ] `prompt.go` teaches each `execute_*` tool with a concrete intent → tool mapping table covering: send, swap (same-chain + cross-chain), EVM contract call, THORChain LP add, THORChain LP remove.
- [ ] Explicit precedence rule documented: prefer `execute_*` over any surviving `build_*`; explicit "`build_fin_swap` is FIN-DEX-on-THORChain only — never use for EVM↔EVM swaps."
- [ ] Surviving `build_*` tools (Rujira DeFi, Pumpfun, 6 token-transfer escape hatches) remain documented as escape hatches, not first-class.
- [ ] `prompt_test.go` updated; `prompt_launchsurface_test.go` removed. `go test ./internal/service/agent/...` passes.
- [ ] `tool_filter.go` allowlist / deny-list aligns with mcp-ts' `surgical/v-qjvi-execute-foundation` registered tool set.
- [ ] Manual smoke against vultisig/vultiagent-app PR #242 + vultisig/mcp-ts#49: a fresh-conversation prompt of "swap 0.0001 ETH on arb to USDC on arb" lands directly on `execute_swap` (single tool call in the thinking block), not on `build_fin_swap`. Loop breaker does not fire.

## Gotchas

- Don't strip the surviving `build_*` tools from the prompt — they're real tools with non-overlapping use cases. The prompt should teach `execute_*` as the primary path AND keep escape-hatch documentation for the kept `build_*` family.
- `build_fin_swap` (Rujira) is not the same thing as `build_swap_tx` (deleted). Easy to conflate by name; the FIN one is THORChain CosmWasm via Rujira's FIN DEX, not a generic EVM swap.
- mcp-ts may add more `execute_*` tools after #49 (`execute_query`, `execute_crud`, etc. per follow-up tickets v-uowg / v-zlov). The prompt rewrite here covers the five action-class tools currently in #49 only.
- `prompt_launchsurface_test.go` deletion is intentional in #128 — its test fixtures were tied to legacy build_* invocation patterns. If new launch-surface coverage is wanted, file it as a separate ticket.
- Cache-bust concern: if the LLM provider has long-lived KV-cache for the system prompt, expect a brief tail of cached-context misroutes after deploy. The agent-backend's `cache_ratio` log field can confirm.

## Notes

**2026-04-28T02:21:09Z**

Rewrote prompt.go (1220→962 lines) to teach execute_send/swap/contract_call/lp_add/lp_remove. Added Tool Routing table, explicit Precedence section ('execute_* over build_*'), and Surviving Escape Hatches section (Rujira FIN/GHOST/staking/CCL, Pumpfun, 6 token-transfer specials). Removed all references to retired build_swap_tx / build_evm_tx / build_*_send from LLM-facing prompt content (one Go developer comment retains the names for context — not in the prompt). Updated structuredOutputSection, behavioralHardeningSection, toolExamplesSection to reference execute_* instead of retired build_* family. Preserved launchsurface flag-aware machinery (renderSystemPrompt + chainsForPrompt + EVM_CHAINS substitution + scheduling on/off gating), BuildVaultInfoSection P0-8 privacy fix, and BuildSystemPartsForCaching cache-aware infrastructure. Removed dead ConfirmActionTool / BuildPolicyTool. Deleted prompt_launchsurface_test.go (per ticket — its assertions were tied to the old SAME-CHAIN SWAPS / provider-substitution prompt). Rewrote VA-169 fiat test to assert execute_*'s natural-language amount strings ('$10', '50%', 'max') instead of build_*'s amount_usd/value_usd. Added regression tests: (1) zero deleted-tool references, (2) surviving escape hatches still documented + FIN scoping, (3) explicit Precedence section. tool_filter.go is category-based (no name-level allowlist), no changes needed. go test ./internal/service/agent/... passes; go build ./... clean. Manual smoke against vultiagent-app PR #242 + mcp-ts#49 still pending — flagged as outstanding.
