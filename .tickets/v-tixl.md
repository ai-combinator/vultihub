---
id: v-tixl
status: closed
deps: []
links: []
created: 2026-04-14T23:35:21Z
type: task
priority: 1
assignee: Jibles
---
# CLI agent-friendliness pass: token registry, prompt leaks, structured echoes, hints, introspection

## Objective
Make `vsig` reliably usable by AI agents end-to-end by closing the gaps exposed during a real Claude Code session that swapped SOL→ETH on Arbitrum and then tried to swap ETH→USDC on Arbitrum. The architecture is sound (structured `VasigError`, JSON-default on non-TTY, `--non-interactive` flag, exit-code taxonomy) — this ticket closes the leaks around it so agents don't hang on hidden prompts, hit dead-end errors, or get inconsistent token resolution.

## Context & Findings

Observed during live session:
1. **Canonical tokens missing from Arbitrum.** `vsig tokens Arbitrum` returned ~empty. USDC (`0xaf88d065e77c8cC2239327C5EDb3A432268e5831`) had to be manually added.
2. **`tokens --add` crashed with interactive prompt in non-TTY context.** Exit code 7, "User force closed the prompt with 0 null", from a hidden `? Enter token name` prompt. Only succeeded when `--name` was passed explicitly.
3. **Silent success on `--add`.** Second attempt with all flags produced zero output — no confirmation, no resolved metadata echo.
4. **Address lookup fails after successful add.** `--to-token 0xaf88...5831` returned "Token not found" even after add; only `--to-token USDC` (symbol) worked.
5. **Error hint referenced internal API.** `TokenNotFoundError` hint said "Add it with `vault.addToken()`" — that's a JS method, not a CLI command an agent can run.
6. **Portfolio didn't include Arbitrum** until `chains --add Arbitrum` was run, and there was no hint suggesting it.

Root causes (verified in code):
- **`TokenDiscoveryService.ts:33-65`** — `resolveToken()` keys on `knownTokensIndex[chain][contractAddress.toLowerCase()]`. CLI-added tokens persist to `vultiagent-cli/src/auth/config.ts` but are never written into this index, so SDK-layer resolvers never see them.
- **`SwapService.ts:254-305`** (`resolveCoinInput`) — only paths: static `knownTokensIndex` → chain RPC metadata. No fallback to user config tokens.
- **Interactive prompt in SDK layer** — `tokens.ts:82` calls `vault.resolveToken()`, which appears to prompt for missing metadata via an SDK-level `inquirer` call. The CLI's `--non-interactive` guard doesn't extend into the SDK.
- **No token registry** — there is no static curated list of major tokens (USDC/USDT/WETH etc.) per chain bundled with the SDK.
- **`errors.ts:1-215`** — `VasigError` base already supports `hint` and `suggestions` fields; they're just under-used and occasionally reference non-CLI APIs.
- **No schema introspection command.** Agents currently need 2-3 `--help` round-trips to discover command shape.

Research anchors (clig.dev, Anthropic MCP docs, gh/stripe/aws CLI patterns): JSON-first dual-mode output, structured actionable errors with runnable hints, non-TTY fail-fast (never prompt), normalize-and-echo identifier resolution, machine-readable command catalogs, `--dry-run` that returns the full planned action.

**Rejected approaches:**
- Splitting into 3 tickets (token registry / prompt audit / introspection) — user requested a single coherent CLI pass; the changes share files and testing surface.
- Building a general "any ERC-20 auto-discovery" service — out of scope; a curated registry + user-add write-through solves the observed pain without opening a new surface area.
- Removing `--to-token` by-symbol support — breaking change, no benefit; fix address resolution to be consistent instead.

