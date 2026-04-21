---
id: v-unku
status: closed
deps: []
links: []
created: 2026-04-21T01:28:27Z
type: chore
priority: 2
assignee: Jibles
---
# Rip out historical-conversation tool-part migrations in agent-backend

## Objective

The v-pxuw consolidation branch shipped ~800 LOC of read-time migration logic in `agent-backend/internal/storage/postgres/convert.go` to keep pre-consolidation conversations rendering their tool cards. We've decided historical conversations aren't worth carrying. The frontend already degrades gracefully on unknown tool names (silent skip, no crash), so the entire migration layer can be removed. Net: ~−800 LOC, no SQL migration, no archive sweep, no app changes.

## Context & Findings

- **Graceful degrade is already built in.** `vultiagent-app/src/features/agent/components/AssistantMessageView.tsx:93-108` returns `null` when `resolveToolUI(toolName)` is undefined. Surrounding text bubbles still render. `toolUIRegistry.test.ts:151-173` already pins this behaviour for every legacy tool name.
- **`migrateResolvedEchoShape` is migrating a shape that can't pre-exist.** Only `execute_*` tools carry the `resolved` field, and those tools shipped in this branch — no old rows have the flat shape it rewrites. Pure paranoia code.
- **`rebuildConfirmationPartFromMetadata` stays.** It's not historical — it rebuilds the live mutable confirmation part from `metadata.confirmation` (which is mutated via `jsonb_set`). Unrelated to the rename migration.
- **One live-behaviour trade-off.** `RenamedToolNewName` has a single non-test caller at `agent.go:2106`. It dedupes the case where the LLM hallucinates an old tool name inside a `respond_to_user` action against a live new-name tool call. The prompt no longer advertises old names, so this is a tail-risk edge — accept the small regression. Replace the call with `ta.Type` directly.
- **Rejected alternatives:**
  - SQL archive migration (soft-set `archived_at = NOW()` on all existing conversations) — unnecessary given graceful degrade; adds a one-way data mutation for no gain.
  - Keep `renamedToolMap` + `RenamedToolNewName` as live-dedup protection — only one caller, tail-risk scenario, not worth the ongoing maintenance.
  - Hard `DELETE` of old conversations — irreversible, no user consent, provides no benefit over the silent-skip path.
  - Per-conversation `schema_version` column — overkill; new column carried forever for a one-time concern.

## Files

- `agent-backend/internal/storage/postgres/convert.go` — delete `migrateRenamedToolParts`, `migrateResolvedEchoShape`, `applyToolInputMigration`, `rewriteResolvedEchoInOutput`, `renamedToolMap`, `renamedToolMigration`, `RenamedToolNewName`, `legacyToolName`, and the two call-sites at lines 164 and 170. Keep `rebuildConfirmationPartFromMetadata`.
- `agent-backend/internal/storage/postgres/convert_test.go` — delete every test case covering the deleted helpers (`TestMessageFromDB_RenamedToolPartsMigration`, `TestMessageFromDB_LegacyPartTypesMigrateToCurrent`, `TestRenamedToolNewName`, shape-migration tests, unknown-tool pass-through test). Keep tests covering `rebuildConfirmationPartFromMetadata` and unrelated `messageFromDB` behaviour.
- `agent-backend/internal/service/agent/agent.go:2106` — replace the `postgres.RenamedToolNewName(ta.Type)` call with `ta.Type` directly; drop the now-unused import if no other references remain.
- `agent-backend/internal/service/agent/process_actions_test.go` — remove or update any test that asserted the dedup-via-rename behaviour (commented reference at line 17).

## Acceptance Criteria

- [ ] `convert.go` no longer references `renamedToolMap` or any of the `migrate*` / `apply*` / `rewrite*` / `legacyToolName` / `RenamedToolNewName` helpers
- [ ] `messageFromDB` still runs `rebuildConfirmationPartFromMetadata` and the nil-Parts guard at the end
- [ ] `agent.go:2106` uses `ta.Type` directly, no `postgres.RenamedToolNewName` callsite remains anywhere outside deleted test files
- [ ] `go build ./...` and `go test ./...` pass
- [ ] golangci-lint clean
- [ ] No new imports; unused imports removed

## Gotchas

- Don't delete `rebuildConfirmationPartFromMetadata` — it handles the live mutable-confirmation pattern and the legacy `data-confirmation` / `tool-schedule_task` part rebuild from `metadata.confirmation`. Its call-site at `convert.go:158` stays.
- Verify no other package imports `postgres.RenamedToolNewName` before deleting — grep showed only `agent.go` and test files, but double-check.
- `process_actions_test.go:17` has a comment referencing `RenamedToolNewName("add_coin") == "vault_coin"`. If that test still asserts dedup behaviour it needs updating or removing.
- Old conversations will render text bubbles but no tool cards — expected. Do not add frontend guards or banners for this.

## Notes

**2026-04-21T01:39:43Z**

Implemented on refactor/v-unku-rip-tool-migrations off refactor/v-pxuw-tool-consolidation. Commit f9f234a: -780 net LOC across 4 files. convert.go: deleted migrateRenamedToolParts, migrateResolvedEchoShape, applyToolInputMigration, rewriteResolvedEchoInOutput, renamedToolMap, renamedToolMigration, RenamedToolNewName, legacyToolName (-246 LOC). convert_test.go: deleted 6 migration test fns + jsonEqual helper (-476 LOC). agent.go: replaced postgres.RenamedToolNewName(ta.Type) with direct calledMCPTools[ta.Type] check; postgres import retained for other uses. process_actions_test.go: dropped TestProcessActions_DedupsLegacyActionAgainstRenamedTool, kept direct-name dedup + pass-through tests. rebuildConfirmationPartFromMetadata intact at convert.go:180 (called at :158). Verified: go build ./... clean, go test -count=1 on affected packages pass, gofmt clean.
