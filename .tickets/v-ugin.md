---
id: v-ugin
status: open
deps: [v-wdyq]
links: [tic-ef85]
created: 2026-03-22T00:38:44Z
type: feature
priority: 1
assignee: Jibles
---
# MCP: Create @vultisig/mcp server package

## Objective

Create a stdio MCP (Model Context Protocol) server that exposes Vultisig vault operations as tools for AI agents like Claude Code. The server imports `@vultisig/sdk` directly and reads credentials from the same keyring/config that `vultiagent-cli` (`vasig auth`) sets up. This means the human authenticates once via the CLI, and the MCP server works automatically.

## User Story

A developer using Claude Code wants to check their crypto portfolio, send tokens, or execute swaps by chatting naturally. The MCP server makes Vultisig operations available as tools that Claude can call directly.

## Design Constraints

- Stdio transport only (spawned as subprocess by Claude Code) — no HTTP server
- Must not write anything to stdout except JSON-RPC messages — all logging goes to stderr
- Reads credentials from system keyring (set by `vasig auth`) — never prompts for passwords
- Uses `@vultisig/sdk` directly for all operations — does NOT shell out to the CLI
- Tool names use snake_case per MCP convention
- All tool inputs have JSON Schema definitions
- Errors are clear and actionable (not raw stack traces)
- Should be installable via: `claude mcp add vultisig -- npx @vultisig/mcp`

## Context & Findings

### Architecture: SDK-direct, not CLI wrapper

The MCP server imports `@vultisig/sdk` and calls vault methods directly (balance, send, swap, etc.). It does NOT wrap the CLI as a subprocess. Reasons:
- No subprocess overhead per tool call
- Native structured data (no JSON stdout parsing)
- Type safety between MCP tool schemas and SDK types
- The CLI and MCP are siblings that both consume the SDK — not parent/child

### Auth: Shared with CLI

The MCP server reads from the same credential sources as the CLI:
1. System keyring: `vultisig/<vaultId>/server` for server password, `vultisig/<vaultId>/decrypt` for vault decryption
2. `~/.vultisig/config.json` for vault file path and vault ID
3. `VAULT_PASSWORD` / `VAULT_DECRYPT_PASSWORD` env vars as fallback (useful for `claude mcp add` config)

If no credentials are found, the MCP server returns a tool error: "Vault not authenticated. Run `vasig auth` in a terminal first."

The credential-reading code should be extracted into a shared package or module that both the CLI and MCP import. This is the `auth/credential-store.ts` and `auth/config.ts` from the CLI ticket — either publish as a separate package or copy the module.

### MCP spec compliance

Per the MCP spec (modelcontextprotocol.io):
- Stdio transport: no MCP-level OAuth needed — server is a trusted local subprocess
- Credentials come from environment or local files/keyring
- `@modelcontextprotocol/sdk` handles JSON-RPC transport — don't implement manually

### Future: Session tokens

When tic-ef85 (server session tokens) lands, the MCP server benefits automatically — the credential store will return a session token instead of a password. No MCP-specific changes needed.

### Minimum tool set

| Tool | Description | SDK Method |
|------|-------------|------------|
| `get_balances` | Get token balances for a chain or all chains | `vault.balances()`, `vault.balance(chain)` |
| `get_portfolio` | Get portfolio with fiat values | `vault.balancesWithPrices()` |
| `get_address` | Get derived address for a chain | `vault.address(chain)` |
| `send` | Send tokens to an address | `vault.prepareSendTx()` → `vault.sign()` → `vault.broadcastTx()` |
| `swap` | Swap tokens between chains | `vault.getSwapQuote()` → `vault.prepareSwapTx()` → `vault.sign()` → `vault.broadcastTx()` |
| `swap_quote` | Get a swap quote without executing | `vault.getSwapQuote()` |
| `vault_info` | Show vault name, type, chains, signer count | Vault metadata |
| `supported_chains` | List chains the vault supports | `vault.getSupportedSwapChains()` |

### Confirmation for destructive operations

`send` and `swap` tools should include a two-step pattern:
1. First call returns a preview (amounts, fees, recipient) and asks the agent to confirm
2. Agent presents preview to user, gets confirmation
3. Second call with `confirmed: true` executes the transaction

This prevents accidental execution — the agent must explicitly confirm after seeing the preview.

## Files

New package (location TBD — could be in `premiumjibles/vultiagent-cli` monorepo or separate repo):
```
src/
  index.ts              — MCP server entry point (stdio transport)
  tools/
    balances.ts         — get_balances, get_portfolio tools
    addresses.ts        — get_address tool
    send.ts             — send tool (with confirmation step)
    swap.ts             — swap, swap_quote tools
    vault-info.ts       — vault_info, supported_chains tools
  auth/
    credential-store.ts — Shared with CLI: keyring + env var credential reading
    config.ts           — Shared with CLI: ~/.vultisig/config.json reading
  lib/
    errors.ts           — MCP error formatting
package.json
```

Reference:
- `vultisig-sdk/clients/cli/src/commands/swap.ts` — swap flow SDK usage
- `vultisig-sdk/clients/cli/src/commands/transaction.ts` — send flow SDK usage
- `vultisig-sdk/clients/cli/src/commands/balance.ts` — balance query SDK usage
- MCP SDK docs: https://modelcontextprotocol.io/docs

## Acceptance Criteria

- [ ] MCP server starts via stdio and registers all tools with JSON Schema definitions
- [ ] `get_balances`, `get_portfolio`, `get_address`, `vault_info`, `supported_chains` work end-to-end
- [ ] `send` and `swap` work end-to-end with two-step confirmation pattern
- [ ] `swap_quote` returns quote without executing
- [ ] Server reads credentials from keyring (set by `vasig auth`) without prompting
- [ ] Falls back to VAULT_PASSWORD env var if keyring unavailable
- [ ] Returns clear error if no credentials found ("Run vasig auth first")
- [ ] All logging goes to stderr, only JSON-RPC on stdout
- [ ] Claude Code can discover and use the server via `claude mcp add vultisig -- npx @vultisig/mcp`
- [ ] Error responses are structured and actionable
- [ ] Lint and type-check pass

## Gotchas

- MCP stdio servers must not write ANYTHING to stdout except JSON-RPC — console.log will break the protocol. Use console.error or a stderr logger.
- The `@modelcontextprotocol/sdk` server class handles transport — don't implement JSON-RPC manually
- WASM libraries (DKLS, Schnorr) load async — SDK must be initialized before registering tools
- The SDK's `Vultisig` constructor takes `onPasswordRequired` callback — wire to credential store, not a prompt
- `keytar` (keyring access) requires native modules — may need prebuild configuration for npx usage
- For the two-step send/swap confirmation: MCP doesn't have built-in confirmation — implement via tool input params (`confirmed: true/false`) or separate preview/execute tools
- The shared credential-store code between CLI and MCP should be a lightweight extraction, not a full shared package initially — can be promoted to a package later if needed
