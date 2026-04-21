---
id: v-wzgs
status: closed
deps: [v-bauf]
links: []
created: 2026-04-17T00:50:57Z
type: task
priority: 1
assignee: Jibles
---
# vultiagent-app: unify execute_* action cards behind TxStepCard shell

## Objective

The `execute_*` tool UIs in vultiagent-app (send / swap / LP / contract-call) currently render as three visually disconnected pieces — a floating "PREPARING SEND" timeline entry, a bordered summary card with rows, and a bare stepper sibling below the card — plus, on failure, a separate red bordered error box with a "Try again" button. All four pieces are already driven by a single tool call and a single flow hook; the disunity is purely in layout composition. Introduce a single action-card shell that all four cards compose, so frame + header + preview rows + stepper + error footer live inside one border with a consistent preparing / ready / errored set of variants.

## Context & Findings

**Symptoms** (user-reported, 2026-04-17, prompt "Can we self send 0.0001 ETH on arb?"):
- Floating `TimelineEntry` "PREPARING SEND" stacked above the card
- Bordered `PreviewBlock` "Send transaction" summary with Amount / Chain / To
- Vertical stepper (Prepare / Sign / Broadcast) rendered outside the border
- On failure: separate red bordered box "Failed to derive Ethereum address" with a "Try again" button, rendered below the stepper

**Root cause** (layout composition, not data flow):
- `SendFlowCard.tsx:43-65` renders `<PreviewBlock>` and `<ExecutionStepper>` as **siblings**, so the stepper falls outside the card frame
- `PreviewBlock` (`shared.tsx:86-133`) is the only thing that draws the frame today, and it brackets rows only
- `ExecutionStepper.tsx:36-70` hardcodes vertical layout (SVG connectors at :100-124) and ships its own `ExecutionErrorFooter` (:237-291) as a separate red-bordered box with a Try-again `Pressable`
- `PrepLoading` (`shared.tsx:36-39`) is a bare `TimelineEntry` with no frame — preparing state has zero visual continuity with the post-prep card
- Pattern is **replicated across all four cards** (SendFlowCard, SwapFlowCard, LpFlowCard, ContractCallCard)

**History — why the shell is missing:**
- **v-pxuw Task 9** (commit `60362b2`, 2026-04-16) ported the shapeshift `Execution.*` compound but **missed porting `TxStepCard`** (the shapeshift outer shell)
- **v-pxuw Task 12** (commit `45ef60c`, 2026-04-17) flattened the compound because the context layer was stateless and introduced stale-closure risk in the flow hook's callback refs. Defensible in isolation — but with no `TxStepCard`, the frame responsibility silently shifted into `PreviewBlock`, which only ever bracketed rows
- This is exactly what the parent epic `.tickets/v-bauf.md` already flagged as an alignment target

**Shapeshift reference pattern** (the north star):
```
<Execution.Root state={...}>
  <Execution.HistoricalGuard>
    <TxStepCard.Root>
      <TxStepCard.Header>...</TxStepCard.Header>
      <TxStepCard.Content>...rows...</TxStepCard.Content>
      <Execution.Stepper>...</Execution.Stepper>
      <Execution.ErrorFooter />
    </TxStepCard.Root>
  </Execution.HistoricalGuard>
</Execution.Root>
```
Key: frame + header + rows + stepper + error footer all live inside the same `TxStepCard.Root` border. Error is inline red text, not a separate box. No Try-again button. Preparing is handled via skeleton rows + `overrideStatus` on the stepper's first step — no floating timeline entry.

**Decisions:**
- **Add the missing shell** (call it `TxStepCard` or `ExecuteActionCard`) in `ExecuteCards/shared.tsx`. Owns frame, title/subtitle, rows slot, stepper slot, error slot, and preparing / ready / errored variants
- **Keep `ExecutionStepper` flat** — Task 12's flatten argument still stands; no context-threading wrapper, no callback refs
- **Tame `ExecutionErrorFooter`, don't delete it** — strip the bordered red box + Try-again `Pressable`, render as inline red text inside the shell (matches shapeshift)
- **Delete the per-step `Tap to retry`** + Pressable wrapper on the failed step
- **Horizontal stepper** — deliberate divergence from shapeshift's vertical. Product choice for the narrow mobile viewport. Worth revisiting if a 4th step (e.g. token approve on non-swap flows) is ever needed
- **Replace `PrepLoading`** with a "preparing" variant of the shell so preparing / ready / errored are all the same frame
- **No data-flow changes.** `useExecuteToolFlow`, `parsePrep`, `ExecutePrepResult`, `ToolExecutionState`, `StepperConfig` all stay

