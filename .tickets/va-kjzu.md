---
id: va-kjzu
status: open
deps: []
links: []
created: 2026-04-13T03:50:13Z
type: task
priority: 2
assignee: Jibles
---
# Refactor transaction UI and action system toward per-tool-card pattern

## Objective

Refactor the transaction rendering, action system, and backend response structure to follow a per-tool-card pattern where each MCP tool call renders its own lifecycle-aware UI component. Currently the architecture has competing mechanisms (respond_to_user actions, data-tx_ready side-channel, clientSideActionTypes, local text message injection) that produce fragmented, inconsistent transaction UX. The target state: backend chains MCP tools, frontend renders a card per tool via the existing TOOL_UI_REGISTRY, tool components own their lifecycle including signing. No side-channels, no action-button round-trips, no injected text messages.

## Context & Findings

### Current architecture (broken)

**respond_to_user** is a tool the LLM MUST call for every response. It packages text + actions + suggestions + intent into one structured call. The backend parses it, extracts actions, stores them in Redis, emits them as separate SSE events. The system prompt enforces: "EVERY response MUST use respond_to_user."

**DataActions** renders action buttons from respond_to_user's actions array. Tapping an action button calls doSend(action.title) — sends the button's title as a plain text user message. The action's params are thrown away. The LLM then re-parses intent from the title text and proceeds. This is a full round-trip for what should be a direct tool call.

**data-tx_ready** is a side-channel: when ANY build_* MCP tool executes, the backend detects the build_ prefix (agent.go:945) and emits the tool result as a duplicate data-tx_ready SSE event. The frontend watches for this in AgentChatArea.tsx:109 to trigger signing via onTxReady → setPendingTx. The tool output already exists in the tool part — this duplicates it.

**Transaction rendering is scattered across 4+ components:**
- TxPreviewCard — renders from data-tx_ready parts
- SignTransactionTool — registered for sign_tx in TOOL_UI_REGISTRY, reads from Zustand store, but the backend prompt says "NEVER emit sign_tx" so it rarely/never renders
- TxResultCard — only child of SignTransactionTool, therefore orphaned
- TxStatusCard — renders for evm_tx_info/get_tx_status, shows raw status like "NOT_FOUND"
- Local text messages injected via makeLocalAssistantMessage in useTransactionSigning for signing state

**Skills contradict the system prompt:**
- send-transfer.md says: emit build_send_tx as respond_to_user action, then after confirmation emit sign_tx action
- swap-trading.md says: emit build_swap_tx as respond_to_user action, then emit sign_tx
- System prompt says: "NEVER emit sign_tx. The app handles signing automatically after any successful build."
- build_swap_tx is BOTH an MCP tool name AND an action type, meaning different things in each context

### Reference: ShapeShift agentic pattern (target)

ShapeShift's agentic wallet follows a simpler, more robust architecture:
- **No respond_to_user** — LLM calls tools, text is text. AI SDK handles both natively.
- **No DataActions** — tool components render their own UI (buttons, steppers, previews)
- **No data-tx_ready** — tool component reads part.output directly when state becomes output-available
- **Per-tool cards** with execution lifecycle hooks — each tool component shows preview, drives multi-step execution (network switch → approve → sign → broadcast), displays result
- **Tool UI registry** maps tool names to components (same pattern Vultisig already has)

### What Vultisig keeps that ShapeShift doesn't have

- **MCP tool architecture** — tools live on separate server, shared across app/CLI/web
- **LLM tool chaining** — LLM calls evm_tx_info → sees nonce/gas → calls build_evm_tx with those values. Reasons between steps.
- **Conversation titles** — backend generates via cheap Haiku call on first message, no LLM tool needed

### Rejected approaches

- **Anchoring on data-tx_ready as the lifecycle card** — too coupled to current side-channel mechanism. Fragile if backend changes.
- **One mega-card for the whole tx flow** — doesn't match the architecture where the LLM chains 2-3 independent MCP tools. Per-tool cards naturally combine into one if tools consolidate later.
- **Keeping respond_to_user but fixing around it** — the fundamental problem is that it conflates UI directives with chat messages. Fixing the symptoms leaves the root cause.

