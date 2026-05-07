---
id: v-nncg
status: open
deps: []
links: []
created: 2026-05-07T05:04:44Z
type: task
priority: 0
assignee: Jibles
---
# Persist token breakdown and add usage-audit CLI

## Problem
Production usage data is not detailed enough to explain OpenRouter spend incidents from the database alone. The backend now exposes useful `prompt_breakdown` fields in responses/SSE and local harnesses, but those fields are not persisted per AI call and there is no canonical report for “why did this user spend $X on this day?” Local benchmark diffs also need to show component-level before/after deltas so prompt/tool/loop PRs can prove what they improved. Solved means the backend stores enough per-call token/cache/provider/component data, ships a CLI/report that can diagnose user/day spend by conversation/message/loop/model/provider/component, and updates perf/benchmark diff output to report component-aware deltas.

## Background
### Findings
- Overarching context from the token-cost investigation: we need to make cost visible at the same granularity as the architecture levers we are changing. Otherwise we cannot prove whether prompt shrink, tool filtering, loop reduction, result compaction, or cache behavior actually helped.
- Cost philosophy: OpenRouter/provider token spend is separate from user-facing credits. The system needs to diagnose real inference cost by user, conversation, visible message, AI call, loop iteration, model/provider, and prompt component.
- Measurement philosophy: every optimization ticket should be able to prove its effect with before/after component deltas, not just total tokens.
- Recent local audit added response/SSE prompt attribution fields such as `usage.prompt_breakdown` and `usage.prompt_breakdown_by_call`.
- Current debug/harness surfaces can show component splits locally, but production forensics still rely on partial DB fields, logs, PostHog/SSE receipts, or OpenRouter dashboard totals.
- Existing `agent_token_usage` persists prompt/completion/total tokens plus `system_prompt_tokens`, `tool_definition_tokens`, latency, model, call type, loop iteration, and estimated cost. It does not persist enough detail for cache/provider/component analysis.
- Missing or weakly persisted production-forensics fields discussed:
  - `cached_tokens`
  - `cache_write_tokens`
  - `provider`
  - requested model vs reported/routed model where available
  - `system_stable_tokens`
  - `system_dynamic_tokens`
  - `tool_definition_tokens` by actual selected tool set
  - `user_message_tokens`
  - `assistant_message_tokens`
  - `assistant_tool_call_tokens`
  - `tool_result_tokens`
  - `reasoning_replay_tokens`
  - `estimated_prompt_tokens`
  - `prompt_breakdown JSONB` for future fields
- Existing admin/dashboard aggregates are useful for totals and charts, but they do not answer the incident questions cleanly: which visible message caused spend, how many model loops happened, which component dominated, whether cache worked, which provider/model OpenRouter used, and whether a tool result caused the spike.
- The OpenRouter dashboard was the source of the reported `$5/day` observation, so internal tooling needs to reconcile with provider-facing token/cost reality rather than only user-facing credits.
- Local benchmark scripts already capture some usage breakdown fields, but before/after reports are not component-aware enough. For optimization PRs we want diffs that say whether `system_stable`, `tool_definition`, `tool_result`, `assistant_tool_call`, cache, cost, and AI-call-count actually moved.

## Current Thinking
These should be delivered as one ticket, not two, because persistence and the report validate each other. The CLI is the practical acceptance test for the DB migration: if the report cannot explain a user/day spend from persisted data, the storage work is incomplete.

Ideal shape:

```text
1. Persist complete per-AI-call usage/component fields.
2. Add a usage-audit CLI that reads those persisted rows.
3. Use the CLI against local backend-e2e/curl-replay runs to prove the stored fields are sufficient.
4. Add component-aware perf/benchmark diff output using the same breakdown vocabulary.
5. Use the same CLI for production incidents before building dashboard UI.
```

CLI invocation shape:

```bash
usage-audit --public-key <pk> --from <iso/date> --to <iso/date>
usage-audit --conversation-id <id>
usage-audit --message-id <id>
```

Minimum useful report shape:

