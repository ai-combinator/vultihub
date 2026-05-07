---
id: v-kmev
status: closed
deps: []
links: []
created: 2026-05-01T07:37:01Z
type: bug
priority: 2
assignee: Jibles
---
# agent-backend + app: balance flow simplification — drop prompt injection, non-blocking send, tracked_coins

## Problem

Three independent issues with one tightly-scoped fix:

1. **30s gate before "Analyzing"** — `useAgentTools.buildAgentContext` `await`s `ensureQueryData(balancesQueryOptions(...))` before posting to the backend. When DOGE/ZEC blockchair hangs ~40s before returning HTTP 500, the user sees ~30s of silence before the backend even hears the message.
2. **Prompt anchoring hallucination** — the `### Balances (live)` system-prompt block writes prefetched balances as ground truth. Real bug 2026-04-30: model fabricated a confident causal story ("the swap may not have broadcasted") around a stale value the prompt asserted as live, ignoring its own tool-call results that disagreed.
3. **Tracked-token disambiguation gap** — when the user adds UNI to the vault, no path returns UNI's balance (prefetch is native-only; `evm_get_balance` is native-only). Model has no way to tell "absent because dust" from "absent because not in vault" and falls back to confused asking.

"Solved" = SendMessage lands at the backend within ~100ms regardless of fan-out state; model never anchors on a prompt-asserted balance; model can disambiguate dust vs untracked tokens; mental model collapses to "balances live behind one tool".

## Background

### Findings

Current architecture has **eight moving parts for one concept** (balance retrieval); data-flow diagram is genuinely hard to draw:

1. App `fetchAllBalances` (`src/features/agent/lib/queries.ts:100`) — EVM/SOL fast paths + SDK fan-out for non-EVM/Sol + token fan-out via `fanOutWithLimit` (BALANCE_FAN_OUT_CONCURRENCY=8).
2. App `usePrefetchBalances` hook (`src/features/agent/hooks/usePrefetchBalances.ts`) — three triggers (mount, foreground, new conversation), fingerprint-keyed throttle (5s), AppState listener.
3. App RQ cache, 60s TTL, key includes ethAddress + solAddress + sorted address-map + sorted token-set (`balancesQueryOptions` in queries.ts:364).
4. Wire field `req.Context.Balances` — input AND output (cache merge re-injects on subsequent turns).
5. Backend 24h `cachedVaultContext` (`agent-backend/internal/service/agent/context.go:14-21,35-42,149-150`) — `mergeContext` falls back to cached `Balances` when request comes in with empty/null balances.
6. Prompt `### Balances (live)` block (`agent-backend/internal/service/agent/prompt.go:771-787`) — sourced from #4 or #5, whichever is non-empty.
7. ~25 per-chain MCP balance tools (`get_evm_balance`, `get_sol_balance`, `get_utxo_balance`, etc.), gated by `_meta.categories: 'chain_balance'` keyword filter.
8. Two top-level agent tools (`get_balances` AND `get_portfolio` in `data_tools.go`) with overlapping responsibilities.

Concrete code paths the planner will touch:

