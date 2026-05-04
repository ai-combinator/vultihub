---
id: v-zlov
status: open
deps: [v-bvsj]
links: []
created: 2026-04-23T23:07:53Z
type: feature
priority: 2
assignee: Jibles
---
# vultiagent-app: CRUD cards for action-discriminated tools (VaultCoinTool, VaultChainTool, AddressBookTool)

## Objective

Add UI cards for the three new action-discriminated CRUD tools from ticket 12: `VaultCoinTool` (renders `vault_coin(action: "add"|"remove")`), `VaultChainTool` (renders `vault_chain(action: "add"|"remove")`), `AddressBookTool` (renders `address_book(action: "add"|"remove")`). Keep the legacy single-purpose cards (`AddChainTool`, `AddCoinTool`, `RemoveChainTool`, `RemoveCoinTool`) registered during transition so historical messages still render.

## Context & Findings

### Why keep legacy cards

#164 unregistered the 4 legacy Add/Remove single-purpose cards from `toolUIRegistry`, which means any historical conversation containing an old `add_coin` or `remove_chain` tool-call part renders as blank. Ticket 12 adds server-side aliases to rewrite old part names to new ones at read time, but the alias replay assumes the new cards exist client-side. Until users are all on the new app build:
- NEW messages use new tools (`vault_coin(action)`) → new cards render
- OLD messages replay as new tools via ticket 12's alias → new cards render them
- As an extra safety net, keep the 4 legacy cards registered under their old keys so if any reply path slips the alias, rendering still works

Once all clients are on the post-ticket-13 build and telemetry shows no legacy-card hits for several weeks, a follow-up PR removes them.

### Reviewer findings this PR must address

**Gomes B8 — defensive per-row rendering (applies to 8 cards across #164, CRUD cards are in that class):**
Each card must defensively validate inputs. `vault_coin(action: "add", coin: {...})` with a malformed `coin` → skip the row / show fallback, do NOT throw.

**Do NOT include (from #164 that's out of scope):**
- ScheduleCreate / ScheduleStatus cards (depend on `schedule_task` → `schedule_create` rename — ticket 12 explicitly defers that; skip these cards)
- CreditsCard (depends on `check_credits` → `credits` rename — deferred; skip)
- ManagePluginTool (depends on `create_policy` / `delete_policy` rename — deferred; skip)
- The old AgentApprovalBar / ApprovalContext changes (PR 229 handles)
- The old `useTransactionFlow` hook (ticket 10 defers)

### What this ticket covers

**Cherry-pick from `refactor/v-pxuw-tool-consolidation` (vultiagent-app):**
- Parts of `7db747b` — feat(agent): add CrudCards + toolUIRegistry wiring (v-pxuw Task 12) — SCOPE to vault_coin, vault_chain, address_book only
- Parts of `5b9fb98` — refactor(agent): simplify ExecutePrepResult + split CRUD cards per action — CRUD half only

**Adjust during cherry-pick:**
- Keep `AddChainTool`, `AddCoinTool`, `RemoveChainTool`, `RemoveCoinTool` **registered** in toolUIRegistry for legacy rendering
- Skip CrudCards for schedule / credits / plugin (pure renames, deferred)
- Add defensive per-row validation in all 3 new cards

### Dependency chain

- **Must depend on ticket 12** being in an agent-backend deploy candidate — the new tools must be registered server-side
- Can open as a draft alongside ticket 12 — both need to ship together in a coordinated release

## Files

**New:**
- `src/features/agent/components/tools/CrudCards/VaultCoinTool.tsx`
- `src/features/agent/components/tools/CrudCards/VaultChainTool.tsx`
- `src/features/agent/components/tools/CrudCards/AddressBookTool.tsx` (if not already present)
- `src/features/agent/components/tools/CrudCards/vaultCoin.ts`, `vaultChain.ts` — parser helpers
- `src/features/agent/components/tools/CrudCards/index.ts`
- Tests: `VaultCoinTool.test.ts`, `VaultChainTool.test.ts`

**Modified:**
- `src/features/agent/lib/toolUIRegistry.ts` — add 3 new entries; KEEP existing `AddChainTool`, `AddCoinTool`, `RemoveChainTool`, `RemoveCoinTool`, `AddressBookTool` entries

**Do NOT include:**
- `VaultChainAddStub.tsx` (was `*Stub` component from #164; Gomes called it a "bad smell" — don't ship stub cards. If a chain isn't supported, don't register a fallback card for it.)
- `ScheduleCreate.tsx`, `ScheduleStatus.tsx`, `CreditsCard.tsx`, `ManagePluginTool.tsx`

## Acceptance Criteria

- [ ] PR opened as **draft** against vultiagent-app main, title `feat(agent): CRUD cards for vault_coin / vault_chain / address_book (action-discriminated)`
- [ ] 3 new cards registered in toolUIRegistry
- [ ] Legacy cards `AddChainTool`, `AddCoinTool`, `RemoveChainTool`, `RemoveCoinTool` still registered (comment: "kept for legacy message rendering until ticket v-ujuc retires")
- [ ] All cards defensively validate inputs; malformed row → skip/fallback, not throw
- [ ] No `*Stub` cards registered
- [ ] `pnpm test`, `pnpm lint`, `pnpm tsc --noEmit` clean
- [ ] PR body has **"Blockers" section**: "Do NOT merge before ticket 12 is in an agent-backend deploy candidate"

## Gotchas

- `VaultChainAddStub` from #164 is a "Chain X isn't supported yet" card. Gomes: *"Shipping stubs named `*Stub` with user-facing copy is a bad smell regardless of which of the above you pick."* Do not ship it. If a chain is unsupported, the backend should reject the tool call with a clear error — handled client-side by generic error row.
- If the 3 new tool schemas emit `coin: null` or `chains: []` edge cases, handle them gracefully — not every action shape is populated symmetrically
- Keep the legacy cards rendering visually identical to what's on main; this PR should not redesign them
- `AddressBookTool.tsx` may already exist on main in some form — check and extend rather than duplicate
