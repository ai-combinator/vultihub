---
id: v-hykj
status: open
deps: []
links: []
created: 2026-05-07T06:20:36Z
type: task
priority: 0
assignee: Jibles
---
# Model-visible tool result envelope

## Problem
Many tools need to return rich payloads for app rendering, signing, security scans, persistence, and debugging, but the model often only needs compact status, labels, next-action guidance, and stable references. Today raw or downstream-shaped tool results can be appended back into the LLM transcript, increasing token cost and making retries/final narration inefficient. Solved means the backend has a shared result-envelope pattern that preserves full tool outputs for app/backend consumers while sending the model a compact, capped, decision-shaped result.

## Background
### Findings
- Overarching context from the token-cost investigation: the agent should be made cheaper and more reliable by shaping the LLM's environment, not by piling on more prompt warnings. The model should see fewer tools, shorter instructions, smaller results, and only fields it can actually decide from the conversation.
- Core philosophy: hard invariants belong in code/tool contracts/validators/structured errors; the prompt should carry only concise behavior and routing guidance where model judgment is genuinely needed.
- Cost philosophy: optimize both normal-path spend and outlier protection. Normal paths were dominated by stable system/tool context and repeated calls; outliers can still come from raw tool/search result dumps.
- Tool-contract philosophy: split app/backend/signing payloads from LLM-visible content. The model does not need to reread full calldata, rich card payloads, provider internals, or raw JSON just to decide the next action.
- Recent usage probes showed normal paths are mostly dominated by system prompt/tool schema tokens, but historical outliers suggest raw tool/search outputs can still create expensive individual turns.
- `v-giou` identified output-side tool contracts as a distinct problem: tools can dump more data into the model than the next reasoning step needs.
- Current card/signing flows need full payloads for mobile UI and signing, so simply truncating or deleting tool results is unsafe.
- This work complements:
  - `v-glle`: deterministic/canned final responses after presentational tool success.
  - `v-nncg`: component-aware token reporting, including `tool_result_tokens`.
  - `v-blbw`: structured errors and terminal retry policy.
  - `v-skxz`: escape-hatch send absorption, which can reuse the envelope pattern for raw signing payloads.

## Current Thinking
Introduce a shared backend/tool-result boundary:

```text
Full result / sidecar:
  rich payload for app cards, signing, security scans, persistence, debugging

LLM-visible result:
  compact status, labels, chosen/available options, next-action guidance, refs/ids
```

Ideal shape, names negotiable:

```text
ToolExecutionResult {
  raw_for_client
  raw_for_persistence_or_ref
  llm_content
  summary
  truncated
  raw_chars
  llm_chars
  policy
}
```

The LLM transcript should receive `llm_content`, not the full raw result, wherever possible. The app/signing/security paths must continue to receive the complete data they require.

Suggested phases:

1. Define the common envelope/result policy and where it is applied before `ai.ToolMessage` is appended.
2. Apply to one tx/card-producing tool and one read/search-style tool as proof of shape.
3. Add caps/tests/metrics proving full app payload survives while model-visible content is bounded.

## Constraints
- Do not truncate or mutate the payload before mobile/signing/security consumers receive it.
- Do not store raw prompts or sensitive payloads in usage telemetry by default.
- Preserve enough evidence for the model to answer truthfully and avoid fabrication.
- Preserve stable references so follow-up calls can refer to a selected result without replaying raw payloads.
- Result caps must be deterministic: defined ordering, returned count, total count, and continuation/ref behavior.

## Assumptions
- Some tools can adopt this pattern via backend wrapping before tool implementations are changed.
- Long-term, tools may return structured metadata that lets the backend choose the correct LLM-visible projection.
- `v-nncg` metrics can verify reductions in `tool_result_tokens` once this lands.

## Non-goals
- Do not define every per-tool summarizer in this ticket.
- Do not solve Polymarket/search output policy here beyond proving the infrastructure can support it.
- Do not skip final LLM calls after card success; that belongs to `v-glle`.

## Open Questions
- Should the envelope be owned in agent-backend, mcp-ts tool outputs, or both?
- What default cap should apply to unknown raw results?
- How should full raw payloads be referenced from compact LLM content: ids, hashes, sidecar metadata, persisted tool-call rows?
- Which tx/card tool and which read/search tool should be the first proof cases?