## Files

### Frontend (vultiagent-app)

**Modify:**
- src/features/agent/lib/toolUIRegistry.ts — register build_evm_tx, build_swap_tx, and other build_* tools with lifecycle components
- src/features/agent/components/AssistantMessageView.tsx — remove data-actions, data-tx_ready, data-suggestions part handling (suggestions get a new mechanism)
- src/features/agent/components/AgentChatArea.tsx — remove onTxReady/data-tx_ready detection
- src/features/agent/screens/AgentHomeScreen.tsx — remove onTxReady prop, simplify handleActionPress
- src/features/agent/hooks/useAgentChat.ts — remove data-tx_ready proposal syncing, remove handleActionPress title-as-text mechanism
- src/features/agent/hooks/useTransactionSigning.ts — signing triggered by build tool component, not pendingTx from store. Remove makeLocalAssistantMessage injection for signing state.
- src/features/agent/hooks/useAgentContext.ts — recentActions mechanism may need adjustment
- src/features/agent/stores/chatSessionStore.ts — pendingTx may be internalized into tool component state
- src/features/agent/components/tools/TxStatusCard.tsx — map NOT_FOUND to friendly "Pending" / "Confirming" display
- src/features/agent/lib/coalesceParts.ts — consider adding pre-build info tools (evm_tx_info when used for nonce/gas) to COALESCE_TOOLS

**Remove:**
- src/features/agent/components/tools/SignTransactionTool.tsx — orphaned, replaced by build tool cards
- src/features/agent/components/tools/TxResultCard.tsx — absorbed into build tool card lifecycle
- src/features/agent/components/TxPreviewCard.tsx — absorbed into build tool card lifecycle
- src/features/agent/components/DataActions.tsx — action buttons replaced by tool component UI
- src/features/agent/components/DataSuggestions.tsx — dropped (non-functional, no onPress handler; re-introduce later as a real feature)

**Add:**
- Build tool lifecycle components (e.g. BuildEvmTxCard, BuildSwapTxCard or a shared BuildTxCard) — show preview from tool output (read-only), show signing progress/result
- Confirmation panel mode for bottom input bar — when a build tool output arrives, input bar switches from chat input to "Approve or edit the action..." with confirm button. Confirm triggers password/biometric → signing. Edit allows parameter adjustment without a new chat turn.

### Backend (agent-backend)

**Modify:**
- internal/service/agent/prompt.go — remove "EVERY response MUST use respond_to_user" requirement. LLM generates text naturally and calls tools directly. Client-side tools (add_coin, create_vault, etc.) called as tool invocations, not wrapped in respond_to_user actions.
- internal/service/agent/agent.go — simplify/remove processActions pipeline. Remove data-tx_ready emission from build_* tool processing (lines 944-949). Simplify clientSideActionTypes — these should be direct tool definitions the LLM can call, not action types that get converted.
- internal/service/agent/tools.go — add client-side tool definitions (add_coin, create_vault, etc.) so the LLM can call them directly
- internal/service/agent/types.go — simplify response types, remove/reduce ToolResponse/Action structures
- internal/service/agent/agent.go — remove processSuggestions and Redis suggestion storage (suggestions dropped, non-functional)
- internal/service/agent/agent.go — add title generation: after first assistant response in a conversation, if title is null, fire a cheap Haiku call (max_tokens: 20) with first user message to generate title. Persist via existing convRepo.UpdateTitle(), emit existing data-title SSE. No new tool schema, no LLM instruction changes, frontend unchanged.

**Reference:**
- internal/service/agent/protocol_v1.go — V1ToolInputAvailable is the mechanism to emit client-side tool calls. This stays.
- internal/service/agent/interceptor.go — action router may become unnecessary

### Skills (mcp repo)

