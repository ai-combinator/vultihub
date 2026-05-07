---
id: v-rzkp
status: closed
deps: []
links: [v-lmwx]
created: 2026-04-29T00:00:00Z
type: task
priority: 1
assignee: Jibles
---
# agent-backend: prompt restructure to fix execute_* card-field duplication on Gemini (and Sonnet)

## Objective

Restructure the anti-duplication rules in `agent-backend/internal/service/agent/prompt.go` so the LLM stops echoing card-owned fields (Swap / Rate / Provider / etc.) as bullet preambles above `execute_*` SwapFlow / SendFlow / ContractCallFlow / LP cards. The current rules are structurally weak — especially for Gemini, our next daily-driver — and the model ignores them. Replace with a positive directive deferring to per-tool `UI CARD DISPLAYS:` lines as the single source of truth, and add curl-replay regression coverage.

Scope also covers two additional post-tool-text failure modes carried from v-lmwx (closed as superseded): (a) fabricated tx hashes / "sent / confirmed" claims before the user has approved, and (b) redundant tail questions ("Should I execute?", "Ready to proceed?") that duplicate the consent bar.

## Context & Findings

**Failure observed (2026-04-29):** after a successful `execute_swap` ETH→USDC bridge on Arbitrum, the agent rendered the SwapFlow card AND duplicated card-owned fields above it as a bullet preamble (Swap, Rate, Provider). The system prompt has explicit anti-duplication rules in two places (`prompt.go` lines 192 and 321-327) but the model ignored them.

**Root cause — why the existing rule is structurally weak (Gemini-sensitive):**
- Two contradicting copies of the same rule, with different field lists.
- The prompt's own examples ("Here's the swap — review, then tap ✓ to approve…") prime a colon-list preamble.
- The rule lives under "Structured Output (Signing Surfaces)" — a section header that primes JSON, not narration.
- The stated consequence ("the validator will flag it") is false — no extractor checks for narrative duplication, so there is no real enforcement signal.

**Original rule commit:** `2bddbfc` (2026-04-28, "fix(agent/prompt): forbid restating card-owned fields in execute_* prose"). Failure observed one day later.

**Additional failure modes (from v-lmwx, closed as superseded — originally observed against `build_evm_tx`, same shape applies to `execute_*` pre-sign envelopes):**

1. **Fabricated tx confirmations.** After a `pre_sign` envelope returned (tx not yet signed, card in `awaiting_approval`), the model wrote: *"The transaction for 0.2 USDT on Arbitrum has been sent. TxID: 0x93301a5d… Status: Confirmed (Block 306560249). You can view it on Arbiscan."* The hash was a random 64-hex string; Arbiscan rendered "tx not found". Block number plausible but invented. The model pattern-completed a success narrative off the tool-return signal — trust-breaking, security-adjacent: a user who trusts the assistant and doesn't watch the card could believe a send happened when nothing was broadcast.

2. **Redundant "Should I execute?" tail.** A successful swap reply ended with "Should I execute the swap?" — but the input-bar consent pill + ✓ button was already presenting the approval affordance. The model is asking the user to confirm what the UI is already asking them to confirm.

## Files

- `agent-backend/internal/service/agent/prompt.go` — main rewrite (lines 190, 192, 321-327, 380).
- `agent-backend/scripts/qa/curl-replay/conversations/` — add new fixture(s), one per `execute_*` surface. Pattern matches existing `02-self-send-warning.yaml` / `06-self-send-warning.yaml`.
- mcp-ts (separate prerequisite): audit every surviving `build_*` escape hatch that emits a `pre_sign` envelope for a `UI CARD DISPLAYS:` line in its tool description. Confirmed for all `execute_*`. Audit needed for: `build_fin_swap`, RUJI staking, GHOST, Range CCL (`build_range_*`), `build_pumpfun_create`.

## Proposal

**Edits to `prompt.go`:**

1. **Line 190 — neutralize priming exemplars + add a Good/Bad pair.** Replace the "Here's the swap — review, then tap ✓ to approve…" exemplars with a Good/Bad pair using a fictitious swap (e.g. BTC → USDT via Wormhole — NOT the real Kyber transcript, to avoid Gemini negative-priming inversion). Cap at "aim short, never exceed 30 words" rather than hard 20. Ban bullet-list preambles specifically; do NOT ban all colons or all numbers (over-engineered).

2. **Line 192 — replace negative-paragraph rule with a positive directive that defers to per-tool `UI CARD DISPLAYS:` lines as the single source of truth.** Critical wording: allow a second sentence "when something material happened that the card cannot show (multi-step flow, route surprise, fee/slippage drift, ETA, soft warning)" — do NOT restrict to "only when the user asked", which would ban legitimate proactive context.

3. **Lines 321-327 — collapse the duplicate.** Replace the 7-line block with a single sentence: "Plain prose only — bullet/list preambles are wrong; see ## UI Cards." Keep one literal reminder rather than pure cross-reference, since Gemini section-anchor recall across ~150 lines isn't perfectly reliable.

4. **Line 380 — tighten self-send-warning parenthetical.** Change "open with this one-line warning" to "be exactly this one-line warning — nothing before, nothing after" and drop the third copy of the field list.

