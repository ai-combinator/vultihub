---
id: v-ucht
status: open
deps: []
links: []
created: 2026-04-29T04:11:05Z
type: feature
priority: 2
assignee: Jibles
---
# mcp-ts + agent-backend: optimistic token resolution with structured ambiguity error

## Objective

`execute_swap` and `execute_send` should resolve tokens optimistically from `(chain, symbol)` and surface a typed `AmbiguousTokenError` only when the symbol is genuinely ambiguous on the target chain. The agent prompt should be rewritten to call `execute_*` first and fall back to `search_token` only on ambiguity — replacing today's pre-emptive "always search before swapping non-vault tokens" rule that adds unnecessary round-trips on common tokens like USDC.

## Context & Findings

**Half the machinery already exists.** `mcp-ts/src/lib/resolveToken.ts:228` runs an optimistic chain: native → known list → on-chain RPC → CoinGecko. `resolveKnownBySymbol` (line 145-152) already throws when multiple known-list entries match for `(chain, symbol)`:

```
throw new TokenResolutionError(
  `Ambiguous symbol '${symbol}' on ${chain} — multiple contracts found: [${addrs}]. Please specify \`address\`.`
)
```

**What's missing — three gaps:**

1. The error is a flat string with addresses joined; no structured candidate list, no machine-readable tag. `execute_swap.ts:506-526` catches resolution failures generically and surfaces them as opaque `textError`s. The agent has no signal to call `search_token` as a fallback vs. abandoning the request.

2. `resolveViaCoingecko` (`resolveToken.ts:190-218`) silently picks the FIRST match on multi-deployment results — a latent correctness bug independent of the agent UX (silent wrong-asset swap possible). PR #49 had a CR Critical comment on related silent reassignment; the multi-match loop case slipped through. Same `matches.length > 1` ambiguity guard belongs here too.

3. `agent-backend/internal/service/agent/prompt.go` has THREE sites pushing pre-emptive `search_token` (lines ~110-112 "Token not in vault: search_token → user picks → vault_coin → proceed", ~200-208 "When the user mentions a token NOT present in their vault use search_token", ~290 "ALWAYS use search_token before saying X doesn't exist"). For tokens already in the user's "Coins in Vault" context the canonical contract address is right there — the model should just pass it as `from_address` / `to_address` and skip resolution entirely.

**`mismatchCheck` (resolveToken.ts:92-103) is the existing template** for guidance-bearing structured errors ("Re-fetch token info via search_token or omit `decimals` to use the canonical value") — the new ambiguity path should follow the same shape.

**`execute_swap` schema today (`mcp-ts/src/tools/execute/execute_swap.ts:441-459`)** already has `from_address` / `to_address` / `from_decimals` / `to_decimals` as optional overrides — the explicit-address re-call path is already supported by the schema; only the agent's policy needs to change.

## Files

- `mcp-ts/src/lib/resolveToken.ts` — add `AmbiguousTokenError`, mirror multi-match detection in `resolveViaCoingecko`, upgrade `resolveKnownBySymbol` to throw the typed variant with full candidate metadata
- `mcp-ts/src/tools/execute/execute_swap.ts` (~lines 506-526) — specialize the `Promise.all([resolveToken, resolveToken])` catch to emit a JSON-tagged structured error on `AmbiguousTokenError`
- `mcp-ts/src/tools/execute/execute_send.ts` — same change, mirroring execute_swap
- `agent-backend/internal/service/agent/prompt.go` — consolidate the three "search-first" sites (~110, ~200, ~290) into one "execute-first, search-on-error" rule; add a "if the token IS in `Coins in Vault`, copy its contract address into `from_address` / `to_address`" nudge
- Reference: `mcp-ts/src/lib/resolveToken.ts:92-103` (`mismatchCheck`) for the structured-guidance error template

## Acceptance Criteria

- [ ] New `AmbiguousTokenError extends TokenResolutionError` carrying `{ side: 'from' | 'to', symbol, chain, candidates: [{ address, decimals, symbol, source }] }`
- [ ] `resolveViaCoingecko` no longer returns the first deployment silently — throws `AmbiguousTokenError` when more than one deployment on the target chain matches the symbol (this is a latent correctness fix and stands on its own, even if the agent UX change is deferred)
- [ ] `resolveKnownBySymbol` throws the typed error with the full candidate list (not the joined-addresses string)
- [ ] `execute_swap` and `execute_send` catch `AmbiguousTokenError` and return a `textError` containing both human-readable text and a JSON block the agent prompt teaches the model to recognize (e.g. `error: "ambiguous_token"`)
- [ ] Three "search-first" prompt sites collapsed into one "call `execute_*` optimistically; on `ambiguous_token` error, call `search_token` to disambiguate, then re-call with explicit `from_address` / `to_address`" rule
- [ ] Prompt nudge added: "if the token IS in the `Coins in Vault` context, pass its contract address as `from_address` / `to_address` to skip resolution"
- [ ] Unit tests cover: clean single-match resolve, multi-match in known list (typed error), multi-match in CoinGecko (typed error, no silent first-pick), explicit `from_address` override path still works, `mismatchCheck` decimals path unaffected
- [ ] Reproduction passes: vanilla "swap 0.0001 USDC on arb to ETH on arb" reaches `execute_swap` with zero `search_token` calls
- [ ] `pnpm typecheck`, `pnpm lint`, `pnpm test` clean in mcp-ts; `go test ./...`, `golangci-lint run` clean in agent-backend

## Gotchas

- `mismatchCheck` (caller-supplied vs resolved decimals) is a separate path — don't conflate with new ambiguity logic
- Some chains have legitimate same-symbol ambiguity (e.g. Avalanche USDC + USDC.e); error message should name each candidate's `source` (`known` / `coingecko` / `rpc`) so the model and user can tell the canonical from the bridged variant
- Keep human-readable text alongside the JSON in the textError body — models sometimes drop structured payloads; the prose fallback keeps the failure intelligible
- `searchToken()` in the resolver is the underlying CoinGecko helper; do not confuse with the `search_token` MCP tool the agent calls
- The prompt rewrite touches three sites — review the QA fixture suite at `scripts/qa/curl-replay/` for token-resolution flows; add or rebaseline fixtures so the new "execute-first, search-on-ambiguity" path has regression coverage
- This ticket pairs with v-vvep (per-chain balance fan-out cleanup) — both surfaced from the same reproduction; landing them together gives the cleanest swap UX, but they are independent and either can ship first
