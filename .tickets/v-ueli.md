---
id: v-ueli
status: closed
deps: []
links: [v-wcgz]
created: 2026-05-03T22:41:44Z
type: bug
priority: 0
assignee: Jibles
---
# Conversation titles leak refusal text and assistant prose

## Problem
9 of 47 conversations in Felix's 2026-05-01 dump have titles that are leaked refusal/disclaimer text from the assistant's first paragraph. Sidebar reads like a generic chatbot, not a wallet. Examples (from `/home/sean/Downloads/2026-05-01-debug-export/2026-05-01-debug-export/conversations.jsonl`):

- `"I'm Claude, an AI assistant. I can help you with:\n\n- **Wr..."` (conv `f7b9b981`)
- `"I don't have access to real-time price data. Bitcoin's pr..."` (conv `6b9039ee`)
- `"I don't have real-time data access, so I can't tell you t..."` (conv `5b579479`)
- `"I can't monitor prices or send you alerts. I don't have r..."` (conv `87690619`)
- `"AAVE Token Price Overview\n\nI don't have real-time price d..."` (conv `ec56bde0`)
- `"Checking Largest DeFi Protocol TVLs\n\nI don't have real-ti..."` (conv `d3fb9a54`)

Note: the underlying assistant responses in these conversations are usually fine — it's the title summarizer that's regressing into refusal mode. So this is a title-generation bug, not a primary-response bug.

## Background
### Findings
- Title generation appears to grab the first paragraph (or summarized form) of the assistant's response.
- Many leaked titles aren't even from the user-visible response — they look like the title summarizer is being prompted independently and refusing on its own (e.g. "AAVE Token Price Overview\n\nI don't have real-time price data..." has structure suggesting a summarization-prompt response, not an in-conversation message).
- Length: titles include `\n` characters and run >60 characters in many cases — independent of refusal content, the format is wrong.

## Current Thinking
Investigate title-summarizer prompt + truncation behavior. Likely fixes:
- Make the title summarizer aware it's a wallet (so "I don't have access to real-time data" never makes sense as a title).
- Filter known-bad starts (`"I'm "`, `"I don't"`, `"I can't"`, `"I'm sorry"`) and retry summarization.
- Hard-truncate to a single line + ~50 char cap.

Pick one or all — diagnose first.

## Constraints
- Don't regress titles that are already fine (the dump has plenty of clean ones: "Polymarket Fed Chair Bet", "Bitcoin Price Today", etc.).

## Non-goals
- Re-titling existing conversations. Forward-only fix.

## Open Questions
- Where does title generation happen — SendMessage handler, separate worker, the app? Trace this first.
- Is the title summarizer a separate LLM call or post-processing of the assistant's first response?
- Should we add user-editable titles regardless?

## Notes

**2026-05-05T05:32:29Z**

migrated to GH issue: https://github.com/vultisig/vultiagent-app/issues/430

**2026-05-05T05:34:06Z**

kept open locally — GH issue is a duplicate, local remains canonical scratch

**2026-05-06T22:17:42Z**

Superseded by v-wcgz. GH #430 / app PR #441 closed the user-visible sidebar leak, and backend #271 added marker guards, but the remaining root-cause work is a backend title-persistence invariant: all title writes should pass through one canonical normalizer/validator before persistence.
