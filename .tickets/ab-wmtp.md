---
id: ab-wmtp
status: open
deps: []
links: []
created: 2026-05-01T01:03:23Z
type: task
priority: 2
assignee: Jibles
---
# Convert quick_actions from fenced-JSON-in-prose to a tool call

## Objective

`quick_actions` (the chip surface under category/option-list assistant turns) currently rides the prose channel: the model emits a fenced ```quick_actions JSON block at the end of its text reply, and the server runs a regex over the response to fish it back out. Convert it to a real tool call so structured data stops travelling through the user-visible text channel. This closes the original issue's recommended path (#204 option (a) — `present_options(actions: [...])` — explicitly called "cleanest, fits existing tool surface" by paaao) and deletes the known live-stream leak as a side effect.

## Context & Findings

**The shipped design is a known-fragile workaround, not the intended one.**

- Original issue (vultisig/agent-backend#204, by paaao) explicitly proposed two emission mechanisms: (a) a tool call `present_options(actions: [...])`, and (b) a structured-output path through the parts accumulator. paaao recommended (a) for the first pass.
- Shipped implementation (PR #228, NeOMakinG, merged 2026-04-30) chose neither. It invented a third path — fenced JSON inside the prose channel + a server-side regex extractor — which was not argued for in the PR or the issue.
- The PR shipped with a known LOW finding: *"Live-stream UX: user briefly sees the raw fenced JSON during streaming before server extraction. Documented tradeoff — no fix this PR."* That is exactly the bug currently visible in the app: under "ANALYZED FOR 5s" turns the chip JSON appears as a code block above the rendered chip row.
- Mechanism of the leak: the server's extractor strips the fenced block from the persisted DB row and emits a separate `data-quick_actions` SSE part for the chips, but it cannot un-stream the text deltas that have already been forwarded to the client char-by-char during generation. On reload the JSON disappears because the app re-reads the cleaned text from DB; in the live session it stays.

**Why a tool call is the right shape:**

- Every other structured surface in this app (`BuildTxCard`, `ReceiveCard`, `PriceCard`, `PaymentRequestCard`, typed signing cards) flows from a tool call rendered by a tool-keyed component. Quick-actions is the lone exception.
- A tool call moves the chip array onto the LLM API's structured side-channel. It never enters the text stream, so there is nothing to leak, nothing to strip, no live-vs-persisted divergence, no permissive regex (currently widened to `quick[-_]?actions` because the model drifts on the spelling — itself an admission the contract is fragile).
- The persisted wire shape consumed by the app (`data-quick_actions` SSE part with `{quick_actions: [...]}`) does not need to change. The app's existing `parseQuickActions` keeps working unchanged. The change is upstream of that part — how the chips get *into* the part — not downstream.

**Rejected alternatives (already discussed in this conversation):**

- *Strip the fenced block client-side as deltas arrive.* Bandaid. Leaves the smuggling in place; every new client (web station, future integrations) has to re-implement the same secret-tag stripper.
- *Suppress the fence at stream time on the server.* Smaller bandaid. Same family — adds another sieve over a channel that shouldn't be carrying structured data in the first place. Also has edge cases (partial fences at stream end, legitimate code blocks that look like the tag, model drift on fence spelling).
- Leaving as-is until reload masks it. Persistence/live divergence is a permanent footgun and the JSON dump on a "test" message is a visible quality regression in the live session.

**Scope boundaries:**

- This ticket is server-side + prompt change. The app's `parseQuickActions` and `MessageQuickActionChips` stay as-is — they consume `data-quick_actions` SSE parts which keep their wire shape.
- Out of scope: redesigning chip UX, adding new chip kinds, expanding the chip cap (currently 8), reworking the categories the prompt surfaces.

## Files

**Backend (agent-backend):**

- `internal/service/agent/prompt.go` — replace the "quick_actions emission shape" rule (lines ~234-256) and the "Capability questions and option lists" examples (lines ~205-232). The new rule tells the model to call the tool instead of emitting a fenced JSON block. Drop the canonical-fence wording entirely — there is no fence anymore.
- `internal/service/agent/quick_actions.go` — delete the regex extractor (`quickActionsBlockRE`, `extractQuickActions`, `parseQuickActionsJSON`). Keep the type definitions (`QuickAction`, `QuickActionsPayload`, `MaxQuickActionsPerMessage`) and the `coerceQuickActions` validation — those are still load-bearing for tool-arg coercion.
- `internal/service/agent/agent.go` — every call site of `extractQuickActions` (currently around lines 3441, 3556, 4079, 4188 per earlier grep) becomes a tool-call handler. Find the new tool's invocation in `resp.ToolCalls`, lift its argument onto `quickActions`, emit the existing `data-quick_actions` SSE part the way it does today. The `replaceTextParts` re-snapshot dance (lines ~4225-4233) goes away because the text parts never contained the JSON.
- `internal/service/agent/quick_actions_test.go` — the fence-spelling matrix and extractor tests get deleted; tool-arg coercion tests remain. Rewrite the suite around tool-call inputs.
- `internal/service/agent/prompt_test.go` — update the `TestBuildFullPromptContainsQuickActionsRule` assertions (currently pin the canonical-fence text). Pin the tool-call rule instead.
- Tool registration site — wherever the agent's tool list is assembled (look near where `update_memory`, `set_vault`, `schedule_task` are registered; agent-backend CLAUDE.md says these are the local tools alongside MCP-served ones). Add `present_options` (or whatever name lands; suggest `present_options` to match paaao's original spec, but `quick_actions` works too) with arg schema mirroring `QuickAction`. Tool execution is a no-op for the agent loop — the result is empty / a sentinel — and the `terminal_handler` lifts the args onto the response surface.
- `scripts/qa/curl-replay/conversations/47-quick-actions-emit.yaml` — update the fixture's expected response shape if it currently checks for the fenced block in the streamed text.

**App (vultiagent-app) — verify only, no changes expected:**

- `src/features/agent/lib/parseQuickActions.ts` — already consumes `data-quick_actions` parts; should keep working unchanged.
- `src/features/agent/components/MessageQuickActionChips.tsx` — same.
- `src/features/agent/components/AssistantMessageView.tsx` — same; it suppresses `data-quick_actions` in the generic part renderer and reads chips via `parseQuickActions` separately. Verify nothing assumes the JSON is in the text.

**Reference:**

- Original issue: vultisig/agent-backend#204 (read paaao's "Emission mechanism — pick one" section; option (a) is what we're now implementing).
- Existing tool registration patterns: `update_memory`, `set_vault`, `schedule_task` in `internal/service/agent/`. `schedule_task` in particular is a good shape reference because its tool arguments are also surfaced onto the response (`schedule_task_input` / confirmation flow), which is structurally similar to "lift the chip array onto the response."

## Acceptance Criteria

- [ ] New tool registered (suggested name: `present_options`) with arg schema covering id/label/prompt/kind, mirroring the existing `QuickAction` struct.
- [ ] Prompt rule rewritten to instruct the model to call the tool for capability/option-list turns. The "fenced JSON block" wording is gone from the prompt.
- [ ] Server lifts the tool-call argument onto the existing `data-quick_actions` SSE part — wire shape consumed by the app is byte-identical to today.
- [ ] Server lifts the tool-call argument onto the `QuickActions` field of `SendMessageResponse` for non-streaming clients.
- [ ] `extractQuickActions`, `parseQuickActionsJSON`, and `quickActionsBlockRE` are deleted; `replaceTextParts` is no longer called for chip-stripping (other callers — harmony rewrite, ActionBlock — keep using it).
- [ ] `coerceQuickActions` (per-chip validation: empty-string drop, dedupe by id, unknown-kind strip, 8-chip cap) still applies to tool-call args.
- [ ] App's `parseQuickActions` and chip rendering work unchanged — verified by sending a "what can you do" turn and seeing chips with no JSON in the bubble, both during stream and on reload.
- [ ] No prose-text-contains tests added (per project memory: brittle, verifies copy not behavior). Regression coverage goes through the QA replay matrix.
- [ ] Curl-replay fixture `47-quick-actions-emit.yaml` passes with the new shape.
- [ ] `go test ./...` green.
- [ ] PR title tagged appropriately; PR body lists the deleted regex/extractor and links back to #204 + this ticket.

## Gotchas

- The app's `parseQuickActions` accepts both `data: [...]` (bare array) and `data: {quick_actions: [...]}` (wrapped). Keep emitting the wrapped form to match `data-title` precedent — don't switch to bare just because tool-call args lend themselves to it.
- Tool-call args from the model are not pre-validated. Run them through `coerceQuickActions` server-side before emitting the SSE part — same defense-in-depth as today's extractor.
- `quick_actions` is in `CanonicalDataKinds` (`internal/types/parts.go`) and the parts accumulator whitelist. Those stay — the change is in how the part gets populated, not the part itself.
- Don't emit the chip part on signing turns or specific actionable requests. The current prompt rule has explicit "Do NOT emit" cases — port those exclusions into the tool's call-site guidance.
- If the model calls the tool *and* emits prose listing the same options, the user gets the duplicate-list problem the original issue was trying to kill. The prompt rule should still say "the chips ARE the answer; the prose is a one-line lead-in, not a parallel list."
- Tool execution returns nothing useful to the agent loop. Make sure the terminal handler treats the call as final-ish (or pairs it with the model's text reply) rather than looping on it expecting a meaningful tool result.
- The streaming text-end ordering matters: if the model writes its prose, then calls the tool, the existing `text-start/delta/end` events for the prose should fire before `data-quick_actions`. Mirror the order today's code uses for `data-tokens` / `data-actions`.
- Ping NeOMakinG and gomes before merging. NeOMakinG implemented the current shape; gomes wrote the original anti-wall-of-text prompt rule. Both should see the change.

## Notes

**2026-05-05T05:32:53Z**

migrated to GH issue: https://github.com/vultisig/vultiagent-app/issues/434

**2026-05-05T05:34:06Z**

kept open locally — GH issue is a duplicate, local remains canonical scratch
