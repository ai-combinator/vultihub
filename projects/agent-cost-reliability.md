# Agent Cost and Reliability Mission

## Mission

Make the Vultiagent wallet agent materially cheaper, easier to reason about, and more reliable under normal user behavior.

The immediate trigger was OpenRouter dashboard spend that looked too high for normal Gemini Flash usage. Local probes showed the dominant normal-path cost is not completion tokens; it is repeated prompt-side context: a large stable system prompt, broad tool schemas, and extra loop iterations. Historical evidence also suggests raw tool/search result payloads can create expensive outlier turns.

This project is the coordinated effort to reduce both everyday inference cost and outlier risk without weakening fund-safety behavior.

## Principles

- Shape the LLM's environment instead of adding more warnings.
- Keep the global prompt small; reveal workflow instructions only when relevant.
- Move hard invariants into code, tool contracts, validators, structured errors, and tests.
- The model should choose user intent: what action, which asset, which chain, how much, and who/where.
- Backend/tool code should resolve plumbing: wallet addresses, token metadata, decimals, gas, nonce, routes, provider details, and signing payloads.
- App/signing/security paths can keep rich payloads; the model should receive compact decision-shaped results.
- Successful presentational/card flows should not pay for another full LLM call when deterministic copy is enough.
- Retrying is good only when the next tool call can realistically fix the issue.
- Every optimization must be measurable by component: system prompt, tool schemas, tool results, assistant tool calls, AI call count, cache, and cost.

## Success Criteria

- Normal wallet flows use fewer AI calls per visible user turn.
- Stable system prompt tokens drop substantially in backend-e2e probes.
- Tool schema tokens drop on intent-scoped turns.
- Tool result tokens are bounded for search/list/read tools.
- Terminal failures stop cleanly without loop-breaker thrash.
- Prompt text becomes shorter and more behavior-bearing, with safety enforced outside prompt prose where possible.
- Production usage can be explained from persisted data without relying on screenshots or dashboard guesswork.

## Recommended Execution Order

1. **`v-nncg` — Persist token breakdown and add usage-audit CLI**

   Do first so every later optimization can prove what moved: system prompt, tool schemas, tool results, calls, cache, and cost.

2. **`v-glle` — Canned responses for presentational tool outputs**

   Fastest direct cost win. Removes whole final LLM calls from successful card flows.

3. **`v-blbw` — Classify tool errors for bounded LLM retry vs terminal response**

   Reduces expensive failure loops and gives structured error behavior that later prompt distillation can rely on.

4. **`v-hykj` — Model-visible tool result envelope**

   Infrastructure for separating rich app/signing payloads from compact LLM-visible content. Unblocks cleaner swap/search/send result shaping.

5. **`v-vpeb` — Progressive disclosure for prompts and tools**

   Start structural prompt shrink: relevant prompt packs, fail-closed tools, and no broad post-success widening.

6. **`v-scht` — Distill prompt text to behavior-bearing minimum**

   Run alongside or just after `v-vpeb`. As sections move into packs, rewrite them shorter instead of moving verbose prose unchanged.

7. **`v-mnan` — Clean up execute_swap LLM contract**

   High-value focused tool-contract cleanup. Benefits from `v-hykj` and `v-blbw`.

8. **`v-qzbp` — Compact search and market tool outputs**

   Outlier protection and cleaner choice sets. Benefits from `v-hykj`.

9. **`v-skxz` — Absorb transfer escape-hatch build tools into execute surface**

   Important, but heavier signing/parser work. Plan and slice by chain/token family.

10. **`v-tkhz` — Catalog remaining wallet and DeFi plumbing cleanup**

    Later discovery/backlog. Use after the high-value surfaces are handled.

11. **`v-tjwm` — Polymarket graduation gate**

    Keep disabled for v1. Revisit only when Polymarket is back in product scope.

## Ticket Map

- `v-nncg`: Measurement, production forensics, component-aware benchmark diffs.
- `v-glle`: Deterministic final copy for successful card/presentational tool outputs.
- `v-blbw`: Retryability taxonomy and terminal tool-error handling.
- `v-hykj`: Shared raw-result vs model-visible-result envelope.
- `v-vpeb`: Progressive prompt/tool disclosure by intent and phase.
- `v-scht`: Prompt distillation and safety migration out of prose.
- `v-mnan`: `execute_swap` user-decision vs plumbing cleanup.
- `v-qzbp`: Search/market/list output compaction.
- `v-skxz`: Transfer escape-hatch builder absorption into execute surface.
- `v-tkhz`: Remaining wallet/DeFi plumbing cleanup catalog.
- `v-tjwm`: Polymarket re-enable gate.

## Notes

The broad audit ticket `v-giou` was retired after being crystallized into the concrete follow-up tickets above.

