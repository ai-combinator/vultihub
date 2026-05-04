---
id: v-xemk
status: open
deps: []
links: []
created: 2026-05-03T22:45:16Z
type: feature
priority: 1
assignee: Jibles
---
# Rujira FIN aggregator (fast-follow after v1)

## Problem
Rujira FIN is dropped from chat for v1 (separate release-critical ticket). The fast-follow goal is reinstating Rujira FIN swap support behind `execute_swap` so users can do RUNE↔USDC, RUNE↔BTC, RUNE↔RUJI, etc., without leaving chat.

The current MCP tools (`get_fin_swap_quote`, `build_fin_swap`, `get_fin_pairs`) cover single-pair single-hop quoting and building. Multi-hop routing, slippage budgeting across legs, and partial-fill recovery are NOT covered — they were being asked of the LLM, which doesn't reliably perform graph search or transactional state management. That gap is what made Felix's `8ae9bf54` conversation spiral.

## Background
### Findings
- Rujira FIN is a CosmWasm orderbook on THORChain — one contract per pair. The "graph" is essentially a star with USDC at the center plus a few side edges (e.g. `thor.ruji/thor.rune`, `eth-eth/btc-btc`).
- 36 pairs available (per `get_fin_pairs` output in Felix's dump). Two-hop routing covers virtually every reachable pair via USDC.
- Existing primitives in `mcp-ts/packages/rujira/dist/assets/index.d.ts`: `SwapRouter` class with `quote(from, to, amount)`, `quoteThorchainLP`, `quoteRujiraFIN`. Single-hop only — no multi-hop, no graph traversal.
- Asset-format gotcha: each asset has 3 forms (`l1`, `thorchain`, `fin`). Example RUJI: `l1: "RUJI"`, `thorchain: "THOR.RUJI"`, `fin: "x/ruji"`. FIN pairs use the `fin` form — model couldn't keep these straight in v1.
- Upstream Rujira contract is currently reverting RUNE↔RUJI smart-query with `gasWanted: 3000000, gasUsed: 3000732`. Backend bug — fixable only by Rujira. Until they fix it, even a perfect aggregator can't quote RUJI specifically.

### External Context
- Rujira's Liquidy team is building a multi-hop split-path router for RUJI Swap (Rujira 2026 roadmap, https://blog.thorchain.org/rujiras-2026-roadmap/). No public SDK/API timeline. *"The Liquidy team has been working on a new multi-hop, split-path router that we will integrate into the RUJI Swap frontend."*
- SwapKit (https://swapkit.dev) covers THORChain ecosystem swaps; provider list does not include Rujira FIN.
- Rango Exchange supports CosmWasm and THORChain but no Rujira FIN integration listed.

## Current Thinking
Two paths, decide before implementing:

**Option A — wait + integrate Liquidy's router**: Track Liquidy roadmap. When/if they ship the multi-hop router as an SDK or REST endpoint, integrate it into the Rujira slot of our chain-family pipeline. Lowest engineering cost; depends on Rujira's timeline. Probably also depends on relationship play — ask them to expose it.

**Option B — build internal aggregator**: ~500 LOC TypeScript in `mcp-ts/packages/rujira` — pair-graph builder, two-hop router (Dijkstra-equivalent over the star + side-edges), leg-by-leg quote walker, normalized output matching `ExecutePrepResult`. Slippage budget per-leg, partial-fill aware. ~1 week of focused work.

Either way the result hooks into `execute_swap` via the chain-family pipeline so the user-facing flow is unified — agent calls `execute_swap`, server routes Rujira FIN intents through the aggregator, returns a (possibly multi-step) Cosmos `wasm_execute` envelope, app dispatches via `parseServerTx`. No `get_fin_*` / `build_fin_*` tools re-exposed to the LLM.

## Constraints
- The chat agent's tool surface unchanged from v1 — `execute_swap` only. No re-introduction of FIN-specific tools.
- Output envelope must match `ExecutePrepResult` so the existing app-side cosmos-msg parser handles it (PR #320).
- Don't start until v1 has shipped.

## Assumptions
- Once Rujira fixes the upstream contract gas bug, RUNE↔RUJI quotes work. If they don't, aggregator can route around RUJI but headline use case stays broken regardless.
- Liquidy's router (if it ships as an SDK) covers the multi-hop split-path logic we'd otherwise have to write ourselves.

## Non-goals
- THORChain native swaps (BTC↔ETH↔RUNE etc.) — those go through `execute_swap`'s existing native path.
- LP / range positions / orderbook limit orders / GHOST lending — separate Rujira products with their own tickets.
- Direct user access to FIN orderbook tools from chat — keep them server-only.

## Dead Ends
- Letting the LLM aggregate at the prompt level. Tried in v1 (`get_fin_swap_quote` + `get_fin_pairs` exposed to model). 96% failure rate; the loop spiraled on RUJI. LLMs don't reliably do graph search + multi-step transactional state.

## Open Questions
- A vs B — depends on Liquidy timeline. Track it; pick based on whichever ships sooner.
- If A: integration shape — REST API, embeddable SDK, or smart contract calls? Affects effort.
- If B: do we vendor Rujira's `SwapRouter` from `@vultisig/rujira-sdk` (already in `mcp-ts/packages/rujira`) as the per-pool quote primitive, or call it via npm dep?
- Multi-hop signing UX — single approval covering the multi-leg bundle, or one approval per leg? Affects frontend stepper.
