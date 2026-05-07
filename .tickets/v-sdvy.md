---
id: v-sdvy
status: closed
deps: []
links: []
created: 2026-04-21T03:30:26Z
type: task
priority: 2
assignee: Jibles
---
# Goal-state consolidation: core/experimental tier split + scope pruning

## Problem Statement

The agent-backend + mcp-ts + vultiagent-app + vultisig-sdk stack has accumulated 105+ tools, 37 chains, and numerous exotic protocol integrations (Rujira, Polymarket, Tornado, Pump.fun, Uniswap V3 advanced, Yield.xyz, DeFiLlama, THORChain LP) — far beyond what a reliable, fast, cheap basics-first wallet agent needs. Shipping velocity suffers because every increment must coordinate across four repos carrying this baggage. "Solved" means core user flows (send, swap, portfolio, price, onboarding) run on a narrow end-to-end tool surface against popular chains, with experimental work safely isolated so a single teammate can keep iterating without adding agent/user-visible load.

## Research Findings

### Current State

**Tool inventory in mcp-ts (post-v-pxuw, 105 tools):**
- 5 execute_* end-to-end tools (`execute_send`, `execute_swap`, `execute_contract_call`, `execute_lp_add`, `execute_lp_remove`) — truly end-to-end, return `ExecutePrepResult` envelopes with natural-language input parsing and server-side token/decimal/gas/approval resolution
- 10 core utility tools (`set_vault_info`, `get_address`, `get_price`, `get_tx_status`, `search_token`, and 5 more)
- 28 building blocks (9 EVM helpers like `abi_encode`/`evm_call`/`evm_check_allowance`, 14 per-chain balance tools, 6 fee-rate tools)
- 6 custom-send escape hatches (SPL, XRP, TRC-20, TON Jetton, Sui native + token) — "retired behind gated categories" per `mcp-ts/src/tools/index.ts:109` comment
- 53 exotic integrations: Rujira (18), DeFiLlama + yield.xyz + THORChain LP (17), Polymarket (8), Tornado (8), Uniswap V3 advanced (3), Pump.fun (2)
- 4 plugin/verifier tools

**Agent-visible surface (intended post-cleanup, 11 tools):**
1. `execute_send` — end-to-end send on popular chains
2. `execute_swap` — end-to-end swap, internal approval handling
3. `get_balances` — unified (va-bqks subsumes `get_portfolio`); token/chain filters; `include_defi` flag
4. `get_price` — market price lookup
5. `get_address` — address + QR-renderable card payload
6. `search_token` — token resolution (symbol or contract)
7. `get_tx_status` — post-broadcast confirmation
8. `create_vault` — triggers app MPC keygen flow (card)
9. `import_vault` — triggers app vault import flow (card)
10. `add_chain` (`vault_chain` action="add") — enable chain in vault view
11. `add_token` (`vault_coin` action="add") — track specific token on chain

**Tool cards in vultiagent-app (`src/features/agent/lib/toolUIRegistry.ts`):** 26 mapped
- 4 execute cards (Send, Swap, ContractCall, Lp)
- 11 query cards (DeFi, Polymarket ×3 read-only, Uniswap ×3)
- 5 CRUD cards (vault_coin, vault_chain, schedule, credits)
- 4 utility cards (price, tx status, build_tx, sign_typed_data)
- 3 system cards (CreateVault, ImportVault, VerifyEmail)
- Confirmed: **Tornado, Pump.fun, Rujira have 0 UI cards** — deletion/relocation is UX-neutral

**Chain breadth in SDK (37 chains):** 13 EVM (generic code, ~390 LoC shared), 6 UTXO (shared base ~577 LoC + Cardano 1K), 10 Cosmos (4.8K LoC, heaviest family due to IBC), 8 other (Solana, Sui, Polkadot, Bittensor, TON, Ripple, Tron, QBTC testnet). SDK is 101K LoC well-factored; chain surface can be reduced in UI without touching SDK.

**Current prompt structure (agent-backend/internal/service/agent/prompt.go):**
- Stable prefix (cacheable, ~2,500 tokens): 29-chain list, tool routing table, execute_* routing, fiat/amount rules, UI card rules, on-demand skills list
- Dynamic suffix (per-turn): UTC time + wallet context (vault name, balances, addresses, coins, address book, recent actions)
- Balances flow: app sends per-turn in `SendMessage Request.context.balances` → backend renders into `### Balances (live)` section
- v-pxuw added explicit "no per-chain balance calls needed" instruction; per-chain balance tools deprecated in routing but still reachable

