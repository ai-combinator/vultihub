---
id: v-giou
status: closed
deps: []
links: []
created: 2026-04-29T05:39:16Z
type: task
priority: 0
assignee: Jibles
---
# audit: apply 'user decision vs plumbing' lens across tool contracts

## Objective

Investigation, not implementation. Survey the remaining LLM-facing tool contracts across `mcp-ts/src/tools/` and the agent-backend prompt/tool plumbing that props them up. Apply the `execute_send` lens broadly:

1. **Input side:** identify tool schema fields the LLM is asked to provide even though the backend could inject or resolve them from vault context, RPC, token registry, address book, balances, or existing server state.
2. **Output side:** identify tool results that dump more data back into the model than the next reasoning step needs, especially raw search/API payloads, full quote/debug objects, repeated context, or presentation-card fields that the UI already consumes directly.

Produce a ranked set of concrete follow-up tickets where shrinking the tool contract would reduce token cost, reduce loops, or delete prompt instructions. Each recommendation should include expected value, difficulty, blast radius, gotchas, and the evidence that supports it.

This ticket exists to encode the methodology so the audit can happen as a single coherent pass, rather than us re-deriving the principles on every individual tool.

## The lens we're applying

A tool contract exists at the LLM boundary for one of two reasons: it encodes a **user decision** the model must infer from conversation, or it exposes **implementation plumbing** the server needs to execute a request. The LLM should only see the first category. Everything else is footgun surface and token tax.

This applies in both directions:

- **Inputs:** do not ask the model to copy or invent values the backend already knows.
- **Outputs:** do not send the model large raw payloads when a smaller typed result, capped list, deterministic UI card, or server-side summary would preserve the decision-making signal.

### Concrete tells that a field is plumbing in disguise

- The param description includes warnings like "do not omit, do not fabricate" or "copy from the user's context" — the server already has it; we're asking the LLM to copy a value the server could just inject.
- The field is documented as "auto-fetched server-side when omitted" — the override path is unused in practice; real users never say "send 0.1 ETH with priority fee 2 gwei." The field exists "just in case" but its only real consumer is the LLM mis-populating it.
- The prompt has a section coaching the LLM about how to populate a specific field (cross-chain destination rules, address book usage, EVM address universality, etc.) — that coaching is the prompt compensating for a schema that puts the wrong question to the model.
- The error returned when the field is missing is generic schema validation ("required at recipient") rather than behavioral coaching ("ask the user — never fabricate"). Generic errors leave the LLM nothing to do but retry; behavioral errors give it a clear next step.
- Multiple overlapping tools accept similar fields (e.g. `execute_send` + `build_evm_send` + `build_swap_tx`) — the routing layer that disambiguates them (intent guards, tool filters, forced `tool_choice`) is a downstream consequence of the schema being wider than the user-decision surface.

### Concrete tells that a tool result is too broad

- The result returns a raw third-party API response or full search result list when the model only needs a ranked subset, a chosen quote, or a small disambiguation set.
- The result includes fields consumed only by mobile UI cards, signing payloads, analytics, or persistence, then sends those same fields back through the LLM context.
- The result repeats vault context, token metadata, balances, or prompt-known facts already present in the request.
- The result includes debug data, full calldata envelopes, provider traces, nested quote internals, or error stacks when the model only needs a status, reason, and next allowed action.
- The result is large enough to make retries/second-pass narration expensive, especially when a successful presentational tool output will be followed by a canned response or UI card.

### What stays in the LLM's input schema

The user's actual decision: which asset, how much, to whom, optional memo. That's it. Everything else — `from`, gas, nonce, denom, account_number, sequence, fee_rate, token_address, token_decimals, family-specific overrides — is server-resolvable from vault context, RPC, or token registry.

For non-send tools, the same principle applies: keep market/search/filter parameters that express the user's intent; remove addresses, keys, chain plumbing, quote internals, display-only fields, and override knobs the user did not actually ask to control.

### What should come back from tools

