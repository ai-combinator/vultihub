---
id: v-jerl
status: closed
deps: []
links: []
created: 2026-05-04T05:11:53Z
type: feature
priority: 2
assignee: Jibles
---
# execute_swap: auto-derive destination for cross-family non-EVM swaps

## Problem

ETH→BTC (and other cross-family non-EVM destination) `execute_swap` calls thrash on `get_receive_info` and never reach `execute_swap`. The model needs the user's BTC destination address; with `get_address` hidden from the LLM-facing tool surface, the only visible "address-fetching" tool is `get_receive_info`, which is a UI-rendering tool that emits a ReceiveCard surface as a side-effect. The LLM loops calling it (sometimes with different `variant` values), the loop-breaker trips, the swap is never built. "Solved" means cross-family `execute_swap` calls succeed without the LLM ever needing to call `get_receive_info` for swap-destination resolution.

## Background

### Findings

- **Target site**: `mcp-ts/src/tools/execute/execute_swap.ts:839-850`. Today's destination resolution:
  ```ts
  const sameChain = fromChain === toChain
  const bothEvm = EVM_CHAIN_SET.has(fromChain) && EVM_CHAIN_SET.has(toChain)
  const destination =
    args.destination && args.destination.length > 0
      ? args.destination
      : sameChain || bothEvm
        ? args.sender
        : undefined
  if (!destination) {
    return textError(
      \`execute_swap: cross-chain swap from \${fromChain} to \${toChain} requires an explicit destination address.\`,
    )
  }
  ```
  EVM↔EVM and same-chain default destination to sender. Cross-family non-EVM (ETH→BTC, ETH→SOL, ETH→ATOM, etc.) errors out unless the LLM passes `destination`.

- **Derivation pattern to mirror**: `mcp-ts/src/tools/utility/get-receive-info.ts:130-145` — calls `deriveAddressFromKeys({chain, ecdsaPublicKey, eddsaPublicKey, hexChainCode})` from `@vultisig/sdk`. The SDK helper picks ECDSA/EdDSA internally via `getPublicKey` (`vultisig-sdk/packages/sdk/src/tools/address/deriveAddressFromKeys.ts:56-64`), so the caller passes both keys and the SDK selects per-chain. No need to mirror `get_receive_info`'s `eddsaChains` set — that drives the `key_type` label only.

- **Vault keys are injected uniformly**: `execute_swap.ts:815-821` declares `ecdsa_public_key` / `eddsa_public_key` / `chain_code` as optional inputs and sets `injectVaultArgs: true`. Backend reads `_meta.inject_vault_args` and force-overwrites those args on every opt-in MCP tool call (`agent-backend/internal/service/agent/executor.go:3409-3470`). Keys WILL be present when the LLM omits `destination`.

- **In-file precedent for vault-derivation**: the Skip branch at `execute_swap.ts:609-628` already vault-derives `toAddress` and only falls back to `llmDestination` if derivation fails. The proposed fix lifts the same pattern to the top-level destination resolution.

- **Downstream destination consumers**: THORChain/Maya native branch passes `destination` verbatim to `findSwapQuote` (`execute_swap.ts:1009`). This is where the auto-default is load-bearing. Skip branch already self-derives, so the fix is mostly redundant there (defense in depth).

- **`assertSafeDestination` (line 857) won't false-positive**: it rejects burn/null/system-program addresses by chain family (`mcp-ts/src/lib/dangerous-addresses.ts:78-88`). Vault-derived addresses are deterministic outputs of secp256k1/ed25519 over user keys — they cannot equal `0x0`, `0xdead`, the Solana System Program, or BTC null-script vanity addresses.

- **SDK chain coverage**: SDK's `Chain` enum (`vultisig-sdk/packages/core/chain/Chain.ts:83-90`) is a strict superset of `get_receive_info`'s 37-chain `supportedChains` (`get-receive-info.ts:16-22`). All swap-relevant destinations (BTC, BCH, LTC, DOGE, DASH, ATOM, RUNE, CACAO, KUJI, SOL, LUNA, LUNC, OSMO) derive cleanly. Two SDK-only Chain values: `Dydx` (in receive-info) and `QBTC` (post-quantum, no WalletCore signing — would throw).

- **Why this is "pre-existing"**: not introduced by recent v-gopv / v-lckx work. Structural since `get_address` was added to `agent-backend/internal/service/agent/tool_filter.go:191 llmFacingDropList`. The combination of "hide `get_address`" + "make `get_receive_info` the canonical address tool" + "tell the model to read swap destinations from inline Addresses context" leaves a gap any time a cross-family swap needs a non-EVM destination.

- **No existing tests assert the error path**: `grep "requires an explicit destination"` returns zero hits in tests/fixtures (only the source string at `execute_swap.ts:849`). `mcp-ts/src/tools/execute/__tests__/execute_send.test.ts` is the only test in that dir and tests `execute_send`. The QA fixture `agent-backend/scripts/qa/curl-replay/conversations/04-swap-btc-sol.yaml` is permissive (accepts ask-OR-default-OR-refuse paths). Zero blocking churn.

