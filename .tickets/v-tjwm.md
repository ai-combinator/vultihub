---
id: v-tjwm
status: open
deps: []
links: []
created: 2026-05-07T05:30:52Z
type: task
priority: 0
assignee: Jibles
---
# Polymarket graduation gate: rework before enabling

## Problem

Polymarket is not release-critical and was cut from v1, but the repo still has Polymarket read/action tooling, app card work, and historical tickets that could make it look closer to shippable than it is. Recent token-cost investigation also flagged Polymarket/search-style tools as a plausible historical outlier: broad search/result payloads can become expensive if raw API output is fed back into the model, and local probing showed the current path is gated off rather than proven healthy.

Solved means Polymarket stays disabled/experimental for release, and there is a clear graduation gate before it can be exposed to normal agent users again. Re-enabling requires a serious rework of the tool surface, result shaping, order lifecycle, tests, and usage-budget guardrails.

## Background

Existing tickets establish the current status:

- Overarching context from the token-cost investigation: experimental/breadth features should not silently burden the core agent surface. Disabled features must stay out of default tools/prompts until they are reliable, bounded, and measured.
- Product philosophy: Polymarket is not release-critical. It should graduate only after tool contracts, result sizes, action lifecycle, and usage budgets are proven.
- Cost philosophy: broad search/market payloads and looping action flows can create outlier spend even if normal wallet paths are optimized.

- `v-xfzc` investigated repeated Polymarket order failures (`Invalid order payload`) and was killed because Polymarket was cut from v1 scope.
- `v-xbtm` was the follow-up fix ticket and was also killed because Polymarket was cut from v1.
- `v-sdvy` classifies Polymarket as experimental/non-core and notes action tools/stubs should move to experimental rather than being part of the core agent surface.
- `v-zhop` covers Polymarket read-tool consolidation, but it is not enough to make the product safe to expose. It mainly addresses read-tool naming/schema compatibility.
- Current backend behavior in local probing did not call Polymarket tools for a Polymarket search prompt because launch-surface gating kept Polymarket off.
- Direct MCP probing of `polymarket_search` failed locally with upstream fetch/TLS issues, so the path has not been validated end-to-end in the current environment.

## Current Thinking

This is not a launch blocker because Polymarket should remain off for v1. It is a post-v1 graduation ticket: do not re-enable Polymarket for normal users until this gate is satisfied.

The rework should cover both reliability and token cost:

1. **Tool exposure**
   - Keep Polymarket tools out of the default/core LLM-visible surface.
   - Experimental/dev exposure is fine, but it must be explicit via tier/flag.
   - Production re-enable should require a positive flag/tier decision and passing probes.

2. **Read/search tools**
   - Consolidate around a small read surface (`search`, `market`, `my positions/orders`) rather than many overlapping tools.
   - Cap and rank search results deterministically.
   - Never return raw third-party API payloads to the LLM.
   - Split rich UI/card payloads from the compact LLM-visible result.
   - Add token-budget assertions for broad searches like sports/team/event queries.

3. **Action/order lifecycle**
   - Root-cause and fix `Invalid order payload` before allowing bets.
   - Validate balance/fee/min-size math with real captured CLOB responses.
   - Ensure approvals, signing, order submission, cancellation, and post-order status all have deterministic states and user-visible recovery.
   - Do not let the model invent action types or fall back to generic contract-read tools for Polymarket flows.

4. **Prompt/tool contract**
   - Remove global Polymarket instructions from the core prompt; only include a Polymarket prompt pack when the feature is enabled and intent-matched.
   - Tool schemas should ask only for user decisions: market/outcome/side/amount/order preference. Addresses, balances, approvals, CLOB plumbing, salts, fees, and signing details should be resolved server-side.
   - Tool outputs should provide compact next-action guidance rather than raw CLOB/market data.

5. **Tests and probes**
   - Backend-e2e probes for read-only search, market detail, positions, and a disabled-state prompt.
   - If action flows are enabled: a sandbox or controlled small-real-money test plan for approval -> sign -> submit -> status.
   - Usage probes that record prompt tokens, tool-result tokens, tool-definition tokens, and number of model calls for representative Polymarket turns.
   - Regression test that flagged-off Polymarket does not expose Polymarket tools or global Polymarket prompt text.

## Acceptance Criteria

- [ ] Polymarket remains disabled/experimental for v1/default production agent surface.
- [ ] There is an explicit feature/tier gate for Polymarket LLM visibility.
- [ ] When disabled, Polymarket tools are not visible to the LLM and core system prompt contains no bulky Polymarket workflow section.
- [ ] Read tools return capped, ranked, compact LLM-visible results; rich raw/API/card payloads are kept out of model context unless strictly needed.
- [ ] Action tools cannot be enabled until `Invalid order payload` is root-caused and fixed with captured evidence.
- [ ] Polymarket action flow has deterministic state handling for approvals, signing, submit, failure, retry, cancel, and status.
- [ ] Usage probes demonstrate bounded token cost for representative search/detail/position/action turns.
- [ ] E2E probes pass for disabled-state behavior and any enabled experimental path.
- [ ] Follow-up implementation tickets are filed for each separately sized piece: read-result shaping, action-order fix, prompt-pack/gating, UI card payload split, and e2e/usage probes if they are not handled in one PR.

## Non-goals

- Do not make Polymarket a v1 release requirement.
- Do not re-enable Polymarket just because read consolidation lands.
- Do not solve broad progressive prompt disclosure here; that belongs to `v-vpeb`.
- Do not solve general tool-contract auditing here; that belongs to `v-giou`.
- Do not add new Polymarket product scope beyond making the existing intended flows reliable, cheap, and gated.

## Gotchas

- Read consolidation (`v-zhop`) is necessary but not sufficient. A smaller read surface does not prove order submission, result-size caps, or prompt gating.
- Hidden-by-flag is not the same as healthy. Before re-enable, test with the feature on and inspect actual token usage.
- Polymarket can create both reliability and cost failures: broad market searches can bloat context, while action flows can loop through approvals/signing/submission if errors are not model-readable.
- If no sandbox exists, define a controlled real-money smoke plan with hard spend limits before action tools graduate.
