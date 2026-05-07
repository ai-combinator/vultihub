---
id: v-tkhz
status: open
deps: []
links: []
created: 2026-05-07T06:21:37Z
type: task
priority: 0
assignee: Jibles
---
# Catalog remaining wallet and DeFi plumbing cleanup

## Problem
Beyond the highest-priority swap/send/search surfaces, there are likely remaining wallet, contract, staking, yield, and DeFi tools that still ask the model to copy addresses, gas, base units, token metadata, or opaque protocol payloads. Some of those are worth cleaning up, but bundling all of them into the first tool-contract cleanup pass would create a vague mega-project. Solved means the leftover plumbing smells are cataloged, ranked, and converted into scoped implementation tickets only where the value justifies the work.

## Background
### Findings
- Overarching context from the token-cost investigation: the agent should be made cheaper and more reliable by shaping the LLM's environment, not by piling on more prompt warnings. The model should see fewer tools, shorter instructions, smaller results, and only fields it can actually decide from the conversation.
- Core philosophy: hard invariants belong in code/tool contracts/validators/structured errors; prompt text should shrink as tool contracts and deterministic backend behavior improve.
- Prioritization philosophy: do not clean up every smell for its own sake. Focus on high-frequency, LLM-visible, cost/reliability-relevant tools first; leave obscure or experimental surfaces alone unless they block product goals.
- `v-giou` identified a broad class of remaining tool-contract smells after the focused candidates:
  - `execute_contract_call` may still ask for nonce/gas fields that backend could fetch.
  - Cosmos staking/read-write flows may expose address or base-unit plumbing.
  - Yield/read/action tools may ask for wallet address copying.
  - Rujira signing/build tools likely contain protocol-specific plumbing, but are larger and more product-specific.
- Higher-priority concrete workstreams now exist:
  - `v-mnan`: clean up `execute_swap` LLM contract.
  - `v-skxz`: absorb transfer escape-hatch build tools into execute surface.
  - `v-hykj`: model-visible tool result envelope.
  - `v-qzbp`: search/market output compaction.
- This ticket is deliberately lower-priority discovery/backlog hygiene for leftovers.

## Current Thinking
Use the same user-decision vs plumbing lens, but keep this as a cataloging pass unless a tool clearly warrants immediate implementation.

Audit each candidate with:

```text
Tool / family
Current LLM-visible fields
Which fields are user decisions vs plumbing
Where plumbing could be resolved from
Current output size/shape risk
Prompt sections that would become deletable
Usage frequency / product importance
Risk if changed
Recommendation: no-op, implementation ticket, or fold into existing workstream
```

Likely candidates:

- `execute_contract_call`: backend-fill `from`, nonce, gas, EIP-1559 fields where safe; preserve raw calldata escape hatch.
- Cosmos staking and related account tools: address/base-unit/account plumbing.
- Yield/DeFi read and action tools: wallet address injection and compact outputs.
- Rujira signing/build tools: likely important but should be handled as a separate protocol-specific execute-surface project if pursued.
- Any remaining card-producing tools whose raw payloads are still sent back to the model.

## Constraints
- Do not turn this into a giant implementation PR.
- Do not absorb Rujira or other protocol-specific DeFi flows into generic send/swap without a dedicated product/architecture decision.
- Preserve raw calldata/advanced contract-call escape hatches for users who explicitly need them.
- Use telemetry/usage frequency to avoid spending time on obscure tools with no current product value.

## Assumptions
- This ticket should run after or alongside the higher-priority tool-contract tickets, because those will likely eliminate many obvious examples.
- Some tools may be best left alone if they are internal, experimental, disabled, or rarely LLM-visible.

## Non-goals
- Do not redesign `execute_swap`, escape-hatch sends, search/market output, or the shared result envelope here; those have dedicated tickets.
- Do not create implementation tickets for every smell automatically. Rank by value and risk.
- Do not make experimental DeFi/Rujira work release-critical.

## Open Questions
- Which leftover tools are actually visible to the LLM in production/default launch-surface state?
- Which tools are high-frequency enough to justify cleanup?
- Should `execute_contract_call` gas/nonce fetching be a focused follow-up ticket or folded elsewhere?
- Should Rujira get its own `execute_rujira_*` surface, or stay experimental/protocol-specific for now?
