---
id: v-uawr
status: open
deps: []
links: []
created: 2026-05-03T22:42:19Z
type: bug
priority: 0
assignee: Jibles
---
# Hallucinated assistant action types crash the frontend

## Problem
The agent emits assistant message metadata containing `actions[].type` values that aren't in the registered action-type list. Frontend tries to dispatch them and crashes. Two confirmed cases in Felix's 2026-05-01 dump:

- conv `8ce2cc9e`: `actions[0].type = "read_evm_contract"` for "Check Polymarket Positions". Frontend errored: `outputTypes.map is not a function. (In 'outputTypes.map((t, i) => ({ name: \`out${i}\`, type: toAbiType(t) }))', 'outputTypes.map' is undefined)`.
- conv `ce25b4e7`: `actions[0].type = "thorchain_query"` with `params: { asset: "sports", query_type: "polymarket_search" }`. Pure invention — model fabricated an action type to satisfy a sports-search request it couldn't fulfill.

Neither type is registered in `vultiagent-app/src/features/agent/lib/toolLabels.ts` (canonical action-type registry).

## Background
### Findings
- Action-type registry: `vultiagent-app/src/features/agent/lib/toolLabels.ts` — flat dict of known types (`get_price: 'Fetching price'`, etc., 100+ entries).
- Action emission shape from backend: assistant message `metadata.actions: [{ id, type, title, params }]`.
- Frontend dispatch maps `type` to a handler. Unknown types fall through to a code path that assumes a specific param shape (`outputTypes.map`) and crashes.
- Dump locations: `full-dump.sql.gz` rows where `metadata->'actions'->0->>'type' IN ('read_evm_contract','thorchain_query')`.

## Constraints
- Validator must be on the server side, not just the frontend, so older app builds don't keep crashing on bad types.

## Non-goals
- Adding new action types to handle the user intents the model was reaching for. Goal here is just to stop the crash; legitimate gaps in agent capability are tracked elsewhere.

## Open Questions
- On unknown type, what's the fallback — drop the action silently, replace with a "this isn't supported yet" text response, or surface a structured error?
- Where does the registry live canonically — `toolLabels.ts` is frontend; is there a server-side equivalent we should validate against, or do we need to ship one?
- How does the model invent these — is the prompt suggesting type names indirectly, or pattern-matching from examples? Worth a glance during planning.
