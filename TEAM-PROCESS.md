# Team Process

We optimize for speed, AI-first workflows, and autonomy with alignment. No sprints, story points, standups, blocking PR reviews, or backlog grooming.

---

## Missions

Shaped pieces of work with a clear goal. Each mission has:

- **Goal**: what "done" looks like
- **Priority**: 1 (highest) through 3
- **Blocked by**: other missions that must finish first, or "none"

When you finish a mission, pick the highest-priority unblocked mission available. Missions are tracked as [GitHub issues](https://github.com/ai-combinator/vultihub/issues).

Engineers decompose missions however they want — BYO task tracking.

---

## Rhythm

| Cadence | What | Format |
|---------|------|--------|
| **2-week cycles** | New/updated missions posted | Async |
| **Weekly** | Course correction, architecture drift, blocked decisions | 30 min sync |
| **Daily** | Automated git activity digest | Async, bot-generated |
| **End of cycle** | 2-day cool-down: refactor, explore, clean up | Unstructured |

### Team Pulse (Daily, Automated)

A scheduled agent summarizes each engineer's git activity (PRs merged, active branches, areas touched) and posts to Discord. No manual writing.

Footer: **"Any blockers or FYIs?"** — reply in thread only if you have something.

### Weekly Sync

Not a status update — everyone knows status from the pulse. Agenda:
- Course corrections ("I think we should change approach on X")
- Blocked decisions that need real-time discussion

### Cool-Down

2 days at the end of each cycle. Unstructured. Refactor, pay down debt, explore, write an ADR, clean up.

---

## Git & PRs

- **Commit prefixes**: `feat:`, `fix:`, `chore:`
- **PRs required**, no direct push to main
- **Self-test, self-merge.** CI must pass. No human approval required.
- **Squash or rebase**, your choice per PR. No merge commits.
- **Pre-merge review required for**: key material, signing flows, address derivation. Tag someone and wait.
- Post-merge review is optional and pull-based.

### Main Stays Green

If main is broken, fixing it is everyone's top priority — especially whoever broke it. Revert or fix, then go back to your mission. Never leave main broken.

### Cross-Cutting Changes

If your changes break interfaces or affect multiple areas, drop a Discord message: what changed, what breaks, what others need to do.

### SDK Interface Changes

Post the proposed change and rationale. Give people a day to object before merging.

---

## CI

Target: **under 2 minutes**, blocking merge.

| Check | Tool |
|-------|------|
| Lint + format | `biome check .` |
| Type check | `tsc --noEmit` |
| Unit/integration tests | `vitest run` |
| Go lint | `golangci-lint` |
| Go tests | `go test -short` |

Testnet integration tests run on a schedule, non-blocking.
