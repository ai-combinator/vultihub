---
id: v-ygnw
status: open
deps: [v-llqd]
links: []
created: 2026-04-24T04:44:32Z
type: chore
priority: 2
assignee: Jibles
---
# Collapse ExecutionHistoricalGuard into useToolExecution as first-class 'skipped' step

## Objective

Replace the `ExecutionHistoricalGuard` wrapper component with a first-class `{ step: 'skipped' }` state in `useToolExecution`. Cards read the hook's state and render a stub inline when `step === 'skipped'`. This moves the policy "don't re-arm approval on a replayed historical tool call without a persisted outcome" from the component tree into the state machine that owns it, so new cards inherit the defense automatically instead of needing to remember the wrapper.

## Context & Findings

### Why this work exists

`useToolExecution` auto-fires approval on mount. The `ExecutionHistoricalGuard` wrapper is the sole barrier preventing a replayed-but-unresolved historical tool call from re-arming approval: if a card forgets the wrapper, the footgun fires. The hook itself now has enough information (`historicalToolIds` + `recentActions`) to refuse the auto-fire at the source, making the wrapper redundant.

### Rejected / prior approaches

- **Tried and reverted once:** PR #242 commit `7c123ce` landed this collapse as part of `surgical/v-llqd-execute-cards`, reverted in `1378177` because v-llqd was framed as a surgical extraction and listed `ExecutionHistoricalGuard` as a named deliverable. The reverted code is referenceable from git history as the starting point for this ticket's implementation — the design is known-working (445/445 tests passed). Not worth re-deriving from scratch.
- **Leave as-is with the wrapper:** current state. Works, but adds a latent footgun each time a new card is authored (v-uowg query cards, v-zlov CRUD cards both incoming).

### Design sketch (informational — implementer decides)

- `ToolExecutionStep` gains `{ step: 'skipped' }` (no data payload). Existing `{ step: 'historical' }` absorbed or removed.
- `state` memo returns `historicalStep ?? { step: 'skipped' }` for the historical branch; eliminates the `isHistorical + hasOutput + !isError` sub-branch that currently produces `'historical'`.
- Auto-fire effect already gates on `!isHistorical`; no change needed there.
- Each card gets an early `if (state.step === 'skipped') return <SkippedStub label="..." />` at the top of the render body. Cards lose their `ExecutionHistoricalGuard` wrap around every return path.
- `SkippedStub` lives in `ExecuteCards/shared.tsx`; renders the same "X skipped (no saved data)" row the guard currently produces.

## Files

- `src/features/agent/hooks/useToolExecution.ts` — add `'skipped'` to `ToolExecutionStep`; simplify historical branch in `state` memo
- `src/features/agent/components/tools/ExecuteCards/shared.tsx` — add `SkippedStub` export; update `mapPhase` exhaustiveness
- `src/features/agent/components/tools/ExecuteCards/{Send,Swap,ContractCall,Lp}FlowCard.tsx` — drop the guard import + wrapping; add skipped early-return
- `src/features/agent/components/tools/ExecutionHistoricalGuard.tsx` — delete
- `src/features/agent/components/tools/BuildTxCard.tsx` — verify historical replay rendering unchanged; update the `'historical'` comment if enum is renamed
- Reference: commit `7c123ce` in `vultisig/vultiagent-app` has the full shape of this change; commit `1378177` is the revert that restored the wrapper pattern

## Acceptance Criteria

- [ ] `ExecutionHistoricalGuard.tsx` deleted
- [ ] `useToolExecution` emits `{ step: 'skipped' }` for historical tool calls with no persisted outcome
- [ ] All 4 execute cards render `SkippedStub` via inline branch, not via wrapper
- [ ] `BuildTxCard` historical replay still renders as a plain preview (no callout, no approval re-arm)
- [ ] No component imports `ExecutionHistoricalGuard` anywhere
- [ ] `mapPhase` handles the new step exhaustively (exhaustiveness `never` check passes)
- [ ] Agent test suite green (expect ~445 tests)
- [ ] Manual: reopen a chat with a completed execute_send — success card renders, no re-prompt
- [ ] Manual: kill app mid-approval then reopen — skipped stub renders, no re-prompt
- [ ] Lint + tsc clean

## Gotchas

- Enum rename risk: grep the whole repo for `state.step === 'historical'` and `'historical'` step matches before landing — any telemetry or analytics that compares on the string breaks silently.
- `BuildTxCard` also consumes `useToolExecution` — its render path uses `'data' in state`, not `state.step === 'historical'`, so structurally it should tolerate a rename, but manual-verify historical replay before merging.
- The auto-fire effect already short-circuits on `isHistorical`; don't add a second gate for `'skipped'` — single source of truth.
- Ticket v-llqd's acceptance criteria specified the wrapper as a deliverable. This ticket supersedes that choice — call it out in the PR description so reviewers don't re-cite v-llqd.
- If v-uowg (query cards) or v-zlov (CRUD cards) land first and follow the wrapper pattern, fold them into this migration instead of shipping the refactor in two waves.
