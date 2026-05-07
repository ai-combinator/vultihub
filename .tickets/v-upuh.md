---
id: v-upuh
status: closed
deps: []
links: []
created: 2026-05-03T22:42:01Z
type: bug
priority: 0
assignee: Jibles
---
# Address-book accepts short hex strings as Ethereum addresses

## Problem
Address-book validator accepts `0x235232664375` (14 hex chars) as a valid Ethereum address. Reproduced in Felix's 2026-05-01 dump, conv `f7b9b981` msg #14:

> User: `add 0x235232664375 as Omar in the address book`
> Agent: `Successfully added Omar (Ethereum) to your address book.`

Real Ethereum addresses are 40 hex chars after the `0x` prefix (42 total). The 14-char string passed validation, was saved, and is now indistinguishable from a real address in the book. If the user later tells the agent "send 0.1 ETH to Omar", the agent would try to send to that malformed address.

For comparison, the validator did correctly reject `0xe253425yg` later in the same conversation (non-hex chars), and rejected the same string for Bitcoin (`Bitcoin addresses typically start with 1, 3, or bc1`). So character validation works; some other property is missing.

## Background
### Findings
- Tool name in the dump is `address_book` with action `add`. Lives in agent-backend or mcp-ts side; spot-check during planning.
- The validator successfully checks hex-character composition (rejecting `y`, `g`).
- Tool call from dump:
  ```json
  { "tool_name": "address_book",
    "input": { "entry": { "name": "Omar", "chain": "Ethereum", "address": "0x235232664375" }, "action": "add" } }
  ```

### External Context
- EIP-55 checksum addresses + raw 0x40-hex are both valid forms for Ethereum.

## Constraints
- Don't break legitimate non-EVM chain addresses. Per-chain semantics matter: Bitcoin bc1 is variable-length, Solana base58 is variable, Cosmos bech32 is prefix+hash, etc.

## Non-goals
- Network-level address verification (does it have a balance? is it a contract?). Static format check only.
- ENS / SNS / TNS resolution.

## Open Questions
- Existing malformed entries already in users' address books — do we sweep and warn, or leave them?

## Notes

**2026-05-05T05:32:32Z**

migrated to GH issue: https://github.com/vultisig/vultiagent-app/issues/431

**2026-05-05T05:34:06Z**

kept open locally — GH issue is a duplicate, local remains canonical scratch

**2026-05-07T00:00:00Z**

Closed: fixed in agent-backend PR #298 (`fix(agent): validate EVM address-book entries`), merged 2026-05-05. GitHub issue vultisig/vultiagent-app#431 closed as stale/duplicate of the backend fix.
