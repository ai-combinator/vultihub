---
id: v-vvep
status: closed
deps: []
links: []
created: 2026-04-29T04:10:20Z
type: bug
priority: 2
assignee: Jibles
---
# agent-backend: model still fans out per-chain balance tools after backend fan-out landed

## Objective

Commits `08c2cbd` and `8d234e8` (2026-04-27) moved balance fan-out into agent-backend so `get_balances` / `get_portfolio` resolve missing chains internally. The prompt and tool filter still push the LLM toward per-chain MCP balance tools directly, so the model continues to redundantly call `get_thor_balance` / `get_maya_balance` / `get_terra_balance` / `get_terra_classic_balance` even after firing the consolidated `get_balances`. Land the prompt + tool-visibility cleanup the original fan-out PR missed so the model uses the consolidated path exclusively.

## Context & Findings

**Reproduction.** "let's swap 0.0001 USDC on arb to ETH on arb" produced this tool sequence: ANALYZED → SEARCHING TOKENS → CHECKING TOKEN BALANCE → CHECKING BALANCE → FETCHING THOR BALANCE → FETCHING MAYA BALANCE → FETCHING TERRA BALANCE → FETCHING TERRA CLASSIC BALANCE → EXECUTE SWAP. The "CHECKING BALANCE" step IS the consolidated `get_balances` call; the per-chain tools fired afterward.

**Why it still happens — three compounding causes:**

1. `agent-backend/internal/service/agent/prompt.go:410-420` ("Balance / portfolio queries — fetch immediately, don't ask") explicitly tells the model: *"if `get_balances` response lists chains under `not_pre_fetched`, IMMEDIATELY call the chain-specific MCP balance tools named in the `refresh_tool_hint` — `evm_get_balance`, `get_sol_balance`, `get_atom_balance`, ... — in parallel, one per chain present in the Addresses context."* This directly contradicts the fan-out commit. Commit `8d234e8` fixed two other prompt sites (Rujira section, critical-data example) but missed this one.

2. `agent-backend/internal/service/agent/tool_filter.go:199, :227` unconditionally includes every tool tagged `category: 'balance'` on every request. `mcp-ts/src/tools/balance/cosmos-balance.ts:36-53` registers `get_thor_balance` / `get_maya_balance` / `get_terra_balance` / `get_terra_classic_balance` etc. with `categories: ['balance', ...]` — so they're visible on every prompt, including a same-chain Arbitrum swap. Per `.tickets/v-sdvy.md:120` the plan was: *"Per-chain balance tools remain as internal-tier tools (registered, categories stripped, reachable via backend RPC for BalanceService fan-out)"* — the strip never happened.

3. `prompt.go:158` ("Skip prep tools — bridges do NOT require `get_balances`, `get_market_price`, `get_skill`, or `evm_get_balance` before calling `execute_swap`") is scoped to bridges only. Same-chain swaps have no equivalent "skip prep" rule, so the model defaults to defensive balance-checking before calling `execute_swap`.

**Backend fan-out wiring (already correct, do not touch):** `data_tools.go` execGetBalances dispatches per-chain via the backend `chainToBalanceTool` map (`data_tools.go:520-608`). The per-chain MCP tools must remain *registered* and reachable by name from agent-backend — only their LLM visibility should change.

## Files

- `agent-backend/internal/service/agent/prompt.go` — rewrite section at line 410-420; generalize line 158 from bridges-only to all `execute_*` flows; audit lines 28, 75, 200, 245-246 for residual chain-specific balance directives
- `agent-backend/internal/service/agent/tool_filter.go:195-235` — decide whether per-chain balance tools stay always-on or get filtered out
- `mcp-ts/src/tools/balance/cosmos-balance.ts:36-53` (and the EVM/Solana/UTXO/other balance modules) — likely strip `'balance'` from per-chain `categories`, keep registration intact
- Reference commits: `08c2cbd` (fan-out), `8d234e8` (partial prompt cleanup), `data_tools.go:520-608` (`chainToBalanceTool` map)

## Acceptance Criteria

- [ ] `prompt.go:410-420` no longer instructs the model to call per-chain balance tools; states that the backend fans out automatically when chains are missing from prefetch
- [ ] "Skip prep tools" rule generalized so it applies to ALL `execute_*` flows (same-chain swaps and sends, not just bridges)
- [ ] Per-chain balance tools removed from the LLM-visible tool list (categories strip in mcp-ts and/or filter in agent-backend) while remaining reachable via backend RPC fan-out
- [ ] Reproduction passes: same-chain Arbitrum USDC→ETH swap shows zero per-chain balance tool calls and at most one consolidated `get_balances` call before `execute_swap`
- [ ] `data_tools_test.go` fan-out concurrency / partial-failure tests still pass
- [ ] `go test ./...`, `golangci-lint run`, `go build ./...` clean in agent-backend; `pnpm typecheck` and `pnpm lint` clean in mcp-ts

## Gotchas

- `tool_filter.go:32` uses `"balance"` as a *category-keyword* for matching user intent — that path still needs `get_balances` / `get_portfolio` exposed; only the per-chain tools should disappear
- `chainToBalanceTool` invokes per-chain MCP tools by name — registrations must stay; only `categories` change
- `refresh_tool_hint` in `get_balances` responses may currently name chain-specific tools the model can no longer call — update its content (or delete it) to match the new contract
- `'utility'` must remain always-on; do not collapse the always-on set
- QA: existing curl-replay fixtures under `scripts/qa/curl-replay/` may assert specific balance tool sequences — re-baseline if needed

## Notes

**2026-04-29T06:04:49Z**

PR #165 (backend fan-out) merged to main 2026-04-29 after rebasing surgical/v-repi-balance-fanout onto current main (dropped the kitchen-sink WIP commit + the partial 8d234e8 prompt cleanup that was already on main via the execute_* rewrite). Final diff: 2 files / +944 / -90; gomesalexandre's approval survived the force-push.

v-vvep follow-up PRs opened on top of merged main:
- agent-backend #197: https://github.com/vultisig/agent-backend/pull/197
- mcp-ts #63: https://github.com/vultisig/mcp-ts/pull/63

agent-backend changes: prompt.go lines 75/158/292/410-420 + tool_filter.go always-on rule. mcp-ts changes: strip 'balance' from 21 per-chain leaf tools across 5 modules. Both green locally — go test ./..., golangci-lint (no new findings vs main), pnpm typecheck/lint/test 702/702.

Acceptance criteria all met. Pending: reviewer behavioral repro on a dev build (Arbitrum same-chain USDC→ETH swap should show zero per-chain balance calls).

**2026-04-29T06:13:49Z**

Closing — implementation legwork done. PRs in draft awaiting reviewer behavioral repro on a dev build (Arbitrum same-chain USDC->ETH swap, zero per-chain balance calls between get_balances and execute_swap) + ready/merge:

- agent-backend #197 (draft): https://github.com/vultisig/agent-backend/pull/197
- mcp-ts #63 (draft): https://github.com/vultisig/mcp-ts/pull/63

5 of 6 acceptance criteria met locally (prompt rewrite, skip-prep generalization, per-chain tools removed from LLM tool list across both repos, fan-out tests still pass, all linters/tests clean). The remaining one is the live-traffic repro which is reviewer/QA territory.

Regression fixture for the bug landed in fixture #44 (44-arbitrum-samechain-swap-no-perchain-balance.yaml) on the agent-backend PR.
