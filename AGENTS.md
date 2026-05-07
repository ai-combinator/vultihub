# Vultisig

## Repos
- vultisig-sdk/ — Unified TypeScript SDK, all chain logic and signing (yarn)
- agent-backend/ — Go AI chat service, Claude integration, MCP tool calls
- vultiagent-app/ — React Native mobile app (Expo), chat-based wallet (npm)
- vultiagent-cli/ — CLI agent interface (npm)
- mcp/ — MCP server for tool discovery, skills, and chain interactions (Go)
- vultiserver/ — Verifier/relay service (Go)
- station/ — Station wallet web app (npm)
- vultisig-android/ — Android native app
- vultisig-ios/ — iOS native app
- vultisig-windows/ — Windows app (Go + yarn)

Each sub-repo has its own CLAUDE.md. This file applies to all.

## Architecture
Chain ops, signing, and address derivation live in vultisig-sdk. UI flow, request handling, and platform glue stay in their app/CLI/backend layer.

## Project Tracking
Missions and task planning live in GitHub Project "Vultiagent" (https://github.com/orgs/vultisig/projects/20).

## PR Bodies
Write PR descriptions for reviewers who only have GitHub context. Describe work chunks by behavior or subsystem, not by Sean's local-only `tk` ticket IDs or branch tags. Local tags like `v-qbzv`, `v-xvcz`, `v-duhk`, or `guide:tix` are fine in commits/notes for Sean, but they are meaningless in a public PR body unless expanded into plain English.

Examples:
- Good: "Tool-result persistence: stores swap execution outcomes and reloads confirmed/cancelled cards from the conversation sidecar."
- Bad: "v-qbzv core."
- Good: "Quest follow-up: derive first-swap quest completion from server-persisted tool metadata."
- Bad: "v-xvcz."

## Comments
WHY-only, ≤2 lines. Don't restate what the code does, don't reference the task/ticket/PR — those belong in the commit/PR body and rot in code. Don't mimic existing long comment blocks; shorten them when you touch them. The bar: would removing this confuse a future reader? If no, delete it.

## Safety
For changes touching key material, signing flows, or address derivation:
- Add or update a test exercising the affected derivation/signing path.
- List affected chain IDs in the PR body.
- Tag the SDK area in the PR title (e.g., `[sdk/signing] ...`).