**Modify:**
- internal/skills/files/send-transfer.md — rewrite for direct MCP tool flow. Remove "emit build_send_tx as respond_to_user action" pattern. Remove "emit sign_tx after confirmation." Describe: LLM calls chain-specific build tool, frontend component handles confirmation and signing.
- internal/skills/files/swap-trading.md — same rewrite. Remove respond_to_user action pattern. LLM calls build_swap_tx MCP tool directly.
- Other skills that reference respond_to_user action patterns (check: evm-token-transfer.md, utxo-transfer.md, custom-transactions.md, eip712-signing.md)

## Acceptance Criteria

- [ ] Build tool MCP calls (build_evm_tx, build_swap_tx, chain-specific build tools) registered in TOOL_UI_REGISTRY with lifecycle-aware components
- [ ] Build tool components show tx preview from part.output (read-only card), show signing progress, show success/error result
- [ ] Bottom input panel switches to confirmation mode when build tool output arrives — "Approve or edit the action..." with confirm button. Confirm triggers password/biometric → signing. No auto-fire, no render timing hacks.
- [ ] evm_tx_info/get_tx_status renders friendly status: NOT_FOUND maps to "Pending" or "Confirming", not raw string
- [ ] DataActions component and action-button-sends-title-as-text mechanism removed
- [ ] data-tx_ready side-channel removed — no duplicate SSE event from build_* tools, no onTxReady in AgentChatArea
- [ ] SignTransactionTool, TxResultCard, TxPreviewCard removed (functionality absorbed into build tool cards)
- [ ] No local text messages injected via makeLocalAssistantMessage for signing state — tool component owns the UI
- [ ] respond_to_user no longer required for every LLM response — LLM text is streamed directly, tools called directly
- [ ] Client-side tools (add_coin, create_vault, remove_coin, etc.) still work — emitted as tool_call SSE events without respond_to_user wrapper
- [ ] Suggestions dropped — DataSuggestions component and processSuggestions/Redis storage removed
- [ ] Conversation titles generated by backend Haiku call on first message (no LLM tool, frontend unchanged)
- [ ] send-transfer and swap-trading skills updated for direct MCP tool flow, no action/sign_tx emission
- [ ] Signing flow works end-to-end: user requests send → LLM chains MCP tools → build tool card shows preview → user approves → signing → result in card
- [ ] Existing tool cards (AddCoinTool, CreateVaultTool, PriceCard, etc.) continue working

## Gotchas

- build_swap_tx is both an MCP tool name AND an action type name — they mean different things. The MCP tool builds the actual tx; the action was a UI confirmation button. After refactor, only the MCP tool meaning should exist.
- Client-side tools (add_coin, create_vault, etc.) currently flow through respond_to_user → processActions → clientSideActionTypes → V1ToolInputAvailable. They need a path that bypasses respond_to_user but still emits the tool_call SSE event. The backend needs to recognize these as client-side and emit without executing.
- The signing flow in useTransactionSigning is deeply coupled to the pendingTx store state and the data-tx_ready trigger. Refactoring signing to be triggered by the confirmation panel requires removing the auto-fire useEffect and exposing an imperative trigger. The biometric/password approval, chain-specific signing logic, and tx receipt verification all remain intact — only the entry point changes.
- recentActions mechanism (passing tx results to backend context on next message) should survive the refactor — the backend needs to know what happened with signing.
- The non-streaming path (buildLoopResponse) also uses respond_to_user/processActions — both paths need updating.
- ShapeShift uses execution hooks (useExecuteOnce, useSwapExecution) with step-based state machines for multi-step tx flows. Good pattern to reference but the implementation details differ (Vultisig has chain-specific signing in useTransactionSigning, not in individual tool components).
- Conversation title extraction currently comes from respond_to_user's conversation_title field. Replaced by backend Haiku call on first message — uses existing data-title SSE and UpdateTitle, frontend unchanged.

## Notes

**2026-04-13T04:02:12Z**

## Design: Per-Tool-Card Transaction Refactor (Big-Bang)

### Architecture

Coordinated backend + frontend change. Remove respond_to_user as the mandatory response wrapper. LLM streams text directly and calls MCP tools directly. Backend emits suggestions and titles as standalone data parts/events (not wrapped in respond_to_user). Frontend renders build tool cards via TOOL_UI_REGISTRY, and those cards own the tx preview lifecycle.

