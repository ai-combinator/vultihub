---
id: v-bauf
status: open
deps: []
links: []
created: 2026-04-15T21:53:59Z
type: epic
priority: 2
assignee: Jibles
---
# Simplify AssistantMessageView to match shapeshift pattern

## Why
AssistantMessageView diverges from shapeshift-agentic's simple parts-iteration pattern because backend quirks forced frontend workarounds. After the data-tools migration (agent-backend 205c038), the backend shape is closer to shapeshift's, but the frontend compensating layers remain.

Reference: /home/sean/Repos/shapeshift-agentic/apps/agentic-chat/src/components/AssistantMessage.tsx — 40 lines: text, registered tool component, or null. No timeline entries, no aggregate chips, no generic fallback, no data-* custom handling.

## Children
- Consolidate MCP balance fan-out server-side (deletes coalesceParts + AggregateToolIndicator)
- Model multi-build swap flows as one tool with internal stepper (deletes findLastBuildToolIndex)

## Already done in plan vivid-riding-creek
- Drop GenericToolIndicator fallback
- Move data-confirmation into a schedule_task tool card
- Backend: split clientSideTools() and relocate data-tool defs to data_tools.go
