---
id: v-wldr
status: closed
deps: []
links: []
created: 2026-04-19T21:47:43Z
type: task
priority: 1
assignee: Jibles
---
# Decimals/amounts robustness: typed label/raw split + centralized token resolution

## Objective

The `execute_*` MCP tool family leaks raw base-unit integer strings into UI-facing fields (shipped symptom: `Min. received: 77574000` on a \$1 ETH→USDC swap via MayaChain, where the value should read `0.77574 USDC`). The defect class is structural, not a one-field miss: `ExecutePrepResult.resolved: Record<string, string>` erases the distinction between pre-formatted human labels and raw base units, and the LLM is load-bearing for ferrying per-token `decimals` between `search_token` and `execute_*` inputs — a path with three separate failure modes (CoinGecko staleness, LLM hallucination, wrong deployment picked). Close the regression + remove the LLM-as-decimal-courier role in one pass, borrowing shapeshift-agentic's "asset is the unit of coherence + formatted summary is the output shape" patterns without adopting their hardcoded-only asset registry.

## Context & Findings

**Observed bug (v-pxuw escape).** `mcp-ts/src/tools/execute/execute_swap.ts` assigns raw base-unit strings directly into `resolved.minimum_output` and `resolved.quote_summary` in both branches (native: lines 405–410; general: lines 443–448). The UI (`SwapFlowCard.tsx:64`) renders them verbatim. THOR/Maya native quotes use a fixed 8-decimal convention (SDK's `getNativeSwapDecimals` returns THORChain's 8 for any destination not on MayaChain itself); 1inch/LiFi "general" quotes are in destination-token decimals. The two conventions look identical to TypeScript — only author-memory distinguishes them.

**Same bug class on `main`.** `mcp-ts/src/tools/swap/build-swap-tx.ts` emits the same raw `expected_output`/`minimum_output`; it is latent on main only because the legacy `TransactionPreview.tsx` doesn't display those fields. VA-185 (`humanizeNativeSwapFee`) already patched the fee leg of this bug after a `0.00000000000000016 ETH` dogfood sighting — local patch that didn't generalize, so the new swap card re-hit it on a sibling field. That confirms the structural fix is the right altitude.

**Decimals today come from four sources, layered implicitly.** `vultisig-sdk/packages/core/chain/coin/chainFeeCoin.ts` (static native), `packages/core/chain/coin/knownTokens/index.ts` (curated ~750-line list), `packages/core/chain/coin/token/metadata/resolvers/{evm,solana,cosmos,tron,cardano}.ts` (live on-chain reads — viem `decimals()` for EVM etc.), and CoinGecko `detail_platforms[*].decimal_place` via `packages/sdk/src/tools/token/searchToken.ts`. There is no single entry point that layers them in priority order; instead the LLM calls `search_token` → picks a deployment → ferries `to_decimals`/`to_address` into `execute_swap` as flat args. No cross-check, no authoritative resolution.

**Shapeshift-agentic pattern (borrowed from, not copied wholesale).** `apps/agentic-server/src/tools/initiateSwap.ts` calls `resolveAsset(input, walletContext)` internally — the agent passes intent (symbol + network), the tool returns a canonical Asset with `precision` baked in. The tool then emits `{summary, swapTx, approvalTx, swapData}` where `summary` (via `createSwapSummary`) is pre-formatted human (`sellAsset.amount`, `exchange.rate: "1 X = Y Z"`, `networkFeeCrypto`, `priceImpact`) and `swapTx` is transport-only. Raw base units can't reach the UI by construction. Their asset registry is hardcoded (bundled with the app); we will **not** adopt that — our live on-chain + CoinGecko discovery is a real coverage advantage for an agent that handles long-tail tokens, so keep the sources but fold them behind one resolver.

**Why a "patch execute_swap" local fix is wrong altitude.** The v-pxuw Tier D test harness marked D.4 swap ✅ by asserting `tool_calls` fired — it never asserted rendered values. The bug shape survives as long as `resolved: Record<string,string>` permits raw integers and the flat-arg `to_decimals` is agent-supplied. Every new `execute_*` tool (estimated-LP-units, contract-call return values) is a fresh site. Fixing now is cheap: 5 `execute_*` tools + 4 flow cards exist; in two months this touches a dozen more.

**Three moves, one ticket, one PR-per-repo.** (1) Type-enforced split of labels vs raw on the tool output; (2) centralized `resolveToken` inside each `execute_*` with the four-source priority layered as fallbacks; (3) echo the resolved token back as a label so the LLM sees what actually got used and can self-correct next turn.

