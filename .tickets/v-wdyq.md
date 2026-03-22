---
id: v-wdyq
status: open
deps: []
links: [tic-ef85]
created: 2026-03-22T00:38:08Z
type: feature
priority: 1
assignee: Jibles
---
# CLI: Agent-friendly Vultisig CLI with keyring auth (vultiagent-cli)

## Objective

Build a standalone, agent-first CLI for Vultisig (`vultiagent-cli`, command: `vasig`) that enables AI agents like Claude Code to perform crypto operations (balance, send, swap) without interactive prompts. Includes a keyring-based auth system where humans authenticate once in a real terminal, and agents consume stored credentials transparently.

Repo: https://github.com/premiumjibles/vultiagent-cli
Published name: `vultiagent-cli` (binary: `vasig`)

## User Story

An AI agent (Claude Code, MCP server, or any automated tool) needs to check balances, send tokens, and execute swaps through Vultisig vaults without interactive password prompts or access to raw credentials.

## Design Constraints

- Depends on `@vultisig/sdk@0.5.0+` and `@vultisig/rujira@1.0.0+` from npm (not workspace links)
- Must be fully non-interactive when credentials are available — zero `inquirer` prompts in agent mode
- `vasig auth` is the ONLY interactive command — it's always run by a human in a real terminal
- All other commands fail with structured errors if auth is missing
- References `vultisig-sdk/clients/cli/` for SDK usage patterns (swap flow, send flow, balance queries) but is a fresh implementation, not a fork
- Auth system stores passwords in OS system keyring (not plaintext files) — designed so swapping to server-issued session tokens later is a small change in one function
- Must handle both encrypted and unencrypted .vult files

## Context & Findings

### Why a new repo instead of modifying the existing CLI

The existing `@vultisig/cli` at `vultisig-sdk/clients/cli/` works but is built for human-interactive use:
- Uses `inquirer` for password prompts, confirmations, vault selection — all of which hang or fail when run by an agent
- `password-manager.ts` resolution chain: cache → env var → interactive prompt (no keyring)
- Import flow (`vault-management.ts:249`) always prompts for password interactively
- No structured error output (errors are chalk-formatted strings, not JSON)
- Limited `--help` descriptions without examples

We don't have full write access to the vultisig-sdk repo. A standalone repo gives full control and avoids blocking on upstream approval. Both packages consume `@vultisig/sdk` from npm, so they're independent.

### Auth system design

**`vasig auth` flow:**
1. Searches common locations for `.vult` files: `~/.vultisig/`, `~/Documents/Vultisig/`, current directory, and any path in existing config
2. Presents found vaults, or accepts `--vault-file <path>` to skip discovery
3. Loads the vault file:
   - If unencrypted: loads directly, no decryption password needed
   - If encrypted: prompts for decryption password (masked), decrypts to validate
4. Prompts for VultiServer password (masked) — this is the password sent to the server during 2-of-2 MPC signing
5. Validates server auth works (test request to VultiServer)
6. Stores in system keyring:
   - `vultisig/<vaultId>/server` → server password
   - `vultisig/<vaultId>/decrypt` → decryption password (only if vault is encrypted)
7. Stores in `~/.vultisig/config.json`:
   - Vault file path, vault ID, vault name (not sensitive)
8. Prints summary: vault name, type, chains, "Ready for agent use"

**Credential resolution (all non-auth commands):**
```
getServerPassword(vaultId):
  1. keyring lookup (vultisig/<vaultId>/server)
  2. VAULT_PASSWORD env var (fallback for CI/testing)
  3. throw AuthRequiredError with message "Run vasig auth"

getDecryptionPassword(vaultId):
  1. keyring lookup (vultisig/<vaultId>/decrypt)
  2. VAULT_DECRYPT_PASSWORD env var
  3. throw AuthRequiredError with message "Run vasig auth"
```

**Future upgrade path (when server session tokens land via tic-ef85):**
Change `getServerPassword` to `getServerCredential` — look up session token from keyring first, fall back to password exchange. One function change, not a refactor.

### CLI agent-friendliness requirements

- Every command has `--help` with usage, required/optional args, and 2-3 examples
- `--output json` gives structured JSON with consistent schema across all commands
- `--yes` skips confirmation on send/swap (agent passes this flag)
- No interactive prompts anywhere except `vasig auth`
- Grouped help categories: Authentication, Wallet, Trading, Vault Management
- Exit codes: 0 = success, 1 = bad args/usage, 2 = auth required, 3 = network/server error
- JSON errors always include `code`, `message`, and `hint` fields