### Transaction Flow (new)

```
User: "swap 5 ETH to USDC"
  → LLM chains: evm_tx_info → build_swap_tx
  → build_swap_tx output arrives as tool-output-available SSE
  → Frontend: TOOL_UI_REGISTRY["build_swap_tx"] → BuildTxCard
  → Card renders preview (action, route, fee, receive amount)
  → Password/biometric prompt appears in bottom panel
  → User enters password → signing proceeds
  → Card updates with signing progress → success/error result
```

### Key Changes

**Backend (agent-backend)**
- Remove respond_to_user from system prompt and tool definitions
- LLM text streams directly as text-delta events (already works for non-respond_to_user text)
- Remove data-tx_ready emission from build_* tool processing — the tool output SSE is sufficient
- processActions simplified: no more action routing for tx actions. Client-side tools (add_coin, create_vault, etc.) become direct tool definitions the LLM calls, emitted as tool-input-available without the respond_to_user → processActions → clientSideActionTypes pipeline
- Suggestions dropped entirely (non-functional — no onPress handler). Remove processSuggestions, Redis storage, DataSuggestions component. Re-introduce later as a real feature.
- Title generated by backend via cheap Haiku call (max_tokens: 20) on first message only, if title is null. Uses existing convRepo.UpdateTitle() and data-title SSE. Frontend unchanged.
- buildLoopResponse and emitLoopResponse both updated to match
- Non-streaming path updated in parallel

**Frontend (vultiagent-app)**
- Register build_evm_tx, build_swap_tx, and other build_* tools in TOOL_UI_REGISTRY with BuildTxCard component
- BuildTxCard: reads part.output for tx details, renders preview (action, route/chain, estimated fee, receive amount), triggers signing flow when output is available, shows signing progress and result — all within the card
- Signing trigger moves from pendingTx auto-fire to confirmation panel: bottom input bar switches to "Approve or edit the action..." mode when build output arrives. User confirm tap calls into useTransactionSigning imperatively. No auto-fire useEffect.
- Remove: DataActions.tsx, TxPreviewCard.tsx, SignTransactionTool.tsx, TxResultCard.tsx
- Remove: data-tx_ready handling from AgentChatArea.tsx and useAgentChat.ts
- Remove: makeLocalAssistantMessage injection for signing state in useTransactionSigning
- Remove: handleActionPress title-as-text mechanism
- DataSuggestions removed (non-functional, dropped from this refactor)
- TxStatusCard: map NOT_FOUND to friendly "Pending" / "Confirming"
- recentActions mechanism preserved — build card reports tx results the same way

**Skills (mcp repo)**
- Rewrite send-transfer.md, swap-trading.md, and other tx skills: LLM calls build_* MCP tool directly, no "emit as respond_to_user action" pattern, no "emit sign_tx after confirmation"

### BuildTxCard Design

Single shared component for all build_* tools. Renders:
- **Action** line: "SWAP 5 ETH → USDC" or "SEND 0.5 BTC → bc1q..."
- **Route/Chain**: THORChain, Ethereum, etc.
- **Est. fee**: from tool output
- **You receive** (swaps) or **Recipient receives** (sends): amount

Lifecycle states: preview → awaiting_approval → signing → success | error

The card reads tool output to populate fields. Different build_* tools have different output shapes — the card normalizes them (similar to how parseServerTx already handles chain-specific formats).

### Signing UX

Card renders preview (read-only). Bottom input panel switches from chat input to confirmation mode: "Approve or edit the action..." with a confirm button. User reads the card, then either confirms or edits. Confirm tap triggers password/biometric → signing proceeds. The user's tap is the synchronization point — no render gates, no requestAnimationFrame hacks, no InteractionManager timing. The useEffect auto-fire in useTransactionSigning (pendingTx → confirmTx) is replaced with an imperative call from the confirmation handler.

### What Stays Unchanged

