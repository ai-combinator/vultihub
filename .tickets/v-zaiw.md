---
id: v-zaiw
status: closed
deps: []
links: []
created: 2026-04-29T23:23:50Z
type: bug
priority: 2
assignee: Jibles
---
# agent-backend: drop balance context injection; agent fetches via tool with as_of + force_refresh

## Objective

Stop injecting a "Balances (live)" block into the system prompt and stop merging balance data into the request context from the conversation cache. The agent already has a `get_balances` tool — it should use it. Balance results carry an `as_of` timestamp so the model can decide for itself whether prior numbers in chat history are fresh enough to quote, and a `force_refresh` flag lets it bypass any optimization the backend introduces later. Caching itself is out of scope for this ticket.

## Why

Real-world bug (2026-04-30): user with `0x376F5E…9abC` had ~0.000122 ETH on Arbitrum after a swap. Chatbot kept reporting `0.00137943 ETH ($3.11)` across multiple turns including after the user explicitly said "refresh." The model even fabricated a confident causal story ("the swap did not decrease your ETH balance, suggesting it may not have broadcasted") around the stale number — a hallucination grounded in something the system told it was true.

Two compounding sources of staleness:

1. **System prompt asserts the wrong number every turn.** `prompt.go:771-787` writes `### Balances (live)` directly into the system prompt from `MessageContext.Balances`. The "(live)" label is actively misleading — the values come from whatever the mobile app prefetched, possibly minutes old, with no `as_of`. The model treats this as ground truth and ignores its own tool-call results that disagree.

2. **24h Redis MessageContext cache outlives reality.** `context.go:12` caches `MessageContext` (incl. Balances) for 24h keyed per-conversation. `mergeContext` (`context.go:149-150`) falls back to cached Balances when the request comes in with empty/null balances. Even when the model dutifully calls `get_balances`, `gatherBalances` reads from `req.Context.Balances` first — which was just filled from the 24h cache.

Decision: simplify rather than add freshness machinery. Trust the model — give it `as_of` on tool results and a `force_refresh` it can call when judgment says so. Don't try to scrub old tool-call results from chat history; the agent is fine remembering and re-using them when `as_of` is acceptable.

## Context & Findings

