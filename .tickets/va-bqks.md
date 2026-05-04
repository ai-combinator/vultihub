---
id: va-bqks
status: closed
deps: []
links: []
created: 2026-04-21T02:58:26Z
type: task
priority: 2
assignee: Jibles
---
# Backend-owned portfolio fetch + unified get_balances tool

## Objective

Move portfolio/balance fetching entirely to agent-backend, served by one shared endpoint that backs both the portfolio UI and the agent's tool. Replace the ~25 per-chain MCP balance tools and the existing `get_balances` / `get_portfolio` agent-backend tools with a single `get_balances` tool whose schema is designed for weak-model reliability (Gemini Flash 3, Haiku 4.5). Same backing cache feeds an auto-injected summary on every agent turn so simple "net worth" questions don't require a tool round-trip.

## Context & Findings

**Today's split is structurally awkward and has a real product gap.** Frontend pre-fetches EVM native + SOL balances on every send (blocking, 60s React Query cache, ~25 RPC calls in parallel). Backend's `get_balances` / `get_portfolio` fans out to MCP per-chain tools for the rest. The split:
- Two code paths fetching the same conceptual data (`vultiagent-app/src/features/agent/lib/queries.ts:fetchAllBalances` vs `agent-backend/internal/service/agent/data_tools.go:gatherBalances`)
- Frontend doing chain-specific business logic in violation of repo CLAUDE.md ("business logic in vultisig-sdk")
- LLM gets only natives in `MessageContext.Balances`; routing to "call get_balances for the rest" is an implicit prompt instruction at `agentContext.ts:68` that weak models miss
- **Real gap:** after `vault_coin add UNI`, no path exists for `get_balances` to return UNI's balance. Pre-fetch is native-only; fan-out via `evm_get_balance` is native-only; the only escape is the gated `evm_get_token_balance` behind `chain_balance` keyword filter — unreliable
- Send is blocked by the prefetch (~1-3s on first message in 60s)

**Cache architecture:**
- Redis (already in agent-backend per CLAUDE.md), key `balances:{public_key}:{hash(sorted_coin_ids + addresses)}` so vault mutations automatically miss cache (free correctness — no explicit invalidation cascade for `vault_coin`/`vault_chain`)
- TTL 60s; explicit invalidation on action mutations (after `execute_send` completes, drop source-chain entry) — TTL alone leaves an action-completion stale window
- `singleflight.Group` (`golang.org/x/sync/singleflight`) on cache miss to dedupe concurrent app-startup + agent-tool + UI-mount requests for the same key

**Tool schema** — researched against weak-model failure modes, validated against v-pxuw §3 schema rules and Gemini function-calling docs:

```
get_balances({
  chain?:        string   // "eth" | "Ethereum" | "eth,sol,btc" — single string, comma-list parsed server-side
  token?:        string   // "USDC" | "0xA0b8…48" — symbol or contract; regex disambiguates
  fiat?:         string   // ISO code, default "USD"
  include_defi?: boolean  // default false; flips on for "show positions" / "net worth" prompts
  refresh?:      boolean  // default false; only set when user explicitly asks to refresh
})
```

Five flat optional fields. Zero-arg call is the most common path. Comma-list `chain` (not `chains: string[]`) chosen because Flash routinely emits `chain: "eth"` against array schemas (forgetting the wrapper) and the call rejects; single string + server split mirrors the existing `amountString` natural-language pattern. `token` accepts symbol OR contract address with server-side regex disambiguation rather than `oneOf` (Gemini doesn't support oneOf). `chain` + `token` AND together. No enums (chain set is >25, violates v-pxuw's <10 enum cardinality rule).

**Result shape echoes interpretation per v-pxuw §3 design rule:**

```
{
  as_of, cache: {hit, stale_seconds},
  resolved_filters: {chains, token, fiat, include_defi, refreshed},
  balances: [{chain, symbol, amount, usd_value, contract_address?, source, decimals}],
  totals: {total_fiat, total_fiat_defi_only, by_chain: [{chain, fiat, row_count}]},
  failed_chains: [{chain, error}]
}
```

`source` tag (`native|erc20|spl|cosmos|staked|lp|lending`) prepares the surface for the future `include_defi` work. Pre-computed totals because Flash and Haiku reliably miscount when summing arrays. `failed_chains` makes partial failure recoverable rather than aborting.

