---
id: v-beyw
status: open
deps: [v-kmev]
links: []
created: 2026-05-01T07:38:38Z
type: task
priority: 2
assignee: Jibles
---
# agent-backend: BalanceService + Redis cache + REST endpoint + portfolio UI consolidation (extends v-kmev)

## Problem

After `v-kmev` (bang-for-buck balance flow simplification) lands, the LLM contract is clean ("balances live behind one tool, results carry `as_of`, vault metadata in `tracked_coins`") but two parallel fetch paths still exist (app `fetchAllBalances` for portfolio UI; backend per-chain MCP fan-out for the agent tool). Each balance question pays a fan-out round-trip (no backend cache). `get_balances` schema works but isn't designed for weak-model reliability. `totals` aren't pre-computed (Flash/Haiku miscount when summing arrays).

"Solved" = one `BalanceService` in agent-backend; one `POST /agent/balances` endpoint feeding both portfolio UI and the agent tool; Redis cache (60s TTL, singleflight, coin-set-hash key); schema redesigned for weak-model reliability; portfolio UI migrated off `fetchAllBalances`.

This ticket is the **infrastructure-consolidation half of the original `va-bqks`** — the LLM-contract half (drop prompt injection, `as_of`, `force_refresh`, `tracked_coins`, drop `get_portfolio`, hide per-chain MCP tools from LLM, drop `extras.balances` wire) shipped via `v-kmev`. What's left here is purely infrastructure.

## Background

### Findings

State-of-the-world after `v-kmev` (assume it has shipped):

- App still uses `fetchAllBalances` (`src/features/agent/lib/queries.ts`) for the portfolio UI screen. Same per-task-timeout vulnerability mitigated but not eliminated by `v-kmev`'s `fanOutWithLimit` timeout.
- Agent `get_balances` tool fan-out has no Redis cache. Repeat balance questions in the same conversation pay full fan-out cost each time.
- `get_balances` current schema works but Flash routinely emits `chain: "eth"` against array schemas (the existing schema may use arrays — verify and replace with comma-list strings during implementation).
- `MessageContext.Coins` already feeds `tracked_coins` injection (from `v-kmev`) — reuse, don't re-implement.
- `extras.balances` wire field has been dropped (from `v-kmev`) — backend has no fast-path to populate `balances` injection except by reading cache.
- Per-chain MCP balance tools are already invisible to the LLM (categories stripped in `v-kmev`); reachable only via `mcpProvider.CallTool` from the chain-dispatch map.

Reference (canonical pattern): vultisig-windows `core/ui/agent/orchestrator/AgentContextService.ts#fetchBalances` — iterates vault coins (native + tokens) and dispatches via `getCoinBalance` from the SDK's core-chain resolver tree. Same dispatch shape we'd use server-side.

### External Context

