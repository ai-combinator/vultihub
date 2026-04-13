---
id: v-sdtg
status: closed
deps: []
links: []
created: 2026-04-07T05:04:53Z
type: task
priority: 2
assignee: Jibles
---
# Eliminate client-side auto-action loop — server-orchestrated tool execution

**Branch: `feat/chat-state-unification`** in vultiagent-app — contains prerequisite work (Zustand message store, FlatList migration, streaming unification). Build on top of this branch.

## Problem Statement

The vultiagent-app client runs a multi-iteration auto-action loop in `doSend()` that duplicates server-side MCP tool capabilities. When the user asks "what's my balance?", the backend emits a `get_balances` action, the client executes it locally, reports the result back via a second SSE stream, the backend generates a new response, and this repeats up to 5 times. This produces multiple SSE streams per user message, causes the server to store intermediate responses that differ from what the user sees, makes message state reconciliation impossible to get right, and results in visual bugs (duplicate messages, content swapping, layout jumping). "Solved" means one SSE stream per user message, server stores what the user sees, and the client auto-action while loop is deleted.

## Research Findings

### Current State

**Backend** (agent-backend):
- System prompt in `internal/service/agent/prompt.go` (lines 15-40) defines an actions table telling Claude to emit client-side actions: `get_balances`, `get_portfolio`, `get_market_price`, `search_token`, `build_swap_tx`, `build_send_tx`, `add_coin`, `create_vault`, etc.
- `autoExecutableActions` whitelist in `internal/service/agent/agent.go` (lines 1140-1158) marks these for auto-execution: `get_balances`, `get_portfolio`, `get_market_price`, `add_chain`, `remove_chain`, `add_coin`, `remove_coin`
- Claude emits actions via a `respond_to_user` tool defined in `prompt.go` (lines 407-447). Action types are schema-enforced via enum.
- An `interceptor.go` (lines 43-100) currently only intercepts `search_token` to route through MCP server-side. All other actions pass through to the client.
- MCP tools are auto-discovered from `MCP_SERVER_URL` via `mcp/client.go` GetTools(). The MCP server registers 70+ tools including `evm_get_balance`, `get_sol_balance`, `get_ton_balance`, `get_price`, `search_token`, `build_solana_tx`, `build_gaia_send`, `build_evm_tx`, etc. — equivalents for everything the client auto-executes.

**Frontend** (vultiagent-app):
- `useAgentChat.ts` `doSend()` contains a `while` loop that auto-executes actions and reports results back via `reportActionResultStream`, creating multiple SSE streams per message.
- `useAgentTools.ts` `executeAutoAction()` (lines 43-608) is a ~560-line function handling every action type with a giant switch statement. Most are API calls that duplicate backend MCP tools.
- `isAutoExecuteAction` in `lib/helpers.ts` (lines 172-186) matches on a hardcoded set + `build_*` prefix + explicit `auto_execute` flag.
- After the auto-action loop, the final response content differs from what the server stored. This makes server reconciliation impossible.

**Prerequisite work already done** (on `feat/chat-state-unification` branch):
- Created `chatMessagesStore.ts` — single Zustand store for all messages
- Streaming content lives IN the message array (`isStreaming` flag), not as separate state
- Replaced ScrollView with FlatList in AgentChatArea
- Deleted `useStreamingState.ts` (6 separate streaming useState vars)
- Deleted `useChatMessages.ts` (3-source merge)
- Simplified SSE callbacks from 6 setters to single `updateStreamingMessage`
- Removed post-send `fetchConversation` (was causing content swap + duplicates)

**Reference implementation** (shapeshift-agentic):
- Server orchestrates all tool calling via AI SDK protocol. Client receives one SSE stream per message.
- Tools that need client execution are rendered as inline UI components via a `TOOL_UI_REGISTRY` — a 1:1 mapping from tool name to React component.
- Each tool component uses `useToolExecution` (step-based state machine) and `useExecuteOnce` (guards single execution).
- No client-side orchestration loop. No reconciliation.

### Available Tools & Patterns

