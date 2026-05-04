---
id: v-wgok
status: closed
deps: []
links: []
created: 2026-04-23T23:56:26Z
type: bug
priority: 2
assignee: Jibles
---
# agent-backend: Block build_* tools on IntentSchedule turns (narrow tool slice + prompt)

## Objective
Stop the LLM from emitting `build_evm_tx` / `build_*_send` / any other calldata-producing tool on a turn classified as `IntentSchedule`. These turns should call `schedule_task` only — the scheduled task builds and signs its own tx at execution time. Currently the model calls both in parallel, producing a spurious SEND tx preview alongside the schedule preview.

## Context & Findings
- Reproduction: user asks "schedule to send 0.0001 ETH on Arbitrum once in 5 minutes." Agent emits `evm_tx_info` → `build_evm_tx` → `schedule_task`. Frontend renders a SEND ActionCard with auto-fired approval bar AND a separate ScheduleConfirmationCard. Nothing is pre-signed — both are independent tool emissions. Confirmed `execScheduleTaskPreview` (`executor.go:2024-2066`) writes Redis only; `approveSchedulePreviewApi` promotes without signing.
- Root causes:
  1. **Tool slice too permissive.** `internal/service/agent/turn_state.go:188-192` — `categoriesForIntent(IntentSchedule)` keeps `send`, `swap`, `evm`, `token`, `fee` MCP categories. `narrowToolsForIntent` also keeps `produces_calldata: true` tools (line 160). Combined with `tool_choice=required` for forcing turns, `build_evm_tx` is visible AND the model must call something.
  2. **Prompt imbalance.** `prompt.go:253-266` prescriptively instructs `evm_tx_info → build_evm_tx` for ANY EVM send, with no scheduled-send carve-out. The Scheduled Tasks section is one sentence, 400 lines later (`prompt.go:627-632`): "Tasks cannot sign" does not imply "do not call build_*".
  3. **Missing skill file.** `prompt.go:619` advertises `get_skill("scheduling")` but there is no `scheduling.md` under `mcp-ts/skills/` or in the backend; `mcp/client.go:646` returns `skill not found` if the model tries to load it.
- Gomes's `feat/agent-reliability-rearchitecture` adds a post-response validator pipeline (`internal/service/agent/validator/`) with extractors for factual drift, invented capability, etc. Checked — no extractor flags "schedule_task + build_* in the same turn" (both succeed, narration is accurate). Not blocked on that branch.
- Rejected: app-side render suppression when `schedule_task` sibling is present — moves the bug, adds debt, server still emits tx_ready and queues calldata.
- Rejected: frontend approval-slot precedence rule — defensive plumbing for a backend bug that tool-slice narrowing makes structurally impossible (LLM physically cannot call a tool absent from its list).

## Files
- `internal/service/agent/turn_state.go` — `categoriesForIntent(IntentSchedule)` and the `produces_calldata` handling in `narrowToolsForIntent`. Target: `IntentSchedule` slice contains `schedule_task` + read-only utility tools only; NO `build_*`, NO `erc20_approve`, NO `produces_calldata: true` tools.
- `internal/service/agent/prompt.go` — tighten `scheduledTasksSectionForPrompt` (line 627) with an explicit "do not call build_* on scheduled tx/send" rule. Either create `mcp-ts/skills/scheduling.md` or remove the `get_skill("scheduling")` bullet from `schedulingSkillLineForPrompt` (line 619).
- `internal/service/agent/turn_state_test.go:283` — existing "Schedule intent must keep schedule_task in narrowed slice" test; add inverse assertion that `build_evm_tx` and friends are absent.

## Acceptance Criteria
- [ ] For `IntentSchedule` turns, the narrowed tool slice excludes every `produces_calldata: true` tool.
- [ ] Narrowed slice still includes `schedule_task` + read-only utilities (`search_token`, `get_address`, `get_price`, `get_market_price`, `get_balances`, `get_portfolio`, `convert_amount`, `get_address_book`, `list_vaults`, `get_skill`).
- [ ] Repro prompt produces exactly one tool call: `schedule_task`. No `evm_tx_info`, no `build_evm_tx`.
- [ ] Prompt's Scheduled Tasks section contains an explicit rule forbidding `build_*` calls on scheduled tx/send turns.
- [ ] `get_skill("scheduling")` either resolves to a real file or is removed from the prompt.
- [ ] Existing schedule-intent tests (`turn_state_test.go:283`, `scheduling_flag_test.go`) still pass.
- [ ] New test asserts `build_*` tools are absent from the IntentSchedule slice.
- [ ] `gofmt` + `golangci-lint` clean.

## Gotchas
- Narrowing is per-intent — keep build tools visible for `IntentSend` / `IntentSwap`.
- `demoteIntentForFlags` (`intent.go:148`) already demotes `IntentSchedule → IntentNone` when the UI flag is off; tool narrowing only runs when `forcing()` is true. Don't disturb.
- Observer schedules (`action_type: observe`) also classify `IntentSchedule` but have no build tool anyway.
- If Gomes's branch has merged by implementation time, use `ToolsForModel` / `ToolChoiceForModel` (capability-aware) instead of `Tools` / `ToolChoice`.
- If creating `scheduling.md`, the slug must match the prompt bullet (`scheduling`) and the `SkillFlagOn` mapping in `internal/launchsurface/flags.go:178`.

## Notes

**2026-04-24T00:08:53Z**

Implemented: IntentSchedule narrows away all produces_calldata:true tools + categories trimmed to {balance,utility}; prompt adds explicit 'DO NOT build_*' rule; get_skill('scheduling') bullet removed (no scheduling.md exists); new test TestTurnState_Schedule_ExcludesBuildTools pins the fix + IntentSend regression guard. Files: turn_state.go, prompt.go, turn_state_test.go. All tests pass, gofmt clean on changed files, no new lint issues.
