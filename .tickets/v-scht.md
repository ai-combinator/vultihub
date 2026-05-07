---
id: v-scht
status: open
deps: []
links: []
created: 2026-05-07T05:56:06Z
type: task
priority: 0
assignee: Jibles
---
# Distill prompt text to behavior-bearing minimum

## Problem
Progressive disclosure will stop the model from seeing irrelevant prompt sections, but the prompt sections themselves are still too verbose, defensive, and incident-shaped. A lot of prompt text reads like historical regression notes or repeated guardrail prose rather than concise behavior instructions. Solved means the global prompt and any prompt packs are rewritten to the shortest behavior-bearing form, with hard invariants moved into tool contracts, validators, structured errors, or tests wherever possible.

## Background
### Findings
- Overarching context from the token-cost investigation: the agent should be made cheaper and more reliable by shaping the LLM's environment, not by piling on more prompt warnings. Progressive disclosure reduces which sections are visible; distillation reduces the size and noise of the sections that remain.
- Prompt philosophy: hard invariants belong in code/tool contracts/validators/structured errors/tests. Prompt text should carry concise behavior/routing instructions only where model judgment is genuinely needed.
- Safety philosophy: this is not about deleting safety. It is about moving safety to durable enforcement layers and keeping prompt instructions short, direct, and behavior-bearing.
- Existing prompt work is tracked under `v-vpeb` (`Progressive disclosure for prompts and tools`), which focuses on showing only relevant prompt/tool packs by intent/phase.
- Progressive disclosure does not automatically make each prompt section concise. A smaller selected pack can still be bloated if it carries long warnings, historical examples, repeated `DO NOT` rules, and tool-field coaching.
- Recent token-cost audit showed normal backend-e2e prompt-side spend dominated by stable system prompt and tool definitions, not tool results. Structural disclosure helps, but text distillation is still needed to reduce the baseline inside each selected prompt/pack.
- Current prompt text includes many sections that likely encode real regressions but may be better enforced elsewhere: validators, tool schemas/descriptions, deterministic MCP errors, backend terminal responses, or behavior-level tests.
- This complements related tickets:
  - `v-vpeb`: progressively disclose prompt/tool packs by intent/phase.
  - `v-giou`: audit tool contracts so model-facing fields/results stop carrying implementation plumbing.
  - `v-blbw`: classify errors as retriable vs terminal so prompt prose does not need to teach every recovery case.
  - `v-nncg`: component-aware usage/perf diffs to prove prompt shrink actually reduced `system_stable`/prompt tokens.

## Current Thinking
This is not about deleting safety. It is about moving safety to the right layer.

Target principle:

```text
Move hard invariants into code.
Keep prompt instructions only where model judgment is actually needed.
Write those instructions as short positive behavior/routing rules.
```

Examples of the desired transformation:

```text
Before:
Long warning explaining past CW20 hallucinations, USTC/LUNC contract-address mistakes,
and repeated NEVER/DO NOT guidance.

After:
Prompt: CW20 is only for contract tokens; native Cosmos/Terra sends use execute_send.
Tool/code: build_cw20_transfer rejects native denoms with a structured terminal error.
Tests: USTC/LUNC prompts route to execute_send, not CW20.
```

```text
Before:
Many paragraphs of execute_send/execute_swap field-level coaching.

After:
Prompt: Use execute_send for transfers and execute_swap for swaps. If required user decisions are missing, ask one clarifying question.
Tool contracts/errors: validate/resolve sender, gas, nonce, token metadata, destination, route availability.
```

Suggested audit classification for each prompt paragraph/section:

```text
keep_global          -> core identity/style/safety/routing needed every turn
move_to_pack         -> only relevant for a specific intent/protocol/phase
replace_with_tool    -> belongs in tool description/schema/result contract
replace_with_code    -> should be validator/resolver/error taxonomy, not prompt prose
replace_with_test    -> historical regression should be pinned as behavior test
delete_obsolete      -> no longer true or no longer needed after execute_* migration
```

The output should be measurable. Every PR in this line should report before/after token counts for:

```text
global prompt
selected prompt packs
system_stable tokens in backend-e2e
probe pass/fail count
```

The first implementation can start before full `v-vpeb` pack architecture if it safely distills obvious global text and behavior tests pass. Long-term, this ticket should make both the global prompt and packs concise.

## Constraints
- Do not remove safety-critical behavior unless it is enforced by code/tool contract/tests in the same PR or already covered elsewhere.
- Do not rely on exact prompt wording tests as the main safety net. Prefer behavior/e2e assertions when rewriting prose.
- Preserve fund-moving safety: no fabricated transactions, no auto-signing, no success/broadcast claims before real signing/broadcast evidence, and clear confirmation before user approval.
- Keep prompt text direct and positive where possible. Prefer "Use X for Y" over repeated negative/caps-lock guidance.
- Any distillation PR must include before/after token measurements and run the relevant backend-e2e/curl-replay probes.
- Coordinate with `v-vpeb`: if a section is being moved into a pack, distill it as part of the move rather than preserving old verbose prose.

## Assumptions
- Some prompt bloat exists because tool/code layers are missing structured enforcement. Distillation may uncover follow-up tickets rather than deleting every paragraph immediately.
- The first pass can likely remove or shorten duplicated/obsolete text without waiting for every tool-contract cleanup ticket.
- Component-aware perf diffs from `v-nncg` will be available or can be approximated locally while this work starts.

## Non-goals
- Do not build the prompt-pack routing architecture here; that belongs to `v-vpeb`.
- Do not redesign MCP tool schemas broadly here; that belongs to `v-giou` and follow-up implementation tickets.
- Do not delete real production regression protections without replacement.
- Do not attempt a one-shot rewrite of the entire prompt if safer incremental slices are available.

## Dead Ends
- Treating progressive disclosure as sufficient was rejected. It reduces irrelevant sections but does not make the remaining selected text concise.
- Treating prompt distillation as pure copy editing was rejected. Many paragraphs encode behavior or safety assumptions and need code/test/tool-contract replacement, not just shorter wording.
- Big-bang prompt deletion was rejected because the current prompt contains real regression history and fund-moving safety constraints.

## Open Questions
- What is the target token budget for the global prompt before packs? 3k? 5k? Another threshold based on backend-e2e results?
- Which prompt sections can be safely distilled first without waiting for code changes?
- Which exact prompt wording tests need conversion to behavior-level tests before substantial rewrites?
- Should prompt paragraphs carry ownership metadata/classification comments in code, or is a one-time audit document enough?
- Should this work land as one audit + multiple implementation tickets, or as incremental PRs against `prompt.go` with token-budget gates?
