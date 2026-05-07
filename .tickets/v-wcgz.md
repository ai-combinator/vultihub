---
id: v-wcgz
status: open
deps: []
links: [v-ueli]
created: 2026-05-06T22:17:37Z
type: bug
priority: 1
assignee: Jibles
---
# Centralize backend conversation title validation before persistence

## Problem
Conversation titles can still originate as free-form LLM text from multiple backend ingress paths. The current app-side sanitizer fixes the sidebar symptom, and backend persona-leak guards reject known bad strings, but the stronger invariant is missing: only title-shaped data should ever be persisted as a conversation title. Solved means every backend title write goes through one canonical normalizer/validator before `UpdateTitle`, with app sanitization kept only as legacy-data/read-side defense.

## Background
### Findings
- Original local ticket `v-ueli` reported conversation titles leaking assistant refusal/prose text into the sidebar. Examples included `"I'm Claude, an AI assistant..."`, `"I don't have access to real-time price data..."`, and multi-paragraph titles with newline content.
- GitHub issue `vultisig/vultiagent-app#430` was closed as completed by merged PR `#441` (`Fix conversation title sanitizer`) on 2026-05-05.
- PR `vultiagent-app#441` sanitized titles on the app read/update path:
  - `vultiagent-app/src/features/agent/lib/helpers.ts` has `formatConversationTitle(rawTitle)` with a 40-character cap, first-paragraph extraction, whitespace collapse, escaped newline handling, and bad-start regex fallback to `New Conversation`.
  - `vultiagent-app/src/features/agent/lib/queries.ts` applies `formatConversationTitle(conversation.title ?? '')` when normalizing fetched conversations.
  - `vultiagent-app/src/features/agent/hooks/useConversations.ts` applies `formatConversationTitle(title)` on live `updateConversationTitle` cache updates.
- That app fix is display/read-side. It does not prevent a bad title from being generated or persisted server-side.
- Backend `agent-backend` currently has source-side guards:
  - `internal/service/agent/agent.go` `generateTitle(parentCtx, userMessage)` strips markdown, runs `looksLikePersonaLeak(title)`, logs a structured warning, and returns empty instead of persisting known bad persona/refusal titles.
  - `buildLoopResponse` and `emitLoopResponse` also strip `toolResp.ConversationTitle`, run `looksLikePersonaLeak(stripped)`, and only then call `convRepo.UpdateTitle`.
  - Tests exist in `internal/service/agent/title_persona_leak_test.go`, including marker coverage and a source-shape pin that both `toolResp.ConversationTitle` ingress sites call `looksLikePersonaLeak(stripped)`.
- The backend guards were introduced by `agent-backend` commit `e6fb6f78 feat(agent/title): drop persona-leak responses from cheap-model title generator (paaao spike #7) (#271)`. Its own commit body explicitly says it addressed the symptom/title leak, not the deeper cause of the model treating the title prompt as a chat prompt.
- Backend `origin/main` contains the persona-leak and `toolResp.ConversationTitle` gates.

## Current Thinking
Where we landed: the app patch was useful but is not the root-cause fix. It hides bad persisted titles and protects existing polluted rows, so keep it as defense-in-depth. The backend marker guard is closer to correct because it blocks known bad values before persistence, but it is still a set of string filters around free-form LLM output.

The pragmatic correct fix is to centralize title persistence behind a single backend invariant. Shape should be roughly: all backend code that might write a conversation title calls one helper such as `normalizeConversationTitle(raw) (title string, ok bool)` before `convRepo.UpdateTitle`. That helper should enforce title-shaped data, not just known-bad strings: trim, strip harmless wrappers, collapse whitespace, reject empty, reject paragraph/newline prose, reject assistant/refusal/persona starts, cap by rune length, and return `ok=false` for anything that should fall back to untitled/default UI.

The planner should prefer making invalid persisted titles unrepresentable over adding another call-site filter. If direct `convRepo.UpdateTitle` remains public and widely callable, the invariant can still be bypassed. Consider either a service-level `updateConversationTitle` wrapper used by all agent paths, or moving normalization into the repository method if every caller should get the same invariant.

Keep the frontend `formatConversationTitle` in place after the backend change. It is still valuable for legacy rows and as a final UI guard, but it should no longer be the main correctness mechanism.

## Constraints
- Do not re-title existing conversations as part of this ticket. Original `v-ueli` explicitly scoped this as forward-only.
- Do not remove the app sanitizer in `vultiagent-app`; it protects existing persisted bad data and streamed edge cases.
- Preserve clean existing title behavior such as `Polymarket Fed Chair Bet`, `Bitcoin Price Today`, `Send 1 ETH`, and similar short wallet/action titles.
- Keep comments WHY-only and short per repo guidance.
- This spans `agent-backend` primarily. `vultiagent-app` changes should be unnecessary except perhaps test/docs references.

## Assumptions
- The intended remaining work is backend invariant hardening, not reopening GH issue `vultiagent-app#430`.
- All active backend title persistence paths are in `internal/service/agent/agent.go`, but the planner should grep for `UpdateTitle`, `ConversationTitle`, and any create/update endpoint that accepts title before finalizing the plan.
- A deterministic or schema-constrained title-generation rewrite is desirable but may be larger than this ticket. The pragmatic target is one canonical validation path before persistence.

## Non-goals
- Replacing the title generator model wholesale unless planning shows it is low-risk and clearly cheaper than centralized validation.
- Migrating or cleaning existing database rows.
- Removing the frontend sanitizer.
- Changing visible title fallback copy beyond existing `New Conversation` behavior.

## Dead Ends
- App-only sanitizer: already shipped in `vultiagent-app#441`; fixes the sidebar symptom but allows bad values to exist in durable backend state.
- Per-call-site marker checks only: better than app-only, and currently present, but still easy to bypass when a new title write path is added.
- Relying on prompt wording alone: title generation is probabilistic; even explicit “ONLY the title” prompts have already needed defensive scrubbing.

## Open Questions
- Should normalization live in the service layer around title writes, or inside `ConversationRepository.UpdateTitle` so persistence itself enforces the invariant?
- Should the canonical backend max length match app display length (`40`) or backend storage/title emission length (`60` today)? If they differ, document why.
- Should rejected titles emit only structured logs, metrics, or both?
- Is there any remaining endpoint or admin flow that intentionally needs to persist arbitrary user-supplied titles, and should it share or bypass this invariant?
