---
id: v-mfno
status: closed
deps: []
links: []
created: 2026-05-03T22:40:34Z
type: chore
priority: 0
assignee: Jibles
---
# Drop Rujira FIN tools from chat for v1

## Problem
Felix's 2026-05-01 debug-export shows `get_fin_swap_quote` was called 51× with 49 errors (96% failure). Conv `8ae9bf54` (`/home/sean/Downloads/2026-05-01-debug-export/2026-05-01-debug-export/conversations.jsonl`) has the agent making ~50 failed FIN tool calls trying to swap RUNE→RUJI, returning "I couldn't complete that in a reasonable number of steps" 3 times before giving up. RUJI/Rujira FIN coverage is "ideal not crucial" for release. "Solved" = FIN tools no longer reachable by the agent in v1; user requesting Rujira swaps gets a clean redirect to the Rujira tab.

## Background
### Findings
- Tool registrations: `mcp-ts/src/tools/rujira/index.ts:29-72` — `getFinSwapQuote`, `buildFinSwap`, `getFinPairs` registered with categories `['rujira','swap']`.
- Tool filter: `agent-backend/internal/service/agent/tool_filter*.go` — these are pulled into agent's visible toolset whenever conversation matches `swap` or `rujira` categories.
- `build_fin_swap` was called 0× in the dump despite being in scope. `get_fin_swap_quote`: 51× / 49 errors.
- Of the 49 errors: 33 "Unknown asset" (model invented strings like `x/ruji/thor.rune/x/ruji`), 9 "out of gas" (Rujira CosmWasm contract reverting `gasWanted: 3000000, gasUsed: 3000732` — upstream bug), 7 "no FIN contract found" (pair doesn't exist).
- Even with perfect prompting, upstream gas bug means RUNE↔RUJI quotes fail at the contract layer — RUJI is unreachable today regardless of agent quality.
- v-kmev's `tool_filter.llmFacingDropList` is the established mechanism for hiding tools from the LLM while keeping them registered for direct `mcpProvider.CallTool` paths.

### External Context
- Rujira's Liquidy team is building a multi-hop split-path router for RUJI Swap (Rujira 2026 roadmap — https://blog.thorchain.org/rujiras-2026-roadmap/). No public SDK timeline. May be the basis for a v1.1 fast-follow (separate ticket).

## Current Thinking
Two-part filter:
1. **Backend side**: drop `get_fin_swap_quote`, `build_fin_swap`, `get_fin_pairs` from the agent's visible toolset via `tool_filter.llmFacingDropList` (same mechanism v-kmev used for per-chain balance tools). Keep them registered server-side.
2. **Prompt side**: when user asks for RUJI / FIN / Rujira-DeFi swap, route to a "Rujira swaps live in the Rujira tab for now — want to get there?" response.

## Constraints
- Filter scoped to agent's view only — frontend / scheduler / other call paths must continue to reach these tools via `mcpProvider.CallTool`.
- Don't lose tool registrations in `mcp-ts` — the v1.1 fast-follow ticket needs them.

## Non-goals
- Building a FIN aggregator. Separate v1.1 fast-follow ticket.
- Fixing the upstream Rujira contract gas bug. Not ours.
- Re-enabling `build_fin_swap` for power users.

## Open Questions
- Exact filter mechanism — drop list vs category strip — pick whichever matches v-kmev precedent.
- Copy for the redirect: tone consistency may need product input.

## Notes

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/agent-backend#268 (open)