Tool results should be shaped for the next model decision, not for every downstream consumer. When the mobile app or backend needs rich data, keep it in structured sidecar fields, metadata, persistence, or card payloads that do not have to be re-read by the model.

Good LLM-visible outputs are usually:

- a small success/failure status;
- the selected or top few relevant options;
- model-readable next action guidance;
- compact identifiers or labels the model may reference;
- enough evidence to avoid fabrication, but not full raw payloads.

### What address-book / contact resolution should look like

The `recipient`-style fields should accept either an address or a contact name, with server-side resolution and structured "no match, here's what's available" errors when the name doesn't resolve. That removes an entire class of prompt coaching about address book usage.

### Where forced tool-choice belongs

It usually doesn't. Forced `tool_choice=required` removes the LLM's ability to reply with text, which means it can't ask the user a clarifying question even when it should. Deterministic clarifier short-circuits (resolver-written plain text, LLM bypassed) are a separate, useful pattern — keep those. Lock-out is the dangerous one.

## Background — the pattern PR this generalizes

The first application of this lens was the `execute_send` rework (PR landing alongside this ticket): dropped `from`, gas/nonce/EIP-1559 plumbing, denom, fee_rate, token_address/decimals from the LLM-visible schema; accepted contact-name recipients via address-book lookup; dropped the intent guard's forced-tool-choice + tool-list narrowing while keeping the deterministic clarifier short-circuit; closed the `TokenKindNative` bypass.

Result: smaller schema, server-side resolution of derivable fields, behavioral errors instead of generic validation, and the prompt sections that coached for the now-removed fields came down naturally.

Recent token-cost work adds the cost reason to do this now: normal backend-e2e probes showed prompt-side spend dominated by stable system/tool context, while historical outliers suggest raw tool/search outputs can still create very expensive individual turns. This audit should therefore treat tool contract cleanup as both a reliability project and a cost-control project.

## Areas to investigate

In rough priority order based on user-flagged interest and likely footgun density:

1. **`execute_swap`** — explicitly flagged. `sender` likely has the same problem as `from`. `destination` has cross-chain resolution rules that currently live in the prompt (lines 163-168 of `prompt.go`) but could be server-resolvable for known chains (vault context already has per-chain addresses). Likely also has plumbing fields worth removing.

2. **Gated escape-hatch send tools** — `build_spl_transfer_tx`, `build_xrp_send`, `build_trc20_transfer`, `build_ton_jetton_transfer`, `build_sui_token_transfer`, `build_cw20_transfer`. Each individually small, but collectively they re-introduce the footgun surface that `execute_send`'s simplification eliminated. Same fields (`from`, decimals, contract addresses) probably present.

3. **Other `execute_*` tools** that emit `pre_sign` envelopes — particularly anything in the `execute/` directory we haven't touched. Same lens: required fields with "do not fabricate" warnings? Auto-fetched plumbing exposed as overrides?

4. **Read-only tools that take `address`** (balance queries, account lookups, etc.) — do they require an LLM-supplied address when vault context already has it? Same `injectVaultArgs` `_meta` mechanism could apply.

5. **Search and market tools** — especially Polymarket, token search, quote search, price lookup, portfolio/balance fan-out, and any tool that wraps a third-party API. Check whether they return raw/capped/summarized results, whether they can explode on broad queries, and whether the LLM-visible result is smaller than the backend/UI payload.

6. **Presentation-card tools** — send/swap/bridge/schedule/receive/card-producing flows. Verify whether the tool result sent to the model includes fields that only the frontend needs, and whether final narration still needs any of those fields once canned response work lands.

7. **Anywhere else the prompt has coaching for specific tool fields** — `prompt.go` sections like "MCP Address Resolution", "EVM address universality", address-book usage hints, cross-chain destination rules, search-result handling, or card restatement rules. Each is a tell that some tool's contract is putting the wrong question or too much raw data in front of the model.

## What good output looks like

A concrete audit report that is ready to convert into tickets. For each surveyed tool or tool family, answer:

