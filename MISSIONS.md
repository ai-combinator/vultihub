# Missions

## Priority 1

### Unify Chain Logic into SDK
- **Goal**: All transaction building (EVM, Cosmos, Solana, Sui, TON, Tron, UTXO) lives in vultisig-sdk. App, CLI, and backend all call the same functions.
- **Appetite**: 2 weeks
- **Blocked by**: none
- **Domain**: SDK Core
- **Boundaries**: Focus on consolidation — making app/CLI/backend consume the SDK instead of their own chain logic. UTXO and signing orchestration already exist in the SDK. This is about wiring the consumers, not rebuilding the internals.
- **Rabbit holes**: Don't try to abstract away chain differences — each chain module can have its own shape.
- **Done looks like**: You can delete the duplicated chain logic from vultiagent-app, agent-backend, and vasig. Every send/swap/sign call goes through vultisig-sdk. No consumer has its own transaction building code.

### Stabilize Agent + CLI/MCP
- **Goal**: Make swap and send rock solid across all interfaces — agent, CLI, and MCP. Currently CLI sends/swaps timeout ~70% of the time. Diagnose and fix. Agent conversation flow should be reliable: no silent tool call failures, no context dropping, clear error messages. The bread and butter operations must just work.
- **Appetite**: 2 weeks
- **Blocked by**: none
- **Domain**: Agent Backend, External Access
- **Boundaries**: Fix what's broken, don't add new capabilities yet.
- **Done looks like**: A user can send and swap through the CLI and agent with >95% success rate. When something does fail, the error message tells you why. Agent conversations don't lose context mid-flow. Tool call failures are surfaced, not swallowed.

### Model Research & Commercial Analysis
- **Goal**: Understand our token consumption and build a pricing model. Measure token usage for standard operations (swap, send, balance check, chat conversation). Model free tier limits, subscription tiers, and per-operation costs. Evaluate which models we can use for which operations — where can we use a cheaper model without losing accuracy, where do we need the best? Deliverable is a recommendation doc with real numbers.
- **Appetite**: 1 week
- **Blocked by**: none
- **Domain**: Agent Backend
- **Done looks like**: A document with: (1) token cost per operation type with real measurements, (2) proposed free tier limits with rationale, (3) subscription tier pricing with unit economics, (4) model-to-operation mapping (e.g. "balance queries can use Haiku, swaps need Sonnet") with accuracy tradeoff data.

### Team Infra & Process Setup
- **Goal**: Set up the operational foundations for the team. Unified linters/formatters across all repos (Biome for TS/JS, golangci-lint for Go). Automated daily team pulse (git activity digest to Discord). Centralised CLAUDE.md files so every repo inherits team conventions. Clean up hub repo — PROPOSAL.md is superseded, finalize TEAM-PROCESS.md and MISSIONS.md as the source of truth.
- **Appetite**: 1 week
- **Blocked by**: none
- **Domain**: All
- **Done looks like**: Every repo has consistent lint/format config. Daily automated digest is posting to Discord. Hub CLAUDE.md is inherited by all sub-repos, each sub-repo has its own for repo-specific context. Process docs are clean and current.

