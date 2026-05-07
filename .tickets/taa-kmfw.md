---
id: taa-kmfw
status: closed
deps: []
links: []
created: 2026-05-01T06:23:13Z
type: bug
priority: 2
assignee: Jibles
---
# POST 'failed' on every terminal error path (PR #339 follow-up)

## Problem

PR #339 (vultiagent-app, branch `worktree-va-qbzv`) introduces a contract: every terminal tool-execution state (success / failed / cancelled) must POST to `/agent/messages/:msgId/tool-results/:toolCallId/execution` so reopened conversations hydrate from server-side state. The contract holds on the happy path. **It systematically leaks on failure paths** — when a tool throws, the user sees a "FAILED" card but no row lands in `tool_call_results`, so the LLM never sees the failure on the next turn and the reopened card hydrates as INCOMPLETE forever. "Solved" means every reachable terminal failure path POSTs `failed` exactly once with the surfaced error message.

## Background

### Current State

The PR adds `usePostToolExecutionResult` (`src/services/network/txOutcomeApi.ts`) — a React Query mutation that POSTs the outcome and patches the conversation cache on success. Each tool wires its own POST callsites at terminal-state lines. The successful path got wired carefully because that's where developers test. Catch blocks were missed because errors flow through indirect paths the author didn't see.

Specific gaps verified by reading the code:

- **`useAutoRunOnMount` (`src/features/agent/hooks/useAutoRunOnMount.ts:36-39`)** swallows executor rejections into local `setStatus('error')` and renders the failure UI without ever POSTing. Five tools depend on this hook and inherit the gap:
  - `AddChainTool.tsx:41` — `throw new Error('Missing chain name')`
  - `RemoveChainTool.tsx:39-40` — `!activeVault` and empty-chains throws
  - `AddCoinTool.tsx:57` — `chainRegistry[chain].nativeDecimals` is a TypeError if the chain isn't registered
  - `RemoveCoinTool.tsx:32-33,45` — `!activeVault`, `!ticker`, "token not found in vault"
  - `CreateVaultTool.tsx:33,43-44` — "Missing fields" throw, plus `createFastVault` / `storeVaultPassword` rejection (no `catch`, only `finally`)

- **`useToolExecution.ts`** has two failure branches that don't POST:
  - `:283-285` — JSON-parse error on the tool result
  - `:326-340` — `verifyFastVaultPassword` network error / 5xx (most reachable failure, runs on every flaky network)

- **`ImportVaultTool.tsx:50-55`** — `addVault` rejection sets `setImportState('error')` and returns; no POST. Doesn't use `useAutoRunOnMount`.

- **`VerifyEmailTool.tsx:78-81`** — verify exception sets local `state='error'`; no POST. Doesn't use `useAutoRunOnMount`.

- **`useTransactionFlow.ts:226-232`** — `runSigning`'s vault-missing branch dispatches `CANCELLED` and throws. No POST. (`useTransactionFlow` runs the EVM/Solana/Cosmos/XRP/Sui/Tron transaction flows; this catch sits before the signing block.)

- **Misleading comment**: `txOutcomeApi.ts:14-16` claims `submitted` is "POSTed only on the EVM signing flow." Code disagrees: `useTransactionFlow` POSTs `submitted` for any chain whose receipt-poll has a probe (evm/solana/cosmos/xrp/sui/tron), and chains without a probe POST `submitted` then immediately resolve to `success`. Behavior is correct — comment is stale.

### Validated Behaviors

- `useAutoRunOnMount.useEffect`'s `.catch((err) => { setError(...); setStatus('error') })` swallows the rejection. No POST callsite exists in this hook.
- Backend's `tool_call_results` upsert (`ON CONFLICT (tool_call_id) DO UPDATE WHERE status = 'submitted'`) makes terminal writes immutable. Duplicate terminal POSTs are silent no-ops; missing-POST is the only failure mode that leaks data.
- `usePostToolExecutionResult` exposes `mutate` which is React Query v5 stable — safe in dep arrays. `postOutcome` callbacks already exist in every relevant file's render scope; threading them into `useAutoRunOnMount` is mechanical.
- `useTransactionFlow.runSigning` already wraps signing in try/catch and POSTs `failed` on the catch (line 367-371 in current code). The vault-missing branch sits *above* the try and short-circuits before reaching it.

### Available Patterns

- `useTransactionFlow.ts` already shows the right pattern: try/catch around the risky work, with an explicit `postOutcome({ status: 'failed', error: message })` in the catch before re-dispatching/throwing.
- `useUnsafeToRestartStore.getState().enter('label')` + `leaveUnsafe()` in `finally` is the established side-effect-pairing convention; mirror its discipline for POSTs.
- The `postOutcome` helper signature in each tool is already `(status, error?) => void` (auto-run tools) or `(result: ToolExecutionResult) => void` (transaction hook). Pick whichever shape fits the per-tool callsite.

## Current Thinking