- Which fields are user decisions, which are plumbing.
- For plumbing fields: where would the server resolve them from? (vault context, RPC, registry, etc.) Does that resolution path already exist for sibling tools?
- What does the tool return to the LLM today? Is it minimal, capped, and decision-shaped, or is it a raw/downstream payload?
- If the output is too broad, what should the LLM-visible result become, and where should the richer data live instead?
- Which prompt sections become deletable if this tool's schema shrinks.
- Rough blast-radius tier (matches the framework from the `execute_send` PR: pure subtraction of known footgun -> low-risk plumbing removal -> broader refactor).
- Estimated value: high/medium/low for token reduction, loop reduction, and prompt deletion.
- Estimated difficulty: small/medium/large, with the risky files called out.
- Test shape: exact unit/e2e/probe that would prove the follow-up ticket worked.
- Any genuine plumbing fields the LLM legitimately needs (counterexamples — important to surface so we don't over-rotate).

The output is *input to follow-up tickets*, not a single mega-PR. Each surveyed tool that warrants action gets its own scoped ticket sized like the `execute_send` work.

Minimum deliverable:

- ranked top 10 tool-contract cleanup opportunities;
- at least top 5 written in ticket-ready form;
- explicit "do nothing" list for tools that are already fine;
- list of prompt sections likely deletable after the recommended tickets land;
- notes on any telemetry or usage-audit gaps that made impact hard to estimate.

Suggested scoring rubric:

```text
value_score =
  0-3 token savings
+ 0-3 loop/retry reduction
+ 0-2 prompt deletion potential
+ 0-2 safety/reliability improvement

difficulty_score =
  1 small localized schema/result trim
  2 server-side resolver already exists, wiring needed
  3 resolver exists but needs chain-family parity tests
  4 new resolver or downstream payload split needed
  5 signing/card payload compatibility risk
```

## Out of scope

- Implementation. This is reconnaissance.
- The intent guard / send_intent.go itself — that gets the same treatment as `execute_send` did (mech 1 dropped, mech 2 + native bypass fix kept). If the audit reveals intent-guard-shaped layers around other tools, flag them, but the existing one is handled.
- Prompt restructure as a standalone refactor. The prompt will continue to shrink naturally as load-bearing rules for each toolʼs schema become obsolete. Don't pre-empt that with a cosmetic pass.
- Progressive prompt/tool disclosure as a routing system. That belongs to `v-vpeb`; this ticket should identify schema/result cleanup that makes those prompt packs smaller and safer.
- Deterministic final responses for presentational cards. That belongs to `v-glle`; this ticket may identify which card tool outputs become LLM-invisible once that lands.

## Gotchas

- **Don't confuse "the LLM gets it wrong sometimes" with "the LLM should never see it."** Some fields are genuine user decisions that are simply hard. The lens is "could the server know this without the LLM telling it?" — not "has the LLM ever fumbled this field?"
- **Auto-fetched-when-omitted is a strong tell, but verify the auto-fetch path actually works for every chain family before recommending removal.** The `execute_send` gas auto-fetch has been in production; an analogous field on a less-trafficked tool may be paper-only.
- **Server-side resolution requires the data to actually be in scope.** Vault context has addresses + balances; it does not have the user's external counterparties. If a field's resolution requires data the server doesn't have, it stays.
- **Some forced tool-choice usage is legitimate** (e.g., locking the model to a specific surface during a guided multi-step flow). Audit each instance individually rather than blanket-removing.
- **Do not hide evidence the model needs to answer truthfully.** Shrinking outputs should preserve the facts needed for final claims, error explanations, and user choices.
- **Do not break UI/signing payloads by shrinking the wrong layer.** If the frontend needs rich fields, split LLM-visible content from structured metadata instead of deleting the data entirely.
- **Result caps need deterministic behavior.** For search/list tools, define ordering, truncation, and continuation behavior so capping does not make the model randomly miss the best result.

## Notes

**2026-05-07T06:22:44Z**

Retired after audit crystallised into concrete follow-up tickets: v-hykj, v-mnan, v-qzbp, v-tkhz, plus existing v-skxz for escape-hatch sends.
