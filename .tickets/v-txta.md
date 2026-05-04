---
id: v-txta
status: in_progress
deps: []
links: []
created: 2026-04-23T23:04:12Z
type: bug
priority: 2
assignee: Jibles
---
# vultiagent-app: Trustwallet Maven plugin — read TRUSTWALLET_PAT via Gradle providers

## Objective

Extract the Trustwallet Maven plugin Gradle-providers fix (commit `ee3776b` on `refactor/v-pxuw-tool-consolidation`) into its own tiny drive-by PR. Fix: `plugins/withTrustwalletMavenRepo.js` reads `TRUSTWALLET_PAT` via Gradle providers instead of `System.getenv`, which gives a clearer error when the env var is missing.

## Context & Findings

### Reviewer findings

**0xApotheosis on #164:**
> "`getOrElse("")` means a dev who forgets to export TRUSTWALLET_PAT gets a cryptic GitHub Packages 401 (bearer with empty token) instead of a clear 'env var missing' error."

**Gomes escalated to scope drift on #164** (Suggestion 11):
> "flagging this as scope drift: this Maven auth rewrite isn't mentioned in the PR summary… One of the harder debugging experiences is 'I pulled main and now my Android build fails opaquely' — a tiny dedicated PR makes it trivially greppable later."

This ticket implements gomes' recommendation.

### Coordination needed

Gomes owns a parallel branch `feat/trustwallet-maven-plugin` (sha `3d81585`, no PR open) that has earlier Maven plugin work (via already-merged PR #152 shipping the plugin file itself) but does NOT have commit `ee3776b`. **Before opening this PR, check with gomes**: does he want this fix on his branch, or as a new PR?

### What this ticket covers

**Cherry-pick:** commit `ee3776b` (`fix(plugins): read TRUSTWALLET_PAT via Gradle providers, not System.getenv`) from `refactor/v-pxuw-tool-consolidation`.

Apply the `getOrElse("")` → fail-fast change so missing env var surfaces a clear error.

## Files

- `plugins/withTrustwalletMavenRepo.js` — the single file affected

## Acceptance Criteria

- [ ] PR opened as **draft** against vultiagent-app main, title `fix(android): Trustwallet Maven plugin surfaces clear error when PAT missing`
- [ ] Missing `TRUSTWALLET_PAT` env now fails fast with clear message, not silent empty-bearer 401
- [ ] `pnpm android:build` (or equivalent) succeeds when PAT is set; fails clearly when not
- [ ] PR notes this is **independent and mergeable off main**
- [ ] PR body quotes 0xApot's finding verbatim as context
- [ ] Coordinated with gomes before opening (note in PR body)

## Gotchas

- Do not pull in any other commits from `refactor/v-pxuw-tool-consolidation` — this is a surgical single-commit extract
- Android build verification needs an Android dev environment; if not available, state in PR body as "manual Android verification pending"
- If gomes' `feat/trustwallet-maven-plugin` branch ends up merging first, this PR becomes a smaller delta — rebase accordingly

## Notes

**2026-04-23T23:43:10Z**

Draft PR opened: https://github.com/vultisig/vultiagent-app/pull/239
