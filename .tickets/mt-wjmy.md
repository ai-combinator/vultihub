---
id: mt-wjmy
status: open
deps: []
links: []
created: 2026-05-04T00:04:09Z
type: feature
priority: 2
assignee: Jibles
external-ref: gh-236,gh-346,gh-76
---
# Unified balance tool: SDK-backed get_unified_balance + JIT-fetch in execute_* validator

## Problem

`assertSourceBalanceSufficient` (`agent-backend/internal/service/agent/balance_validator.go:444`) — the only pre-dispatch fund-safety gate for `execute_send` / `execute_swap` — fails open when balance context is empty (cold context) OR when the requested asset is a token (`chainToBalanceTool` registry only routes to native-balance MCP tools). PR #236's removal of wire-side balances on the app side (`vultiagent-app#346`) makes cold context the default path, materially widening the gap. Gomes flagged this as MAJOR fund-safety in the cluster review; his takeover commit (`870859da`) addressed Neo's two preferably-blocking findings but did not close the fail-open. "Solved" means `execute_*` cannot dispatch with unverified balance for either native OR token assets across every chain the SDK covers.

All three PRs in the v-kmev cluster are on branch `v-kmev` (`agent-backend#236`, `vultiagent-app#346`, `mcp-ts#76`). This ticket is response-to-feedback work that lands on the existing branches — **no new branches**.

## Background

### Findings