**Backend MCP equivalents for every client auto-action:**
| Client auto-action | MCP tool equivalent |
|---|---|
| `get_balances` / `fetch_balances` / `get_portfolio` | `evm_get_balance`, `get_sol_balance`, `get_ton_balance`, `get_sui_balance`, `get_trx_balance`, `get_atom_balance`, `get_xrp_balance` |
| `search_token` | `search_token` (already intercepted server-side) |
| `get_market_price` | `get_price` |
| `build_send_tx` (Solana) | `build_solana_tx`, `build_spl_transfer_tx` |
| `build_send_tx` (Cosmos) | `build_gaia_send` |
| `build_send_tx` (TON) | `build_ton_send`, `build_ton_jetton_transfer` |
| `build_swap_tx` | `build_swap_tx` MCP tool |
| `get_chains`, `vault_info`, `get_chain_address` | `set_vault_info`, `get_address` |

**Actions that genuinely need client-side execution:**
- `create_vault` — generates vault keys locally
- `verify_vault_email` — needs user OTP input
- `import_vault` — needs file picker UI
- `sign_tx` / `sign_typed_data` — needs local vault keys + password/biometric
- `add_coin` — modifies local vault store

### Validated Behaviors
- The backend's `interceptor.go` already routes `search_token` through MCP server-side — this is the pattern to extend
- Claude receives ALL MCP tools in its tool list alongside `respond_to_user` — it can already call them directly
- The `respond_to_user` action types are schema-enforced via enum — removing a type from the enum prevents Claude from emitting it
- `autoExecutableActions` is a simple Go map — removing entries stops the auto-execute flag

## Constraints
- Backend changes must not break the Vultisig main app (iOS/Android native) if it uses the same API
- MCP tools for balance queries need vault public keys — already sent in the `context` object
- `create_vault` and `verify_vault_email` have multi-step flows that can't be fully server-side
- Signing must remain client-side (private key material never leaves the device)

## Dead Ends
- **Client-side reconciliation after sends**: Tried content matching, ID patching — fails because auto-actions produce different content than server stores. The right fix is eliminating the mismatch.
- **Keeping auto-actions but fixing the UI**: Fixed worst symptoms with Zustand store + FlatList but edge cases keep appearing. Architecture needs to change.

## Open Questions
- Does the Vultisig native app consume the same `actions` SSE events? Removing action types from the backend prompt could break it.
- Should `add_coin` move server-side or stay client-side?
- For remaining client-side actions, should the backend emit them as a new SSE event type (e.g. `client_action`) or keep them in `actions`?
- Do `build_*` MCP tools already have the enrichment context the client currently provides (contract addresses)?

## Implementation Scope

### Backend: agent-backend

**1. Remove read/query action types from system prompt**
- File: `internal/service/agent/prompt.go`
- Remove from actions table (lines 15-40) and `respond_to_user` enum (lines 407-447): `get_balances`, `get_portfolio`, `get_market_price`, `get_chains`, `get_chain_address`, `vault_info`, `sign_in_status`, `get_coins`
- Claude will use MCP tools directly. `tool_progress` SSE events already stream status.

**2. Remove build_* action types from system prompt**
- Remove: `build_send_tx`, `build_swap_tx`, `build_custom_tx` and any `build_*` types
- Claude will use MCP tools (`build_evm_tx`, `build_solana_tx`, etc.) directly

**3. Remove from autoExecutableActions whitelist**
- File: `internal/service/agent/agent.go` (lines 1140-1158)
- Remove: `get_balances`, `get_portfolio`, `get_market_price`, `add_chain`, `remove_chain`

**4. Keep in actions table (client-side only)**
- Keep: `create_vault`, `verify_vault_email`, `import_vault`, `sign_tx`, `sign_typed_data`, `add_coin`, `remove_coin`

### Frontend: vultiagent-app

**1. Create tool UI registry and execution hooks**
- New: `src/features/agent/lib/toolUIRegistry.ts` — maps action types to components
- New: `src/features/agent/hooks/useToolExecution.ts` — step-based state machine
- New: `src/features/agent/hooks/useExecuteOnce.ts` — guards single execution

**2. Create tool UI components for client-side actions**
- Extract from `useAgentTools.ts` into: `VaultCreationUI`, `VaultVerificationUI`, `VaultImportUI` (adapt existing `AgentVaultImportCard`)
- Render inline in messages, each component owns its lifecycle

