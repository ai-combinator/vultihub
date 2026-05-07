---
id: v-vjww
status: open
deps: []
links: []
created: 2026-05-05T00:16:44Z
type: bug
priority: 2
assignee: Jibles
---
# Cross-chain "move" phrasing misclassified as send → self-send

## Problem
User message "Then let's move over 0.004 ETH on arb to ETH on ethereum" should classify as a cross-chain swap (bridge) but classifies as IntentSend, which forces `execute_send` with the user's own Arbitrum address as recipient. Result: self-send confirmation card on Arbitrum (the destination chain "ethereum" silently dropped), with the from==recipient warning as the only safety net. "Solved" means cross-chain phrasings without the literal word "from" route to IntentBridge / `execute_swap` — or the regex classifier stops forcing the wrong tool when the underlying model would have picked the right one unaided.

## Background
### Findings
- Intent classifier: `agent-backend/internal/service/agent/intent.go:43-82` (`ClassifyBuildIntent`). Two-stage gate — `hasCryptoShape` (≥2 of amount/asset/address/chain/staking-verb signals) then a verb-based switch.
- `sendVerbRe` (intent.go:231) = `\b(?:send|transfer|pay|move|withdraw)\b` — "move" is a send verb.
- `bridgeVerbRe` (intent.go:233) = `\bbridge\b|\bcross[ -]?chain\b` — only matches the literal words.
- Cross-chain fallback (intent.go:66): `sendVerbRe.MatchString(m) && hasCrossChainFromTo(m)` → IntentBridge.
- `hasCrossChainFromTo` (intent.go:109-120) uses `fromToChainRe` (intent.go:253) = `\bfrom\s+([a-z][a-z0-9-]+)\s+.*?\bto\s+([a-z][a-z0-9-]+)\b` — **requires literal `from`**. The user's "on arb to ETH on ethereum" has no "from" → returns nil → bridge route skipped → falls through to `sendVerbRe` alone → IntentSend.
- Existing test pinning the behavior: `intent_test.go:37` — `{"move my rune", "move my RUNE to maya", IntentSend}`.
- Chain canonicalization exists and should be reused: `canonicalizeChain` + `isKnownChainName` (intent.go:134-149) — handles eth/ethereum, sol/solana, btc/bitcoin, etc., backed by `knownCanonicalChainSet` (intent.go:157-171).
- After classification the loop forces `tool_choice=required` and narrows tools to the build slice for that intent. Originating motivation (intent.go:8-15): *"Gemini-flash reliably hallucinates tx-preview prose without calling any build_*_tx tool, after one successful tool call has landed in the conversation."*
- Self-send warning rule: `agent-backend/internal/service/agent/prompt.go:612-634` — when `execute_send` returns with from==recipient, response opens with the "⚠️ Heads up: this sends to YOUR OWN address" warning. This is what produced the screenshot.
- `execute_send` requires a recipient (`mcp-ts/src/tools/execute/execute_send.ts:389`); backend injects the user's own address as fallback when no explicit recipient is parsed (`injectFromAddress: true` pattern, ~line 423).
- Default agent model: `anthropic/claude-sonnet-4.5` (`agent-backend/CLAUDE.md`). The classifier was built for Gemini-flash's failure mode, not Sonnet's.

## Current Thinking
Two viable directions, both worth weighing in plan mode rather than picking one off-cuff. Not mutually exclusive.

**Direction A — narrow regex fix.** Loosen the cross-chain detector so phrasings without literal "from" still trigger the bridge route. Concrete shape: when `sendVerbRe` matches but `hasCrossChainFromTo` returns false, scan the message for two distinct known chains via `chainRe` + `canonicalizeChain` (alias-collapsed, same distinctness check as today's bridge gate). If found → IntentBridge. Cheap, deterministic, fits the existing pattern. Risk: false positives like "move 1 ETH on arb to my friend, the one on ethereum" — needs a guard or confidence test (e.g., the second chain must follow a `to`/`→`/`onto` preposition, not just appear anywhere).

**Direction B — re-examine whether the classifier should fire under Sonnet at all.** The classifier's stated purpose (intent.go:8-15) is mitigating a Gemini-flash *multi-step* hallucination — "after one successful tool call has landed." With the `execute_*` family, that intermediate state doesn't exist: `execute_send` / `execute_swap` are end-to-end, single-shot. The dominant trigger for the original bug doesn't apply in this flow. Sonnet 4.5 would likely have called `execute_swap(from_chain=arbitrum, to_chain=ethereum)` from the natural language given the full tool set. The classifier here actively prevented the right answer. Possible shapes: gate the classifier behind model identity (only narrow/force on a known-weak-model allowlist), or short-circuit it for execute_* intents on Sonnet-class models.

A is small and unblocks this specific phrasing today. B is the architectural question of whether the classifier still earns its keep on the current default model. If B lands first, A may become unnecessary.

The self-send warning (prompt.go:612-634) is doing real work as a last-line safety net here and should stay regardless of which direction wins. Without it the screenshot would have been a silent self-send.

## Constraints
- The self-send "YOUR OWN address" warning (prompt.go:612-634) must keep firing whenever `execute_send` returns with from==recipient. It is the final guard against any classifier-or-LLM mistake that ends up in execute_send with the user's own address as recipient.
- New chain-identity logic must reuse `canonicalizeChain` / `isKnownChainName` / `knownCanonicalChainSet`, not duplicate the alias table.
- Existing `intent_test.go` cases must be preserved or updated explicitly with rationale — silent test edits hide intent shifts.

## Assumptions
- The misclassification is reproducible from the message text alone — prior conversation turns didn't influence the classifier. (`ClassifyBuildIntent` takes only `userMessage`, but worth confirming the call site doesn't pre-compose context.)
- `execute_swap` accepts `from_chain != to_chain` and handles cross-chain end-to-end. Inferred from IntentBridge's comment (intent.go:25-26: *"execute_swap with from_chain != to_chain"*); not re-verified.
- The model in production for the screenshot was Sonnet 4.5 (configured default). If `AI_MODEL` was overridden to a Gemini variant, Direction B's premise weakens.

## Non-goals
- Rewriting the classifier as an LLM-based router. Larger architectural change with its own latency/cost tradeoffs; if Direction B leads anywhere, it's "skip the classifier under capable models," not "replace it with a smarter one."
- Touching the self-send warning copy or UI.
- Expanding the chain alias set in `canonicalizeChain`.

## Open Questions
- Direction A, B, or both? If A only, what's the false-positive policy on phrasings like "move 1 ETH on arb to my friend, the one on ethereum"?
- For Direction B: which models go on the "needs classifier" list — just Gemini-flash, or anything below a capability threshold? How is it configured (env var? list in code? derived from `AI_MODEL`)?
- Third lever not fully explored: should the recipient-injection path in `execute_send` (mcp-ts/src/tools/execute/execute_send.ts ~line 423) fail-closed when no explicit recipient appeared in the message, instead of silently injecting the user's own address? Catches this class of bug regardless of classifier behavior.

## Notes

**2026-05-05T05:32:39Z**

migrated to GH issue: https://github.com/vultisig/vultiagent-app/issues/433

**2026-05-05T05:34:06Z**

kept open locally — GH issue is a duplicate, local remains canonical scratch