**Rejected approaches.**
- *Bundle a hardcoded asset DB shapeshift-style.* Loses long-tail token support; duplicates `knownTokens` + CoinGecko work already done.
- *Nest tool inputs as `from_token: {chain, address, symbol, decimals}`.* Gemini Flash API does not support nested object schemas for function calling (ticket v-pxuw design #4). Keep flat args; make `from_decimals`/`from_address` optional overrides that the resolver fills when absent and verifies when present.
- *Format values in the app (`SwapFlowCard` divides by decimals).* Two conventions (THOR/Maya 8-dec vs 1inch/LiFi dst-dec) mean the UI would need provider-specific math — business logic the app shouldn't own per project architecture (CLAUDE.md).
- *Patch only `execute_swap`.* Same bug class exists for every sibling that will echo provider numbers (LP estimated-units, contract-call return values). VA-185 already demonstrated local patches don't generalize.
- *Backwards-compat alias `resolved: Record<string,string>` for one release.* Historical-message replay via `agent-backend/internal/service/agent/convert.go` already migrates renamed tool parts (v-pxuw C6); extend the same migration rather than ship a dual shape that the next author has to remember to avoid.

## Files

**Primary changes (mcp-ts):**
- `mcp-ts/src/tools/execute/_base.ts` — `ExecutePrepResult.resolved` shape change: `{labels: Record<string,string>, raw?: Record<string, {baseUnits: string; decimals: number; symbol: string}>}`.
- `mcp-ts/src/tools/execute/_amountResolver.ts` — extend with `formatTokenAmount(baseUnits, decimals, symbol)`, `formatNativeSwapAmount(baseUnits, toCoin)`, and `formatRate(fromAmt, toAmt, fromSymbol, toSymbol)` helpers. Existing `baseUnitsToDecimalString`/`resolvedLabel` is the pattern.
- `mcp-ts/src/lib/resolveToken.ts` — NEW. Signature: `resolveToken({chain, symbol, address?, decimals?, ownerAddress?}) → Promise<{chain, contractAddress, decimals, symbol, priceUsd?, source: 'native'|'known'|'rpc'|'coingecko'}>`. Priority: `chainFeeCoin` → `knownTokens` (exact contract-address match, case-insensitive) → on-chain metadata resolver (`getTokenMetadata` from `@vultisig/sdk`) → CoinGecko via `searchToken`. If caller supplies `decimals`, verify against resolved value and return an LLM-readable mismatch error instead of silently overriding.
- `mcp-ts/src/tools/execute/execute_swap.ts` — thread `resolveToken` for both tokens; populate `resolved.labels.{from_token, to_token, minimum_output, quote_summary, rate}` via the new formatters; populate `resolved.raw.minimum_output` for LLM reasoning. Fix both native (8-dec via `getNativeSwapDecimals`) and general (dst-dec) branches.
- `mcp-ts/src/tools/execute/execute_send.ts`, `execute_contract_call.ts`, `execute_lp_add.ts`, `execute_lp_remove.ts` — migrate to `{labels, raw}` shape and call `resolveToken` for token inputs. LP tools don't echo output-side numbers today, but still migrate to the new shape so a future "estimated LP units" field can't regress.

**UI consumers (vultiagent-app):**
- `src/features/agent/components/tools/ExecuteCards/SwapFlowCard.tsx` + `SendFlowCard.tsx` + `ContractCallCard.tsx` + `LpFlowCard.tsx` — mechanical rename `r.foo` → `r.labels.foo`. No formatting added; all math stays server-side.
- `src/features/agent/lib/executePrep.ts` — `parsePrep` / `ExecutePrepResult` type alignment.

**Reference patterns (read, don't modify):**
- `vultisig-sdk/packages/core/chain/swap/native/utils/getNativeSwapDecimals.ts` — authoritative 8-dec-for-non-Maya rule; reuse, don't replicate.
- `vultisig-sdk/packages/core/mpc/swap/native/utils/nativeSwapQuoteToSwapPayload.ts` — canonical `fromChainAmount(expected_amount_out, toDecimals).toFixed(toDecimals)` pattern (lines 44–48).
- `vultisig-sdk/packages/core/chain/coin/chainFeeCoin.ts`, `knownTokens/index.ts`, `token/metadata/resolver.ts` — the four decimal sources `resolveToken` composes.
- `vultisig-sdk/packages/sdk/src/tools/token/searchToken.ts` — CoinGecko adapter; `resolveToken` reuses as fallback.
- `shapeshift-agentic/apps/agentic-server/src/tools/initiateSwap.ts:154–197` — `createSwapSummary` shape for the label-vs-raw split.
- `.tickets/v-pxuw.md` — design decisions #2 (echo interpretation), #4 (flat schemas), #5 (stepper ownership); preserve all three.

**Historical-message migration:**
- `agent-backend/internal/service/agent/convert.go` — if the `resolved` field shape in persisted `data-message` parts changes, add a replay migration entry alongside v-pxuw Task 6's `renamedToolMap`. If the wire shape is unchanged (only nested reorg), no migration needed.

## Acceptance Criteria

- [ ] `ExecutePrepResult.resolved` is typed as `{labels, raw?}`; raw base-unit integer strings cannot typecheck in a label slot.
- [ ] `formatTokenAmount`, `formatNativeSwapAmount`, `formatRate` helpers live in `_amountResolver.ts` and are the only site that calls `fromChainAmount` inside `execute_*`.
- [ ] `resolveToken` helper implements the four-source priority chain, returns `source` tag, and surfaces a mismatch error when caller-supplied `decimals` disagrees with resolved value.
- [ ] All five `execute_*` tools call `resolveToken` for every token input; `from_decimals`/`to_decimals`/`from_address`/`to_address` are optional overrides, not required.
- [ ] `labels.from_token` / `labels.to_token` echo canonical resolution — e.g. `"USDC (0xaf88…5831 on Arbitrum, 6 dec)"`.
- [ ] `SwapFlowCard` on a \$1 ETH→USDC swap via MayaChain renders `Min. received: 0.77574 USDC` (not `77574000`), and the quote_summary subtitle reads human decimals with symbol.
- [ ] General (1inch/LiFi) quote branch formats via destination-token decimals; native (THOR/Maya) branch formats via `getNativeSwapDecimals`.
- [ ] Each flow card (`Send`/`Swap`/`ContractCall`/`LpAdd`/`LpRemove`) reads exclusively from `r.labels.*` — no formatting math in the app.
- [ ] New unit tests assert `resolved.labels.minimum_output` is human-formatted with unit symbol for both native and general quote paths.
- [ ] New unit tests assert `resolveToken` mismatch-detection: caller passes wrong `decimals`, gets an LLM-readable error string; caller omits `decimals`, gets resolved value; each of the four sources exercised.
- [ ] Historical conversations replay cleanly: a persisted pre-migration `data-message` part with the old `resolved: Record<string,string>` shape renders in the new card without crashing (extend `convert.go` if wire shape changed).
- [ ] `pnpm test` in mcp-ts, `yarn test` in vultiagent-app, and `go test ./...` in agent-backend all green; typecheck + lint clean in each repo.

## Gotchas

- MayaChain's CACAO fee coin is 10-decimal, non-fee Maya tokens use native decimals — `getNativeSwapDecimals` already encodes this; do not re-implement.
- `searchToken` hits CoinGecko's community-curated `decimal_place` and can be stale; that's why it's last in the priority chain, after on-chain RPC.
- Gemini Flash function-calling does not support nested object inputs (Firebase AI Logic docs); keep tool inputs flat and use optional override fields.
- `evmCheckAllowance` in `execute_swap` currently takes `{owner, spender, amount}` as raw `0x`-prefixed strings; `resolveToken` returning a typed object must not break that call site.
- `resolved.quote_summary` is used as the flow-card subtitle AND consumed by the LLM for follow-up reasoning — format with symbols so both audiences work (e.g. `"0.000438 ETH → ~0.77574 USDC via MayaChain"`).
- `mcp-ts/src/tools/swap/build-swap-tx.ts` and other gated legacy `build_*` tools are on the deprecation path (`_meta.categories: ["custom_send"|"contract"|...]`) — do NOT migrate them to the new shape; they stay as escape hatches with their existing raw-string output.
- v-pxuw Tier D asserted "tool_calls fired", not "card rendered correctly" — add at least one rendered-value assertion per flow card to close the test-harness gap that let this ship.
- `_base.ts` `ExecuteTxArgs` / `txArgs` (the signer-transport payload) stays untouched — base units belong there; this ticket only re-shapes `resolved`.
- When extending `convert.go` migration, follow the v-pxuw C6 pattern (`renamedToolMap` + `TestMessageFromDB_*` coverage); don't invent a new migration mechanism.

## Notes

**2026-04-19T23:22:07Z**

Implementation complete on branch refactor/v-pxuw-tool-consolidation across 4 repos (vultisig-sdk, mcp-ts, vultiagent-app, agent-backend). All tests + typecheck + lint green: mcp-ts 291/291, vultiagent-app 577/577, agent-backend all pkgs pass. Final review caught 3 field-name drifts (value_echo/value, slippage/tolerance_bps, missing args_preview) — fixed.
