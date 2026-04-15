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
Business logic goes in vultisig-sdk, not app/CLI/backend layers.

## Code Style
- Biome for TS/JS — check each repo's package.json for the correct package manager (npm, yarn, or pnpm)
- gofmt + golangci-lint for Go

## Project Tracking
GitHub Project "Vultiagent" is the source of truth for missions and task planning.

## Safety
Changes to key material, signing flows, or address derivation require extra scrutiny. Flag in PR descriptions.
