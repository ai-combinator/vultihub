---
id: v-slzh
status: in_progress
deps: []
links: []
created: 2026-04-29T05:44:17Z
type: task
priority: 1
assignee: Jibles
---
# execute_send: shrink LLM schema to user-decision fields + close native-send safety net

## Objective

Shrink `execute_send`'s LLM-visible schema to only the fields that encode an actual user decision (asset, amount, recipient, optional memo), and let the server resolve everything else from vault context, RPC, and token registry. Close the intent-guard gap that lets native-token sends bypass the deterministic clarifier safety net. Together these eliminate the failure mode where "Let's send 0.0001 ETH" loops to the agent's thrash breaker instead of producing a clarifying question.

## Context & Findings

**Bug observed:** User typed "Let's send 0.0001 ETH" — agent ran for ~14s, hit the loop breaker at `agent-backend/internal/service/agent/loop_breaker.go:49`, and returned the canned "I couldn't complete that in a reasonable number of steps. Please try a more specific request — e.g. include the chain, token, and amount explicitly." The canned message is itself misleading because the user *did* give amount and token; what was missing was the recipient.

**Root cause is three layers stacked:**

1. **Intent guard skips native sends** (`agent-backend/internal/service/agent/send_intent_integration.go:97`):
   ```go
   if !intent.IsSend || intent.TokenKind == TokenKindNative || intent.TokenKind == TokenKindUnknown {
       return tools, "", intent
   }
   ```
   The pre-LLM rules-based clarifier (the only thing that would deterministically short-circuit ambiguous sends) opts out for `TokenKindNative`. ETH never gets the safety net.

2. **High-confidence predicate doesn't check recipient** (`send_intent.go:180`): TokenKind + Chain + FromAddress + AmountBase = HIGH. Recipient isn't in the predicate, so even if the guard ran for native sends, "Let's send 0.0001 ETH" would resolve as high-confidence and proceed.

3. **`execute_send`'s schema asks the LLM to populate fields the server already has or auto-fetches.** `from` is required and described as "do not omit, do not fabricate" (`mcp-ts/src/tools/execute/execute_send.ts:363`) — but the server already has it in vault context, and `from_address_validator.go` exists *only* to validate the LLM's copy. Gas/nonce/EIP-1559 fees are documented as "fetched server-side when omitted" (lines 370-373) — the override path is unused in practice but the LLM sees fields whose handling it can mishandle. `denom`, `fee_rate`, `token_address`, `token_decimals` are similarly server-derivable from chain+symbol or family classification.

**Why the LLM thrashes:** When it does call `execute_send` without a recipient, MCP returns a generic Zod schema validation error ("required at recipient"). Compare `from`-missing, which returns a behavioral hint at `execute_send.ts:569`: `'execute_send (EVM): \`from\` is required to fetch nonce/gas.'` — actionable, model-readable. The recipient error gives the LLM nothing to learn from, so it retries with permutations. Loop breaker trips at iteration 6 with same-tool repeats.

**The intent guard's mechanism 1 (forced tool_choice + tool-list narrowing) makes this worse on the cases it does fire.** `turn_state.go:77-91` sets `tool_choice=required` on high-confidence intents, which removes the LLM's ability to reply with text — it cannot ask a clarifying question even when the resolver's confidence call was wrong. The clarifier short-circuit (mechanism 2) is unambiguously good and stays; the lock-out is the dangerous half.

**Decoupling that makes the schema shrink safe:** `execute_send`'s LLM input schema and the tool's output prep payload are independent. Mobile cards (`vultiagent-app/src/features/agent/components/tools/ExecuteCards/SendFlowCard.tsx`), signing flow, and SDK all consume the server-built prep payload. As long as the payload shape preserves the resolved `from` / chain / token fields (which it does — server still builds them, just from vault context instead of LLM args), zero downstream changes are needed.

**The `injectVaultArgs` `_meta` flag mechanism** (`agent-backend/internal/service/agent/executor.go:3167-3220`) is already in production for injecting ECDSA/EDDSA/chain_code keys into other tools' args server-side. Same proven pattern applies to `from`.

**Address book exists but the LLM does the lookup in-context.** `prompt.go:827` says "You have the user's address book. When they refer to contacts by name, resolve the address directly." The recipient field should accept contact names directly with server-side resolution, eliminating that prompt section.

**Rejected scope (Tier 3):**
- Deleting the intent guard entirely. Mechanism 2 (deterministic clarifier short-circuit) is genuinely useful — it returns a plain-text response without an LLM roundtrip on ambiguous sends. Worth keeping.
- Restructuring `prompt.go` as a standalone refactor. The prompt will shrink naturally as load-bearing rules for now-removed fields become deletable in the same diff. No separate restructure project warranted.

**Out of scope (covered by separate tickets):**
- Same lens applied to `execute_swap`, build_*_send escape hatches, other `execute_*` tools — see audit ticket v-giou.
- Loop-breaker fallback message rewrite (current "include the chain, token, and amount" text is misleading).
- Output validators for prompt rules currently enforced as defensive coaching (deprecated-testnet names, card-restating, etc.).

## Files

