---
id: va-lsys
status: open
deps: []
links: []
created: 2026-04-29T02:06:47Z
type: feature
priority: 2
assignee: Jibles
---
# Polish ExecuteCards stepper: spinner, checkmark, dotted connectors

## Objective

The execute_* flow cards (Send / Swap / ContractCall) currently render their stepper as four flat colored dots that differ only by fill color — IN_PROGRESS has no motion, COMPLETE has no checkmark, NOT_STARTED is just a faded dot. Users can't tell at a glance which step is active or finished. Closed PR #164 had a much richer treatment (real `ActivityIndicator` spinner, check-in-circle SVGs, dashed-ring for not-started, dotted connectors between steps, mono uppercase labels) that didn't survive the split into surgical PRs. Restore the high-value visual elements while staying inside the architectural constraints that helped close #164.

## Context & Findings

- Plumbing is already correct. `StepperRow` in `src/features/agent/components/tools/ExecuteCards/shared.tsx:77` derives `StepStatus` (NOT_STARTED / IN_PROGRESS / COMPLETE / FAILED) per step from `txFlowReducer`'s phase via `getSigningStepStatus`. **No reducer or state-machine changes needed.**
- The visual gap is purely in the `StepDot` component (`shared.tsx:125`). All four states render as a 10px filled `View` circle, only `backgroundColor` and `opacity` change.
- PR #164 (`refactor/v-pxuw-tool-consolidation`, closed — superseded by surgical/v-* PRs) had a richer `StepIcon` using `react-native-svg` + `ActivityIndicator`. The original implementation can be retrieved with `git fetch origin refactor/v-pxuw-tool-consolidation && git show FETCH_HEAD:src/components/ui/ActionCard.tsx` — relevant SVG shapes are at lines 411–457 of that file.
- Per-status visuals from PR #164 worth porting:
  - NOT_STARTED → 18×18 SVG circle, transparent fill, dashed grey 1.5px ring
  - IN_PROGRESS → real `<ActivityIndicator size="small" color={purple} />` (visible motion)
  - COMPLETE → 18×18 SVG circle, 15% opacity green halo fill, green checkmark stroke
  - FAILED → 18×18 SVG circle, 15% opacity red halo fill, red X stroke
- Plus dotted SVG connectors between adjacent steps and uppercase mono labels (`Fonts.mono.regular`, letterSpacing 0.8) to match the type system used by `ActionCard.Status` ("APPROVED" / "REJECTED").
- IN_PROGRESS color choice: PR #164 used `Colors.accent.purple`. Today's blue clashes with the blue `Callout` shown during `awaiting_confirm` — two competing blues at the consent moment.

### Why PR #164 was closed (so we don't repeat)

Closed with: "superseded by the smaller surgical/v-* PRs (hipx, llqd, uowg)." Reviewer pushback was on architecture and signing-flow safety, **not** on the stepper visuals:

- `ActionCard` lived at `src/components/ui/ActionCard.tsx` (shared) but imported from `features/agent/lib/txFlowReducer` — shared primitive depending on a feature module.
- The Stepper auto-collapsed into Status at terminal phases, deepening the same coupling.
- Other concerns (B1–B4 signing-flow bugs, scope drift) are unrelated to this ticket.

To avoid re-introducing those concerns: **keep the new visuals inside `features/agent/components/tools/ExecuteCards/shared.tsx`**, where the dependency direction is already correct. Do **not** modify `src/components/ui/ActionCard.tsx`. Do **not** port the "Stepper collapses into Status" coupling — leave the current pattern of explicit `<StepperRow>` + (separately) `<ActionCard.Status>` after a divider in each card.

### Rejected / deferred from PR #164

- `ActionCard.Pill` subcomponent (small inline chip for "ERC-20 approval required" etc.) — nice-to-have, not blocking the visual improvement.
- `'incomplete'` Status kind (`NOT SUBMITTED` for replayed-but-unfinished cards) — only matters once historical replay is fully fleshed out.
- Stepper-collapses-into-Status — clean UX but reopens the architectural smell. Skip.
- Compound `Row` / `Label` / `Value` API — orthogonal to the stepper visuals; defer.

## Files

- `src/features/agent/components/tools/ExecuteCards/shared.tsx` — primary change site. Replace `StepDot` (rename to `StepIcon` if useful) with SVG-based per-status visuals. Add a `DottedConnector` helper between adjacent steps. Adjust `StepperRow` label styling.
- `src/components/ui/ActionCard.tsx` — **do not modify.** Keep the architectural separation that the surgical-PR split established.
- Reference (read-only): retrieve PR #164's `ActionCard.tsx` via `git show FETCH_HEAD:src/components/ui/ActionCard.tsx` after fetching `refactor/v-pxuw-tool-consolidation`. SVG shapes at lines 411–457; `DottedConnector` at lines 393–409.
- `src/styles/shared` — confirm `Colors.accent.purple` and `Fonts.mono.regular` exist (both already used elsewhere in this codebase).

## Acceptance Criteria

- [ ] IN_PROGRESS step renders a visible spinner (motion, not a static dot)
- [ ] COMPLETE step renders a circle with a checkmark inside
- [ ] FAILED step renders a circle with an X inside
- [ ] NOT_STARTED step renders a dashed/hollow ring rather than a faded filled dot
- [ ] Dotted connectors render between adjacent steps in the row
- [ ] Step labels are mono, uppercase, with letter-spacing — matching the `ActionCard.Status` type style
- [ ] IN_PROGRESS uses `Colors.accent.purple` (or another non-blue accent) so it doesn't clash with the blue `awaiting_confirm` Callout
- [ ] `src/components/ui/ActionCard.tsx` is unchanged — no new imports from `features/agent/*` introduced into the shared `ui/` layer
- [ ] Existing tests in `ExecuteCards/__tests__/SwapFlowCard.test.tsx`, `SendFlowCard.test.tsx`, `ContractCallCard.test.tsx` still pass
- [ ] `npx tsc --noEmit` clean and `npx biome check src/` clean
- [ ] Manual: verify swap flow shows progressive stepper states (quote → sign → broadcast) with spinner advancing and dots filling in green as each completes

## Gotchas

- Do not move `ActionCard` into the feature module or have it import from `features/agent/lib/*` — that's the architectural concern that helped close #164.
- Do not bundle the "stepper collapses into Status at terminal phases" refactor here. Keep the explicit `<StepperRow>` + `<ActionCard.Status>` composition each card already uses.
- Today's fallback step list in `shared.tsx:22` is `['sign', 'broadcast']` (2 dots). PR #164 used a 3-dot fallback (`quote / sign / broadcast`) so the pre-prep state has more visual presence — consider matching while you're in the file.
- Icon size bump (10 → 18px) adds ~8px row height; sanity-check on small-screen layouts.
- `react-native-svg` is already a dependency (used by other cards) — confirm import path matches existing usage.
- The "Spin the first dot during prep fetch" override in `shared.tsx:97-99` should still apply with the new `StepIcon` — keep that branch so the card has motion before any real step is in flight.
