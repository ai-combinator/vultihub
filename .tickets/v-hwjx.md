---
id: v-hwjx
status: closed
deps: []
links: []
created: 2026-04-29T01:36:50Z
type: bug
priority: 2
assignee: Jibles
---
# Execute_* cards: show clean loading preview while waiting for prep

## Objective
The execute_swap / execute_send / execute_contract_call cards currently render in a Frankenstein half-built state while the agent is still streaming the tool call: rows blink in token-by-token from the streamed input args, the stepper shows the hardcoded `[Sign, Broadcast]` fallback, and then everything "pops" the moment `output-available` lands and the resolved prep parses. The card should instead show a small, coherent preview of whatever streamed-input fields we already have (e.g. `SWAP 0.1 ETH → USDC`) plus a single loading indicator, then upgrade in one frame to the full card once prep is resolved — so the user sees a normal-looking loader, not a half-broken card.

## Context & Findings
- Reproduces every time the agent triggers `execute_swap` (and the same shape applies to `execute_send` / `execute_contract_call` — they share the pattern).
- Two information sources arrive on different timelines:
  1. **Streamed input args** (`from_symbol`, `to_symbol`, `amount`, `from_chain`, etc.) — dribble in token-by-token as the model types the tool call.
  2. **Resolved prep** (`txArgs`, `stepperConfig`, `resolved.labels.*` like provider, rate, min received, slippage, fee) — atomic, only present once the tool finishes (`output-available`).
- The card has no notion of "card as a whole isn't ready yet." Each `ActionCard.Row` self-gates on truthiness, so resolved-source rows stay hidden during streaming while input-source rows blink in independently.
- `StepperRow` falls back to `FALLBACK_STEPS = [{id:'sign'}, {id:'broadcast'}]` when `stepperConfig` is null. That fallback is ALSO a legitimate degraded-mode display for backends that genuinely don't emit a stepper config — it must not be deleted, only narrowed so it can't be reached during the pre-prep wait.
- The reducer's initial phase is already named `fetching_prep` — the state machine knows we're loading. The card just doesn't branch on it; today it only spins the first stepper dot during this phase.
- `ToolStatusCard` exists in the codebase and is already used by these cards for error states; reuse its visual pattern (or whatever the team's standard inline loader is) rather than introducing a new loading widget.

**Rejected approaches:**
- Hiding the card entirely until `output-available`. The agent's input-streaming is informative — the user benefits from seeing "swap is starting, it's roughly 0.1 ETH → USDC" immediately. We just shouldn't pretend we have the resolved data when we don't.
- Removing the `[Sign, Broadcast]` stepper fallback. It's still a valid degraded-mode display for a backend that omits `stepperConfig`. Keep it, but only after prep has been received.

## Files
- `src/features/agent/components/tools/ExecuteCards/SwapFlowCard.tsx` — primary card (matches user repro)
- `src/features/agent/components/tools/ExecuteCards/SendFlowCard.tsx` — same pattern, same fix
- `src/features/agent/components/tools/ExecuteCards/ContractCallCard.tsx` — same pattern, same fix
- `src/features/agent/components/tools/ExecuteCards/shared.tsx` — `StepperRow` + `FALLBACK_STEPS`; the fallback's allowed-reach must be scoped to "prep received but no stepperConfig"
- `src/features/agent/components/tools/ToolStatusCard.tsx` — reference for existing inline status visual
- `src/features/agent/lib/executePrep.ts` — `parsePrep` returns null when output is absent/unparseable; no changes expected
- `src/features/agent/hooks/useTransactionFlow.ts` — exposes `phase.kind === 'fetching_prep'` already; the card can branch on this OR on `prep == null && partState !== 'output-error' && partState !== 'output-available'`

## Acceptance Criteria
- [ ] During `input-streaming` / `input-available` (i.e. before prep is resolved and not in an error state), each Execute_* card renders a single coherent preview: the streamed-input summary line (e.g. \`SWAP 0.1 ETH → USDC\`, \`SEND 5 USDC to 0x12…ab\`) plus one inline loading indicator. No empty resolved rows, no `[Sign, Broadcast]` fallback stepper visible.
- [ ] When prep arrives (`output-available` with parseable prep), the card transitions to the full ActionCard layout with all rows + the real `stepperConfig` stepper in a single frame. No row-by-row blink-in.
- [ ] The `[Sign, Broadcast]` `FALLBACK_STEPS` constant remains in place and continues to work for the case where prep IS received but `stepperConfig` is null (legitimate backend degradation). Verify by mocking that scenario in the existing test.
- [ ] Existing terminal/error rendering paths (output-error, signed/done, failed/retry, timed-out, requiresExplicitTap) are unchanged.
- [ ] `ExecutionHistoricalGuard` replay path is unaffected — historical hydration still renders the terminal state of past tool calls correctly.
- [ ] Tests in `ExecuteCards/__tests__/SwapFlowCard.test.tsx`, `SendFlowCard.test.tsx`, `ContractCallCard.test.tsx` are extended to cover (a) `input-streaming` with partial inputs renders the loading preview, (b) `input-streaming` does NOT render the fallback stepper, (c) `output-available` with no `stepperConfig` still renders the `[Sign, Broadcast]` fallback.
- [ ] Lint and type-check pass.

## Gotchas
- The card must NOT remount between the loading preview and the full card — `useTransactionFlow` holds reducer state (consent registration, prep-received guard via `prepReceivedRef`). Use a conditional render INSIDE the same component, not an early return that changes the component identity tree above the hook.
- The streamed-input summary line for swap depends on which fields have arrived: handle `from_symbol`-only, `from_symbol + to_symbol`, `from_symbol + to_symbol + amount` gracefully — don't show stray separators or `undefined`.
- Do not pull resolved-only labels (provider, rate, slippage, fee, min received) into the loading preview — they don't exist yet and trying to show them re-creates the original bug.
- Keep the loading preview tight visually so it doesn't cause a height jump when the full card swaps in. If a height jump is unavoidable, that's acceptable — the goal is "everything appears at once when ready," not animated layout continuity.
- The `fetching_prep.timedOut` case (45s no PREP_RECEIVED) is a separate state with its own banner — make sure the loading preview is replaced by the timeout messaging, not stacked with it.

## Notes

**2026-05-04T23:09:10Z**

auto-closed: merged via vultisig/vultiagent-app#294
