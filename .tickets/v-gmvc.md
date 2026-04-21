---
id: v-gmvc
status: open
deps: []
links: []
created: 2026-04-15T21:54:24Z
type: task
priority: 2
assignee: Jibles
parent: v-bauf
---
# Swap flow: one tool with internal stepper (drop multi-build collapsing)

## Why
Swap flows emit multiple build_* tool calls per user turn (quote → final tx, approval → swap). The frontend papers over this in AssistantMessageView.tsx (currently lines 114, 184-199) via findLastBuildToolIndex — earlier build_* tool parts collapse to a compact TimelineEntry so only the final card renders. Shapeshift-agentic avoids this: the swap is a single tool (InitiateSwapUI.tsx:38-127) with an <Execution.Stepper> driving named sub-steps (QUOTE, NETWORK, APPROVE, SWAP) from one evolving tool state.

## What
- Reshape the swap flow so the backend emits one tool call (e.g. execute_swap) whose streamed output transitions through named sub-steps.
- Build a SwapFlowTool.tsx with an internal stepper — model on shapeshift-agentic/apps/agentic-chat/src/components/tools/InitiateSwapUI.tsx and Execution.tsx.
- Delete findLastBuildToolIndex, lastBuildToolIndex, and the collapse branch from AssistantMessageView.tsx.
- Audit other multi-step flows (send with approval, Polymarket bet + sign) for the same pattern.
- Do NOT delete TimelineEntry yet — still used by BuildTxCard and SignTypedDataTool; separate cleanup.

## Acceptance
- A swap quote + sign produces one visual card with internal progress.
- lastBuildToolIndex / collapsing logic removed.
- Existing swap e2e / maestro tests pass.