**Cross-repo branch state for v-pxuw consolidation:**
- agent-backend: 60 commits, 10.7K insertions, 2.1K deletions, 84 files
- mcp-ts: 65 commits ahead of main
- vultiagent-app: 177 commits ahead (bulk of refactor lives here)
- vultisig-sdk: 66 commits ahead
- Total: 368 commits across 4 repos

### Available Tools & Patterns

- **Folder organization in mcp-ts**: tools already grouped by category (`src/tools/rujira/`, `src/tools/polymarket/`, etc.) — structural convention ready for promotion to tier boundaries
- **`_meta.categories` field**: controls LLM visibility via agent-backend's `tool_filter.go` keyword-based gating. Stripping categories hides tools from LLM while keeping them registered for backend RPC
- **MCP transport**: mcp-ts is an HTTP MCP server — supports multiple instances at different URLs with different tool subsets
- **`MCP_SERVER_URL` env var in agent-backend**: single config value determines which MCP instance the agent consumes → supports deploying core vs all-tool instances for different consumers
- **Prompt caching split** (`prompt.go:188-198`): stable prefix + dynamic suffix, cache breakpoint between them. Tier-related changes will live in stable prefix
- **Tool card registry** (`vultiagent-app/src/features/agent/lib/toolUIRegistry.ts`): controls which tool results render custom UI; cards not in registry fall back to generic rendering
- **va-bqks pattern** (in flight): Redis cache + singleflight + backend BalanceService; per-chain tools stay registered but categories stripped; backend reaches them via `mcpProvider.CallTool(ctx, name, args)`. **This is the reference implementation for "core/internal" tools — apply at scale to fee rates, EVM helpers, custom-send builders**
- **Launch-surface PostHog flag gating** (shipped in v-pxuw commit 5664b9c): another control dimension; needs audit post-big-bang to determine if still load-bearing
- **Benchmarking infrastructure**: `testdata/benchmark-v2.mjs`, `testdata/benchmark-report.md`, `testdata/smoke-test.mjs` already exist — extend for regression gating
- **Message parts storage migration** (v-pxuw): persisted conversation format changed; historical conversation replay handled by `937deac fix(storage): replay all v-pxuw Task 6 renamed tool parts`
- **va-tixe retirement pattern**: UTXO + Cosmos `build_*_send` tools cleanly retired in va-tixe Phase 3 — comments document the retirement, tests updated. Same pattern applies to per-chain balance + fee rate + escape hatch tools

### Validated Behaviors

- Confirmed: prompt on refactor/v-pxuw-tool-consolidation already tells agent "no per-chain balance calls needed" (commit `cf982d3`) — per-chain balance tools are effectively dead weight in intended agent paths
- Confirmed: backend tool execution is fully server-side (`executor.go:1020` `s.mcpProvider.CallTool(ctx, name, input)`) — no client round-trips for tool dispatch
- Confirmed: frontend has **0 cards** for Tornado, Pump.fun, Rujira despite tools existing — moving these to experimental is UX-neutral (Polymarket has 3 read-only cards but action tools are stubs)
- Confirmed: EVM chains share generic code in SDK — chain breadth in EVM family doesn't cost meaningfully
- Confirmed: Cosmos family is disproportionately heavy at 4.8K LoC — IBC message types + per-chain staking/LP
- Confirmed: no parallel implementations or significant dead code — refactor is deletion, not rewrite (only `respond_to_user` deprecated and cordoned off; one "Not yet implemented" stub in `polymarket_submit_order`)
- Confirmed: `execute_*` tools are truly end-to-end — single tool call returns `ExecutePrepResult` covering approval + main tx steps, UI stepper config included
- Confirmed: `get_balances` / `get_portfolio` umbrella tools fan out across chains via per-chain MCP tools dispatched by backend (`data_tools.go:520-608` `chainToBalanceTool` map)
- Confirmed: agent receives balance context from app each turn via `MessageContext.Balances` — simple balance Qs answerable without tool call
- Confirmed via v-pxuw commit log: cleanup is actively in progress (va-tixe Phase 1/3, v-pxuw Tasks 6/8, v-bpzs UI card pattern, v-wldr storage) — this ticket follows on from that work, doesn't compete with it

### External Context