**Rejected:**
- Reintroducing the full `Execution.*` compound (Root/Context/Step/ErrorFooter) — Task 12's rationale stands; the value was the shell, not the context-threading
- Killing `ExecutionErrorFooter` entirely — user asked for failures to appear in the stepper area; shapeshift's inline red-text version satisfies that without losing the error surface
- Keeping vertical stepper — product wants horizontal
- Shallow fix (move stepper into `PreviewBlock` inline in each card) — same file surface area as structural, leaves the fifth execute card free to re-learn the nesting

## Files

**New shell:**
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/shared.tsx` — add `TxStepCard` (or `ExecuteActionCard`) with variants for preparing / ready / errored; replace `PrepLoading`; `PreviewBlock` either absorbed or retained as a ready-variant internal

**Stepper changes:**
- `vultiagent-app/src/features/agent/components/Execution/ExecutionStepper.tsx` — horizontalize `ExecutionStep` layout (icons-in-row with horizontal connectors, labels below), fold the bordered `ExecutionErrorFooter` into an inline-red-text variant rendered inside the shell, drop the per-step `Pressable` retry wrapper and "Tap to retry" text

**Card migrations** (all four collapse to the same shell):
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/SendFlowCard.tsx`
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/SwapFlowCard.tsx`
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/LpFlowCard.tsx`
- `vultiagent-app/src/features/agent/components/tools/ExecuteCards/ContractCallCard.tsx`

**Reference patterns (read, don't modify):**
- `/home/sean/Repos/shapeshift-agentic/apps/agentic-chat/src/components/ui/TxStepCard.tsx` — the shell we're porting
- `/home/sean/Repos/shapeshift-agentic/apps/agentic-chat/src/components/Execution.tsx` — stepper + inline error footer pattern
- `/home/sean/Repos/shapeshift-agentic/apps/agentic-chat/src/components/tools/SendUI.tsx` — how a tool card composes the shell end-to-end

**Related tickets:**
- `.tickets/v-bauf.md` (parent epic — shapeshift-pattern alignment)
- `.tickets/v-vtmi.md` (P1 derive-address failure — will surface via the new failed-step visual; validate the UI with that bug present)
- `.tickets/v-bpzs.md` (P2 LLM text duplication — separate; do not conflate)
- `.tickets/v-pxuw.md` Task 9 (added the compound) and Task 12 (flattened it)

## Acceptance Criteria

- [ ] A single action-card shell (`TxStepCard` or equivalent) exists in `ExecuteCards/shared.tsx` with variants for preparing / ready / errored states
- [ ] Frame + header + preview rows + stepper + error state all render inside the same border for every `execute_*` card
- [ ] Stepper is **horizontal**, rendered inside the card
- [ ] Failure state renders as inline red text inside the card — no separate bordered red box, no "Try again" button, no per-step "Tap to retry"
- [ ] Preparing state uses the same shell frame (no floating `TimelineEntry`); visually continuous with ready state
- [ ] All four cards (`SendFlowCard`, `SwapFlowCard`, `LpFlowCard`, `ContractCallCard`) migrated to the shell; no card reimplements frame + stepper sibling layout
- [ ] Net LOC delta is **negative** (shell replaces duplicated outer trees in 4 cards)
- [ ] `useExecuteToolFlow`, `parsePrep`, `ExecutePrepResult`, `ToolExecutionState`, `StepperConfig` unchanged (data flow untouched)
- [ ] `ExecutionHistoricalGuard` still wraps cards correctly; historical replay path unaffected
- [ ] Lint + typecheck + unit tests pass; add/update tests for the shell's three variants

## Gotchas

- The parent epic `v-bauf` is the north star — frame this as completing v-pxuw Task 9's missing port, not as a new abstraction
- Task 12's flatten argument still stands — do **not** reintroduce `Execution.Root` / `Execution.Context` context-threading or `onComplete` / `onError` callback refs in `useExecuteToolFlow` (stale-closure bug)
- Horizontal stepper is a deliberate divergence from shapeshift's vertical — it's a product choice, not an oversight
- `ExecutionStepper.tsx:1-8` header comment describes the Task 12 flatten; update it when the shell lands so the rationale stays legible
- React rules (`/home/sean/Repos/setup_and_configs/claude/rules/react.md`): no `useEffect` for derived state (prefer computed/memo), `&&` for conditional JSX, explicit readable JSX over DRY abstractions, functional `setState`
- `PreviewBlock` is exported from `shared.tsx`; check for external consumers before removing/replacing
- v-vtmi (derive-address failure) is a good live regression check for the new failed-step visual — validate the UI against it, but do not fix v-vtmi here

## Notes

**2026-04-17T02:21:59Z**

Implemented in vultiagent-app commit ba3e155. Single TxStepCard shell with preparing/ready/errored variants now wraps frame + header + rows + horizontal stepper + inline error for all four execute_* cards. PreviewBlock kept (external consumers). Net delta -41 lines. typecheck/lint/554 tests pass.
