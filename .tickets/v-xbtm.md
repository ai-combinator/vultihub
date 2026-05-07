---
id: v-xbtm
status: closed
deps: []
links: []
created: 2026-05-03T22:45:24Z
type: bug
priority: 1
assignee: Jibles
---
# Polymarket 'Invalid order payload' fix (depends on investigation)

## Problem
Implement the fix for Polymarket order submission failures with `"Invalid order payload"`. Reference: separate P0 investigation ticket — that ticket reproduces, root-causes, and decides whether to fix or hide for release. This ticket implements the fix once that decision is made.

If the investigation hides Polymarket from the LLM for v1, this ticket is the un-hide + fix work for v1.1. If the investigation lands a fix pre-release, this ticket may not be needed (close as duplicate).

## Background
### Findings
Refer to the investigation ticket for findings — full reproduction, root cause, and chosen fix shape live there.

## Constraints
- Don't start work on this ticket until the investigation ticket has produced its decision.
- Investigation outcome may make this ticket redundant — re-evaluate scope before starting.

## Open Questions
- All — root cause and fix shape are determined by the investigation ticket. This ticket is a placeholder for the fix work.

## Notes

**2026-05-05T02:40:49Z**

killed: Polymarket cut from v1 scope (was gated on v-xfzc)