- **Prompt rule reference**: `agent-backend/internal/service/agent/prompt.go:255-260` documents the three destination cases (EVM↔EVM omit; in Addresses → read; not in Addresses → ASK). The fix preserves case-3 as the fallback when derivation fails.

### External Context

- `@vultisig/sdk` `deriveAddressFromKeys` is the existing helper. It throws on unsupported chains / missing WalletCore.

## Current Thinking

- **Fix**: extend the auto-default at `execute_swap.ts:841-850` to derive destination from injected vault keys when destination is omitted AND the swap is cross-family non-EVM. The LLM no longer needs `get_receive_info` for swap-destination resolution; the structural trap goes away.

- **Wrap derivation in try/catch with fallback to the existing error**. If `deriveAddressFromKeys` throws (unsupported chain, missing WalletCore for chains like QBTC), surface the existing "requires an explicit destination address" `textError` so the LLM gets a recoverable error and the prompt's case-3 (ask user) path remains meaningful. Without the try/catch, the handler crashes opaquely.

- **Mirror `get_receive_info`'s call shape verbatim** — pass both `ecdsaPublicKey` and `eddsaPublicKey` plus `hexChainCode`; the SDK selects key type per-chain. Do not introduce a separate ECDSA/EdDSA mapping in `execute_swap`.

- **Don't touch the Skip branch's existing self-derivation at `execute_swap.ts:609-628`**. It runs after the top-level destination resolution; leaving it as-is gives defense-in-depth for the Skip path.

- **Bonus closed**: the same fix lets EVM→Terra swaps reach the Skip branch, which already vault-derives. Today they're blocked by the destination-required guard at line 847 before the Skip branch sees the args.

- **Don't change the prompt or `tool_filter.go`** in this ticket. The structural fix in `execute_swap` makes the prompt's "read from Addresses" rule less load-bearing without invalidating it. Keep `get_address` in `llmFacingDropList` — the receive-flow ReceiveCard UX is the reason it's hidden, and that reason still holds.

## Constraints

- Preserve EVM↔EVM and same-chain shortcut at `execute_swap.ts:841-846` — destination defaults to sender for those, no derivation call needed.
- The `assertSafeDestination(toChain, destination)` guard at line 857 must still run on the auto-defaulted address.
- Preserve a recoverable error path for chains the SDK can't derive (i.e. preserve the prompt's case-3 ask-user fallback).
- `injectVaultArgs: true` and the existing optional-key declarations must stay — the runtime relies on them to populate keys server-side.

## Assumptions

- Vault keys injected via `injectVaultArgs: true` are present on every `execute_swap` invocation (validated by reading `executor.go:3409-3470`, but worth re-checking if the test fails for an unexpected reason).
- `deriveAddressFromKeys` produces the same address that `get_receive_info` would for the same vault — same SDK call shape with same inputs.

## Non-goals

- Don't touch `get_receive_info` or `tool_filter.go` `llmFacingDropList`. Receive-flow architecture stays as-is; the fix is contained to `execute_swap`.
- Don't rewrite the prompt's destination rule at `prompt.go:255-260`. It still serves as the case-3 fallback narrative.
- Don't re-introduce `get_address` to the LLM-facing surface.
- Don't refactor or remove the Skip branch's self-derivation at `execute_swap.ts:609-628` (defense in depth, leave it).
- Don't change `execute_send` — different tool, different surface area.

## Dead Ends

- **Re-add `get_address` to the LLM-facing surface**. Reason hidden was the ReceiveCard UX (QR + chain network warning) for user-facing address asks; bringing it back regresses fund-safety.
- **Prompt-engineering only**: add "DO NOT call `get_receive_info` for swap destinations" to `execute_swap`'s tool description and the prompt's cross-chain destination rule. Discussed as a smaller alternative but rejected — the structural server-side fix survives prompt churn and works regardless of model.
- **Interceptor in `turn_state.go`**: when intent is `IntentSwap` and model calls `get_receive_info`, return a behavioral error pointing to the Addresses block. Discussed as a fallback but rejected in favor of the cleaner server-side fix that the LLM doesn't need to learn around.

## Open Questions

- Should we add a regression fixture to `agent-backend/scripts/qa/curl-replay/conversations/` that asserts no `get_receive_info` call appears in the tool trace for a cross-family swap (ETH→BTC, ETH→SOL)? `04-swap-btc-sol.yaml` exists but is permissive; tightening it or adding a new fixture would catch a future regression that breaks the auto-default.
- Error-message wording on the fallback path: keep verbatim ("execute_swap: cross-chain swap from X to Y requires an explicit destination address.") or upgrade to mention "or an Addresses entry for to_chain"? Verbatim is safer for prompt-cache stability.

## Notes

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/mcp-ts#79 (DRAFT)