- **App await site to remove:** `src/features/agent/hooks/useAgentTools.ts:50` (`await queryClient.ensureQueryData(balancesQueryOptions(...))`).
- **App fallback already exists** at `useAgentTools.ts:70` (catch block — sends without balances). Currently never triggers because `Promise.all` doesn't fail; just waits.
- **App opportunistic-read alternative:** `queryClient.getQueryData(...)` returns cache contents synchronously without triggering a fetch.
- **App wire field source:** `src/services/agentContext.ts:82` (the `...(extras?.balances?.length ? { balances: extras.balances } : {})` spread).
- **App routing instruction to drop:** `src/services/agentContext.ts:68` (`Pre-fetched balances (EVM and Solana) are in context when present...`).
- **Backend prompt block to delete:** `prompt.go:771-787`.
- **Backend balance-routing block to rewrite:** `prompt.go:411-426` (currently tells model to call per-chain MCP tools when prefetched context is empty).
- **Backend cache merge to drop:** `context.go:14-21,35-42,149-150` (remove `Balances` field from cached struct + drop the merge fallback). 24h `vaultContextTTL` constant survives — Addresses/Coins/AddressBook still cache for 24h.
- **Backend `Balance` struct:** `agent-backend/internal/service/agent/types.go:64-71` — needs `AsOf string` (`json:"as_of,omitempty"`).
- **Backend tool schema:** `data_tools.go:32-42` (`GetBalancesTool`) — add `force_refresh: bool` (default false).
- **Backend handler:** `data_tools.go:116-135` (`execGetBalances`) and `:312-345` (`gatherBalances`) — when `force_refresh: true`, skip prefetched and force fan-out. Otherwise unchanged.
- **Backend parser:** `data_tools.go:587-648` (`parseBalanceResult`) — propagate per-chain `as_of` to `Balance.AsOf`.
- **mcp-ts balance tools:** `mcp-ts/src/tools/balance/evm-balance.ts` and siblings (`get_sol_balance`, `get_utxo_balance`, `get_xrp_balance`, `get_atom_balance`, `get_cardano_balance`, `get_sui_balance`, `get_trx_balance`, `get_ton_balance`, cosmos multi-denom, etc.) — return `as_of` (ISO-8601, fetch time) in result.
- **Per-chain MCP categories to strip:** `mcp-ts/src/lib/toolCategories.ts` (drop `chain_balance` from Categories or mark internal-only); `mcp-ts/src/tools/index.ts` (strip categories from per-chain balance tools so `tool_filter.go` never surfaces them; tools stay registered for backend RPC dispatch via `mcpProvider.CallTool` and the `chainToBalanceTool` map).
- **Per-task timeout site:** `src/features/agent/lib/queries.ts:78-98` (`fanOutWithLimit`) — wrap each gated task in `Promise.race([task(), timeout()])`.

**Real-world bug evidence (2026-04-30, motivates v-zaiw):**

> User with `0x376F5E…9abC` had ~0.000122 ETH on Arbitrum after a swap. Chatbot kept reporting `0.00137943 ETH ($3.11)` across multiple turns including after the user explicitly said "refresh." The model fabricated a confident causal story ("the swap did not decrease your ETH balance, suggesting it may not have broadcasted") around the stale number — a hallucination grounded in something the system asserted as truth via the prompt block.

**Current 30s repro (2026-05-01):**

App logs: `[buildAgentContext] start` → 30.7s → `[buildAgentContext] done in 30719ms (balances=33)`. During the wait: blockchair DOGE + ZEC return 500 after ~41s; SDK fan-out for Ripple and Polkadot also fail. Backend doesn't see the request until buildAgentContext resolves. Backend then thrashes on `search_token` for another ~45s before producing the swap card. Total: ~75s.

### External Context