- `mcp-ts/src/tools/execute/execute_send.ts` — schema and handler.
- `mcp-ts/src/tools/execute/_base.ts` — shared helpers if applicable.
- `agent-backend/internal/service/agent/executor.go:3167-3220` — `injectVaultArgs` extension.
- `agent-backend/internal/service/agent/from_address_validator.go` — obsolete after `from` leaves the schema.
- `agent-backend/internal/service/agent/send_intent.go:180` — high-confidence predicate.
- `agent-backend/internal/service/agent/send_intent_integration.go:97` — `TokenKindNative` bypass.
- `agent-backend/internal/service/agent/turn_state.go:77-91` — forced `tool_choice` logic.
- `agent-backend/internal/service/agent/prompt.go` — sections that coach for removed fields/behaviors (notably the `from` warning at line 363, address-book hint at line 827, possibly MCP Address Resolution at 251 and EVM address universality at 254).
- `agent-backend/scripts/qa/curl-replay/conversations/` — fixtures that pass any of the removed fields explicitly need updating.
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/SendFlowCard.tsx` and its tests — *not* expected to change; verify by integration test that the prep payload shape is preserved.

## Acceptance Criteria

- [ ] "Let's send 0.0001 ETH" against the agent (with vault holding ETH on Ethereum) does NOT hit the loop breaker. The agent responds with a clarifying question asking for the recipient.
- [ ] `execute_send`'s LLM-visible schema contains only `asset` (or chain+token_symbol), `amount`, `recipient`, optional `memo`. No `from`, no gas/nonce/EIP-1559 fields, no `denom`, no `fee_rate`, no `token_address`, no `token_decimals` exposed to the LLM.
- [ ] Server-side resolution paths exist for every removed field, with parity for every chain family `execute_send` supports today (EVM, UTXO, Cosmos, Solana, Tron, TON, Sui, Cardano).
- [ ] `recipient` accepts contact names. Resolution order: address-shape match → address-book lookup for the resolved chain → structured error naming the available entries on that chain when no match.
- [ ] Intent guard fires for native sends (`TokenKindNative` bypass at `send_intent_integration.go:97` removed).
- [ ] Intent guard's high-confidence predicate (`send_intent.go:180`) requires resolved recipient. "Let's send 0.0001 ETH" classifies as ambiguous and produces a recipient-asking clarifier via mechanism 2.
- [ ] Intent guard's mechanism 1 removed: no forced `tool_choice=required`, no tool-list narrowing on high-confidence intents. Mechanism 2 (deterministic clarifier short-circuit on ambiguity) preserved.
- [ ] `from_address_validator.go` deleted, or its enforcement is moved into the server-side resolution path (no LLM-passed value to validate).
- [ ] Prompt sections coaching for removed fields/behaviors are deleted: the `from` "do not omit, do not fabricate" warning, address-book lookup instructions, any gas/nonce/denom/fee_rate coaching. Sections that remain meaningful (identity, response style, ambiguity policy, multi-step flow framing) are untouched.
- [ ] `execute_send`'s prep payload shape is byte-for-byte preserved for downstream consumers. Verified by: existing `SendFlowCard` tests pass without modification, and at least one integration assertion confirms the prep payload contains the same resolved fields it does today.
- [ ] Curl-replay fixtures that pass any removed field explicitly are updated; `make qa-t1` passes.
- [ ] `agent-backend`: `go test ./internal/service/agent/...` passes, `golangci-lint run` clean, `go build ./...` clean.
- [ ] `mcp-ts`: `pnpm typecheck`, `pnpm lint`, `pnpm test` clean.

## Gotchas

- **Mobile app cards consume the prep payload, not LLM args.** Preserving the payload shape is the contract that keeps this PR's blast radius small. If the resolved-field echo changes shape, `SendFlowCard` and signing flows break silently.
- **`injectVaultArgs` `_meta` flag is the proven pattern for `from` injection** — already shipping for vault keys on other tools. Use it; don't invent a parallel mechanism.
- **Verify the auto-fetch path actually works for every chain family before removing the LLM-visible override.** EVM gas/nonce auto-fetch has been in production; analogous fields on less-trafficked chain families (e.g., Cosmos `denom` resolution) may be paper-only. Run a smoke per family before declaring parity.
- **Don't bundle Tier 3 (prompt restructure as a standalone refactor, intent guard deletion).** The prompt will shrink naturally as load-bearing rules become obsolete in this PR. Separate restructure isn't warranted; bundling it inflates blast radius and makes review harder.
- **Don't expand scope to `execute_swap` / `build_*_send` / other `execute_*` tools.** Those are explicitly covered by audit ticket v-giou; doing them here turns a focused refactor into a sprawling one and forecloses on the audit's ability to apply the same lens cleanly.
- **The intent guard's clarifier short-circuit (mechanism 2) is load-bearing.** It bypasses the LLM entirely on ambiguous sends — saving a roundtrip and ensuring deterministic phrasing. Removing both mechanisms would lose this. Drop only mechanism 1.
- **The loop-breaker's canned fallback message contradicts what the user typed in this case.** Tempting to fix here, but it's a separate concern (the breaker fires on more than just send flows) — file as its own ticket if not already.
- **Worktree at `agent-backend/.claude/worktrees/v-vvep` contains in-progress duplicates of several agent files.** Verify there's no conflict before merging.

## Notes

**2026-04-29T07:50:26Z**

Tasks 1 (intent guard fixes ef41f50) and 2 (schema shrink + injection 779fbcb mcp-ts / 8d2d4c4 agent-backend) implemented and reviewed; b3eaac2 fixes review findings (utxo regex anchor, injection ordering, chain alias normalization, coverage gaps)

**2026-04-29T09:58:32Z**

Draft PRs open: agent-backend#199, mcp-ts#65. Backend-mode E2E verified bug-driver (44 = 4/4) plus Cosmos/XRP/Sui sends. MCP wire probe confirmed schema slim + inject_from_address meta flag. Pre-existing tool-registry gaps for Tron/Cardano native sends documented as out-of-scope. v-slzh services left running on alt ports (mcp 9092, backend 8085) — user's existing dev stack on 9091/8084 untouched.