- **Small-model tool-selection pressure**: Gemini Flash 3 and Haiku 4.5 have documented failure modes with >10-option tool lists, array-wrapped schemas (agent emits scalar instead of array), `oneOf` (unsupported in Gemini), and enum cardinality >10. Core tool surface of ~11 tools with flat schemas is well inside the reliability envelope. va-bqks research already validates these constraints.
- **MCP protocol**: tool discovery is client-driven; `_meta` fields are server-advertised; one MCP server can advertise different tool sets based on identity/auth/env
- **PostHog feature flags**: runtime-toggleable, supports per-user/cohort targeting; currently wrapping launch-surface gating

## Constraints

- **Language boundary**: backend is Go, SDK is TypeScript. Backend → SDK access requires IPC. MCP-as-RPC is current mechanism (Option A) with naming convention to name the pattern. Dedicated HTTP API (Option B) and pushing orchestration to mcp-ts (Option C) both deferred.
- **Dependency order**: v-pxuw must merge first (provides execute_* routing foundation). va-bqks must merge before big-bang (establishes core/internal pattern; its per-chain balance handling overlaps this scope).
- **Cross-repo coordination tax is the real bottleneck**: design must minimize coordination events. Monorepo rejected by team; Go-to-TS backend port deferred. Big-bang = one coordination event across 4 repos.
- **Prompt caching boundary must be preserved**: tier changes live in stable prefix. Don't bloat the dynamic suffix.
- **Existing user conversations must not break**: message parts format, historical conversation rendering, mid-flight tool calls. Rollout plan must handle old app / new backend window.
- **Coworker ownership**: one team member actively develops Rujira/breadth. Experimental tier MUST preserve his iteration workflow (`MCP_TIER=all` dev instance + separate deployed URL for demos).
- **SDK left alone preemptively**: consumers narrow their surface without SDK edits. Reactive changes only — if a core tool audit reveals an SDK gap, fix in same PR. No speculative SDK repackaging.
- **Polymarket `submit_order` is a stub**: "Not yet implemented" — move to experimental, don't delete.
- **Graduation criteria must be written and agreed before tier classification**: political prerequisite, not just doc hygiene. Without agreement, the tier boundary is constantly renegotiated.

## Dead Ends

- **Full rewrite of app/backend/mcp stack**: rejected. 368 commits of inherited + team work already on v-pxuw branch; SDK is 101K LoC well-factored; backend port = 2-3 months with no user-visible value. Architectural wins of this branch (backend tool execution, `execute_*` end-to-end, @ai-sdk/react streaming) would be thrown away.
- **Deleting exotic integrations outright**: rejected. Coworker's Rujira work needs a home; PM wants "cool stuff coming" story. Experimental tier preserves these politically and technically.
- **Splitting into 20 small tickets with sequential merges**: rejected. Intermediate states are non-functional (tier infra without classified tools, deleted tools without prompt cleanup, moved tools without app UI updates, etc.). Cross-repo coordination tax makes incremental strictly worse than big-bang here.
- **Full monorepo consolidation (all 4 repos)**: team pushback — "not necessary right now."
- **TS port of agent-backend (narrower monorepo pitch)**: deferred. Port-first gives no user-visible improvement during port; cleanup-first ships product wins and gives data for the TS port ROI case. Revisit after big-bang ships.
- **Preemptive SDK changes** (Rujira package split, long-tail chain deprecation): deferred. Bundle reactively into big-bang if core tool audits reveal gaps.
- **Option C** (push BalanceService to mcp-ts): deferred. Would move Redis + cache + invalidation hooks to mcp-ts; conflicts with va-bqks design. Revisit if backend accumulates significant MCP-as-RPC surface.
- **Option B** (dedicated internal HTTP API on mcp-ts): deferred. Currently only balance-fetching is RPC'd this way; not enough duplication to justify parallel API surface yet. Revisit at ~10+ RPC sites.
- **Per-chain balance tool deletion**: rejected. They remain as internal-tier tools (registered, categories stripped, reachable via backend RPC for BalanceService fan-out). Name/namespace them so the muddle is visible.
- **Keeping `get_portfolio` separate from `get_balances`**: rejected (via va-bqks). Unified into one tool with `include_defi` flag.

## Open Questions

### Tier architecture
- Exact folder structure inside mcp-ts: `src/tools/core/visible/` + `src/tools/core/internal/` + `src/tools/experimental/`? Alternative shapes? Naming convention for internal-tier tools (`_internal_` prefix, suffixed exports, README callout, ...)
- `MCP_TIER` env switch mechanics: how does server decide tier at boot? Fallback if unset? How do dev/staging/prod configs differ? Does launch-surface PostHog flag still serve a purpose after tier split, or does it become redundant?
- Experimental MCP deployment model: separate instance at distinct URL (new infra), OR same instance with tier exposed via auth/identity? What's the actual path for the coworker's workflow (local `MCP_TIER=all` + shared dev URL)?