```text
Totals:
- prompt tokens
- completion tokens
- estimated cost
- model/provider split
- cached/cache-write tokens
- AI call count

Component split:
- system stable
- system dynamic
- tool schemas
- user/history/assistant text
- assistant tool calls
- tool results
- reasoning replay
- completion/thinking

Top offenders:
- top conversations
- top visible messages
- top AI calls
- top loop-heavy messages
- top tool-result-heavy messages

Per-message trace:
- conversation_id
- message_id
- loop_iteration rows
- call_type
- requested/reported model/provider
- prompt/completion/cached tokens
- component breakdown
```

Local benchmark/perf diff output should include before/after component deltas such as:

```text
system stable:       493k -> 180k (-63%)
system dynamic:       42k ->  38k (-10%)
tool schemas:        159k -> 100k (-37%)
tool results:        4.8k -> 2.0k (-58%)
assistant tool calls:1.1k -> 0.7k (-36%)
AI calls:              20 ->   12 (-40%)
cached tokens:        60k -> 120k (+100%)
estimated cost:      $X  ->  $Y
```

This makes cost-reduction tickets measurable:

- `v-vpeb` should primarily reduce `system_stable` and tool schemas by intent/phase.
- `v-glle` should reduce AI call count and final-call tool schemas.
- tool-result truncation/result-shaping work should reduce `tool_result_tokens`.
- escape-hatch absorption work should reduce send-turn tool schemas and prompt exceptions.

The CLI should distinguish token/cost accounting from user-facing credits. Tool-credit accounting is a separate user billing layer and should not be conflated with OpenRouter inference cost.

Dashboard integration is deliberately not first. A CLI is cheaper, faster, and lets us settle the exact report shape before committing UI.

## Constraints
- Do not store raw prompts, raw tool results, private keys, signatures, or sensitive payloads by default.
- Persist per AI call, not only per visible user message, because loop iteration is central to cost diagnosis.
- Keep existing usage APIs and current `agent_token_usage` consumers working.
- Treat `prompt_breakdown` token estimates as heuristic component attribution; provider-reported prompt/completion totals remain source of truth for billed token counts.
- Must work for both streaming and non-streaming message paths.
- Perf/benchmark output should use the same component names as persisted usage where possible, so local diffs and production audits are comparable.

## Assumptions
- Adding a `prompt_breakdown JSONB` field plus a few scalar columns is acceptable, even if some values are heuristic.
- The existing SQLC/postgres migration workflow is the right place for the storage change.
- Provider/cost fields available from OpenRouter may vary; the CLI should degrade gracefully when exact provider cost is unavailable.
- Existing local probes/curl-replay can be used to generate known rows for CLI validation.

## Non-goals
- Do not build an admin dashboard UI in this ticket.
- Do not implement anomaly alerts in this ticket.
- Do not change model selection, prompt size, or loop behavior in this ticket.
- Do not solve user-facing credits/tool billing reconciliation beyond clearly separating token cost from credits.
- Do not make benchmark thresholds overly strict in this ticket. Start with reporting/diffing; budget gates can be added once baseline variance is understood.

## Dead Ends
- Relying on the existing dashboard alone was rejected. It can show that spend happened, but not why at the level needed to prioritize prompt/tool/loop fixes.
- Creating only a DB migration without a report was rejected because it does not prove the stored fields answer the incident questions.
- Creating only a CLI on current fields was rejected as insufficient because current persistence lacks cache/provider/full component attribution.
- Keeping benchmark diffs at total-token level was rejected because it cannot prove which lever worked. If total tokens drop, reviewers still need to know whether it came from prompt shrink, fewer calls, tool-schema reduction, cache behavior, or smaller tool results.

## Open Questions
- Should component fields be mostly scalar columns, mostly `prompt_breakdown JSONB`, or both? Current thinking: scalar columns for common queries plus JSONB for future-proofing.
- What exact command/package location should own the CLI: `cmd/usage-audit`, `scripts/usage-audit`, or admin tooling?
- Can OpenRouter request/generation IDs or raw `usage.cost` be captured reliably from the AI client response path?
- Should the CLI estimate OpenRouter cost from stored pricing, display provider-reported cost, or both when available?
- Should prompt/tool/schema hashes be included in this ticket or deferred to a later fingerprinting ticket?
- Which existing local scripts should own component-aware diffing first: `scripts/perf/bench.mjs`, curl-replay reports, backend-e2e probe audit output, or a shared helper?