**3. Delete the auto-action loop from doSend()**
- Delete the `while` loop, `isAutoExecuteAction` filtering, `executeAutoAction` calls
- `doSend()` becomes: send → stream → finalize (~50 lines instead of ~250)
- Client actions render as tool UI components via registry

**4. Delete redundant client-side tool execution**
- Delete `executeAutoAction()` from `useAgentTools.ts` (~560 lines → ~100 lines)
- Delete balance querying, price fetching, token searching, transaction building logic

**5. Remove isAutoExecuteAction**
- Delete `AUTO_EXECUTE_ACTIONS` set and function from `lib/helpers.ts`

**6. Clean up reconciliation**
- Simplify or remove `reconcileWithServer` — with one SSE stream per message, local state IS server state

### MCP Server
- Likely no changes — all tools already registered
- Verify balance query tools work when called by backend

## Notes

**2026-04-08T04:31:45Z**

## Design: Server-Orchestrated Tool Execution with Tool UI Registry

### Problem
Client runs a multi-iteration auto-action loop that creates up to 6 HTTP streams per user message, causes server/client state divergence, and produces visual bugs (duplicate messages, content swapping, layout jumping). Complex multi-tool flows are fragile because every client-side step is a separate stream and reconciliation point.

### Solution
Move all read/query/build operations server-side via MCP (one stream). Client-side actions (signing, vault creation) render as self-contained inline tool UI components instead of executing in a loop. Tool components optionally notify the server of results via a lightweight endpoint, not a new conversation message.

### Architecture

BEFORE (per message):
  Client → Stream 1 → Server (Claude) → "get_balances" action
  Client executes locally → Stream 2 → Server (Claude) → "build_swap" action
  Client executes locally → Stream 3 → Server (Claude) → "sign_tx" action
  Client executes locally → Stream 4 → Server (Claude) → "Done!"
  (4 streams, 4 reconciliation points)

AFTER (per message):
  Client → Stream 1 → Server (Claude calls MCP: balance, route, build)
    ← tool_progress: "Checking balances..."
    ← tool_progress: "Finding best route..."
    ← tool_progress: "Building transaction..."
    ← tool_call: { name: "sign_tx", data: { unsignedTx, ... } }
    ← text_delta: "Here's your swap: 1 SOL → 45 USDC"
    ← done
  Client renders SignTransaction component inline
  User approves → component signs → component shows result
  Component calls POST /tool-result (lightweight, no new stream)
  (1 stream, 0 reconciliation)

