---
id: v-rfqt
status: open
deps: [v-qjvi, v-llqd]
links: []
created: 2026-04-23T23:03:54Z
type: task
priority: 1
assignee: Jibles
---
# agent-backend: Prompt.go execute_* routing + tool_filter always-on additions (trimmed cutover)

## Objective

Flip the LLM routing to prefer `execute_*` tools for send/swap/contract-call/LP flows: compact prompt rewrite (delete per-chain send/swap recipes, add routing table) + tool_filter.go additions for the new execute_* always-on category. **Trim aggressively** — do NOT widen other gate categories, do NOT rename CRUD tools, do NOT rewrite starters, do NOT touch analytics classify. This is the minimum-scope cutover PR.

## Context & Findings

### Why trimmed

The original #128 bundled this cutover with: CRUD renames (`schedule_task` etc.), `startersCapabilities` hand-curated rewrite, `AddressBookEntry` JSON tag flip, `tool_classify.go` analytics updates, storage migration, `buildAgentTools() → agentTools()` rename, SSE payload deletions. Reviewers hit multiple preferably-blocking findings on the bundled changes (Gomes' `prompt.go:16` flag-gate regression was the highest-consensus item of the whole review).

### Reviewer findings this PR must fix

**Gomes on `prompt.go:16` (5 rounds, highest consensus):**
> "preferably-blocking: hard-coded chain list drops flag-aware rendering. Caught 5 times across rounds (Codex 2/4/6/8/10). Highest consensus in the whole review."

**0xApotheosis independently:**
> "Static chain list removes launch-surface gating. If Polkadot is still gated via flags in the tool filter, the LLM may now propose Polkadot sends the MCP layer can't fulfill."

**Fix**: preserve flag-aware chain rendering in the prompt. Do NOT hard-code a chain list — keep the flag-driven substitution from main. If the refactor branch has a hardcoded list, restore the flag-driven version when porting.

**Gomes on `executor.go:1345` (preferably-blocking):**
> "`execute_swap` bypasses swap-backend launch-flag enforcement… In prod with 1inch off for cost control, `execute_swap` routes users through 1inch because the allowlist never gets injected. Real money through a disabled provider."

**Fix**: ensure `execute_swap` invocations pass through the same allowed-backends allowlist that `build_swap_tx` does. Verify mcp-ts `execute_swap` accepts `allowed_backends` input (cross-repo check against ticket 1). If it doesn't, this PR cannot land until mcp-ts adds the field OR ticket 1 is updated.

### What this ticket covers

**Cherry-pick from `refactor/v-pxuw-tool-consolidation` (agent-backend):**
- `cea5a2a` — refactor(agent): compact prompt around execute_* routing table (v-pxuw Task 8) — **adjust to preserve flag-aware chain substitution**
- `3f50db7` — refactor(agent): tune classify + filter for v-pxuw execute_* surface — **adjust to add ONLY the execute_* always-on category; do NOT widen other categories**
- `27db9c9` — fix(prompt): drop "tap to sign"; teach the LLM about the consent bar — include if small

**Do NOT include:**
- `7132272` (storage + scheduler for schedule_create rename) — ticket 12
- `aa73e80` (consolidate tool layer per v-pxuw Task 6) — contains CRUD renames, ticket 12
- `00e845d` (drop decimalToBaseUnits) — defer
- `a85cf62` (address_book wire format on entry.name) — ticket 12
- `4cde987` (replay all v-pxuw Task 6 renamed tool parts) — ticket 12
- `fae3234` (finish v-pxuw consolidation — client-side tool bridge + legacy cleanup) — catch-all, cherry-pick selectively
- `e4ea191` (wrap legacy flat resolved maps under labels at read time) — related to labels/raw split; low priority, defer or include if clean
- `06f61b3` (record all tool calls for dedup + unexport loop internals) — defer
- `792431b`, `af29f84` (drop retired build_* refs from tests/classify) — depends on ticket 1 deletion which isn't happening
- `16854d8` (route jsonError MCP results through tool-output-error) — good improvement, defer to own PR
- `738122a` (rip out historical-conversation tool-part migrations) — already done per v-unku (closed)
- `e7bafd0` (balance_<ticker> regression guards) — already on main per v-unku work

### External coordination needed BEFORE opening

**Coordinate with avran on agent-backend PR #133** (`feat/category-filter-fixes` — "filter: fix category drift + better tool context"). Directly overlaps this ticket's tool_filter.go work. Decide one of:
- Merge #133 first, rebase this PR on top
- Merge this PR first, avran rebases #133
- Fold each other's changes

Do not open this PR without talking to avran first.

### Dependency chain

This PR **cannot merge** until:
- Ticket 1 is merged AND deployed to the MCP server (so `execute_*` tools exist to route to)
- Ticket 10 (app-side execute_* consumer) is in an app-store build at least in TestFlight (so new cards exist client-side when LLM routes to execute_*)

PR body must state these dependencies explicitly as "DO NOT MERGE BEFORE…" blockers.

## Files

- `internal/service/agent/prompt.go` — compact routing; preserve flag-aware chain rendering
- `internal/service/agent/tool_filter.go` — add execute_* always-on category only
- `internal/service/agent/tool_filter_test.go` — test coverage for new category keywords
- `internal/service/agent/prompt_test.go` + `prompt_launchsurface_test.go` — test coverage for routing + flag-aware rendering

**Do NOT touch:**
- `internal/service/agent/tool_classify.go`
- `internal/service/agent/data_tools.go` (ticket 5)
- `internal/service/agent/action_tools.go` (ticket 12)
- `internal/service/agent/starters.go`
- `internal/service/agent/types.go` (JSON tag flip — ticket 12)
- `internal/storage/postgres/convert.go` (ticket 12)
- `internal/auth/` (ticket 6)

## Acceptance Criteria

- [ ] PR opened as **draft** against agent-backend main, title `feat(agent): route LLM to execute_* via compact prompt + filter category`
- [ ] `prompt.go` retains flag-aware chain substitution (no hardcoded list)
- [ ] `tool_filter.go` adds execute_* tools to always-on; no other category-table widening
- [ ] `execute_swap` invocations carry `allowed_backends` from the active launch-surface config (cross-verify with ticket 1 that mcp-ts accepts this)
- [ ] All prompt_test / filter_test / launchsurface_test pass
- [ ] `go test ./...`, `golangci-lint run`, `go build ./...` clean
- [ ] PR body has a **"Blockers" section** noting: Ticket 1 must be merged AND deployed; Ticket 10 must be live in an app-store build; coordination with avran's PR #133 complete
- [ ] PR body has a **"Do NOT merge before:"** line listing ticket 1 + ticket 10

## Gotchas

- The flag-aware chain substitution lives in a helper on main; do not delete it while "simplifying" the prompt
- `execute_swap` allowlist: verify mcp-ts ticket 1's `execute_swap.ts` accepts `allowed_backends` in its input schema. If not, this PR opens a P0 gap; coordinate with ticket 1 owner to add it.
- Any `ProcessMessage{,Stream}` dedup changes — OMIT. Those belong to ticket 12's schema track.
- Do not include `fae3234` whole; manually apply just the prompt + filter changes from it
- The repo has coderabbit; expect review-loop iteration. Keep PR scope tight so review is actually useful.