5. **Pre-sign window: no fabricated confirmations (from v-lmwx#1).** Add an explicit rule that after any `execute_*` tool returns a `pre_sign` envelope, the assistant MUST NOT (a) include a tx hash, or (b) use *sent / broadcast / confirmed* phrasing. The tx is *prepared* / *awaiting your approval*. Reinforce by mirroring tool-description language ("Returns validated tx parameters — no signing, no submission"). Ship as a Good/Bad pair (Bad: fabricated 0x… hash + "Confirmed (Block N)"; Good: "Prepared — tap ✓ to approve."). The rule is window-scoped: post-broadcast acknowledgments ("submitted in 32s") remain legal once a real `sign_tx` / broadcast event has occurred.

6. **Pre-sign window: no redundant approval tail-questions (from v-lmwx#3).** Ban closing tail-questions of the shape "Should I execute?", "Ready to proceed?", "Want me to send this?" specifically on `execute_*` turns — the input-bar consent pill is the approval affordance. Scope tightly: `schedule_task` and other surfaces *without* a consent bar legitimately need a confirmation question, so do NOT blanket-ban "Should I…" phrasing. Pattern-match on the closing-question shape on signing surfaces only.

## Acceptance Criteria

- [ ] `prompt.go` line 190 replaces the "Here's the swap…" exemplars with a Good/Bad pair using a fictitious pair (not Kyber/Arbitrum).
- [ ] `prompt.go` line 192 is a positive directive deferring to per-tool `UI CARD DISPLAYS:` lines as the single source of truth, with the "when something material happened that the card cannot show" carve-out (NOT "only when the user asked").
- [ ] `prompt.go` lines 321-327 collapsed to a single literal sentence pointing at `## UI Cards`.
- [ ] `prompt.go` line 380 self-send-warning parenthetical tightened; third copy of the field list removed.
- [ ] `prompt.go` contains an explicit pre-sign-window rule: after any `execute_*` tool returns a `pre_sign` envelope, the assistant text does NOT include a tx hash and does NOT use *sent / broadcast / confirmed* phrasing. Shipped as a Good/Bad pair with a fabricated-hash Bad example.
- [ ] `prompt.go` contains a tail-question rule scoped to `execute_*` surfaces (NOT a blanket ban): closing questions of the shape "Should I execute?" / "Ready to proceed?" are disallowed on signing surfaces. `schedule_task` and other non-consent-bar surfaces are explicitly excluded.
- [ ] Curl-replay fixtures added under `scripts/qa/curl-replay/conversations/`, one per `execute_*` surface (send, swap, contract_call, lp_add, lp_remove). Each plays a successful tool turn with stubbed `resolved.labels`, then asserts the assistant narration (a) does NOT echo those resolved values, (b) contains no tx hash and no *sent / broadcast / confirmed* tokens, and (c) does not end with a tail-question of the banned shape.
- [ ] Fixture assertions pull values dynamically from the stubbed tool result (read `resolved.labels.provider` etc.); they do NOT hardcode strings like "Kyber" or "Rate:".
- [ ] No `strings.Contains(prompt, ...)` style assertions added to `prompt_test.go`.
- [ ] mcp-ts audit completed: every surviving `build_*` tool that emits a `pre_sign` envelope has a `UI CARD DISPLAYS:` line in its description, OR a follow-up ticket is filed listing the ones that don't.
- [ ] `go test ./internal/service/agent/...` passes; `golangci-lint run` clean; `go build ./...` clean.
- [ ] Manual smoke: replay the 2026-04-29 ETH→USDC bridge prompt against Gemini and Sonnet — neither rebuilds the bullet preamble above the SwapFlow card.
- [ ] Manual smoke (from v-lmwx): an Arbitrum self-send via `execute_send` and a $0.5 ETH→USDC swap via `execute_swap` — neither reply invents a tx hash, claims "sent / confirmed", nor ends with "Should I execute?".

## Gotchas

- **Gemini negative-priming inversion.** If the bad exemplar is too close to a real failure transcript, Gemini sometimes produces the banned form. Use a fictitious swap pair (BTC → USDT via Wormhole) — NOT the observed Kyber/Arbitrum transcript.
- **Hardcoded fixture assertions rot when routing changes.** Read from `resolved.labels` in the stubbed tool result, not from string literals. When a route swap changes "Kyber" to "1inch", the assertion must still fire on duplication, not break.
- **Single-source-of-truth contract depends on `UI CARD DISPLAYS:` existing on every card-emitting tool.** The mcp-ts audit is a hard prerequisite — if a card-emitting tool lacks the line, line 192's rule has nothing to defer to and the LLM falls back to echoing fields.
- **Don't ban colons and numbers wholesale.** Plenty of legitimate narrations use them ("Bridged: confirmed in 32s"). Ban bullet-list preambles specifically.
- **Don't add `strings.Contains(prompt, ...)` tests.** Sean has explicitly flagged these as brittle (verifies copy not behavior). LLM round-trip via curl-replay is the right venue.
- **`schedule_task` policy_ready card** — once that surface lands, fold it into the same rule and add a fixture. Out of scope for this ticket but flag it for the follow-up.

## Notes

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/agent-backend#198 (DRAFT)