### What changes
- The system prompt no longer asserts balances. Removed entirely; the model fetches via `get_balances` when it needs the number.
- `MessageContext.Balances` is no longer populated through the conversation cache merge. Balances are not durable context — they are tool output. (Addresses, Coins, AddressBook stay cached; they're stable.)
- `get_balances` (and the per-chain MCP balance tools that back it) stamp every result with an `as_of` ISO-8601 timestamp at fetch time. The model sees this and applies its own freshness judgment.
- `get_balances` accepts `force_refresh: bool` (default false). For now this is a contract-only addition — pass-through to the per-chain MCP tools. No new backend cache exists today, so there's nothing to bypass yet; the param is here so the cache that lands later (separate ticket) doesn't require a tool-schema change.
- `prompt.go` is updated to tell the model: balances are not in the prompt, call `get_balances` for them, and when chat history already contains a recent balance result the model may quote it if `as_of` is within tolerance — otherwise call again with `force_refresh: true`.

### What does NOT change here (deferred)
- No new per-(address, chain) backend cache. User explicitly scoped this out.
- No swap-completion cache invalidation.
- No `as_of`-based scrubbing of older tool results from the conversation window. Model decides.
- The mobile app may still send `Balances` in `req.Context` as a perf hint. We accept it but stamp `as_of` from the request receive time, and we no longer persist it through the cache merge. Whether the app should stop sending it entirely is a follow-up for the app side, not this ticket.

### Considered and rejected
- **Adding a 60s server-side balance cache.** Out of scope per user direction. Easy follow-up once `force_refresh` is wired.
- **Scrubbing old tool results from chat history each turn.** Heavier-handed than needed. The model can reason about `as_of` itself; that's the simpler contract.
- **Renaming "(live)" → "(snapshot, stale OK)" without deleting the block.** Half-measure — the model still anchors to system-prompt-asserted numbers over tool results. Cut it.
- **Two tools (one cached, one fresh).** Confusing schema for the LLM. Single tool + parameter is cleaner.

### Code map (Go side)
- `agent-backend/internal/service/agent/prompt.go:771-787` — the `### Balances (live)` block that writes into the system prompt. Delete.
- `agent-backend/internal/service/agent/prompt.go:411-426` — the existing `Balance / portfolio queries — fetch immediately` section. Update: it currently tells the model to call per-chain MCP tools when prefetched context is empty. After this ticket, prefetched context is never trustworthy for balances; the rule simplifies to "always call `get_balances` for balance questions; pass `force_refresh: true` when chat-history balances look stale or after a state-changing action like a swap."
- `agent-backend/internal/service/agent/context.go:14-21,35-42,149-150` — the `cachedVaultContext` struct caches `Balances`, and `mergeContext` falls back to cached balances when the incoming request has none. Remove `Balances` from the cached struct and the merge.
- `agent-backend/internal/service/agent/data_tools.go:32-42` — `GetBalancesTool` schema. Add `force_refresh` (bool, default false). Update description so the model knows when to set it.
- `agent-backend/internal/service/agent/data_tools.go:116-135` (`execGetBalances`) and `:312-345` (`gatherBalances`) — when `force_refresh: true`, skip the `prefetched` map entirely and force every requested chain through `fanOutBalances`. Otherwise behavior unchanged for now.
- `agent-backend/internal/service/agent/data_tools.go:587-648` (`parseBalanceResult`) — propagate the per-chain `as_of` into the returned `Balance` value.
- `agent-backend/internal/service/agent/types.go:64-71` — add `AsOf string \`json:"as_of,omitempty"\`` to `Balance`. ISO-8601 / RFC 3339.

### Code map (mcp-ts side)
- `mcp-ts/src/tools/balance/evm-balance.ts` and siblings (`get_sol_balance`, `get_utxo_balance`, `get_xrp_balance`, `get_atom_balance`, `get_cardano_balance`, `get_sui_balance`, `get_trx_balance`, `get_ton_balance`, cosmos multi-denom, etc.) — return `as_of` (ISO-8601, fetch time) in the JSON result. For text-formatted results (current EVM/UTXO output), include `as_of` either as an additional line or migrate the result to JSON. Pick whichever minimizes parser churn in `parseBalanceResult`.

## Files

- `agent-backend/internal/service/agent/prompt.go`
- `agent-backend/internal/service/agent/context.go`
- `agent-backend/internal/service/agent/data_tools.go`
- `agent-backend/internal/service/agent/types.go`
- `mcp-ts/src/tools/balance/evm-balance.ts` (+ all sibling balance tools)
- Existing tests in `agent-backend/internal/service/agent/` that build a `MessageContext` with `Balances` and assert prompt output — update to assert the block is absent.
- QA fixtures under `agent-backend/scripts/qa/curl-replay/` that exercise balance queries — update or add one that confirms the model issues a `get_balances` call rather than answering from prompt.

## Acceptance Criteria

- [ ] `### Balances (live)` block is removed from the system prompt builder; no test in the repo expects it.
- [ ] `MessageContext.Balances` is no longer written into or read from the conversation cache (`cachedVaultContext` and `mergeContext`). Addresses / Coins / AddressBook caching is unchanged.
- [ ] `Balance` struct has an `AsOf` field; per-chain MCP balance tools populate it; `parseBalanceResult` propagates it; the `get_balances` JSON output includes it on every entry.
- [ ] `get_balances` schema accepts `force_refresh` (bool, optional, default false); description tells the model when to set it (refresh keywords, after state-changing actions like swap/send, when chat-history `as_of` is stale).
- [ ] When `force_refresh: true`, `gatherBalances` ignores `req.Context.Balances` for the requested chains and fans out to MCP for every one of them.
- [ ] Prompt guidance updated so the model knows: balances live behind `get_balances` only; chat-history balance results are quotable if `as_of` is recent; otherwise re-call with `force_refresh: true`.
- [ ] A QA fixture (Tier 1) that asks "what's my ETH balance on Arbitrum?" expects a `get_balances` tool call (not a prompt-answered response). A second fixture that asks the same question after submitting a swap expects `force_refresh: true` on the second call.
- [ ] `make test` and the existing prompt golden tests pass with updates; lint clean on both sides.

## Gotchas

- The mobile app currently sends `req.Context.Balances` as an optimization. We are intentionally NOT removing the field or breaking the wire shape; we just stop persisting it through the cache and stop letting it dictate `get_balances` output when `force_refresh` is set. App-side cleanup is a separate ticket.
- Some QA fixtures and prompt golden tests likely encode the "Balances (live)" block. Don't suppress those tests — update them to assert the block's absence and that the model calls `get_balances`.
- Per-chain MCP balance tools currently return mixed shapes (text for EVM/UTXO, JSON for most others). Adding `as_of` is cheapest if EVM/UTXO move to JSON, but verify `parseBalanceResult` keeps backward compatibility with any tool that lags the JSON migration.
- The 24h `vaultContextTTL` constant survives — Addresses/Coins/AddressBook still cache for 24h. Don't lower it; it's not the source of the bug once Balances stop riding along.
- Don't add a new server-side balance cache in this ticket. The `force_refresh` parameter is a contract-only addition for now — it has nothing to bypass yet, by design.
- This is signing-adjacent only insofar as wrong balance reporting can mislead the model into building the wrong send/swap. No actual derivation/signing path changes here, so no `[sdk/signing]` PR tag and no chain-ID list required by the repo safety policy.

## Notes

**2026-05-01T07:38:47Z**

Subsumed by v-kmev. v-kmev rolls in everything from this ticket (drop ### Balances (live) block, stop persisting MessageContext.Balances through cache, add Balance.AsOf / per-chain as_of stamping, add force_refresh to get_balances) plus the app-side stop-blocking fix and tracked_coins disambiguation. Closing as superseded — implementation should reference v-kmev.
