# Vultiagent Team Operating Model

Proposal for how we work. Read it, disagree with anything, suggest changes. Nothing here is set in stone.

---

## How This Repo Works

This repo (`vultisig/`) is the team hub. Product repos are nested underneath, gitignored, independently version controlled. Open Claude Code from here and it sees everything — team config, missions, decisions, and all sub-repos.

```
vultisig/                        # hub repo — open Claude Code here
├── CLAUDE.md                    # team conventions (inherited by all sessions)
├── operating-model.md           # this doc (finalized)
├── biome.json                   # shared lint/format config
├── missions/                    # shaped mission pitches
├── decisions/                   # architecture decision records
│
├── vultiagent-app/              # gitignored, independent repo
├── vultisig-sdk/                # gitignored, independent repo
├── agent-backend/               # gitignored, independent repo
└── vasig/                       # gitignored, independent repo
```

Each sub-repo has its own CLAUDE.md for repo-specific context (build commands, architecture). Claude Code reads parent CLAUDE.md files automatically, so team conventions apply everywhere.

GH Issues and milestones live on this hub repo. PRs live on individual product repos and reference hub issues with `vultisig/vultisig#42`.

### Setup

```bash
git clone git@github.com:vultisig/vultisig.git
cd vultisig
git clone git@github.com:vultisig/vultiagent-app.git
git clone git@github.com:vultisig/vultisig-sdk.git
git clone git@github.com:vultisig/agent-backend.git
git clone git@github.com:vultisig/vasig.git
```

---

## Domains

Domains are areas of the system. They're semi-permanent. Each team member has a **primary domain** (you're the decision-maker) and a **secondary domain** (you stay across it, handle overflow, can land changes independently).

Domains aren't walls. Anyone commits anywhere. Domains determine who makes decisions in that area.

| Domain | Scope |
|--------|-------|
| **SDK Core** | Chain logic, signing orchestration, types, configs — the foundation everything calls |
| **Agent Backend** | Go service, Claude integration, chat orchestration, conversation state |
| **Client Surfaces** | vultiagent-app, Station wallet, Chrome extension — all user-facing UIs |
| **External Access** | vasig CLI + MCP server — how 3rd party harnesses consume the SDK |

Primary/secondary assignments TBD — we'll sort this out together based on where people want to go deep.

---

## Missions

Missions are shaped pieces of work with a clear goal and a timebox (appetite). They're temporal — they have a start and an end. A mission might live within one domain or span several.

**Domains are the map. Missions are the journeys across it.**

A mission is NOT a Jira ticket. It's a 1-page pitch shaped by the lead:
- **Goal**: what "done" looks like
- **Appetite**: how much time this is worth (not an estimate — a decision)
- **Intent**: "if nothing else, achieve X because Y"
- **Boundaries**: what's in/out of scope
- **Rabbit holes**: known traps to avoid

The engineer who gets a mission decomposes it however they want. BYO local task tracking — beads, superpowers, tk, markdown files, whatever works for you.

---

## Rhythm

| Cadence | What | Format |
|---------|------|--------|
| **2-week cycles** | Shape missions, assign at betting table | 30 min sync |
| **Weekly** | Course correction — architecture drift, decisions needed | 30 min sync |
| **Daily** | Automated team pulse (git activity digest) | Async, bot-generated |
| **End of cycle** | 2-day cool-down — refactor, explore, clean up | Unstructured |

### Team Pulse (Daily, Automated)

A scheduled agent summarizes each engineer's git activity (PRs merged, active branches, areas touched) and posts to Discord. No manual writing required.

At the bottom: **"Any blockers or FYIs?"** — reply in thread only if you have something. Most days nobody replies. That's fine.

### Weekly Sync

Not a status update — everyone already knows status from the pulse. This is for:
- Course corrections ("I think we should change approach on X")
- Blocked decisions that need real-time discussion
- Betting table at cycle boundaries

### Cool-Down

2 days at the end of each cycle. Unstructured. Refactor, pay down debt, explore something, write an ADR, or just clean up. Prevents burnout and creates natural integration points.