### Existing CLI patterns to reference

The existing CLI at `vultisig-sdk/clients/cli/src/commands/` has correct SDK usage patterns:
- `swap.ts` — full swap flow: quote → preview → prepare tx → sign → broadcast, handles approval payloads
- `transaction.ts` — send flow: prepare → confirm → sign → broadcast
- `balance.ts` — balance/portfolio queries
- `vault-management.ts` — import vault from .vult file (`sdk.importVault(content, password)`)
- `sign.ts` — arbitrary byte signing

These are the reference for how to correctly call `@vultisig/sdk` methods. The SDK patterns (vault.getSwapQuote, vault.prepareSwapTx, vault.sign, vault.broadcastTx) are the same regardless of which CLI wraps them.

### Keyring implementation

Use `keytar` npm package (or `@aspect-build/keytar`) for cross-platform keyring access:
- Linux: libsecret / GNOME Keyring / KDE Wallet
- macOS: Keychain
- Windows: Credential Manager

The keyring is encrypted at rest by the OS, tied to the user's login session. This is the same mechanism `gh` uses for GitHub tokens.

## Files

New repo structure (`premiumjibles/vultiagent-cli`):
```
src/
  index.ts              — Entry point, Commander.js program setup
  commands/
    auth.ts             — vasig auth, auth status, auth logout
    balance.ts          — vasig balance (portfolio view)
    send.ts             — vasig send (token transfers)
    swap.ts             — vasig swap, swap-quote, swap-chains
    vault.ts            — vasig vaults, vault info
    addresses.ts        — vasig addresses
  auth/
    credential-store.ts — Keyring read/write, env var fallback
    vault-discovery.ts  — Find .vult files on disk
    config.ts           — ~/.vultisig/config.json read/write
  lib/
    output.ts           — JSON/table output formatting
    errors.ts           — Structured error types with codes
  types.ts              — Shared types
package.json
tsconfig.json
README.md
```

Reference (read-only, for SDK usage patterns):
- `vultisig-sdk/clients/cli/src/commands/swap.ts` — swap flow
- `vultisig-sdk/clients/cli/src/commands/transaction.ts` — send flow
- `vultisig-sdk/clients/cli/src/commands/balance.ts` — balance queries
- `vultisig-sdk/clients/cli/src/commands/vault-management.ts` — vault import
- `vultisig-sdk/clients/cli/src/core/password-manager.ts` — current password resolution (what we're replacing)

## Acceptance Criteria

- [ ] `vasig auth` discovers vault files, handles encrypted/unencrypted, stores credentials in system keyring
- [ ] `vasig auth status` shows authenticated vaults
- [ ] `vasig auth logout` clears keyring entries
- [ ] `vasig balance` shows balances (table + JSON output)
- [ ] `vasig send` prepares, signs, and broadcasts a transfer with `--yes` flag
- [ ] `vasig swap` gets quote, prepares, signs, and broadcasts a swap with `--yes` flag
- [ ] `vasig addresses` shows derived addresses for active chains
- [ ] All commands work fully non-interactively when keyring credentials exist
- [ ] All commands fail with structured JSON error (code + message + hint) when auth is missing
- [ ] `vasig --help` and `vasig <command> --help` show grouped categories, args, and examples
- [ ] `--output json` works on all commands with consistent schema
- [ ] Exit codes: 0 success, 1 bad args, 2 auth required, 3 network error
- [ ] Published to npm and installable via `npm install -g vultiagent-cli`
- [ ] Lint and type-check pass

## Gotchas

- `@vultisig/sdk` WASM modules (DKLS, Schnorr) need Node.js >=20 — set in engines field
- The SDK's `Vultisig` class requires `onPasswordRequired` callback at init — wire this to the keyring credential store, NOT to an interactive prompt
- `sdk.importVault(content, password)` handles both encrypted and unencrypted — pass `undefined` for password if unencrypted
- `sdk.isVaultEncrypted(content)` exists and should be used during `vasig auth` to detect encryption
- The server password and the vault decryption password may be different — `vasig auth` should prompt for them separately
- `keytar` requires native compilation — consider `@aspect-build/keytar` or prebuild binaries for easier install
- The CLI's `coordinateFastSigning` in ServerManager sends `vault_password` in plaintext over HTTPS — this is the current server API, not something we can change
- For SecureVault (N-of-M), signing requires multiple devices via relay — initial scope should focus on FastVault (2-of-2) which is the agent-friendly path