**Validator state (agent-backend, branch `v-kmev`):**
- `assertSourceBalanceSufficient` at `internal/service/agent/balance_validator.go:444` fires only for `execute_send` / `execute_swap` (Neo's PR #220, comment at `executor.go:2068-2074`).
- Two fail-open paths today: cold context at `balance_validator.go:449` (`len(req.Context.Balances) == 0` → return `""`), and missing-entry at `balance_validator.go:462` (`findBalance(...)` returns `""` → return `""`).
- Comment at `balance_validator.go:464-470` literally describes the JIT-fetch fix as a deferred TODO: *"Per issue: 'if not pre-fetched, call the appropriate balance MCP tool' — that's a follow-up optimization. For now skip rather than block legitimate flows."*
- Call site: `executor.go:2075` inside `executeTool(ctx context.Context, ...)` — `ctx` is in scope, signature change is one hop.
- `chainToBalanceTool` registry at `data_tools.go:445-503` is native-only.
- Dead route: `polkadot → get_dot_balance` at `data_tools.go:488` — that MCP tool does not exist in mcp-ts.
- Existing dispatch primitive: `gatherBalances(ctx, msg, requested, forceRefresh bool)` in `data_tools.go`. PR #236 added the `forceRefresh` parameter.
- `execGetBalances` already writes results back to `req.Context.Balances` (data_tools.go ~line 124) — established precedent for validator mutating context.

**SDK state (`vultisig-sdk`):**
- `getCoinBalance({chain, address, id?})` at `packages/core/chain/coin/balance/index.ts:17-30` dispatches over a single `Record<ChainKind, CoinBalanceResolver>` covering 12 chain families: utxo, cosmos, sui, evm, ton, ripple, polkadot, bittensor, solana, tron, cardano, qbtc.
- `AccountCoinKey<T> = { chain: Chain, address: string, id?: string }` (`packages/core/chain/coin/AccountCoin.ts:7-9`). `id=undefined` ⇒ native (via `isFeeCoin`, `utils/isFeeCoin.ts:3`); `id` for tokens = contract address (EVM/TRON), SPL mint (Solana), denom or wasm contract (cosmos), jetton master (TON), coin_type (Sui), Cardano asset id.
- Resolver returns raw `bigint`. Decimals NOT required by callers.
- All resolvers Node-safe. Polkadot uses raw `state_getStorage` JSON-RPC + `@noble/hashes` + `bs58` specifically to avoid `@polkadot/util-crypto` bundle issues (`polkadot.ts:1-5`).
- Upstreams identical to mcp-ts: `api.vultisig.com/blockchair` for UTXO (`getBlockchairBaseUrl.ts:4-5`, `packages/core/config/index.ts:4`), `*-rest.publicnode.com` for cosmos (`cosmosRpcUrl.ts:5-16`), viem default RPCs for EVM. **No new env vars / API keys.**
- `knownTokensIndex` at `packages/core/chain/coin/knownTokens/index.ts:765-777` covers Ethereum, BSC, Polygon, Avalanche, Arbitrum, Optimism, Base, Blast, zkSync, Mantle, Hyperliquid, Solana (USDC/USDT/JUP/USDS), TON (USDT/NOT/DOGS/CATI/HMSTR/STON/stTON/tsTON), TRON (USDT), plus cosmos/THOR via `knownCosmosTokens`. Symbol→contract lookup: `knownTokens[chain].find(t => t.ticker.toUpperCase() === symbol.toUpperCase())` (`VaultBase.ts:1809`).
- Fallback: `searchToken` at `packages/sdk/src/tools/token/searchToken.ts:71` hits CoinGecko via `api.vultisig.com/coingecko` — already wired in mcp-ts as `searchTokenTool` (`mcp-ts/src/tools/evm/search-token.ts`).
- Ripple semantics caveat: SDK's `getRippleCoinBalance` returns spendable (total minus 10 XRP reserve). mcp-ts's existing `get_xrp_balance` returns total drops with explicit `actNotFound` note (`other-balance.ts:39-46`).

**MCP-ts state (branch `v-kmev`):**
- `@vultisig/sdk: ^0.21.0` is already a direct dep in `apps/agent-mcp/package.json`.
- Existing per-chain balance tools (registered in `mcp-ts/src/tools/index.ts:130-144` + cosmos spread at :135): EVM (13 chains), UTXO (Bitcoin/LTC/Doge/BCH/Dash + Zcash via shared tool), Solana, XRP, TRON, TON, Sui, Cardano; cosmos: Cosmos, THORChain, MayaChain, Osmosis, Kujira, Terra, TerraClassic. Tokens: ERC-20, SPL, TRC-20, TON jetton, Sui token. Each emits a different shape (`evm_get_token_balance` returns `{symbol, balance, decimals}`; `get_sui_token_balance` returns base-units only; `get_ton_jetton_balance` returns wallet address; etc.).
- Conspicuously absent from mcp-ts but present in SDK: Polkadot, Bittensor, Akash, Noble, Dydx, Zcash-as-distinct, QBTC. **SDK is a strict superset.**
- mcp-ts `CLAUDE.md:55` states *"Balance/fee tools use direct RPC calls via `src/lib/rpc.ts` (not SDK, to avoid heavy deps)"* — added by gomes in commit `e11bcf40` (2026-04-07). Rationale is now stale: SDK is already a dep, pulled in by swap-quote logic. Worth deleting / updating in this same PR.

**Cluster context:**
- Gomes' fund-safety MAJOR is on `agent-backend#236` (issue comment `4358759425`). Three suggested fixes: (1) fail-CLOSED, (2) server-side auto-fetch, (3) hard prompt precondition. This ticket implements a fourth: server-side JIT fetch via SDK-backed MCP tool.
- Neo's review of `agent-backend#236` framed the same hole as "doesn't make it worse, fix later" — disagreement on scope, not existence.
- Neo's `mcp-ts#76` review flagged the `as_of` semantics caveat (server-fetch time, not snapshot validity). The new tool inherits the `as_of` contract; should align with PR #236's `parseBalanceResult` regex behavior at `agent-backend/internal/service/agent/data_tools.go:548, 600`.

### External Context

- ShapeShift's comparable agentic repo (`/home/sean/Repos/shapeshift-agentic`) does in-tool balance validation in `apps/agentic-server/src/tools/initiateSwap.ts:233` via `validateSufficientBalance(sellAddress, sellAsset, sellAmountCrypto)` (`utils/balanceHelpers.ts:18`). Single unified `executeGetAccount({ address, network })` call returns native + tokens uniformly. This ticket is the equivalent posture for vultisig.

## Current Thinking

**Add one new MCP-ts tool — `get_unified_balance` (placeholder name)** that internally calls SDK's `getCoinBalance({chain, address, id?})`. Args shape: `{chain, address, contract_address?, ticker?}`. Resolution order inside the handler: (a) if `contract_address` supplied, use directly as `id`; (b) elif `ticker` supplied, resolve via `knownTokensIndex[chain]` lookup, then `searchToken` CoinGecko fallback; (c) else native (`id=undefined`). Returns `{balance: bigint-as-string, decimals?: number, as_of: ISO}` aligned with PR #236's `parseBalanceResult` contract. Tool stays internal-initially (kept off `llmFacingDropList` consideration — added to `tool_filter.go` exclusions so the LLM still uses unified `get_balances`, not this new tool directly). Why: agent-backend is the dispatcher; the new tool is a validator primitive, not a chat surface.

**Agent-backend changes (branch `v-kmev`, on top of PR #236):**
- Thread `ctx context.Context` into `assertSourceBalanceSufficient` signature; update call site at `executor.go:2075` (1 LOC) and the ~25 mechanical call sites in `balance_validator_test.go`.
- After existing `findBalance` returns `""` (or in cold-context branch), JIT-call the new MCP tool via existing `mcpProvider.CallTool` plumbing. On success, write result into `req.Context.Balances` (matches `execGetBalances` precedent).
- Preserve fail-open on (a) `req.Context == nil` — headless / scheduler path can't construct a JIT call (`addressForChain` needs `msg.Addresses`); (b) JIT call returns failed_chains non-empty or MCP unreachable (preserves current zero-value `&AgentService{}` test behavior).
- Replace or simplify `chainToBalanceTool` registry — eventually a single dispatch to `get_unified_balance`. (For this ticket: validator goes through new tool. Existing `get_balances` LLM-facing flow keeps current per-chain registry — migrating that is a follow-up; see Non-goals.)

**Why Path B (SDK-backed unified tool) over Path A (extend registry):**
- SDK is a strict superset of mcp-ts coverage. Path B closes the dead `polkadot → get_dot_balance` route for free and gives Bittensor/Akash/Noble/Dydx/Zcash/QBTC.
- `AccountCoinKey` shape is exact match for what the validator already has — `{chain, contract_address?, ticker}`.
- Token contract knowability: SDK has `knownTokens` + `searchToken` fallback chain. Path A had no built-in resolution and inherited the same problem.
- Path A is realistically ~150 LOC because each existing token-balance MCP tool emits a different shape; response normalization is non-trivial.
- Path B retires the `chainToBalanceTool` registry over time (this ticket starts the move), which advances the strategic end-state where MCP-ts is a thin SDK wrapper.

**Delete the stale "avoid heavy deps" line** in `mcp-ts/CLAUDE.md:55` (or update to reflect that SDK is now an explicit dep and balance tools should migrate). The implicit "don't go there" signal is discouraging the right refactor.

## Constraints

- **No new branches.** All three repos already have `v-kmev` open as PRs against main; this ticket lands as additional commits on those existing branches.
- Validator MUST remain fail-open on `req.Context == nil` — headless task scheduler (`cmd/scheduler`) and any caller without a populated MessageContext can't construct a JIT fetch.
- Validator MUST remain fail-open when MCP unreachable (`s.mcpProvider == nil` in zero-value test agents); otherwise existing `balance_validator_test.go` cases (~25 of them) break with mass mocking changes.
- New tool's `as_of` field MUST parse cleanly through `agent-backend/internal/service/agent/data_tools.go:548, 600` (`parseBalanceResult`'s case-insensitive `(?i)As Of:` regex + JSON `as_of` field lookup) — the existing PR #236 contract.
- TSS signing constraint: this is a balance-read primitive only. Do not couple to swap/sign flows.
- Preserve back-compat on the existing `get_balances` LLM-facing tool — its schema and behavior don't change in this ticket. Validator gets the new tool; `get_balances` migration is deferred.

## Assumptions

- `knownTokensIndex` + `searchToken` fallback resolves enough of the LLM-emitted ticker space for the realistic execute_swap input distribution. Long-tail tokens may require the LLM to supply `contract_address` explicitly — fall back to fail-open with a structured "token contract unknown — supply contract_address" log marker.
- mcp-ts's existing runtime config (env vars, RPC URLs at `lib/rpc.ts`) is sufficient for SDK's `getCoinBalance` — sub-agent verified upstreams are identical, but no live test was run.
- Neo's PR #220 server-side balance precondition is the gating layer that makes this validator work load-bearing; this ticket assumes it stays in place.
- The new MCP tool can be added to mcp-ts on the existing `v-kmev` branch without breaking the current PR #76 `as_of` work (additive change).

## Non-goals

- Migrating `gatherBalances` (the existing LLM-facing `get_balances` fan-out) to use the new unified tool. Separate cleanup ticket.
- Retiring per-chain MCP-ts balance tools (`evm_get_balance`, `get_sol_balance`, `get_atom_balance`, etc.). Separate strategic ticket.
- Adding Stride or any chain the SDK does not currently cover.
- Changing prompt rules for when the LLM should call `get_balances`.
- Touching `execute_swap` / `execute_send` build/sign/broadcast paths.
- Resolving the `as_of`-as-fetch-time semantics nuance Neo flagged on `mcp-ts#76` (block_height augmentation) — separate ticket.
- USD-value enrichment (Neo's #5 deferred finding on `agent-backend#236`).

## Dead Ends

- **Native-only JIT fix in validator** (~50 LOC, prior subagent's plan). Closes only the native-asset half of the fund-safety gap. User explicitly rejected because token swaps would still fail open — half-fix feels pointless when the architectural cost of the full fix is similar.
- **Path A: extend `chainToBalanceTool` to a token-aware registry** routing to existing per-chain token-balance MCP tools (`evm_get_token_balance`, `get_spl_token_balance`, etc.). Realistic ~150 LOC once response normalization for 5 different shapes is handled. Doesn't move the architecture forward, doesn't close the SDK-only chain coverage gap (Polkadot, Bittensor, Akash, Noble, Dydx, Zcash, QBTC), doesn't fix the dead `get_dot_balance` route, and the contract-knowability problem still needs separate handling.
- **Fail-CLOSED + force LLM to call `get_balances` first** (CodeRabbit's post-takeover suggestion + Gomes' option 1). Pushes the burden to the model; no behavioral guarantee. Also still hits the token-coverage gap because the LLM's `get_balances` call would route through the same native-only registry.
- **Refactor MCP-ts balance tools to consume SDK as their backing implementation** (strategy step 3). Too big for a response-to-feedback ticket; deferred. The "avoid heavy deps" rationale that was blocking this is now stale.
- **WASM-compile the SDK so Go agent-backend can call it directly.** Months of work; the realistic multi-language path stays "agent-backend → MCP-ts → SDK."

## Open Questions

- **Tool name.** `get_unified_balance` is a placeholder. Naming convention in mcp-ts? Plan-mode call.
- **Response shape.** Flat `{balance: string, decimals?: number, as_of: ISO}` or richer (matching existing `parseBalanceResult` text-format `Balance: 0.42 BTC | As Of: ...` + JSON `as_of`)? Should align with PR #236's `as_of` contract — verify shape against the existing regex at `agent-backend/internal/service/agent/data_tools.go:548`.
- **Ripple spendable-vs-total.** SDK returns spendable (minus 10 XRP reserve); existing tool returns total. Surface both? Pick one? For fund-safety, stricter (spendable) is arguably correct, but downstream parsers (and any prompt rules that quote "your XRP balance is X") may double-subtract. Decide.
- **LLM-facing or internal-only.** Probably internal-only initially (validator uses it; `get_balances` stays the LLM-facing surface). Confirm; if internal-only, add to whatever exclusion list keeps it off the model's tool catalogue.
- **Caching.** Should the new tool have its own cache, or rely on agent-backend's `req.Context.Balances` write-back as the cache layer? Latter is consistent with existing `execGetBalances` behavior — preferred default.
- **Token contract resolution failure mode.** If `knownTokens` + `searchToken` both miss, fail-open with a structured "token contract unknown" log marker (preserves current behavior on long-tail tokens) — confirm this is acceptable, or escalate to fail-CLOSED for unresolvable tokens.
- **Stale CLAUDE.md line.** Delete or update `mcp-ts/CLAUDE.md:55` ("avoid heavy deps") in this same PR? Probably yes — implicit signal discouraging the broader refactor.