---

## Git & PRs

- **Commit prefixes**: `feat:`, `fix:`, `chore:` — that's the whole taxonomy
- **PRs required** — no direct push to main. The PR is the record, not a gate.
- **Self-test, self-merge.** CI must pass. No human approval required.
- **Squash or rebase** — your choice per PR. No merge commits.
- **Pre-merge review required for**: key material, signing flows, address derivation. Tag someone and wait. This is the one area where speed isn't worth the risk.
- Post-merge review is optional and pull-based. If you want to get across someone's changes, go read their PRs.

### Cross-Cutting Changes

If your changes break interfaces or affect multiple areas, drop a quick Discord message: what changed, what breaks, and anything others need to do (re-pull, run a migration, update imports). Use judgment — a new file in your domain doesn't need an announcement.

### SDK Interface Changes

These affect everyone. Post a short message with the proposed change and rationale. Give people a day to object before merging.

---

## CI

Target: **under 2 minutes**, blocking merge.

| Check | Tool |
|-------|------|
| Lint + format | `biome check .` |
| Type check | `tsc --noEmit` |
| Unit/integration tests | `vitest run` |
| Go lint | `golangci-lint` |
| Go tests | `go test -short` |

Testnet integration tests run on a schedule, non-blocking. No pre-commit hooks — CI catches everything. Run `pnpm fix` locally if you want.

Biome config is shared from this hub repo. Each product repo extends or copies it.

---

## Proposed CLAUDE.md (Hub Level)

This is the team constitution. Every Claude Code session inherits it. We update it frequently as we learn what works.

```markdown
# Vultisig Team

## Repos
- vultiagent-app/ — React Native mobile app (Expo 55), chat-based wallet
- vultisig-sdk/ — Unified TypeScript SDK, all chain logic and signing
- agent-backend/ — Go AI chat service, Claude integration, MCP tool calls
- vasig/ — MCP server + CLI for 3rd party harness access

## Domains
- SDK Core: chain logic, signing, types, configs
- Agent Backend: Go service, Claude integration, chat orchestration
- Client Surfaces: app, Station, Chrome extension — all UIs
- External Access: MCP server + vasig CLI

## Conventions
- Commit prefixes: feat:, fix:, chore:
- Business logic goes in the SDK, not app/CLI/backend layers
- Self-test, self-merge. CI must pass.
- Pre-merge review required for: key material, signing, address derivation
- Big cross-cutting changes: Discord summary of what breaks and what others need to do
- SDK interface changes: post the proposed change, give a day for objections

## Code Style
- Biome for all TS/JS (see biome.json in this repo)
- gofmt + golangci-lint for Go
- Reuse existing code — check imports/utils before writing new ones
- Prefer readable, procedural code over abstractions
- Comments only for non-obvious logic, one line max
```

---

## Proposed Missions (Phase 1: Weeks 1-4)

First pass based on the roadmap. Each mission is roughly 1-2 weeks of appetite.

### Cycle 1 (Weeks 1-2)

| Mission | Domains | Appetite |
|---------|---------|----------|
| **Unify EVM chain logic into SDK** — consolidate all EVM transaction building (send, swap, approve) into one SDK implementation. App, CLI, and backend all call the same functions. | SDK Core | 2 weeks |
| **Unify Cosmos chain logic into SDK** — same as above for Cosmos family (ATOM, OSMO, etc). Send, stake, IBC transfers. | SDK Core | 2 weeks |
| **Unify Solana/SVM chain logic into SDK** — Solana transaction building, token accounts, SPL transfers. | SDK Core | 2 weeks |
| **Stabilize agent backend** — audit and fix conversation flow, tool call failures, context dropping. Clear error messages. | Agent Backend | 2 weeks |
| **Station hype/migration screen** — ship the initial Station presence. Hype screen, migration messaging, LUNA/LUNC token support for airdrop holders. | Client Surfaces | 1 week |
| **MCP/CLI reliability pass** — fix failing tool calls, stabilize balance/send/swap commands, consistent error handling. | External Access | 2 weeks |