- Blockchair returns HTTP 500 with `code:500, error:"Something went wrong"`, `render_time:40.01s` for both DOGE and ZEC dashboards. Server-side hang ~40s before failure response.
- v-pxuw §3 schema rules: enum cardinality <10, no `oneOf` (Gemini Flash doesn't support), prefer flat fields, comma-list strings beat arrays for weak models. Already followed by current `get_balances`; preserved.

## Current Thinking

### App side (`vultiagent-app`)

**Replace the await with a non-blocking read.** In `useAgentTools.buildAgentContext`, replace `await queryClient.ensureQueryData(balancesQueryOptions(...))` with `queryClient.getQueryData(balancesQueryOptions(...).queryKey)`. Read whatever the prefetch hook has warmed; ship without balances if cold.

Why hard-stop drop instead of race-against-deadline: deadline is a magic number papering over the architectural truth that balances aren't load-bearing once `### Balances (live)` is gone and `as_of`-stamped tool results are quotable from chat history. After this ticket, the model is supposed to call `get_balances` for balance questions anyway. A race adds complexity for ~1s of "fast common path" we're already giving up on.

**Keep `usePrefetchBalances` as-is.** Still warms RQ cache for subsequent sends and for the portfolio UI screen. Cache stays useful even after we stop blocking on it.

**Drop the `extras.balances` wire field entirely.** Once the app is non-blocking and tool results carry `as_of`, sending balances via the wire is a third source of truth with marginal value — and it's the input/output overload that caused the staleness bug in the first place. Concretely: app `agentContext.ts:82` stops including `balances`; `AgentContextExtras.balances` removed; backend `gatherBalances` stops reading `req.Context.Balances`. Wire is now exclusively input for tool-call results, never balances themselves. **Removes one of the eight moving parts and resolves the input/output wire-field overload that was structurally causing the staleness bug.**

**Drop the `Pre-fetched balances (EVM and Solana)…` instruction line at `src/services/agentContext.ts:68`.** Redundant after the prompt rewrite and after the wire field is gone.

**Add per-task timeout to `fanOutWithLimit`.** Even though the agent path is non-blocking, the portfolio UI still uses `fetchAllBalances` and is exposed to the same blockchair-hang risk. Wrap each task in `Promise.race([task(), timeout(8000)])` and treat timeout as a rejection — `fetchAllBalances` collectors already log and continue on rejected results. Universal hardening; small diff. Also mitigates prefetch-thrash on cold start so warm-path RQ hit rate stays high.

### agent-backend prompt + context

**Delete `### Balances (live)` block** (`prompt.go:771-787`).

**Rewrite balance routing rules** (`prompt.go:411-426`): "Balances live behind `get_balances`. If chat history contains a recent `as_of`-stamped balance result, you may quote it; otherwise call `get_balances`. After state-changing actions (swap/send), call with `force_refresh: true`."

**Inject `### Tracked coins` block** from `MessageContext.Coins` (always populated by app every turn). Lists every vault coin's `{chain, symbol, contract_address?}` — vault metadata, ~50-200 bytes. Add three routing rules:

- User asks about a token IN `tracked_coins` but NOT in chat-history balances → call `get_balances({token})` (dust or cold cache).
- User asks about a token NOT in `tracked_coins` → suggest `vault_coin add <token>`; do NOT call `get_balances`.
- User says "refresh" / "check again" / "is this current" → call `get_balances({force_refresh: true})`.

This is the load-bearing piece for token-gap disambiguation. Free (vault metadata already on wire); only cost is one prompt block + three rules. Borrows va-bqks's smartest insight at zero infrastructure cost.

**Stop persisting `Balances` through the conversation cache** (`context.go:14-21,35-42,149-150`). Remove `Balances` field from `cachedVaultContext`; remove the merge fallback. Addresses/Coins/AddressBook caching unchanged (those are stable; balances are tool output).

### agent-backend tool + types

**`get_balances` accepts `force_refresh: bool`** (default false). Update tool description so the model knows when to set it. Pass-through to per-chain MCP for now — no backend cache exists to bypass; kept as contract-only addition so the future cache landing doesn't require a schema change.

**`Balance` struct gains `AsOf string`** (ISO-8601 / RFC 3339, `omitempty`).

**`parseBalanceResult` propagates per-chain `as_of` to `Balance.AsOf`.**

**Drop `get_portfolio` tool** (`data_tools.go`). Redundant with `get_balances`. Update prompt routing to point at `get_balances` for portfolio queries. Removes one tool from the LLM surface and one concept from the codebase.

**Strip `_meta.categories: 'chain_balance'` from per-chain MCP balance tools** so `tool_filter.go` never surfaces them. Tools stay registered (backend RPC dispatch via `mcpProvider.CallTool` continues to work). Model now sees one balance tool (`get_balances`) instead of ~25.

### mcp-ts

**Per-chain balance tools stamp `as_of`** (ISO-8601, fetch time) on results. For text-format tools (current EVM/UTXO output), add `as_of` either as an additional line or migrate to JSON. Pick whichever minimizes parser churn in `parseBalanceResult`.

### Resulting mental model

After this ticket: *"Balances live behind one tool (`get_balances`). Tool results carry `as_of`. Vault metadata is in `tracked_coins`. App prefetch warms RQ cache opportunistically for the portfolio UI."* Four nouns. Compare to current: eight moving parts, no clean precedence rules.

This **subsumes `v-zaiw` entirely** (drop prompt block, stop caching `Balances`, add `as_of`, add `force_refresh`) and borrows va-bqks's smartest insight (`tracked_coins` for dust-vs-untracked disambiguation) at zero infrastructure cost. The remaining va-bqks pieces (REST endpoint, Redis cache, schema rewrite, portfolio UI migration, pre-computed totals) become an optional consolidation ticket.

## Constraints

- **Wire-shape compat for `req.Context.Balances`:** field stays in the JSON schema (don't break older app builds in flight) but backend ignores it. App stops sending it on new builds. Defer schema-level removal to a release boundary.
- **Per-chain MCP tools must stay registered** (just hidden from LLM surface via category strip). Backend `mcpProvider.CallTool` invokes them by name from `chainToBalanceTool` map.
- **24h `vaultContextTTL` constant survives.** Addresses/Coins/AddressBook still cache for 24h — only `Balances` stops riding along.
- **No new server-side balance cache in this ticket.** `force_refresh` is contract-only; nothing to bypass yet, by design.
- Signing-adjacent only insofar as wrong balance reporting can mislead the model into building the wrong send/swap. No actual derivation/signing path changes here, so no `[sdk/signing]` PR tag and no chain-ID list required by the repo safety policy.

## Assumptions

- `MessageContext.Coins` is reliably populated by the app on every turn — this is what `tracked_coins` injection depends on. **Spot-check before implementation.** If the app sometimes ships an empty `Coins`, the disambiguation rule degrades to "model can't tell tracked from untracked" on those turns.
- `get_balances` already fans out correctly to all chains today via `gatherBalances` + `chainToBalanceTool` map — so de-registering the 25 per-chain tools from the LLM surface doesn't lose capability, just hides them. Worth confirming the per-chain dispatch covers every chain currently in `chainRegistry` before stripping categories.
- Per-chain MCP balance tools all return parseable results that can be augmented with `as_of` without breaking `parseBalanceResult`. Mixed text/JSON shapes today; assume each can carry an extra line/field.
- App prefetch warming the RQ cache during chat-screen mount + foreground gives a high-enough warm-cache hit rate that "common path" balance answers from chat-history `as_of` are plausible. If the prefetch usually times out (DOGE/ZEC dragging), we may need the per-task timeout (already in scope) just to make the prefetch itself reliable.

## Non-goals

- New `POST /agent/balances` REST endpoint.
- Redis cache + singleflight + coin-set-hash key.
- Action-mutation cache invalidation hooks (`execute_send`/`execute_swap`/`execute_lp_*`).
- Schema rewrite of `get_balances` (5-flat-field comma-list with `chain`/`token`/`fiat`/`include_defi`/`refresh`).
- Pre-computed `totals` injection.
- Portfolio UI migration off `fetchAllBalances`.
- Multi-vault aggregation.
- DeFi position fetching (`include_defi`).
- Scrubbing old tool results from chat history.

All of the above belong to the optional consolidation ticket that extends this one.

## Dead Ends

- **Race-against-deadline in `buildAgentContext`** (e.g. `Promise.race([ensureQueryData, timeout(2000)])`). Magic number; papers over the architectural truth that balances aren't load-bearing once the prompt block is gone.
- **Renaming "(live)" → "(snapshot, stale OK)" without deleting the prompt block.** Half-measure; model still anchors to system-prompt-asserted numbers over tool results.
- **Two tools (one cached, one fresh).** Confusing schema for the LLM; `force_refresh` parameter on a single tool is cleaner.
- **Scrubbing old tool results from chat history each turn.** Heavier than needed. Model can reason about `as_of` itself; simpler contract.
- **Adding a 60s server-side balance cache in this ticket.** Out of scope per the bang-for-buck framing. Easy follow-up once `force_refresh` is wired (then it has something to bypass).
- **Per-task timeout via abort-signal threading through `getCoinBalance`.** Bigger surface change; `Promise.race` wrapping at `fanOutWithLimit` is sufficient and contained.

## Open Questions

- Per-task timeout value for `fanOutWithLimit`: 5s, 8s, 10s? Blockchair currently fails at ~40s. 8s leaves headroom for slow-but-real RPCs; 5s is more aggressive. Pick during implementation after measuring p99 of healthy chains.
- Should `Balance.AsOf` be ISO-8601 string or epoch-ms int? Bias is ISO-8601 (matches v-zaiw spec, human-readable in LLM JSON output); confirm against existing struct serialization.
- For per-chain MCP tools that currently return text (EVM/UTXO), do we add `as_of` as a new line in the text result, or migrate the tool's whole output to JSON? Cheapest for `parseBalanceResult` is probably an extra line, but JSON might be cleaner long-term. Decide during implementation per-tool.
- After stripping `chain_balance` category, does `tool_filter.go` have any other side-effects to clean up? Confirm the filter is the only consumer.
- Is there anywhere else in the prompt that references `### Balances (live)` semantically (examples, anti-examples)? Scan `prompt.go` for stragglers.

## Notes

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/mcp-ts#76 (merged) + vultisig/vultiagent-app#346 + vultisig/agent-backend#236 (open)