**Auto-inject on every turn — comprehensive, not native-only.** Read from cache, injected into `MessageContext`. The summary should make ~95% of balance questions answerable directly from context. Three separate pieces:

1. **`balances`** — every vault row with non-zero balance and `usd_value > $1` (or unknown USD value, in case price lookup failed). Each row: `{chain, symbol, amount, usd_value, source}`. Capped at top 50 by `usd_value` as a safety bound. Typical real user: 5-30 rows, ~2-4KB. Token budget tradeoff is favourable — system prompts are 10-20KB; this saves multiple tool round-trips per session.
2. **`tracked_coins`** — every coin in the vault as `{chain, symbol, contract_address?}` regardless of balance. ~50-200 bytes. **This is the load-bearing piece for "asked about something not in context" cases** — the LLM uses it to disambiguate "absent because dust/cold-cache" (call `get_balances({token})`) from "absent because not tracked" (suggest `vault_coin add`).
3. **`totals`** — `{total_fiat, by_chain[].fiat}`. Pre-computed because Flash and Haiku miscount when summing arrays.

Plus a single `freshness` marker: `{source: "cache"|"cold"|"stale", as_of?, age_seconds?}` so the LLM knows whether to trust the snapshot or call the tool.

**Cold-cache handling (must be designed in, not deferred):** when the auto-inject runs and the cache is empty/stale beyond TTL, inject `tracked_coins` from `MessageContext.Coins` (always available, app sends every turn) but mark `balances: []` and `freshness.source: "cold"`. The LLM sees the universe of tracked tokens but no values, and the prompt instructs it to call `get_balances` to populate. Never block the turn on a fresh fetch. Pre-warm on app startup keeps cold-cache the edge case (long idle, scheduler-only flow).

