---
id: v-apmh
status: open
deps: []
links: []
created: 2026-05-03T22:45:41Z
type: bug
priority: 1
assignee: Jibles
---
# get_tx_status input validation — distinguish address from tx hash

## Problem
`get_tx_status` accepts an Ethereum wallet address as `tx_hash` and fails with a generic error. Felix's 2026-05-01 dump conv `8ae9bf54`:

```json
{ "tool_name": "get_tx_status",
  "input": { "chain": "Ethereum", "tx_hash": "0x62c4bd2f4842338C31374100ae661b99E5f639Bb" } }
```

That's the user's wallet address (40 hex chars), not a tx hash (64 hex chars). The tool returned `"Failed to fetch tx status"` — generic, not actionable.

Symptom: model passes the wrong identifier, gets a meaningless error, then either gives up or contradicts itself in user-visible text. The fix is input validation: reject 40-char hex with a helpful error pointing the model toward the right tool.

## Background
### Findings
- Tool definition lives somewhere in the MCP tool registry; spot-check during planning. Likely `mcp-ts/src/tools/utility/`.
- Ethereum addresses: 40 hex chars after `0x` (42 total). Tx hashes: 64 hex chars after `0x` (66 total). UTXO chains have their own tx-hash formats (Bitcoin: 64 hex, no prefix). Solana: base58.
- Same dump also has a separate generic `"Failed to fetch tx status"` for what looks like a legitimate hash (conv `ec56bde0`, hash `0x9561d8d816...`). That's a different bug (real RPC failure) — out of scope here unless coupled.

## Constraints
- Validation must be per-chain (UTXO hashes are different format from EVM hashes; Solana is base58).
- Don't reject legitimate edge cases (e.g. tx hashes without leading zeros that happen to be 40 chars after stripping — unlikely for EVM but worth checking).

## Non-goals
- Improving the generic "Failed to fetch tx status" error for legitimate tx-not-found cases.
- Resolving "looks like an address — did you mean to look up balance?" automatically. Just fail with a clear hint.

## Open Questions
- Where does `get_tx_status` live — `mcp-ts/src/tools/utility/` is likely. Confirm during planning.
- Strict vs loose validation — strict regex (`/^0x[0-9a-fA-F]{64}$/` for EVM) or just length check?
- Error message wording: name the issue ("looks like an address, not a tx hash") or just "invalid format"?

## Notes

**2026-05-05T05:32:36Z**

migrated to GH issue: https://github.com/vultisig/vultiagent-app/issues/432

**2026-05-05T05:34:06Z**

kept open locally — GH issue is a duplicate, local remains canonical scratch