- **The structural fix is `useAutoRunOnMount` accepting a `postOutcome` callback and calling it from the catch.** One change closes 5 tools (Add/RemoveChain, Add/RemoveCoin, CreateVault) and prevents the same bug in any future auto-run tool. The hook's executor signature stays unchanged; only the hook itself learns to POST `failed` on rejection. Tools still POST `success` themselves at the end of their executor body — symmetry kept on the happy path.
- **`ImportVaultTool` and `VerifyEmailTool` don't use `useAutoRunOnMount`** — they own their own state machines. Add explicit `postOutcome('failed', err.message)` in their existing error-state branches before the structural fix lands, since they don't benefit from it.
- **`useToolExecution` gets two targeted `postOutcome({status: 'failed', error: ...})` calls** at the parse-error and password-verify catches. No structural change.
- **`useTransactionFlow` vault-missing branch** gets a `postOutcome({status: 'failed', error: 'No vault selected.'})` before the throw.
- **Update `txOutcomeApi.ts:14-16` comment** to reflect that `submitted` POSTs whenever a chain has a receipt probe — not "EVM-only."
- **Backend idempotency means we don't need to coordinate ordering.** Double-POST is harmless; the order between dispatch and POST in any single catch can be either way without affecting correctness.
- **Why now, not in PR #339 itself:** the PR is in review and the author may want to iterate the fix in a follow-up commit on the same branch rather than rebasing the existing diff. Spec it as a separate ticket so it's tracked even if it lands as a follow-up commit on `worktree-va-qbzv` or as a new PR on top.

## Constraints

- Lands on or alongside `worktree-va-qbzv`. Cannot ship before PR #339 merges; the contract being fixed only exists in that PR.
- Backend wire shape unchanged. No agent-backend changes.
- Project comment rule (WHY-only, ≤2 lines, no ticket/PR refs) applies to any new comments.
- Cannot regress the success-path POST. Each tool must still POST `success` exactly once on its successful branch.
- The `useAutoRunOnMount` change must not break the existing executed-once-per-mount semantics (`executedRef.current` guard).

## Assumptions

- The five auto-run tools all want identical "POST failed on catch with `err.message`" semantics. If any of them needs a richer mapping (e.g. classify the error before POSTing), the structural fix needs an escape hatch — but this hasn't been observed.
- React strict mode's double-mount is acceptable. `useAutoRunOnMount`'s `executedRef` resets per fresh mount, so dev-mode would POST twice; backend's `(messageId, toolCallId)` upsert makes the second a no-op.
- All five auto-run tools' callsites already have a `postOutcome` defined locally (verified for CreateVaultTool; assumed for the other four — planner should confirm before threading).
- No tool currently relies on the rendered "FAILED" card to be the *sole* signal — i.e., none of them have user-visible logic that says "if no row exists, retry," because that's the bug we're fixing. If anything does, it'll need a parallel fix.

## Non-goals

- ScheduleTaskTool cancel-orchestration race (`ScheduleTaskTool.tsx:75-83`). Already commented in code; needs `AbortSignal` plumbing through `approveSchedulePreviewApi`. Separate work.
- Reopen-mid-poll re-POST of `success`. Each reopen on a `submitted` row re-fires `useReceiptPoll` → re-POSTs `success` until backend lands the upgrade. Idempotent, low-priority cleanup.
- The five review concerns + comment squash (ticket `v-rkks`). Different scope.
- The sidecar-vs-embedded wire rework (ticket `v-aoov`). Different scope.
- Auto-run tool executor refactor beyond the catch path. Each tool's body stays as-is.

## Dead Ends

- **"Gate `submitted` POSTs to EVM-only to match the design comment."** Investigated and rejected. The non-EVM behavior (POST `submitted` then immediately POST `success` for chains without a receipt probe) is correct and the lifecycle works the same way as EVM. Only the comment is wrong; behavior stays.
- **"Per-tool try/catch around each executor body."** Considered as the alternative to the `useAutoRunOnMount` change. Rejected because it duplicates the same five-line catch across 5 files and doesn't prevent the next auto-run tool from re-introducing the bug. Centralized fix wins on maintenance.

## Open Questions

- **`useAutoRunOnMount` signature.** Take a single `postOutcome: (status: 'failed', error: string) => void` parameter, or take a richer `onOutcome` callback that's called with both success and failure? The latter centralizes the success POST too; the former is a smaller diff. Planner's call.
- **Strict mode + structural fix.** Does threading `postOutcome` through the hook's deps cause the executor to re-run on dev double-mount? Need to verify the `executedRef` guard still wins.
- **Should `ImportVaultTool` and `VerifyEmailTool` be migrated to a shared error-handling pattern** (e.g. a small `useToolOutcome` helper) at the same time, or just patched in place? In-place is faster; shared helper is the structural answer if their owners are open to it.
- **Test coverage scope.** One unit test per failure branch is ~9 new tests. Scope OK, or push to a follow-up?
- **Order of merge.** Land this on `worktree-va-qbzv` as a follow-up commit (gets reviewed in the same PR), or open a new PR on top of `worktree-va-qbzv` (cleaner review boundary, but two PRs to coordinate)? Author's preference.
