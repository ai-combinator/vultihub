---
id: v-repi
status: closed
deps: []
links: []
created: 2026-04-23T23:02:39Z
type: feature
priority: 2
assignee: Jibles
---
# agent-backend: balance fan-out for chains missing from SSE context (v-uvyo follow-up)

## Objective

Land the balance fan-out improvement from #128: `get_balances` / `get_portfolio` call mcp-ts per-chain balance tools concurrently (cap 5, 10s per-chain timeout, partial failures → `failed_chains`) for chains missing from SSE-prefetched context. This was one of only two items reviewers praised; stands entirely on its own.

## Context & Findings

### Reviewer praise

0xApotheosis on #128 review summary:
> "Solid consolidation. Test coverage on the fan-out paths (concurrency cap, per-chain timeout, pre-cancelled ctx, partial failures, prefetched-union edge cases) is exemplary."

This is the single clearest endorsement across all three jumbo PRs.

### What this ticket covers

**Cherry-pick from `refactor/v-pxuw-tool-consolidation` (agent-backend):**
- `b33bb84` — feat(agent): fan-out to MCP balance tools for chains missing from SSE context

Plus any follow-up fixes to the fan-out path — check `data_tools_test.go`, `executor.go`, and `context.go` diffs on that branch for fan-out-specific commits (watch for `chainsMissingFromBalances`, concurrency-cap, timeout-per-chain semantics).

### Parent ticket

Closes work implied by `v-uvyo` (already closed) by making the concurrent balance fetch live on main. The v-uvyo ticket notes fan-out was "already done" on the refactor branch; this surgical PR lands it standalone.

### Scope boundaries

**Do NOT include:**
- Tool renames (`schedule_task → schedule_create` etc.) — ticket 12
- Prompt.go rewrite — ticket 7
- tool_filter.go widened categories — ticket 7
- JWT fixes — ticket 6 (separate small PR)
- `decimalToBaseUnits` deletion — that's bundled with ticket 7 or deferred
- `buildAgentTools() → agentTools()` rename — ticket 7 or omit
- `normalizeMCPArgs` field alias changes — defer

**Only touch:**
- `data_tools.go` fan-out additions
- New concurrency helpers
- Test coverage

## Files

Cherry-picks will touch these; others should remain at their main state:
- `internal/service/agent/data_tools.go`
- `internal/service/agent/data_tools_test.go`
- Possibly `internal/service/agent/context.go` (for chains-missing computation)
- Possibly `internal/service/agent/executor.go` (wiring fan-out into execGetBalances / execGetPortfolio)
- MCP client helper if a new concurrency primitive was added

## Acceptance Criteria

- [ ] PR opened as **draft** against agent-backend main, title `feat(agent): balance fan-out for chains missing from SSE context`
- [ ] Concurrency cap of 5; 10s per-chain timeout; partial-failure handling surfaces `failed_chains` in response
- [ ] Pre-cancelled context is respected (no wasted RPC calls)
- [ ] Prefetched-union edge cases covered in tests (all chains prefetched → no fan-out; some missing → fan-out only missing)
- [ ] All existing data-tools tests still pass
- [ ] `go test ./...`, `golangci-lint run`, `go build ./...` clean
- [ ] PR body calls out that this is **independent and mergeable off main**; no dependency on tickets 1/7
- [ ] PR body quotes the 0xApotheosis endorsement as justification

## Gotchas

- Do not accidentally pull in `cea5a2a` (prompt rewrite) or `3f50db7` (classify + filter tuning) — those are ticket 7
- Must not land any mcp-ts tool renames — agent-backend calls mcp-ts tools by their CURRENT main-branch names
- If MCP client calls a balance tool that doesn't exist yet (future tool), handle the error gracefully — falls into the `failed_chains` partial-failure bucket
- Keep the tool_filter/classify categories AS-IS on main during this PR — only the data-tools execution path changes

## Notes

**2026-04-23T23:42:37Z**

Draft PR opened: https://github.com/vultisig/agent-backend/pull/165

**2026-05-04T23:09:10Z**

auto-closed: merged via vultisig/agent-backend#165