## Files
- `vultisig-sdk/packages/sdk/src/vault/services/TokenDiscoveryService.ts` — add static registry seeding + write-through API for user-added tokens.
- `vultisig-sdk/packages/sdk/src/vault/services/SwapService.ts` — extend `resolveCoinInput` (lines 254-305) to consult user-added tokens; normalize case on address lookups.
- `vultisig-sdk/packages/sdk/src/**` — audit for any `inquirer`/`prompts`/`readline` calls; remove all prompt paths from the SDK layer (SDK returns errors; CLI decides whether to prompt).
- `vultiagent-cli/src/commands/tokens.ts` (lines 72-118) — on `--add` success, push token into SDK index; return structured echo of resolved metadata.
- `vultiagent-cli/src/commands/swap.ts`, `send.ts` — echo resolved canonical form (route, from/to token, recipient) in success output; make `--dry-run` emit the unsigned tx; add `--idempotency-key`.
- `vultiagent-cli/src/lib/errors.ts` — replace all `hint` strings that reference JS APIs with runnable `vsig ...` commands; add hint to `TokenNotFoundError`, chain-not-enabled errors, etc.
- `vultiagent-cli/src/index.ts` — verify `--ci` alias wires to `--output json --non-interactive --quiet` end-to-end; add `schema` command.
- `vultiagent-cli/src/commands/schema.ts` (new) — emits full command catalog as JSON (`{command, description, args, flags, examples, output_schema, exit_codes, schema_version}`).
- Reference patterns: `vultiagent-cli/src/lib/output.ts` for format switching, `vultiagent-cli/src/commands/auth.ts` for correct TTY-gated prompt pattern.

## Acceptance Criteria
- [ ] Static token registry seeds canonical tokens (at minimum: USDC, USDT, WETH, native wrapped, per each EVM chain; USDC/USDT SPL for Solana) into `knownTokensIndex` at SDK init.
- [ ] `vsig tokens <Chain> --add <address>` writes the token into both CLI config and the in-session SDK `knownTokensIndex`, so subsequent `--to-token <address>` and `--to-token <symbol>` both resolve in the same process.
- [ ] Token resolution is case-insensitive for both symbols and addresses (EIP-55-checksummed addresses resolve identically to lowercased form).
- [ ] No `inquirer`/`prompts` call survives in the SDK layer; SDK functions return structured errors when metadata is missing, CLI decides whether to prompt.
- [ ] All prompt sites in the CLI layer gate on `!process.stdin.isTTY || program.opts().nonInteractive` and throw `UsageError` (exit 2) with a `hint` naming the missing flag when gated.
- [ ] Every mutating command (`tokens --add`, `chains --add`, `swap`, `send`) emits a structured JSON echo of the resolved canonical state on success in JSON mode.
- [ ] Every `VasigError.hint` is a runnable `vsig ...` command or a concrete next-step action — no references to `vault.addToken()` or other internal APIs. Verified by grep.
- [ ] `vsig schema --json` exists and returns the full command catalog with `schema_version`, flags, args, examples, exit codes, and output shape per subcommand.
- [ ] `--ci` flag is equivalent to `--output json --non-interactive --quiet` (already documented; verify wired through all commands).
- [ ] `swap --dry-run` and `send --dry-run` include the unsigned transaction in the JSON output so agents can display it before re-invoking with `--yes`.
- [ ] `swap` and `send` accept `--idempotency-key <nonce>`; a retried broadcast with the same key returns the prior tx hash instead of re-signing/re-broadcasting.
- [ ] When a user runs `portfolio` and has balance on a chain not enabled, include a `warnings` array in JSON output with a hint: `vsig chains --add <Chain>`.
- [ ] Lint and type-check pass across `vultisig-sdk` and `vultiagent-cli`.

## Gotchas
- `knownTokensIndex` lookups use `contractAddress.toLowerCase()` — when seeding from a static registry or from user input, normalize before keying or lookups silently miss.
- Arbitrum-native USDC (`0xaf88...5831`) is distinct from bridged USDC.e (`0xff97...`); registry must pick the canonical one and document it.
- `vault.resolveToken()` is called from `tokens.ts:82` before CLI-level non-interactive guards can help — the SDK-side prompt must be removed, not just the CLI-side gate tightened.
- Idempotency key needs to persist across CLI invocations (not just in-process) to actually prevent double-broadcasts on agent retry; a small keyed cache on disk keyed by `(vaultId, key)` → tx hash is the minimum.
- `--dry-run` JSON shape for swaps needs to include the full route (provider, hops, expected output, fees) plus the unsigned tx; agents rely on the route for user-facing previews.
- `schema` command output is itself an API — version it (`schema_version: "1"`) so changes can be pinned.

## Notes

**2026-04-22T01:12:40Z**

Closing per user direction (partially done). DONE: curated token registry populated in vultisig-sdk (USDC/USDT/WETH per chain); schema introspection command exists at clients/cli/src/commands/schema.ts; non-TTY requireInteractive() guard in place at tokens.ts:87. REMAINING: user-config tokens still don't write through into TokenDiscoveryService's knownTokensIndex; VasigError hints may still reference non-CLI APIs (e.g. vault.addToken()). Spin a fresh ticket if residual gaps bite an agent session.
