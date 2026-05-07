---
id: v-kkcj
status: closed
deps: []
links: []
created: 2026-04-23T23:03:01Z
type: bug
priority: 1
assignee: Jibles
---
# agent-backend: JWT auth hardening — expiry check on ParseUnverified + DEV_SKIP_AUTH prod guard

## Objective

Land the two security fixes from #128 as a small standalone PR: (a) add expiry check when JWT is parsed via `ParseUnverified`, (b) add a production guard so `DEV_SKIP_AUTH` can never be honored in a prod build. Both were praised by 0xApotheosis; both are strictly defensive; neither belongs anywhere near the tool-consolidation work.

## Context & Findings

### Reviewer praise

0xApotheosis on #128:
> "security fixes (JWT expiry on ParseUnverified, DEV_SKIP_AUTH prod guard) are correct and tested."

### What these fix

**JWT expiry on ParseUnverified**: `jwt.ParseUnverified` skips signature verification — used at spots where the service only needs to read claims (e.g., for logging, routing). But ParseUnverified also skips expiry validation. An expired token's claims were being trusted. The fix: explicitly check `exp` after ParseUnverified.

**DEV_SKIP_AUTH prod guard**: The env var `DEV_SKIP_AUTH=1` bypasses auth for local development. Without a build-mode guard, a production deployment that mistakenly carries this env var would run with auth disabled. The fix: refuse to honor `DEV_SKIP_AUTH` unless a dev/staging build flag is set.

### What this ticket covers

Touch only `internal/auth/jwt.go` + its test file. Extract from #128 branch by diff-ing `internal/auth/` against main.

Do NOT include any of the tool-consolidation work. This PR should be 100 lines or less.

## Files

- `internal/auth/jwt.go` — add the two fixes
- `internal/auth/jwt_test.go` — add coverage for both

## Acceptance Criteria

- [ ] PR opened as **draft** against agent-backend main, title `fix(auth): JWT expiry check + DEV_SKIP_AUTH prod guard`
- [ ] `ParseUnverified` call sites either inline-check expiry or wrap in a helper that does
- [ ] `DEV_SKIP_AUTH` is no-op in prod builds (guard via build tag or runtime env like `ENV=production`)
- [ ] Tests cover: expired-token rejection, still-valid token acceptance, `DEV_SKIP_AUTH` honored in dev, ignored in prod
- [ ] `go test ./internal/auth/...`, `golangci-lint run`, `go build ./...` clean
- [ ] PR notes this is **independent and mergeable off main**
- [ ] PR body quotes the 0xApotheosis endorsement

## Gotchas

- This is security-adjacent — follow the repo CLAUDE.md instruction: "Changes to key material, signing flows, or address derivation require extra scrutiny. Flag in PR descriptions." Not quite that category, but close; flag clearly.
- The prod-guard implementation choice matters: build-tag is compile-time safe but requires build-system cooperation; runtime `ENV` check is more flexible but only as safe as the env. Go with whichever the codebase already uses for similar cases.
- Do not bundle with the fan-out work (ticket 5) — keep as its own tiny PR for clean security-review trail.

## Notes

**2026-04-23T23:43:35Z**

Draft PR opened: https://github.com/vultisig/agent-backend/pull/166

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/agent-backend#166 (DRAFT)