### Cycle 2 (Weeks 3-4)

| Mission | Domains | Appetite |
|---------|---------|----------|
| **Unify remaining chains into SDK** — Sui, TON, Tron, UTXO (Bitcoin, LTC, DOGE, BCH). UTXO requires native wallet-core bindings. | SDK Core | 2 weeks |
| **Signing orchestration in SDK** — migrate signing flow into SDK so all products use the same path. Fast vault signing works end-to-end. | SDK Core | 2 weeks |
| **Station full wallet launch** — vault management, balances, send/receive, swaps. Must feel complete, not beta. Only ship when ready. | Client Surfaces | 2 weeks |
| **SDK capability expansion** — token info lookups, transaction history, portfolio analytics. Available across all surfaces via SDK. | SDK Core, External Access | 2 weeks |
| **Chrome extension scaffolding** — chat UI shell, vault connection, SDK integration via API/MCP. | Client Surfaces, External Access | 1 week |
| **Swap router audit** — which chains/pairs are covered? Fill gaps. Ensure routing finds the best path. | SDK Core | 1 week |

### Phase 2 Missions (Weeks 5-8, Rough Sketch)

| Mission | Domains |
|---------|---------|
| **Chrome extension ships** — core wallet ops, send/swap/balances, chat agent | Client Surfaces, External Access |
| **MCP feature parity** — every SDK capability exposed as an MCP tool | External Access, SDK Core |
| **TSS research + infra** — relay coordination, multi-device sessions, timeout handling | SDK Core, Agent Backend |
| **Staking across chains** — Cosmos validators, Solana stake accounts, ETH staking, unstaking flows | SDK Core, Client Surfaces |
| **Opportunities engine v1** — background monitoring, yield changes, price movements, push notifications | Agent Backend, SDK Core |
| **Advanced DeFi: limit orders + stop-loss** — price monitoring, auto-execution | SDK Core, Agent Backend |
| **Advanced DeFi: LP positions + rebalancing** — enter/exit LP, target allocations, swap set generation | SDK Core |
| **Advanced DeFi: TWAP + dust sweeping** — split large trades, consolidate small balances | SDK Core |

### Phase 3 Missions (Weeks 9-12, Rough Sketch)

| Mission | Domains |
|---------|---------|
| **Full M-of-N TSS signing** — multi-device key share coordination, relay sessions, threshold signing | SDK Core |
| **Agent runtime infrastructure** — containerized agents holding key shares, persistent processes | Agent Backend |
| **Cloud agent deployment** — one-click hosted + self-hosted Docker Compose, agent identity/keys | Agent Backend, SDK Core |
| **Configurable guardrails** — system prompt customization, enforced limits (max tx size, daily spend, asset whitelists) | Agent Backend |
| **Adversarial multi-agent signing** — growth agent vs risk agent, structured debate protocol, both must approve | Agent Backend, SDK Core |
| **Decision audit trail** — log every debate, argument, and decision for user review | Agent Backend, Client Surfaces |
| **End-to-end autonomous demo** — deploy agents, detect opportunity, debate, co-sign, execute, notify | All domains |

---

## What We're NOT Doing

- Sprints, story points, velocity tracking
- Jira or any heavyweight ticket system
- Daily standups or mandatory status writes
- Blocking PR reviews (except for signing/key material)
- Code coverage targets
- Backlog grooming — unshaped work lives in a "raw ideas" list, gets shaped or discarded
- Backwards compatibility (new project, full ownership)

---

## Open Questions

Things to discuss as a team:

1. **Primary/secondary domain assignments** — who wants what?
2. **Monorepo vs multi-repo long-term** — hub repo works for now, but should we consolidate eventually?
3. **Discord channel structure** — do we want `#team-pulse`, `#architecture`, or just one channel?
4. **Biome config details** — any strong opinions on specific rules?
5. **Mission sizing** — are these roughly right or should some be split/combined?
