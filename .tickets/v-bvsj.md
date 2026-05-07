---
id: v-bvsj
status: closed
deps: []
links: []
created: 2026-04-23T23:07:21Z
type: task
priority: 2
assignee: Jibles
---
# agent-backend: CRUD action-discriminator (vault_coin, vault_chain, address_book) with aliases + JSON title|name compat

## Objective

Consolidate the CRUD tool surface: collapse `add_coin`/`remove_coin` → `vault_coin(action)`, `add_chain`/`remove_chain` → `vault_chain(action)`, `address_book_add`/`address_book_remove` → `address_book(action)`. That's a genuine 8→4 tool collapse on the CRUD surface — maps directly to the "better LLM selection on weak models" goal. **This is the only net-negative chunk from the original #128 that's worth keeping**, provided we add the migration scaffolding reviewers flagged as missing.

## Context & Findings

### Reviewer findings this PR must fix

**Gomes preferably-blocking on `types.go:55` (caught 4 rounds):**
> "wire rename Title → json:\"name\" with no backward-compat shim. Every deployed SDK, every cached MessageContext in Redis, every persisted agent_messages.metadata row, every tx_proposals.action_params blob that carries an address-book entry still sends \"title\". Go's JSON decoder silently drops the mismatch → Title == \"\" → `send to Alice` resolves to nothing."

**Fix**: `AddressBookEntry` struct accepts BOTH `"title"` and `"name"` on input (custom UnmarshalJSON or dual-tag via intermediate struct), emits `"name"` on output. Until next major API version.

**Gomes preferably-blocking on `message_parts.go:69`:**
> "preferably-blocking: historical tool-part replay is broken for 12 of 13 renames… Persisted historical messages replay as `tool-<old_name>`. Frontend toolUIRegistry.ts in vultiagent-app #164 only maps new names. Card vanishes, timeline entry renders generic, worst case blank message."