- MCP tool architecture — tools still live on separate server
- LLM tool chaining — LLM still reasons between tool calls
- V1ToolInputAvailable / V1ToolOutputAvailable SSE event format
- Chain-specific signing logic in useTransactionSigning (biometric, password, chain dispatch)
- recentActions for passing tx results to backend context
- Existing non-tx tool cards (AddCoinTool, CreateVaultTool, PriceCard, etc.)
- coalesceParts for collapsing noisy tool cards

### Risks

- **Client-side tool routing**: add_coin, create_vault etc. currently flow through respond_to_user → processActions → clientSideActionTypes. New path: LLM calls them as tools directly, backend recognizes them and emits tool-input-available without executing. Need to verify the backend can distinguish "client-side tool, don't execute on server" from "MCP tool, execute on server."
- **Signing timing**: Resolved — explicit user confirmation via bottom panel. No render timing concerns.
- **Build tool output normalization**: Different chains return different output shapes from build_* tools. BuildTxCard needs a normalizer — can base on existing parseServerTx / enrichBuildResult logic but this is where bugs will hide.

### Tasks

1. **Backend: Add client-side tool definitions and direct emission** — Add add_coin, create_vault, etc. as tool definitions the LLM can call. When backend sees these tool names, emit tool-input-available SSE without executing. Remove dependency on respond_to_user → processActions → clientSideActionTypes pipeline for these. Files: agent.go, tools.go, types.go

2. **Backend: Remove respond_to_user requirement and simplify response pipeline** — Remove respond_to_user from system prompt and tool definitions in prompt.go. Simplify processActions / emitLoopResponse / buildLoopResponse — text streams directly, no action routing for tx actions. Remove data-tx_ready emission from build_* processing. Both streaming and non-streaming paths. Files: prompt.go, agent.go, protocol_v1.go, parts_accumulator.go

3. **Backend: Drop suggestions, add title generation** — Remove processSuggestions, Redis suggestion storage, and suggestion SSE emission. Add title generation: after first assistant response, if conversation title is null, fire a Haiku call (max_tokens: 20) with first user message to generate title. Persist via existing convRepo.UpdateTitle(), emit existing data-title SSE. Files: agent.go, types.go

4. **Frontend: BuildTxCard component** — Create shared BuildTxCard registered for all build_* tools. Renders tx preview (action, route, fee, receive/send amount) from tool output. Normalizes different chain output shapes. Lifecycle states: preview → awaiting_approval → signing → success/error. Triggers signing via useTransactionSigning after card renders. Files: new component, toolUIRegistry.ts

5. **Frontend: Confirmation panel and signing trigger refactor** — Bottom input bar switches to confirmation mode ("Approve or edit the action...") when build tool output arrives. Confirm tap calls into useTransactionSigning imperatively. Remove auto-fire useEffect (pendingTx → confirmTx). Remove makeLocalAssistantMessage injection — card owns signing state display. Remove pendingTx from store (or internalize). Preserve chain-specific signing, biometric/password flow, and recentActions reporting. Files: useTransactionSigning.ts, chatSessionStore.ts, input bar component

6. **Frontend: Remove dead mechanisms** — Remove DataActions.tsx, DataSuggestions.tsx, TxPreviewCard.tsx, SignTransactionTool.tsx, TxResultCard.tsx. Remove data-tx_ready handling from AgentChatArea.tsx and useAgentChat.ts. Remove handleActionPress title-as-text mechanism. Remove onTxReady prop chain. Clean up AssistantMessageView.tsx part rendering for removed types. Files: multiple deletions and edits

7. **Frontend: TxStatusCard friendly status mapping** — Map NOT_FOUND to "Pending"/"Confirming" in TxStatusCard.tsx. Files: TxStatusCard.tsx

8. **Skills: Rewrite tx skills for direct MCP tool flow** — Update send-transfer.md, swap-trading.md, and other tx-related skills. Remove "emit as respond_to_user action" and "emit sign_tx" patterns. Describe direct build_* MCP tool invocation. Files: MCP repo skill files

Tasks 1-3 are backend (can be parallelized). Task 4-5 are sequential (card before signing refactor). Task 6 depends on 4-5. Task 7 is independent. Task 8 is independent (can parallel with everything).