### Core surface scope
- `execute_contract_call` fate: core or experimental? Argument for experimental: not a basic user intent, power-user tool, currently requires `custom-transactions` skill. Argument for core: already shipped, well-tested, some valid basic uses (approve, revoke).
- `execute_lp_add` / `execute_lp_remove` fate: almost certainly experimental (THORChain LP is not a basic), but confirm and remove from prompt routing + skill list.
- 6 custom-send escape hatches (SPL, XRP, TRC-20, TON Jetton, Sui native + token): verify `execute_send` truly covers these end-to-end. If yes, delete. If not, extend `execute_send` inside the big-bang rather than keeping escape hatches.
- 28 building blocks: all move to `core/internal/`, or are any safely deletable (not used by any `execute_*` or BalanceService path)?

### Chain surface
- "Popular chains" firm list: proposal is BTC, ETH, Arbitrum, Base, Optimism, Solana, BSC. Polygon? Avalanche? THORChain as cross-chain infrastructure (not user-holding)? Need explicit sign-off list.
- Chain surface reduction scope in app UI: which screens filter? Vault chain picker, send destination, swap from/to, token search chain filter, portfolio chain display, address book chain options?
- Backend prompt: strip 29-chain list, or keep? Is the list load-bearing for agent chain-name disambiguation, or is `normalizeChain` from SDK sufficient?

### Skills & prompt
- Skills audit: which of `send-transfer`, `scheduling`, `custom-transactions`, `eip712-signing`, `polymarket-trading`, `buy-credits`, `yield-xyz` are core vs experimental? Core skill surface target ≤3.
- Prompt cleanup scope: routing table trim, skills list trim, chain list trim, fiat amount rules, MCP tool values section, UI card rules — what survives, what goes?
- `MessageContext` auto-inject context changes: does balance summary (from va-bqks) extend to other cached data (prices, recent actions) in same pattern?

### Core tool reliability
- `search_token` audit: ambiguous-result handling. "USDT" across 6 chains — does LLM get clean list to pick without hallucinating contract addresses?
- `get_price` audit: works reliably with symbol + contract + chain triples? Flat schema for weak models?
- `get_tx_status` audit: cross-chain consistency. Works for every chain in popular set? Same response shape per chain?
- `get_address` audit: returns enough structured data for QR/copy card, not just string?
- `create_vault` / `import_vault` audit: agent-card handoff ergonomics. Does the conversational flow feel smooth, or is it a half-measure that users bypass to use the app's own flow?
- `add_chain` / `add_token` schema: currently accept arrays. Weak models emit scalars against array schemas. Pivot to comma-string like va-bqks did for `chain` param?

### App surface
- Tool card relocation strategy: delete experimental cards entirely, or move to `components/tools/experimental/` preserved for future graduation? If preserved, opt-in mechanism when a card re-registers?
- Per-chain TX builders in `services/` (10.2K LoC): which survive (popular chains) vs delete (long-tail chains)?

### Measurement
- Performance benchmark targets: exact numbers for first-send latency, balance-question response time, tool-selection accuracy on Haiku 4.5 across intent corpus. Corpus content?
- Regression gate: does `testdata/benchmark-v2.mjs` become CI-gating, or is it run manually?

### Rollout
- Single atomic merge per repo or feature-flagged progressive rollout?
- Old app / new backend compatibility window during deploy — how handled?
- Backend prompt + mcp-ts tool list must stay consistent (prompt references existing tools). Deploy ordering across the 4 repos?
- Rollback plan if core tool audit reveals a regression post-merge?

### Political infrastructure
- Graduation criteria doc location: root `GRADUATION.md`, CLAUDE.md section, Notion, team wiki? Where is it most visible and hardest to ignore?
- Who signs off on graduation? Owner + PM + originating engineer? Written checklist?
- How is the experimental tier messaged internally — "release train," "preview," "lab"? The framing affects coworker buy-in.
- Does the PM roadmap need explicit "Experimental: ship date TBD" rows for Rujira/Polymarket/etc., or does silence work?

### SDK reactive changes
- If a core tool audit reveals SDK gap, fix in same PR or separate ticket? (Recommendation: same PR to minimize coordination; confirm.)
- Any SDK chain-alias / `normalizeChain` extensions needed for the popular-chain reduction?

## Notes

**2026-05-05T02:40:49Z**

killed: per Sean — strategic decision-making, not engineering work