### Station App Takeover
- **Goal**: Own the full Station wallet transition end-to-end. Three threads of work:
  1. **Migration safety**: Ship hype screen with clear messaging warning users to save their seed phrase before the app org transfer (they'll lose access to seeds tied to the current app). Make it impossible to miss.
  2. **Airdrop continuity**: LUNA/LUNC holders currently get airdrops every ~3 weeks. These must continue uninterrupted through the transition. Get contract details from Gary — figure out what needs to migrate or redirect.
  3. **Full wallet**: Vault management, balances, send/receive, swaps. Ship when it feels complete, not beta. Clean up deployment pipeline, verify build/deploy flow end-to-end. Coordinate release plan with Gary.
- **Appetite**: 3 weeks
- **Blocked by**: none
- **Domain**: Client Surfaces
- **Rabbit holes**: Don't redesign the airdrop contracts — just make sure existing holders keep getting paid through the transition. Seed phrase export UX should be dead simple, not a new feature.
- **Done looks like**: Station app is live under our org. Users were warned and had time to save seeds. Airdrop holders didn't miss a cycle. Wallet covers the basics and feels polished. Deployment pipeline is clean and repeatable.

---

## Priority 2

### Chrome Extension
- **Goal**: Full wallet experience matching the native app — send/swap/balances, chat agent. Custom full-page onboarding flow (not a popup). Should feel like a first-class product, not a stripped-down port.
- **Appetite**: 3 weeks
- **Blocked by**: Semi-blocked by Unify Chain Logic into SDK (can start with its own copy of SDK logic, like the native app currently does, and migrate when SDK unification lands)
- **Domain**: Client Surfaces, External Access
- **Done looks like**: A user can install the extension, go through onboarding, connect or create a vault, check balances, send tokens, swap, and chat with the agent. Onboarding is a full-page flow that feels polished. Extension works on Chrome (Brave/Edge get it for free).

### SDK Capability Expansion: Portfolio & Info
- **Goal**: Add read-side capabilities to the SDK — token metadata lookups, transaction history, portfolio valuation and analytics. Wire each new capability to all consumers (MCP, CLI, extension, mobile agent) as part of the work, not as a follow-up.
- **Appetite**: 2 weeks
- **Blocked by**: Unify Chain Logic into SDK
- **Domain**: SDK Core, External Access
- **Done looks like**: From any consumer, a user can look up token info, see their transaction history, and get a portfolio breakdown with valuations. The SDK is the single source for all of it.

### SDK Capability Expansion: Swap Coverage
- **Goal**: Audit which chain/pair combos actually work across our 5 swap providers (1inch, KyberSwap, LiFi, THORChain, MayaChain). Document the gaps. Fill the ones that matter — prioritize by user demand and volume. Ensure the router picks the best path (price, speed, reliability) when multiple providers can handle a pair. Wire improvements to all consumers.
- **Appetite**: 1 week
- **Blocked by**: Unify Chain Logic into SDK
- **Domain**: SDK Core, External Access
- **Done looks like**: A coverage matrix showing what works where. High-priority gaps filled. Router demonstrably picks better paths than before. No regressions on pairs that already worked.

### SDK Capability Expansion: DeFi Operations
- **Goal**: Build out write-side DeFi capabilities in the SDK:
  1. **Staking**: Cosmos validators, Solana stake accounts, ETH staking/unstaking. Users can stake and unstake from any consumer.
  2. **Limit orders + stop-loss**: Price monitoring with auto-execution when targets hit.
  3. **LP management**: Enter/exit liquidity positions, set target allocations, rebalancing generates swap sets.
  4. **TWAP**: Split large trades across time to reduce slippage.
  5. **Dust sweeping**: Consolidate small balances across tokens into a target asset.
  Wire everything to all consumers.
- **Appetite**: 3 weeks
- **Blocked by**: Unify Chain Logic into SDK
- **Domain**: SDK Core, External Access
- **Done looks like**: A user can stake, set limit orders, manage LP positions, execute TWAP trades, and sweep dust from any consumer. Each operation works end-to-end through the SDK.

---

## Priority 3

### TSS Research + Infra
- **Goal**: Improve the threshold signing infrastructure. Investigate and fix relay coordination issues, multi-device session handling, and timeout behavior. Understand what's needed for robust N-of-M signing across devices.
- **Blocked by**: Unify Chain Logic into SDK
- **Domain**: SDK Core, Agent Backend
- **Done looks like**: A clear picture of what works, what doesn't, and what needs building for production-grade multi-device TSS. Key relay issues identified and fixed. Sessions don't silently fail or hang.

### Opportunities Engine v1
- **Goal**: Background monitoring system that watches for actionable opportunities — yield rate changes, significant price movements, arbitrage windows, staking reward changes. Pushes notifications to users when something worth acting on is detected.
- **Blocked by**: none
- **Domain**: Agent Backend, SDK Core
- **Done looks like**: A running service that monitors a configurable set of signals and delivers push notifications. Users can act on opportunities directly from the notification. False positive rate is low enough that users don't mute it.

---

## Future (Unpriced)

These need shaping before they become real missions.

- **Full M-of-N TSS signing** — multi-device key share coordination, relay sessions, threshold signing
- **Agent runtime infrastructure** — containerized agents holding key shares, persistent processes
- **Cloud agent deployment** — one-click hosted + self-hosted Docker Compose, agent identity/keys
- **Configurable guardrails** — system prompt customization, enforced limits (max tx size, daily spend, asset whitelists)
- **Adversarial multi-agent signing** — growth agent vs risk agent, structured debate protocol
- **Decision audit trail** — log every debate, argument, and decision for user review
- **End-to-end autonomous demo** — deploy agents, detect opportunity, debate, co-sign, execute, notify
