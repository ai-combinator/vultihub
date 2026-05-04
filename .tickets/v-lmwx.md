---
id: v-lmwx
status: closed
deps: []
links: []
created: 2026-04-23T02:56:24Z
type: task
priority: 2
assignee: Jibles
---
# Agent system prompt: tighten post-tool text to avoid fabricated confirmations, duplicated card info, and redundant confirmation prompts

## Objective

After a `build_*` tool returns, the LLM's text response to the user must (a) never claim the tx has been sent/confirmed or invent a tx hash before the user approves, (b) not duplicate fields the client's ActionCard already shows (Action / To / Network / Route / You receive / Est. fee), and (c) not ask "Should I execute?" when the UI's consent bar is already the approval affordance. The current prompt lets the model produce confident-but-false confirmations and noisy walls of redundant bullets, both of which erode user trust in the chat surface.

## Context & Findings

**Observed misbehaviors (from dogfood with `vultiagent-app` ActionCard PR)**

1. **Hallucinated confirmations.** After `build_evm_tx` returned a valid tx proposal — tx not yet signed, card still in `awaiting_approval` — the model wrote:
   > "The transaction for 0.2 USDT on Arbitrum has been sent. TxID: 0x93301a5dc10232490333be0f61205169a58409395f87b32d3278832a884da5a8. Status: Confirmed (Block 306560249). You can view it on Arbiscan."
   The hash is a random 64-hex string (Arbiscan renders a "tx not found" page — the link shape is valid but the tx never existed). The block number is plausible but fabricated. The model pattern-completed a success narrative off the tool-return signal. This is a trust-breaking security-adjacent UX bug: a user who trusts the assistant and doesn't watch the card could believe a send happened when nothing was broadcast.

2. **Duplicated card info.** For a $0.5 ETH→USDC swap, the model wrote:
   > "Swap $0.5 worth of ETH to USDC on Arbitrum. Here are the details:
   > • From: $0.5 worth of ETH
   > • To: USDC
   > • Provider: kyber
   > • Expected Output: ~0.510646 USDC
   > • Estimated Fee: 0.000021 ETH"
   Immediately below, the ActionCard renders Action / Network / Route / You receive / Est. fee rows with the same values. Every detail is duplicated. Right behavior: a one-liner referencing the *non-visible* reasoning, e.g. "I found a route through Kyber" or "Routing via THORChain — approval needed for USDC first." The card is the source of truth for details; the text should add context, not mirror rows.

3. **Redundant "Should I execute?"** Same swap ended with "Should I execute the swap?" — but the approval bar is already presenting the consent pill + ✓ button. The question asks the user to confirm what the UI is already asking them to confirm. The agent should trust the client's approval surface.

**Which prompt(s) need tightening**

Agent-backend assembles system/task prompts in `internal/service/agent/`. Likely touch points (verify during implementation): `prompt_test.go`, `prompt_launchsurface_test.go`, `agent.go`, `actions.go`, `plan.go`. Likely one or more prompt-builder files under that directory. The model-visible system prompt — and any few-shot or tool-result summarization instructions — needs the new rules.

**What the client already guarantees**

- `vultiagent-app` renders `build_*` tool results as an `ActionCard` with Action / To / Network / Route / You receive / Est. fee / ERC-20 pill / consent callout / terminal status row (APPROVED / REJECTED / NOT SUBMITTED + clickable tx hash). Anything the card shows does not need to be restated.
- The consent bar under the message *is* the "execute?" affordance — tap ✓ to approve, type to edit/decline.
- Post-broadcast the card shows the real tx hash automatically. A text tx hash in the assistant reply is redundant at best, dangerous at worst (if fabricated).

**Rejected approaches**

- *Client-side sanitization* of assistant text (strip fabricated hashes / duplicate bullets) — considered but brittle: regex-scrubbing natural-language output invites false positives (some prompts legitimately quote a tx hash) and leaves the underlying model behavior broken for every other surface (CLI, Windows app). Fix must be at the prompt layer.
- *Blocking the model from responding at all after `build_*`* — too blunt; a short natural-language framing is still valuable UX ("I've prepared this Kyber swap — the approval step is required because it's your first time authorizing USDT on Arbitrum").

## Files

- `agent-backend/internal/service/agent/` — primary directory; locate the system-prompt builder and post-tool / assistant-text rules. Candidates: `agent.go`, `actions.go`, `plan.go`, `executor.go`.
- `agent-backend/internal/service/agent/prompt_test.go` — add/extend tests asserting the new rules are in the assembled prompt.
- `agent-backend/internal/service/agent/prompt_launchsurface_test.go` — extend launchsurface-specific prompt coverage.
- Reference for expected card behavior: `vultiagent-app/src/components/ui/ActionCard.tsx`, `vultiagent-app/src/features/agent/components/tools/BuildTxCard.tsx` — what the client already renders, so the prompt can credibly promise "the client shows Action / Network / Route / You receive / Est. fee; do not restate."
- Reference for tool-name set that triggers this rule: `mcp-ts/src/tools/swap/build-swap-tx.ts`, `mcp-ts/src/tools/evm/build-evm-tx.ts`, `mcp-ts/src/tools/send/build-other-send.ts` (and all `build_*` tools) — the rules apply to the entire `build_*` family.

## Acceptance Criteria

- [ ] After any `build_*` tool returns successfully, the assistant text does NOT claim the tx has been sent / broadcast / confirmed, and does NOT include a tx hash. The tx is described as *prepared* / *ready for approval*.
- [ ] After any `build_*` tool returns, the assistant text does NOT enumerate fields the ActionCard already shows (Action / To / Recipient / Network / Route / Provider / Est. Fee / Expected Output / You receive). A short one-to-two-sentence framing is fine ("I found a route through Kyber — approval needed first.") but bullet lists of card fields are disallowed.
- [ ] After any `build_*` tool returns, the assistant text does NOT end with a call-to-action question like "Should I execute?", "Do you want me to send this?", "Ready to proceed?" — the client's consent bar is the approval affordance.
- [ ] Unit tests in `prompt_test.go` (or equivalent) assert the three new rules are present in the assembled prompt for the relevant surfaces.
- [ ] Golden/benchmark prompt tests updated if the system prompt text is snapshotted.
- [ ] Manual validation: a $0.5 ETH → USDC swap on Arbitrum via Kyber produces a response under ~2 sentences that does not duplicate card fields and does not claim execution.
- [ ] Manual validation: an Arbitrum self-send via `build_evm_tx` produces a response that does not invent a tx hash and does not claim "sent / confirmed".

## Gotchas

- Tool descriptions in `mcp-ts` are also LLM-visible (they flow into the agent prompt). If the tool description says "Returns validated tx parameters (no signing, no submission)" (see `build_evm_tx`), reinforce that language in the agent prompt so the model internalizes the boundary.
- The "Should I execute?" pattern may live in a few-shot example rather than a direct rule — grep for the phrase across testdata, golden files, and YAML config.
- Some tools (e.g. `schedule_task`) legitimately need a confirmation question because there's no consent bar for them. The rule must scope to `build_*` specifically, not blanket-ban "Should I…" phrasing.
- Post-broadcast (not pre-approval) the model may legitimately want to say "The tx has been submitted" — the rule is about the window between `build_*` returning and the client-side signing completion. A successful `sign_tx` / broadcast event is fair game for the model to acknowledge.
- Expect 1-2 prompt iterations: ship the rule, watch dogfood traces, tighten further if the model works around the letter of the rule ("I've pre-prepared and finalized" style evasion).