### Key decisions
- No Vercel AI SDK — adapt ShapeShift's tool UI pattern to our existing SSE infra
- No Zustand migration needed — current hook structure works fine
- Nothing else uses this backend — safe to remove action types
- Tool result feedback is opt-in per component (signing needs it, add_coin doesn't)
- Confirmation flow actions (create_policy, cancel_scheduled_task, etc.) stay on existing actions pattern

### Backend changes (agent-backend)

**1. Remove read/query/build action types from prompt (prompt.go)**
Remove from actions table: get_balances, get_portfolio, get_market_price, search_token, get_chains, get_chain_address, vault_info, get_coins, sign_in_status, build_send_tx, build_swap_tx, build_custom_tx, read_evm_contract, scan_tx, thorchain_query

**2. Remove from autoExecutableActions (agent.go)**
Remove: get_balances, get_portfolio, get_market_price, list_vaults

**3. Emit tool_call SSE for remaining client-side actions**
When Claude emits a client-side action via respond_to_user, backend emits as tool_call SSE event: { name, toolCallId, data }. Replaces actions event for these types.

**4. Add POST /conversations/{id}/tool-result endpoint**
Accepts { toolCallId, name, result }. Stores as conversation metadata (not a message). Claude picks it up as context on next user message. No Claude invocation triggered.

**Keep in actions table:** create_policy, delete_policy, plugin_install, cancel_scheduled_task, address_book_add, address_book_remove, get_address_book, polymarket_sign_bet (confirmation flow pattern).

### Frontend changes (vultiagent-app)

**1. Tool execution hooks**
- useExecuteOnce(toolCallId, executor) — single-execution guard (~20 lines)
- useToolExecution(toolCallId) — step-based state machine: idle → executing → success|error, Zustand map keyed by toolCallId (~80 lines)

**2. Tool UI registry (toolUIRegistry.ts)**
Maps tool name → React component:
- sign_tx → SignTransactionTool (from useTransactionSigning)
- sign_typed_data → SignTypedDataTool (from useTransactionSigning)
- create_vault → CreateVaultTool (from useAgentTools)
- verify_vault_email → VerifyEmailTool (from useAgentTools)
- import_vault → ImportVaultTool (from useVaultImport + AgentVaultImportCard)
- add_coin → AddCoinTool (from useAgentTools)
- remove_coin → RemoveCoinTool (from useAgentTools)
- add_chain → AddChainTool (from useAgentTools)
- remove_chain → RemoveChainTool (from useAgentTools)

Each receives { toolCallId, data }, manages own lifecycle via useToolExecution, calls notifyToolResult on completion if server needs to know.

**3. Wire tool_call SSE into message rendering**
- sseEventProcessor.ts: handle tool_call events, store on message
- Message bubble: look up registry, render component inline

**4. Delete auto-action loop + legacy code**
- useSendMessage.ts: delete while loop, doSend becomes send → stream → finalize (~50 lines)
- useAgentTools.ts: delete executeAutoAction (~560 lines)
- helpers.ts: delete AUTO_EXECUTE_ACTIONS + isAutoExecuteAction
- agentClient.ts: delete reportActionResultStream, add notifyToolResult

**5. Tool result notify behavior**
| Tool | Notify server? | Why |
| sign_tx / sign_typed_data | Yes | tx hash for future context |
| create_vault | Yes | vault ID to continue conversation |
| verify_vault_email | Yes | verification status |
| import_vault | Yes | vault ID |
| add_coin / remove_coin | No | local vault store only |
| add_chain / remove_chain | No | local vault store only |

### Current codebase state (as of 2026-04-08)
Note: significant refactoring happened since ticket creation. Key differences:
- useAgentChat decomposed into focused hooks (useSendMessage, useConversationManager, useVaultImport, etc.)
- Auto-action loop is in useSendMessage.ts (315 lines), not useAgentChat.ts
- No Zustand chatMessagesStore exists — useChatMessages.ts and useStreamingState.ts still present
- useVaultImport.ts already extracted
- tool_call SSE events already forwarded by agentClient.ts (currently no-op in sseEventProcessor.ts)
- tool_progress already renders via TOOL_LABEL_MAP in sseEventProcessor.ts

### Tasks

1. **Backend: Remove read/query/build action types from prompt** — Remove ~15 types from actions table in prompt.go. Remove stale entries from autoExecutableActions in agent.go. (agent-backend, independent)

2. **Backend: Emit tool_call SSE for client-side actions** — When respond_to_user contains a client-side action type, emit as tool_call SSE event with { name, toolCallId, data }. (agent-backend, depends on 1)

3. **Backend: Add tool-result notify endpoint** — POST /conversations/{id}/tool-result, stores metadata without creating a message or invoking Claude. (agent-backend, independent)

4. **Frontend: Create tool execution hooks** — useExecuteOnce and useToolExecution in src/features/agent/hooks/. (independent)

5. **Frontend: Create tool UI registry + extract components** — Registry file + extract SignTransactionTool, CreateVaultTool, ImportVaultTool, VerifyEmailTool, AddCoinTool, RemoveCoinTool, AddChainTool, RemoveChainTool from existing code. (depends on 4)

6. **Frontend: Wire tool_call SSE into message rendering** — Handle in sseEventProcessor.ts, render via registry in message bubble. (depends on 5)

7. **Frontend: Delete auto-action loop + legacy code** — Remove while loop from useSendMessage.ts, gut useAgentTools.ts, remove AUTO_EXECUTE_ACTIONS/isAutoExecuteAction from helpers.ts, replace reportActionResultStream with notifyToolResult in agentClient.ts. (depends on 6)

8. **Verify MCP tool coverage** — Confirm every removed action type has a working MCP equivalent. Test balance queries with vault public keys, build tools with contract addresses. (parallel with everything)

Tasks 1, 3, 4, 8 are independent. Task 2 depends on 1. Tasks 5→6→7 are sequential. Backend (1-3) and frontend (4-7) can be developed in parallel.
