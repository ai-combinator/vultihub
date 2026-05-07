---
id: v-mnan
status: open
deps: []
links: []
created: 2026-05-07T06:20:55Z
type: task
priority: 0
assignee: Jibles
---
# Clean up execute_swap LLM contract

## Problem
`execute_swap` should feel like `execute_send`: the model chooses what to swap, where, and how much, while backend/tool code supplies wallet addresses, token metadata, route plumbing, and signing payload details. Today swap still carries prompt/schema complexity around sender/destination, token overrides, route/provider details, and raw success payloads. Solved means `execute_swap` exposes only user-decision fields to the model, resolves derivable wallet/token/signing details server-side, and returns compact model-visible success/error output while preserving full app/signing payloads.

## Background
### Findings
- Overarching context from the token-cost investigation: the agent should be made cheaper and more reliable by shaping the LLM's environment, not by piling on more prompt warnings. The model should see fewer tools, shorter instructions, smaller results, and only fields it can actually decide from the conversation.
- Core philosophy: hard invariants belong in code/tool contracts/validators/structured errors; the prompt should carry only concise behavior and routing guidance where model judgment is genuinely needed.
- Tool-contract philosophy: the LLM should choose user intent (`what`, `where`, `how much`, and any true preference), while backend/tool code resolves wallet addresses, token metadata, gas/nonce, route plumbing, and signing payloads.
- Cost philosophy: successful action flows should avoid extra model calls and should not resend raw/rich payloads when compact facts are enough for final user-facing text.
- `v-giou` identified `execute_swap` as the top input-side contract cleanup candidate.
- `execute_send` already moved toward the desired pattern: user-decision fields visible to model; sender/gas/token plumbing resolved by backend/tool code.
- Swap prompt sections still carry cross-chain destination guidance and source/destination rules that may be compensating for tool contract gaps.
- Local loop/cost analysis showed swap-like successful card flows can pay large final-call prompt costs; compact outputs and `v-glle` should combine to reduce those calls.
- `v-blbw` will help distinguish retryable swap errors from terminal route/provider failures like `no_route`.
- `v-hykj` (model-visible result envelope) should provide the shared mechanism for compact swap success output.

## Current Thinking
Target model contract:

```text
LLM supplies:
  from_chain
  from_asset/from_symbol
  to_chain
  to_asset/to_symbol
  amount
  optional user-selected route/slippage preference if genuinely user-controlled

Backend/tool resolves:
  sender/from address
  default/self destination where safe
  chain-specific vault addresses
  token contracts/denoms/decimals
  quote/provider internals
  approval payloads
  calldata/signing envelopes
```

Suggested phases:

1. Audit current `execute_swap` schema and prompt sections for user-decision vs plumbing fields.
2. Backend-resolve sender and safe self-swap/self-bridge destination defaults where possible.
3. Hide/remove optional token plumbing fields from the LLM-visible schema where server resolution is reliable.
4. Convert route/provider/no-route errors into structured retriable/terminal envelopes compatible with `v-blbw`.
5. Compact swap success output using the model-visible result envelope pattern from `v-hykj`.

## Constraints
- Preserve fund-moving safety and approval flow semantics.
- Do not fabricate destinations. If destination cannot be derived safely and is a user decision, ask.
- Preserve full `ExecutePrepResult`, approval payloads, stepper config, and mobile signing/card behavior.
- Preserve route evidence needed for truthful user-facing quote facts: provider, source/destination, rate, minimum received, approval requirement.
- Do not treat general spot price claims as grounded by swap quotes unless the returned fields actually support them.

## Assumptions
- `execute_swap` remains the correct high-level tool; no new swap execute tool is needed.
- Some schema fields may need to remain accepted at runtime for backward compatibility but hidden from LLM-visible schema.
- `v-hykj` can land before or during compact success output work; earlier phases can proceed without it.

## Non-goals
- Do not absorb escape-hatch transfer builders here; that is `v-skxz`.
- Do not solve all search/token registry compaction here.
- Do not redesign all DeFi/Rujira swap flows here unless they are directly in `execute_swap`'s core path.

## Open Questions
- Which current swap args are genuinely user decisions vs compatibility/runtime-only fields?
- Which destinations can be safely backend-derived, and which require explicit user confirmation?
- Should runtime schema and LLM-visible schema diverge for backward compatibility?
- What exact terminal/retriable error codes should swap emit first?
- What before/after probes should prove fewer swap calls, smaller tool schemas, and no behavior regression?