**LLM routing rules (to be encoded in `prompt.go`):**
- "User asks about a balance and the value is in `balances`" → answer from context
- "User asks about a token in `tracked_coins` but absent from `balances`" → call `get_balances({token})` (dust-filtered or cold cache; tool returns precise value)
- "User asks about a token NOT in `tracked_coins`" → suggest `vault_coin add <token>`; do NOT call `get_balances` (it won't find the token)
- "freshness.source is `cold`" → call `get_balances` before answering balance/portfolio questions
- "User says 'refresh' / 'check again' / 'is this current'" → call `get_balances({refresh: true})`

**Endpoint shape:** `POST /agent/balances` with `MessageContext` body (same shape SendMessage already uses — stateless, no per-user vault registration). Returns the result shape above. Used by:
- App startup (after vault loaded, parallel with `/conversations/list`) — pre-warm cache
- Portfolio UI screen — primary data source (replaces app-side `fetchAllBalances`)
- Internal call from agent's `get_balances` tool — same Go service layer, not an HTTP self-call

**Decisions and rejected approaches:**

- **Keep tool name `get_balances`** (not `fetch_balances`) — `get_*` is established read prefix per v-pxuw §3, and it avoids prompt-rewrite churn against the existing routing table
- **Drop `get_portfolio` entirely** — subsumed by `get_balances({include_defi: true})` returning `totals.total_fiat`. One tool, not two
- **Stateless endpoint, not server-side vault registration** — matches existing SendMessage pattern; one less concept to maintain
- **`include_defi` defaults false** — DeFi adds latency and risks overcounting on simple "what's my ETH balance" turns; flip true on explicit intent. Revisit once positions ship
- **No `exclude_chain` param** — second list parameter that weak models will fill incorrectly; workaround is to list the chains you want
- **Single-flight in-process only (per pod)** — accept some cross-pod redundancy; cluster-wide locking is overkill at current scale
- **Migrate portfolio UI in same PR** — shipping the endpoint and tool while the app still uses `fetchAllBalances` leaves three balance code paths during the transition. Better to delete the old path immediately
- **Don't support arbitrary-token lookup outside vault** — product story is "if you care about a token, add it to your vault." Removes the entire `chain_balance`-gated escape hatch category from the LLM surface
- **Multi-vault aggregation deferred** — current `MessageContext` carries one active vault; cross-vault portfolio is a separate ticket

**Per-chain MCP balance tools:**
- Stay registered as MCP tools (agent-backend calls them as RPCs from the new BalanceService)
- Lose their `chain_balance` `_meta.categories` so the LLM never sees them

## Files

**agent-backend (primary):**
- `internal/service/agent/data_tools.go` — replace `GetBalancesTool` + `GetPortfolioTool` with single `GetBalancesTool`; replace `execGetBalances` + `execGetPortfolio` with one handler that calls into the new BalanceService
- `internal/service/agent/balance_service.go` (new) — `BalanceService` with `Fetch(ctx, MessageContext, params) (BalanceResult, error)`. Owns cache lookup, single-flight, fan-out, normalization. Shared between REST endpoint and tool handler
- `internal/api/balances.go` (new) — `POST /agent/balances` handler; thin wrapper that calls BalanceService
- `internal/api/server.go` — register the route under existing JWT auth middleware
- `internal/cache/redis/balances.go` (new) — typed cache layer over existing Redis client; key composition + TTL
- `internal/service/agent/context.go` — auto-inject `{balances, tracked_coins, totals, freshness}` into `MessageContext` on each turn. Read `balances` + `totals` from cache (non-blocking; empty + `freshness.source: "cold"` if cache miss/stale). `tracked_coins` always populated from `MessageContext.Coins` regardless of cache state
- `internal/service/agent/prompt.go` — drop the "for other chains, call get_balances or chain-specific tool" routing instruction. Add the four LLM routing rules from the auto-inject section above (in-context vs not-tracked vs cold vs refresh) — these are what enable the LLM to use the auto-inject correctly
- `internal/service/agent/executor.go` — invalidate balance cache for source chain after `execute_send` / `execute_swap` / `execute_lp_*` success
- `data_tools_test.go`, plus new `balance_service_test.go`, `balances_test.go` (handler), `redis/balances_test.go`

**mcp-ts:**
- `src/lib/toolCategories.ts` — drop `chain_balance` from `Categories` (or mark internal-only)
- `src/tools/index.ts` — strip categories from per-chain balance tools so `tool_filter.go` never surfaces them; tools stay registered for backend RPC calls

**vultiagent-app:**
- `src/features/agent/lib/queries.ts` — delete `fetchAllBalances`, `balancesQueryOptions`
- `src/services/agentContext.ts` — delete the `extras.balances` injection path; delete the `Pre-fetched balances (EVM and Solana)…` instruction line; delete `fetchEthBalance` / `fetchSolBalance` / `fetchEthPrice` / `fetchSolPrice` if no remaining callers
- `src/features/agent/hooks/useAgentTools.ts` — drop the `ensureQueryData(balancesQueryOptions(...))` block; `buildAgentContext` no longer awaits a balance fetch
- App startup site (find via `useConversations` / `listConversationsApi` mount) — fire `POST /agent/balances` in parallel with the conversation list call once vault is loaded
- Portfolio UI screen — migrate from local `fetchAllBalances` to `POST /agent/balances`; delete obsolete app-side balance/price fetchers

**Reference patterns:**
- `mcp-ts/src/tools/execute/execute_send.ts` — flat-schema + Zod `.transform()` + echo-back template; copy this style for the new tool def
- `mcp-ts/src/tools/execute/_amountResolver.ts` — natural-language string parsing with LLM-readable errors
- `agent-backend/internal/service/agent/data_tools.go:520-608` — current `chainToBalanceTool` map; reference for which MCP tool to dispatch per chain
- `vultisig-sdk` `normalizeChain` (added in v-pxuw task #1) — chain alias resolution for `chain` param parsing

## Acceptance Criteria

- [ ] `POST /agent/balances` endpoint exists, accepts `MessageContext` body + the 5 schema params, returns the documented result shape
- [ ] Redis cache keyed on `pub_key + hash(coin_set + addresses)`, 60s TTL, single-flight on miss
- [ ] Action handlers (`execute_send`/`execute_swap`/`execute_lp_*`) invalidate the cache slice for the affected chain on success
- [ ] Single `get_balances` tool replaces both `get_balances` + `get_portfolio` in agent-backend; tool definition matches the recommended schema; `get_portfolio` is fully removed
- [ ] Tool reads cache via the same BalanceService used by the REST endpoint; cache miss triggers synchronous fetch
- [ ] `MessageContext` auto-injection on every turn includes: `balances` (vault rows with non-zero balance and `usd_value > $1`, top 50 by value, from cache), `tracked_coins` (every vault coin's chain+symbol+contract, always present), `totals` (pre-computed grand + per-chain), `freshness` (`{source: "cache"|"cold"|"stale", as_of?, age_seconds?}`). Reading the cache is non-blocking — cold cache yields `balances: []` + `freshness.source: "cold"`, never delays the turn
- [ ] System prompt encodes the four routing rules: in-context → answer; in `tracked_coins` but not in `balances` → call `get_balances({token})`; not in `tracked_coins` → suggest `vault_coin add`; `freshness.source: "cold"` → call `get_balances` before answering
- [ ] Eval / manual test: "what's my UNI balance?" works in three states — UNI in vault with funds (answers from context), UNI in vault but dust (calls tool), UNI not in vault (suggests `vault_coin add`)
- [ ] Frontend `fetchAllBalances` + `balancesQueryOptions` + `useAgentTools` prefetch path deleted; `buildAgentContext` no longer awaits balance work
- [ ] App startup pings `POST /agent/balances` in parallel with `/conversations/list` once vault is loaded
- [ ] Portfolio UI screen reads from `POST /agent/balances`; no remaining direct RPC calls in the app for balance/price work covered by the endpoint
- [ ] Per-chain MCP balance tools stay registered (for backend RPC dispatch) but `_meta.categories` is stripped so they're invisible to the LLM
- [ ] `agentContext.ts:68` routing instruction line is removed; `prompt.go` updated to reference single `get_balances` tool
- [ ] "Add UNI to vault, ask balance" works end-to-end: `get_balances({token: \"UNI\"})` returns the on-chain ERC-20 balance via the SDK fan-out path
- [ ] All test suites green: `go test ./...` (agent-backend), `pnpm test` (mcp-ts), `yarn test` (vultiagent-app); type-check + lint clean in each repo

## Gotchas

- Cache key MUST include the vault coin-set hash, not just `public_key` — otherwise `vault_coin add` returns stale cache that excludes the new token until TTL expiry
- Single-flight is per-pod, not cluster-wide — concurrent requests across pods will both fetch; acceptable at current scale, document the limit
- Comma-list `chain` parsing: split on `,` AND whitespace; lowercase before passing to `normalizeChain` from SDK
- Auto-injected summary must read cache **non-blocking** — cold cache yields `balances: []` + `freshness.source: "cold"`; the LLM is responsible for calling the tool to populate
- `tracked_coins` is the load-bearing disambiguator — without it, the LLM can't tell "absent from `balances` because dust" from "absent because not tracked." Source it from `MessageContext.Coins` (always sent by app), not from the cache (which may be cold). Don't conflate the two
- Dust threshold ($1 USD) is configurable but should default conservatively. Lower thresholds (e.g. $0) bloat context with junk; higher thresholds ($10+) hide legitimate small holdings. Revisit after dogfooding
- Top-50 cap on `balances` is a safety bound for users with hundreds of tokens — log when it triggers; if it triggers often, the answer is pagination via the tool, not removing the cap
- Action-mutation cache invalidation needs to fire AFTER the tx succeeds, not on tool-call dispatch — invalidating on dispatch loses you the cache while the user is mid-confirmation
- Per-chain MCP tools that get unregistered from the LLM surface must stay reachable via direct `mcpProvider.CallTool(ctx, name, ...)` — `_meta.categories` controls LLM visibility, not registration
- Moving the EVM-prefetch off the send-blocking path means first-message latency now depends on `/balances` completing during app startup; if the network is slow, the agent's first call falls through to synchronous tool-side fetch — same total latency, just shifted. Worth measuring before declaring win
- `MessageContext.Coins` does not include balance amounts (only metadata); don't accidentally read from there as a fast path — always go through BalanceService

## Notes

**2026-05-01T07:38:50Z**

Split into v-kmev (LLM-contract half: drop prompt injection, as_of, force_refresh, tracked_coins, drop get_portfolio, hide per-chain MCP tools, drop extras.balances wire) + v-beyw (infrastructure-consolidation half: BalanceService + Redis cache + REST endpoint + portfolio UI migration + schema rewrite + pre-computed totals). v-kmev is the bug-fix; v-beyw is the optional extension. Closing as superseded by the split.
