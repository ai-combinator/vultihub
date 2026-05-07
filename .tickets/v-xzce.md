---
id: v-xzce
status: closed
deps: []
links: []
created: 2026-04-06T00:16:18Z
type: task
priority: 2
assignee: Jibles
---
# vasig → SDK CLI + MCP Migration

## Design: vasig → SDK CLI + MCP Migration

### Context

Issue #13 (MCP Consolidation) calls for merging vasig (vultiagent-cli) into the SDK monorepo. This epic covers the full migration: porting vasig's agent-friendly features into `@vultisig/cli`, creating a new thin MCP package, and deprecating vasig.

### Architecture

```
@vultisig/sdk (business logic)
    ↑                    ↑
clients/cli/          clients/mcp/
(vsig - full CLI)     (thin MCP wrapper)
```

Both consume the SDK directly. CLI is the primary human+agent interface. MCP is a lightweight MCP protocol wrapper over SDK calls with its own auth/signing adapters for statefulness.

Key decision: MCP wraps the SDK, not the CLI. This allows the MCP to diverge for stateful use cases (session-based auth, deferred signing) without coupling to CLI command structure.

### Migration Phases

**Phase 1: Port vasig features into SDK CLI**
- Keyring auth (keytar) — vsig auth setup/status/logout, replaces password-only flow
- Token discovery — vsig tokens --discover
- Dry-run mode — --dry-run on send/swap
- Config persistence — remember authenticated vaults, extra chains, tracked tokens across sessions
- Error codes — typed exit codes (0=success, 2=auth, 3=network, 4=bad input, 5=not found)
- Schema discovery — vsig schema command for machine-readable command structure
- Structured JSON error format alignment

**Phase 2: Create clients/mcp/ package**
- New package in SDK monorepo, depends on @vultisig/sdk
- Port vasig's 8 MCP tools (get_balances, get_portfolio, get_address, vault_info, supported_chains, swap_quote, send, swap)
- Pluggable auth adapter interface (local keyring vs session-based)
- Pluggable signing adapter (local end-to-end vs deferred for TSS)
- Profile flag (--profile harness|defi|full) for tool surface control
- stdio + HTTP transport

**Phase 3: Deprecate vasig**
- Verify feature parity
- Update docs, agent-backend references
- Archive vultiagent-cli repo

### Out of scope
- Porting Go MCP tools (DeFi, Polymarket, contract interaction) — incremental per #6/#8
- CLI argument restructuring — separate follow-up pass
- Agent-backend switchover to TS MCP — later step in #13

### Task Decomposition

1. Port keyring auth to SDK CLI — Add keytar, implement auth setup/status/logout, integrate with password-manager as higher-priority resolution source
2. Port token discovery and config persistence — Add --discover to tokens, persistent config store for vaults/chains/tokens
3. Port dry-run mode — Add --dry-run to send/swap, preview without signing/broadcasting
4. Port error codes and schema discovery — Typed exit codes, vsig schema command
5. Create clients/mcp/ package — Scaffold MCP package, 8 core tools against SDK, auth/signing adapters, profile flag, stdio transport. Independent of tasks 1-4.
6. Verify parity and deprecate vasig — E2E comparison, update references, archive repo. Depends on 1-5.

Tasks 1-4 are independent (parallelizable). Task 5 is independent of 1-4. Task 6 depends on all.

## Notes

**2026-04-06T00:17:01Z**

Linked to GitHub issue #13 (MCP Consolidation). This epic covers step 1 of #13's migration path: CLI merge (vasig → @vultisig/cli) + new MCP package.

**2026-04-06T01:31:37Z**

Tasks 1-5 complete. All merged. 3 critical integration issues fixed. 82 tests passing. Both CLI and MCP packages build.

**2026-05-04T23:56:23Z**

auto-closed: epic overtaken by mcp-ts as separate package + v-pxuw consolidation