- Redis is already provisioned in agent-backend per CLAUDE.md.
- `singleflight.Group` (`golang.org/x/sync/singleflight`) is the canonical Go library for in-process request coalescing.
- v-pxuw §3 schema rules: <10 enum cardinality, no `oneOf` (Gemini Flash doesn't support), flat fields, comma-list strings beat arrays.

## Current Thinking

### Endpoint + service

`POST /agent/balances` with `MessageContext` body (same shape SendMessage already uses — stateless, no per-user vault registration). Returns the result shape below. Used by:

- App startup (after vault loaded, parallel with `/conversations/list`) — pre-warm cache.
- Portfolio UI screen — primary data source (replaces app-side `fetchAllBalances`).
- Internal call from agent's `get_balances` tool — same Go service layer, not an HTTP self-call.

**`BalanceService`** (`internal/service/agent/balance_service.go`, new) — `Fetch(ctx, MessageContext, params) (BalanceResult, error)`. Owns cache lookup, singleflight, fan-out, normalization. Shared between REST endpoint and tool handler.

**`internal/api/balances.go`** (new) — `POST /agent/balances` handler; thin wrapper that calls BalanceService. Register under existing JWT auth middleware in `internal/api/server.go`.

### Cache

Redis key shape: `balances:{public_key}:{hash(sorted_coin_ids + addresses)}`. Including the coin-set hash in the key means `vault_coin add` automatically misses cache (free correctness — no explicit invalidation cascade for `vault_coin`/`vault_chain` add).

TTL 60s. Explicit invalidation on action-mutation success (`execute_send` / `execute_swap` / `execute_lp_*`) — drop source-chain entry. TTL alone leaves an action-completion stale window.

`singleflight.Group` on cache miss to dedupe concurrent app-startup + agent-tool + UI-mount requests for the same key. **Per-pod only** — accept some cross-pod redundancy; cluster-wide locking is overkill at current scale.

`internal/cache/redis/balances.go` (new) — typed cache layer over the existing Redis client; key composition + TTL.

### Tool schema

```
get_balances({
  chain?:        string   // "eth" | "Ethereum" | "eth,sol,btc" — single string, comma-list parsed server-side
  token?:        string   // "USDC" | "0xA0b8…48" — symbol or contract; regex disambiguates
  fiat?:         string   // ISO code, default "USD"
  include_defi?: boolean  // default false; flips on for "show positions" / "net worth" prompts
  refresh?:      boolean  // default false (renamed from `force_refresh` for schema consistency)
})
```

Five flat optional fields. Zero-arg call is the most common path. Comma-list `chain` (not `chains: string[]`) chosen because Flash routinely emits `chain: "eth"` against array schemas (forgetting the wrapper) and the call rejects; single string + server split mirrors the existing `amountString` natural-language pattern. `token` accepts symbol OR contract address with server-side regex disambiguation rather than `oneOf` (Gemini doesn't support oneOf). `chain` + `token` AND together. No enums (chain set is >25, violates v-pxuw's <10 enum cardinality rule).

### Result shape

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

### Auto-inject (extends v-kmev's `tracked_coins`-only inject)

Read from cache, injected into `MessageContext` non-blocking:

1. **`balances`** — every vault row with non-zero balance and `usd_value > $1` (or unknown USD value, in case price lookup failed). Each row: `{chain, symbol, amount, usd_value, source}`. Capped at top 50 by `usd_value` as a safety bound.
2. **`tracked_coins`** — every coin in the vault as `{chain, symbol, contract_address?}` regardless of balance. Already in `MessageContext` from `v-kmev` — keep as-is, source unchanged (`MessageContext.Coins`).
3. **`totals`** — `{total_fiat, by_chain[].fiat}`. Pre-computed.

Plus a single `freshness` marker: `{source: "cache"|"cold"|"stale", as_of?, age_seconds?}` so the LLM knows whether to trust the snapshot or call the tool.

**Cold-cache handling:** when the auto-inject runs and the cache is empty/stale beyond TTL, inject `tracked_coins` from `MessageContext.Coins` (already there) but mark `balances: []` and `freshness.source: "cold"`. The LLM sees the universe of tracked tokens but no values; the prompt instructs it to call `get_balances` to populate. **Never block the turn on a fresh fetch.** Pre-warm on app startup keeps cold-cache the edge case (long idle, scheduler-only flow).

LLM routing rules to add to `prompt.go` (extending the rules added in `v-kmev`):

- "User asks about a balance and the value is in `balances`" → answer from context (no tool call).
- "freshness.source is `cold`" → call `get_balances` before answering balance/portfolio questions.

(The other three rules — token-in-`tracked_coins`-but-not-in-`balances` → tool, token-not-in-`tracked_coins` → suggest `vault_coin add`, refresh keywords → `refresh: true` — already shipped in `v-kmev`.)

### App-side migration

- Delete `fetchAllBalances`, `balancesQueryOptions` (`src/features/agent/lib/queries.ts`).
- Delete `usePrefetchBalances` (replaced by app-startup `POST /agent/balances` ping).
- Delete `fetchEthBalance` / `fetchSolBalance` / `fetchEthPrice` / `fetchSolPrice` from `src/services/agentContext.ts` if no remaining callers.
- App startup site (find via `useConversations` / `listConversationsApi` mount) — fire `POST /agent/balances` in parallel with the conversation list call once vault is loaded.
- Portfolio UI screen — migrate from local `fetchAllBalances` to `POST /agent/balances`; delete obsolete app-side balance/price fetchers.

### Endpoint shape decision: stateless, not registered

`POST /agent/balances` accepts `MessageContext` body (matches existing SendMessage pattern — stateless, one less concept to maintain). Rejected: server-side vault registration. Adds a new lifecycle (register/deregister) for marginal benefit.

### Resulting mental model

*"One BalanceService in agent-backend. One Redis cache, keyed on vault coin-set so vault changes auto-invalidate. One REST endpoint feeds both portfolio UI and agent tool. Auto-inject is non-blocking snapshot. Tool gets schema designed for weak models. Done."* — Four nouns, implementation matches the contract.

## Constraints

- **Cache key MUST include the vault coin-set hash**, not just `public_key` — otherwise `vault_coin add` returns stale cache that excludes the new token until TTL expiry.
- **Action-mutation cache invalidation must fire AFTER tx success**, not on tool-call dispatch — invalidating on dispatch loses the cache while the user is mid-confirmation.
- **Auto-injected summary must read cache non-blocking** — cold cache yields `balances: []` + `freshness.source: "cold"`; LLM is responsible for calling the tool to populate.
- **Per-chain MCP tools stay reachable via `mcpProvider.CallTool`.** `_meta.categories` already controls LLM visibility (handled by `v-kmev`); registration is unchanged.
- **Singleflight is per-pod, not cluster-wide** — concurrent requests across pods will both fetch; document the limit.
- **Migrate portfolio UI in same PR as the endpoint.** Shipping the endpoint and tool while the app still uses `fetchAllBalances` leaves three balance code paths during the transition. Better to delete the old path immediately.

## Assumptions

- `v-kmev` has shipped before this ticket starts. This ticket builds on its primitives (`as_of`, `force_refresh` (renamed `refresh`), `tracked_coins` inject, dropped `extras.balances` wire field, dropped `_meta.categories: 'chain_balance'`, dropped `get_portfolio`).
- Redis is already provisioned and there's a typed Redis client layer to extend.
- `MessageContext.Coins` already feeds `tracked_coins` (from `v-kmev`) — this ticket reuses that source, doesn't re-implement.
- The `chainToBalanceTool` map in `data_tools.go` covers every chain in `chainRegistry`. (Spot-checked in `v-kmev`'s implementation.)

## Non-goals

- Multi-vault aggregation (current `MessageContext` carries one active vault; cross-vault portfolio is a separate ticket).
- DeFi position fetching beyond the `include_defi` schema bit. The bit lands; the actual DeFi-source fetchers are their own ticket.
- Cluster-wide singleflight.
- Arbitrary-token lookup outside vault. Product story: "if you care about a token, add it to your vault." (Already enforced by `v-kmev`'s `tracked_coins` rule.)
- Re-litigating `v-kmev`'s LLM-contract decisions (drop prompt block, `as_of`, etc.).

## Dead Ends

- **Two tools (one cached, one fresh).** Confusing schema for LLM; single tool + `refresh` parameter is cleaner.
- **`exclude_chain` param.** Second list parameter that weak models will fill incorrectly; workaround is to list the chains you want.
- **Server-side vault registration.** Stateless endpoint matches existing SendMessage pattern; one less concept to maintain.
- **Stage rollout (ship endpoint while app still uses `fetchAllBalances`).** Three balance code paths during transition. Delete old path in same PR.
- **Pre-fetched-balance fast path on tool-handler entry.** `extras.balances` wire field is gone (from `v-kmev`); cache is the sole fast path now.

## Open Questions

- Dust threshold ($1 USD) — configurable but should default conservatively. Lower thresholds (e.g. $0) bloat context with junk; higher thresholds ($10+) hide legitimate small holdings. Revisit after dogfooding.
- Top-50 cap on `balances` — log when triggered; if it triggers often, the answer is pagination via the tool, not removing the cap.
- Cache invalidation timing precision: source-chain only, or all-vault? Source-chain is tighter but requires per-chain key shape; all-vault is simpler but more aggressive. Probably source-chain for sends (single chain affected) and all-vault for swaps (two chains).
- Schema parameter rename: `force_refresh` (from `v-kmev`) → `refresh` (in this ticket's redesign). Migration approach: add `refresh` alongside `force_refresh` initially, then deprecate `force_refresh` after a release. Or rename atomically with the schema redesign PR. Decide during implementation.
- Does the existing `get_balances` schema actually use arrays for `chains`, or did `v-pxuw` already migrate to comma-list? Verify before claiming the schema-rewrite is improving on it.
