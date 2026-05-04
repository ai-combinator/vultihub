---
id: v-giou
status: open
deps: []
links: []
created: 2026-04-29T05:39:16Z
type: task
priority: 2
assignee: Jibles
---
# audit: apply 'user decision vs plumbing' lens across remaining tool schemas

## Objective

Investigation, not implementation. Survey the remaining LLM-facing tool schemas across `mcp-ts/src/tools/` (and the agent-backend prompt sections that prop them up) through the lens we just applied to `execute_send`. Produce a ranked list of prime targets where the same pattern would pay off, with rough blast-radius estimates.

This ticket exists to encode the methodology so the audit can happen as a single coherent pass, rather than us re-deriving the principles on every individual tool.

## The lens we're applying

A tool schema field exists for one of two reasons: it encodes a **user decision** (what the user actually wants), or it encodes **implementation plumbing** (what the server needs to build the transaction). The LLM should only see fields in the first category. Everything else is footgun surface.

### Concrete tells that a field is plumbing in disguise

- The param description includes warnings like "do not omit, do not fabricate" or "copy from the user's context" — the server already has it; we're asking the LLM to copy a value the server could just inject.
- The field is documented as "auto-fetched server-side when omitted" — the override path is unused in practice; real users never say "send 0.1 ETH with priority fee 2 gwei." The field exists "just in case" but its only real consumer is the LLM mis-populating it.
- The prompt has a section coaching the LLM about how to populate a specific field (cross-chain destination rules, address book usage, EVM address universality, etc.) — that coaching is the prompt compensating for a schema that puts the wrong question to the model.
- The error returned when the field is missing is generic schema validation ("required at recipient") rather than behavioral coaching ("ask the user — never fabricate"). Generic errors leave the LLM nothing to do but retry; behavioral errors give it a clear next step.
- Multiple overlapping tools accept similar fields (e.g. `execute_send` + `build_evm_send` + `build_swap_tx`) — the routing layer that disambiguates them (intent guards, tool filters, forced `tool_choice`) is a downstream consequence of the schema being wider than the user-decision surface.

### What stays in the LLM's input schema

The user's actual decision: which asset, how much, to whom, optional memo. That's it. Everything else — `from`, gas, nonce, denom, account_number, sequence, fee_rate, token_address, token_decimals, family-specific overrides — is server-resolvable from vault context, RPC, or token registry.

### What address-book / contact resolution should look like

The `recipient`-style fields should accept either an address or a contact name, with server-side resolution and structured "no match, here's what's available" errors when the name doesn't resolve. That removes an entire class of prompt coaching about address book usage.

### Where forced tool-choice belongs

It usually doesn't. Forced `tool_choice=required` removes the LLM's ability to reply with text, which means it can't ask the user a clarifying question even when it should. Deterministic clarifier short-circuits (resolver-written plain text, LLM bypassed) are a separate, useful pattern — keep those. Lock-out is the dangerous one.

## Background — the pattern PR this generalizes

The first application of this lens was the `execute_send` rework (PR landing alongside this ticket): dropped `from`, gas/nonce/EIP-1559 plumbing, denom, fee_rate, token_address/decimals from the LLM-visible schema; accepted contact-name recipients via address-book lookup; dropped the intent guard's forced-tool-choice + tool-list narrowing while keeping the deterministic clarifier short-circuit; closed the `TokenKindNative` bypass.

Result: smaller schema, server-side resolution of derivable fields, behavioral errors instead of generic validation, and the prompt sections that coached for the now-removed fields came down naturally.

## Areas to investigate

In rough priority order based on user-flagged interest and likely footgun density:

1. **`execute_swap`** — explicitly flagged. `sender` likely has the same problem as `from`. `destination` has cross-chain resolution rules that currently live in the prompt (lines 163-168 of `prompt.go`) but could be server-resolvable for known chains (vault context already has per-chain addresses). Likely also has plumbing fields worth removing.

2. **Gated escape-hatch send tools** — `build_spl_transfer_tx`, `build_xrp_send`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build_cw20_transfer`. Each individually small, but collectively they re-introduce the footgun surface that `execute_send`'s simplification eliminated. Same fields (`from`, decimals, contract addresses) probably present.

3. **Other `execute_*` tools** that emit `pre_sign` envelopes — particularly anything in the `execute/` directory we haven't touched. Same lens: required fields with "do not fabricate" warnings? Auto-fetched plumbing exposed as overrides?

4. **Read-only tools that take `address`** (balance queries, account lookups, etc.) — do they require an LLM-supplied address when vault context already has it? Same `injectVaultArgs` `_meta` mechanism could apply.

5. **Anywhere else the prompt has coaching for specific tool fields** — `prompt.go` sections like "MCP Address Resolution", "EVM address universality", address-book usage hints. Each is a tell that some tool's schema is putting the wrong question to the LLM.

## What good output looks like

A short report (notes / draft tickets / inline annotations — whatever's lightest) that for each surveyed tool answers:

- Which fields are user decisions, which are plumbing.
- For plumbing fields: where would the server resolve them from? (vault context, RPC, registry, etc.) Does that resolution path already exist for sibling tools?
- Which prompt sections become deletable if this tool's schema shrinks.
- Rough blast-radius tier (matches the framework from the `execute_send` PR: pure subtraction of known footgun → low-risk plumbing removal → broader refactor).
- Any genuine plumbing fields the LLM legitimately needs (counterexamples — important to surface so we don't over-rotate).

The output is *input to follow-up tickets*, not a single mega-PR. Each surveyed tool that warrants action gets its own scoped ticket sized like the `execute_send` work.

## Out of scope

- Implementation. This is reconnaissance.
- The intent guard / send_intent.go itself — that gets the same treatment as `execute_send` did (mech 1 dropped, mech 2 + native bypass fix kept). If the audit reveals intent-guard-shaped layers around other tools, flag them, but the existing one is handled.
- Prompt restructure as a standalone refactor. The prompt will continue to shrink naturally as load-bearing rules for each toolʼs schema become obsolete. Don't pre-empt that with a cosmetic pass.

## Gotchas

- **Don't confuse "the LLM gets it wrong sometimes" with "the LLM should never see it."** Some fields are genuine user decisions that are simply hard. The lens is "could the server know this without the LLM telling it?" — not "has the LLM ever fumbled this field?"
- **Auto-fetched-when-omitted is a strong tell, but verify the auto-fetch path actually works for every chain family before recommending removal.** The `execute_send` gas auto-fetch has been in production; an analogous field on a less-trafficked tool may be paper-only.
- **Server-side resolution requires the data to actually be in scope.** Vault context has addresses + balances; it does not have the user's external counterparties. If a field's resolution requires data the server doesn't have, it stays.
- **Some forced tool-choice usage is legitimate** (e.g., locking the model to a specific surface during a guided multi-step flow). Audit each instance individually rather than blanket-removing.