**Fix**: `internal/storage/postgres/convert.go` rewrites old tool-part names to new names at read time for ALL CRUD renames covered by this ticket (not just `schedule_task` as #128 did). Specifically: `tool-add_coin` → `tool-vault_coin` (with action: "add"), `tool-remove_coin` → `tool-vault_coin` (with action: "remove"), same pattern for chain + address_book.

**Gomes preferably-blocking on `agent.go:2101`:**
> "preferably-blocking: dedup is keyed on raw tool names with no alias map. Only `schedule_task` ↔ `schedule_create` has this hand-rolled shim. Every other rename has no alias… action double-fires… two preview confirmations."

**Fix**: extend the dedup alias map in `processActions` to cover all CRUD renames in this ticket.

**Gomes preferably-blocking on `tool_classify.go:65` + `:229`:**
> "`knownActions` is frozen at pre-rename names… Every `action_result` PostHog event with a new name gets stamped `action_type=\"unknown\"`."
> "Excludes `execute_send`, `execute_swap`, … Consequence: PostHog `tool_call.chain` is EMPTY for every send/swap through the new path."

**Fix**: update `knownActions` to include BOTH old and new names (map new → same category as old); `chainFromMCPTool` / `chainFromArgs` handles the new action-discriminator shape.

### What this ticket covers

**Cherry-pick from `refactor/v-pxuw-tool-consolidation` (agent-backend):**
- `7132272` — refactor(agent): align storage + scheduler for schedule_create rename (adapt: add coverage for vault_coin, vault_chain, address_book renames too, not just schedule)
- `aa73e80` — refactor(agent): consolidate tool layer per v-pxuw Task 6 (the action-discriminator schemas) — SCOPE: keep only vault_coin / vault_chain / address_book changes; drop the schedule_task / check_credits / plugin_* renames from this PR (those are pure renames, defer)
- `a85cf62` — fix(agent): align address_book wire format on entry.name (v-pxuw C5) — apply WITH dual-tag input compatibility per Gomes
- `4cde987` — fix(storage): replay all v-pxuw Task 6 renamed tool parts (C6) — scope to CRUD renames in this ticket only

**Adjust during cherry-pick:**
- `AddressBookEntry`: custom UnmarshalJSON accepting `"title"` OR `"name"` → stores in unified field; MarshalJSON emits `"name"`
- `convert.go` historical replay covers all 3 CRUD renames × 2 directions (add/remove → vault_coin, etc.)
- `processActions` dedup alias map extended for CRUD renames
- `tool_classify.go` `knownActions` covers both old and new names as same category; new action-discriminator shape recognized for chain attribution

### Scope boundaries

**Include**:
- `vault_coin`, `vault_chain`, `address_book` action-discriminated client-side tool schemas
- Server-side read-time aliases for all 3 renames in `convert.go`
- Dual-tag JSON compat for `AddressBookEntry`
- Dedup alias map extension
- Analytics (`tool_classify.go`) coverage for new names

**Exclude** (defer to later or skip entirely):
- `schedule_task` → `schedule_create` rename — pure rename, no value
- `check_credits` / `get_spending_history` → `credits` rename — pure rename
- `create_policy` / `delete_policy` → `plugin_create_policy` / `plugin_delete_policy` — pure rename
- `store_protocol_state` → `protocol_state_store` family — pure rename
- `AppTool` / `Tools` wire field removal (C15)
- SSE payload type deletions (C16)
- Benchmark testdata delete (C17)
- `decimalToBaseUnits` deletion
- `buildAgentTools() → agentTools()` rename

Pure renames are deferred indefinitely — not worth the migration cost.

### Dependency chain

- **Must depend on ticket 13 (vultiagent-app CRUD cards)** being raised at the same time, since they're the consumer
- Clients currently on older app builds still send `add_coin`/`remove_coin` tool calls — the alias table serves them until app-store ramp
- **Cannot merge before ticket 13 is in an app-store build at least in TestFlight**

## Files

**Modified:**
- `internal/service/agent/action_tools.go` — replace 3 pairs of tools with 3 action-discriminated tools (register BOTH old and new for one release cycle)
- `internal/service/agent/actions.go` — dispatch handlers for new action-discriminated shape
- `internal/service/agent/agent.go` — extend `processActions` dedup alias map
- `internal/service/agent/types.go` — `AddressBookEntry` with dual-tag JSON + custom Unmarshal
- `internal/service/agent/tool_classify.go` — extend `knownActions`, `chainFromMCPTool`, `chainFromArgs`
- `internal/storage/postgres/convert.go` — historical replay for 3 CRUD renames
- `internal/storage/postgres/message_parts.go` — tool-part name rewrite

**Tests:** extend `action_tools_test.go`, `agent_test.go`, `tool_classify_test.go`, `convert_test.go`, `message_parts_test.go`, `process_actions_test.go`.

## Acceptance Criteria

- [ ] PR opened as **draft** against agent-backend main, title `feat(agent): CRUD action-discriminator tools (vault_coin, vault_chain, address_book) with aliases`
- [ ] `vault_coin(action)`, `vault_chain(action)`, `address_book(action)` registered
- [ ] OLD tools (`add_coin`, `remove_coin`, `add_chain`, `remove_chain`, `address_book_add`, `address_book_remove`) remain dual-registered for one release cycle (tag with TODO referencing future retirement ticket)
- [ ] `AddressBookEntry` accepts BOTH `"title"` and `"name"` on JSON input; emits `"name"` on output — unit test covers both input forms
- [ ] `convert.go` rewrites all 3 CRUD rename variants at read time; test coverage includes historical messages with old tool-part names rendering correctly
- [ ] `processActions` dedup alias map covers all 3 CRUD renames (no double-fire when both old and new appear)
- [ ] `tool_classify.go` / `knownActions` map handles both old and new names → same category; chain attribution works for the new action-discriminator shape
- [ ] The `TestMessageFromDB_LegacyDataConfirmationMigratesToTool` test — which #128 deleted — is RESTORED or replaced by equivalent coverage
- [ ] `go test ./...`, `golangci-lint run`, `go build ./...` clean
- [ ] PR body has **"Blockers" section**: "Do NOT merge before ticket 13 is in an app-store build (TestFlight ok); pure CRUD renames (schedule_task, credits, plugin_*) explicitly out of scope"

## Gotchas

- Dual-tag JSON in Go: the cleanest way is a custom `UnmarshalJSON` that tries both keys, then normalizes into the struct field. Don't use two tags on the same field (Go only honors the first).
- Historical replay must be idempotent — replaying an already-new-named part should not double-rewrite
- Dedup alias map key direction matters: map OLD → NEW or NEW → OLD? Check `agent.go` pattern; follow existing `schedule_task` shim's convention
- `action_tools_test.go` and `process_actions_test.go` already exist on main; extend rather than rewrite
- Before merging: confirm ticket 13 (vultiagent-app CRUD cards) is in a TestFlight build. Otherwise the LLM starts calling new action-discriminated tools but old app builds can't render them.
- This ticket is the migration-heavy one. Expect multiple review rounds. Do not rush.

## Notes

**2026-04-23T23:52:47Z**

Draft PR opened: https://github.com/vultisig/agent-backend/pull/167

**2026-05-04T23:09:10Z**

auto-closed: merged via vultisig/agent-backend#167
