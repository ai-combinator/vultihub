---
id: va-urql
status: closed
deps: []
links: []
created: 2026-04-22T00:14:00Z
type: chore
priority: 1
assignee: Jibles
---
# Monorepo POC: collapse SDK + mcp-ts + vultiagent-app into single pnpm workspace

## Problem Statement

The vultiagent team ships across three repos (`vultisig-sdk`, `mcp-ts`, `vultiagent-app`). Any SDK change that touches the agent path currently requires coordinated PRs across 2–3 repos plus waiting on npm publish cycles. The public SDK is consumed by external projects (vultisig-windows, iOS, Android, extension) on a release cadence the agent team does not control. Goal: collapse the three into a single pnpm-workspace monorepo hosted in the current `mcp-ts` repo (to be renamed `vultiagent` after merge), eliminating cross-repo PRs and cutting SDK publish-latency to zero via `workspace:*` references. **POC executed on a branch so the team can evaluate a working migration before committing** — the agent running this ticket must produce a fully working branch pitch-ready by end of the session.

## Why mcp-ts as the host (not vultisig-sdk)

- mcp-ts is already pnpm → no package-manager migration for the host itself
- mcp-ts is team-owned → no political blockers, safe to restructure
- mcp-ts is the smallest of the three → least existing cruft (no changesets, no husky, no 50+ scripts, no 15-package publish pipeline)
- mcp-ts already has a `pnpm.overrides` entry pointing at `../vultisig-sdk/packages/sdk` via `file:` — half the workspace wiring is already intuited
- CI auto-deploys only on push to `main` (see `.github/workflows/ci.yml`), so a branch is production-safe

## Architecture Reality Check

Verified state (as of ticket creation):

- **`vultisig-sdk`** is `yarn@4.13.0` with `nodeLinker: node-modules` (RN-compatible, not PnP). `resolutions` block in root `package.json` needs translation to pnpm overrides syntax (nested paths use `>` not `/`).
- **`mcp-ts`** is pnpm 10.31.0, has `pnpm-workspace.yaml` (currently dockerignored to avoid prod build picking up the sibling SDK override). `package.json` has `pnpm.overrides: { "@vultisig/sdk": "file:../vultisig-sdk/packages/sdk" }` — remove once SDK is a real workspace member.
- **`vultiagent-app`** is npm-based, uses **Biome** already (not ESLint/Prettier). `metro.config.js` already has `unstable_enableSymlinks = true` with a comment referencing local `@vultisig/mpc-native` iteration against the sibling SDK → Metro monorepo prep is largely done.
- **Fly deployment:** app name `vultisig-mcp-ts` is fly-side state, not repo state. URL that `agent-backend` points at does not change across migration. Only `Dockerfile` and `fly.toml` build paths need updating.
- **CI:** `.github/workflows/ci.yml` runs typecheck/lint/test on PRs, deploys to fly on push to `main`. Branch pushes don't deploy.
- **Branch point:** `refactor/v-pxuw-tool-consolidation` on mcp-ts (will be rebased off `main` before this ticket starts — user handles that separately).

## Constraints

- **New branch:** `monorepo-poc` on `mcp-ts`, branching off `refactor/v-pxuw-tool-consolidation`.
- **Host repo:** transform `mcp-ts` **in place**. Do not create a new repo. GitHub repo rename (`vultisig-mcp-ts` → `vultiagent`) happens AFTER merge, out of scope here.
- **Package manager:** pnpm everywhere. No yarn, no npm, no bun.
- **Linter/formatter:** Biome only. Single root `biome.json`. Delete all ESLint + Prettier configs and devDeps across all imported workspaces.
- **Publishing:** fully ripped out. No changesets, no husky, no lint-staged, no `release` scripts. This monorepo does NOT publish to npm. External consumers (Windows, iOS, Android, extension) continue pulling from the public `vultisig-sdk` on npm — unaffected by this fork.
- **Native/WASM deps kept from npm:** `@vultisig/walletcore-native`, `@vultisig/mpc-native`, `@vultisig/mpc-wasm` stay as published npm deps. Do NOT vendor them into `packages/`. The WASM toolchain is gnarly and not on the agent team's critical path.
- **Source imports use copy-in, not git subtree.** One commit per imported tree with upstream SHA recorded in the message. Future graduation back to public SDK happens manually via `git format-patch` on atomic SDK-only commits.
- **Downtime on mcp-ts fly deploy during cutover is acceptable.** `vultiagent-app` is unreleased — zero deployment concern.

## Target layout

```
<repo root — transformed from mcp-ts in place>
├── apps/
│   ├── vultiagent/          # Expo app (was vultiagent-app/)
│   └── agent-mcp/           # @vultisig/agent-mcp — hosted tool server (was mcp-ts src)
├── clients/
│   ├── cli/                 # @vultisig/cli — 3rd-party harness CLI (was vultisig-sdk/clients/cli/)
│   └── wallet-mcp/          # @vultisig/wallet-mcp — 3rd-party local MCP harness (was vultisig-sdk/clients/mcp/)
├── packages/
│   ├── sdk/                 # @vultisig/sdk
│   ├── core/
│   │   ├── chain/           # @vultisig/core-chain
│   │   ├── config/          # @vultisig/core-config
│   │   └── mpc/             # @vultisig/core-mpc
│   ├── lib/
│   │   ├── dkls/            # @vultisig/lib-dkls
│   │   ├── mldsa/           # @vultisig/lib-mldsa
│   │   ├── schnorr/         # @vultisig/lib-schnorr
│   │   └── utils/           # @vultisig/lib-utils
│   ├── rujira/              # @vultisig/rujira
│   ├── client-shared/       # @vultisig/client-shared (used by cli + wallet-mcp)
│   └── mpc-types/           # @vultisig/mpc-types
├── biome.json
├── pnpm-workspace.yaml
├── .npmrc
├── Dockerfile               # monorepo-aware; builds apps/agent-mcp
├── fly.toml                 # app = "vultisig-mcp-ts" (unchanged)
├── package.json
└── tsconfig.base.json
```

## Package naming — critical distinction

Two MCP servers coexist in this monorepo. Do NOT conflate them:

| Package | Path | Purpose | Consumer |
|---|---|---|---|
| `@vultisig/wallet-mcp` | `clients/wallet-mcp/` | 3rd-party local harness, narrow wallet-ops surface | End users via Claude Desktop / Cursor / etc. running locally |
| `@vultisig/agent-mcp` | `apps/agent-mcp/` | Hosted tool server on fly.io, broad surface (wallet + DeFi + analytics + upstream proxies like Nansen/Etherscan/deBridge) | `agent-backend` (Go) over the wire |

`@vultisig/wallet-mcp` was previously `@vultisig/mcp` (inside vultisig-sdk/clients/mcp/). `@vultisig/agent-mcp` was previously `vultisig-mcp-ts` (the flat mcp-ts repo). Rename both during import.

`@vultisig/cli` stays `@vultisig/cli` — no rename.

## Scope — End to End (phased)

### Phase 1: Scaffolding (scaffold-only commit, no moves yet)

- Add `pnpm-workspace.yaml`:
  ```yaml
  packages:
    - apps/*
    - clients/*
    - packages/*
    - packages/core/*
    - packages/lib/*
  ```
- Add `.npmrc`:
  ```
  node-linker=hoisted
  auto-install-peers=true
  strict-peer-dependencies=false
  ```
  `node-linker=hoisted` is the key RN/Expo compatibility setting — makes pnpm install flat like npm, avoids Metro/Gradle/Xcode symlink edge cases.
- Add root `biome.json` — one ruleset, format + lint. Base on the existing `vultiagent-app/biome.json` (team already uses it there). Include overrides for test file globs.
- Rewrite root `package.json`:
  - Remove all `changeset:*`, `release`, `pack:shared`, `docs`, `update`, `prepare` (husky), 20+ `test:e2e:*` variants
  - Keep minimal: `typecheck`, `lint` (= `biome check`), `format` (= `biome check --write`), `test` (= `pnpm -r test`), `build` (= `pnpm -r build`), `dev` (= per-filter helper)
  - Remove `lint-staged` block entirely
  - Remove `husky` devDep
  - Remove all `eslint-*`, `prettier*`, `@typescript-eslint/*` devDeps
  - Keep shared devDeps at root only when truly shared (typescript, biome)
  - Replace `resolutions` with `pnpm.overrides` (see Phase 6 for the exact mapping)
- Add `tsconfig.base.json` at root; each workspace extends it. Strip SDK's `.config/tsconfig.json` layering.
- **Commit:** `chore: scaffold pnpm workspace, biome, npmrc, root package.json`

### Phase 2: Move mcp-ts source to apps/agent-mcp/

- `mkdir -p apps/agent-mcp`
- `git mv src tests skills upstreams.json tsup.config.ts vitest.config.ts tsconfig.json apps/agent-mcp/`
- Copy current `package.json` from root → `apps/agent-mcp/package.json`; rename `"name": "vultisig-mcp-ts"` to `"name": "@vultisig/agent-mcp"`
- Remove `pnpm.overrides` block from `apps/agent-mcp/package.json` (the sibling SDK override — now obsolete)
- Flip deps: `"@vultisig/mpc-types": "^0.1.2"` → `"workspace:*"`, `"@vultisig/rujira": "^10.0.0"` → `"workspace:*"`, `"@vultisig/sdk": "^0.15.1"` → `"workspace:*"`
- Preserve `Dockerfile`, `fly.toml`, `.dockerignore` at root (they'll be rewritten in Phase 9)
- Delete any `CLAUDE.md`, `README.md`, `RECIPES.md`, `RUJIRA_DOCS.md` that belong to the server specifically → move into `apps/agent-mcp/`
- Delete `test-parity.sh` and `eslint.config.js` and any prettier configs
- **Commit:** `chore: move mcp-ts source to apps/agent-mcp/`

### Phase 3: Import vultisig-sdk packages

From `/home/sean/Repos/vultisig/vultisig-sdk` (`main` branch, record SHA in commit message):

- Copy these source trees (NOT their root-level config files):
  - `packages/sdk/` → `packages/sdk/`
  - `packages/core/chain/` → `packages/core/chain/`
  - `packages/core/config/` → `packages/core/config/`
  - `packages/core/mpc/` → `packages/core/mpc/`
  - `packages/lib/dkls/` → `packages/lib/dkls/`
  - `packages/lib/mldsa/` → `packages/lib/mldsa/`
  - `packages/lib/schnorr/` → `packages/lib/schnorr/`
  - `packages/lib/utils/` → `packages/lib/utils/`
  - `packages/rujira/` → `packages/rujira/`
  - `packages/client-shared/` → `packages/client-shared/`
  - `packages/mpc-types/` → `packages/mpc-types/`
- **Explicitly DO NOT copy:**
  - `packages/mpc-wasm/`, `packages/mpc-native/`, `packages/walletcore-native/` — keep consuming these from npm
  - `examples/browser/`, `examples/electron/` — not our concern
  - `.changeset/`, `.config/`, `.husky/`, `scripts/`, `docs/` — SDK's publish/release machinery, dropped
- Strip each imported package's local `.eslintrc*`, `.prettierrc*`, `.prettierignore` files if present
- Strip any husky/lint-staged hooks inside imported packages
- Keep each package's own `tsconfig.json`, `vitest.config.ts`, `package.json`, `README.md`, `src/`, `tests/`
- **Commit:** `chore: import @vultisig/sdk and core/lib/rujira/client-shared/mpc-types from vultisig-sdk@<sha>`
  - Record the exact SHA from `vultisig-sdk` in the commit message body.

### Phase 4: Import clients from vultisig-sdk

- From vultisig-sdk `clients/cli/` → `clients/cli/` (package name unchanged: `@vultisig/cli`)
- From vultisig-sdk `clients/mcp/` → `clients/wallet-mcp/` — rename package in `package.json` from `@vultisig/mcp` to `@vultisig/wallet-mcp`; update `bin` entries if they reference the old name (they're currently `vmcp` and `vultisig-mcp`, can keep or rename to `vwallet-mcp`)
- Both already use `workspace:^` style refs — they'll resolve correctly once pnpm-workspace.yaml picks them up
- Strip `publishConfig` blocks from both (we're not publishing)
- **Commit:** `chore: import clients/cli + clients/wallet-mcp from vultisig-sdk@<sha>`

### Phase 5: Import vultiagent-app to apps/vultiagent/

From `/home/sean/Repos/vultisig/vultiagent-app` (record SHA in commit message):

- Everything → `apps/vultiagent/` except `node_modules/`, `package-lock.json`
- Rename package in `apps/vultiagent/package.json` from `vultiagent-poc` to `@vultisig/vultiagent` (internal-only name, not published)
- Flip `@vultisig/*` deps to `workspace:*`:
  - `@vultisig/sdk`
  - `@vultisig/mpc-types`
  - Keep `@vultisig/mpc-native` and `@vultisig/walletcore-native` as published npm versions (not workspace)
- Keep `jest.config.js`, `babel.config.js`, `metro.config.js`, `biome.json` intact — these are app-specific
- Metro `watchFolders` might need to include the monorepo root; check and update if needed (Expo's monorepo helpers handle most of this). The app's current `unstable_enableSymlinks = true` is still valid.
- Keep `android/`, `ios/`, `app.json`, `eas.json`, `maestro/`, `e2e/` inside `apps/vultiagent/`
- **Commit:** `chore: import vultiagent-app to apps/vultiagent/ from vultiagent-app@<sha>`

### Phase 6: Wire workspace refs, pnpm overrides, install

- Root `package.json` `pnpm.overrides`:
  ```json
  "pnpm": {
    "overrides": {
      "readable-stream": "3.6.2",
      "@noble/hashes": "1.8.0",
      "@lifi/sdk>@solana/web3.js": "1.98.4",
      "@cosmjs/encoding>@scure/base": "1.2.6"
    }
  }
  ```
  Note: nested path uses `>` not `/` (pnpm syntax vs yarn `resolutions` syntax).
- Audit every `@vultisig/*` import across all workspaces; confirm workspace:* resolution
- Delete root-level lockfile from the old mcp-ts (`pnpm-lock.yaml` at repo root) and regenerate:
  ```bash
  rm pnpm-lock.yaml
  pnpm install
  ```
- Fix any resolution errors. Expect some duplicate-version warnings — most are acceptable with `node-linker=hoisted`.
- Watch for React version pinning: SDK root had `@types/react ^19.2.14`, app has `react 19.2.5`. They should align. If two different React instances end up hoisted, Expo throws "Invalid hook call" at runtime → fix by pinning or adjusting peerDeps in `packages/client-shared/`.
- **Commit:** `chore: wire workspace refs, pnpm overrides, lockfile`

### Phase 7: Rip ESLint / Prettier / husky / changesets / lint-staged

- Delete from anywhere they exist across the repo:
  - `.eslintrc*`, `eslint.config.*`, `.eslintignore`
  - `.prettierrc*`, `.prettierignore`, `prettier.config.*`
  - `.husky/` directory
  - `.changeset/` directory
  - `.lintstagedrc*`, `lint-staged` blocks in package.json
  - `CHANGELOG.md` files inside clients (generated by changesets, stale once publish is ripped)
- Remove devDeps from all package.json files:
  - `eslint`, `eslint-config-*`, `eslint-plugin-*`, `@eslint/*`, `@typescript-eslint/*`
  - `prettier`, `eslint-config-prettier`, `eslint-plugin-prettier`
  - `husky`, `lint-staged`
  - `@changesets/*`
  - `typedoc` (SDK docs generation — not needed)
  - `knip` (SDK's dead-code check — keep or drop; team call)
- Remove scripts: `docs`, `docs:watch`, `changeset*`, `release`, `prepare`, `knip*`, `pack:shared`, `check`, `check:all` (regenerate `check` later with Biome if desired)
- **Commit:** `chore: rip ESLint/Prettier/husky/changesets — biome-only lint+format`

### Phase 8: Biome migration / cleanup pass

- One root `biome.json` covers the whole monorepo. Base on current `vultiagent-app/biome.json`. Adjust globs for `apps/**`, `clients/**`, `packages/**`.
- Run `pnpm exec biome check --write` across everything. Expect:
  - Import ordering deltas (Biome re-sorts)
  - Unused-import removals
  - Some TS `any` widening/narrowing rule diffs
  - Possibly RN-specific Babel-runtime patterns that Biome doesn't understand; add ignores as needed
- **Known Biome coverage gaps vs ESLint** (accept, don't re-add ESLint):
  - `no-floating-promises` — no direct equivalent
  - Some stricter `@typescript-eslint/*` rules not in Biome
  - `eslint-plugin-react-hooks` rules (Biome has them but less complete)
- Drop `biome.json` from `apps/vultiagent/` (now redundant with root)
- **Commit:** `chore: run biome check --write across monorepo`

### Phase 9: Update Dockerfile + fly.toml for monorepo

Rewrite root `Dockerfile` using pnpm's canonical monorepo pattern:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-slim AS base
ENV PNPM_HOME=/pnpm
ENV PATH=$PNPM_HOME:$PATH
RUN corepack enable && corepack prepare pnpm@10.31.0 --activate
WORKDIR /app

FROM base AS build
COPY pnpm-workspace.yaml package.json pnpm-lock.yaml .npmrc ./
COPY packages ./packages
COPY clients ./clients
COPY apps/agent-mcp ./apps/agent-mcp

RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile --filter=@vultisig/agent-mcp...

RUN pnpm --filter=@vultisig/agent-mcp build
RUN pnpm --filter=@vultisig/agent-mcp deploy --prod /prod/agent-mcp

FROM node:22-slim AS runtime
ENV NODE_ENV=production
ENV MCP_HTTP_PORT=9091
WORKDIR /app
COPY --from=build --chown=node:node /prod/agent-mcp ./
COPY --chown=node:node apps/agent-mcp/skills ./skills
COPY --chown=node:node apps/agent-mcp/upstreams.json ./upstreams.json
USER node
EXPOSE 9091
CMD ["node", "dist/index.js", "--http"]
```

The `...` suffix after the filter means "this package + all its workspace transitive deps", so `packages/sdk`, `packages/core/*`, `packages/lib/*`, `packages/rujira`, `packages/client-shared`, `packages/mpc-types` all get pulled into the install.

`pnpm deploy --prod` extracts the package + its resolved deps into a flat, self-contained folder suitable for runtime.

Update `.dockerignore`:
- REMOVE the `pnpm-workspace.yaml` exclusion (previous workaround, no longer needed — workspace IS the monorepo now)
- ADD `apps/vultiagent` (don't ship the app into the agent-mcp image)
- Keep `.git`, `node_modules`, `coverage`, test files, `.env*` exclusions

Update `fly.toml`:
- `app = "vultisig-mcp-ts"` — UNCHANGED (agent-backend points at this)
- `[build] dockerfile = "Dockerfile"` — UNCHANGED (still at repo root)
- No other changes needed; env vars, ports, regions all stay

Update `.github/workflows/ci.yml`:
- `pnpm install --frozen-lockfile` step stays
- `pnpm typecheck` / `pnpm lint` / `pnpm test` still work (root scripts)
- `flyctl deploy --remote-only` still works from repo root — NO path changes needed because `fly.toml` is still at root

**Commit:** `chore: Dockerfile + fly.toml for monorepo build context`

### Phase 10: Smoke verification (for pitch readiness)

Agent must verify ALL of the following before declaring done:

1. `pnpm install` at repo root succeeds with no errors (warnings OK)
2. `pnpm --filter @vultisig/sdk build` succeeds
3. `pnpm --filter @vultisig/agent-mcp build && pnpm --filter @vultisig/agent-mcp test` succeeds
4. `pnpm --filter @vultisig/cli build && pnpm --filter @vultisig/cli test` succeeds
5. `pnpm --filter @vultisig/wallet-mcp build && pnpm --filter @vultisig/wallet-mcp test` succeeds
6. `pnpm --filter @vultisig/rujira build && pnpm --filter @vultisig/rujira test` succeeds (if tests exist)
7. `cd apps/vultiagent && pnpm typecheck` succeeds
8. `cd apps/vultiagent && pnpm test` succeeds (Jest)
9. `docker build -t monorepo-test -f Dockerfile .` succeeds locally
10. `cd apps/agent-mcp && pnpm dev:http` boots the MCP server on :9091; `curl localhost:9091/healthz` returns 200
11. **The money shot:** edit a file in `packages/sdk/src/`, see it reflected in both:
    - `pnpm --filter @vultisig/agent-mcp dev` (agent-mcp picks up change)
    - `cd apps/vultiagent && pnpm expo start` (app hot-reloads)
    Document this in a `MIGRATION.md` or `README.md` note for the pitch.

If any smoke check fails, fix it in-session. Do not hand over a broken branch.

## Commit structure (target)

10 commits in phase order. Don't combine phases into one mega-commit; don't split a phase into multiple micro-commits. Huge import commits (3, 4, 5) are reviewed at the `git log -1` level — their size is expected. Wiring commits (1, 2, 6, 7, 8, 9, 10) should be genuinely small and line-by-line reviewable.

```
1. chore: scaffold pnpm workspace, biome, npmrc, root package.json
2. chore: move mcp-ts source to apps/agent-mcp/
3. chore: import @vultisig/sdk and core/lib/rujira/client-shared/mpc-types from vultisig-sdk@<sha>
4. chore: import clients/cli + clients/wallet-mcp from vultisig-sdk@<sha>
5. chore: import vultiagent-app to apps/vultiagent/ from vultiagent-app@<sha>
6. chore: wire workspace refs, pnpm overrides, lockfile
7. chore: rip ESLint/Prettier/husky/changesets — biome-only lint+format
8. chore: run biome check --write across monorepo
9. chore: Dockerfile + fly.toml for monorepo build context
10. chore: MIGRATION.md — pitch summary + smoke verification notes
```

## Dead ends (rejected approaches — do NOT revisit in execution)

- **Fork vultisig-sdk as the host** — rejected. Inherits 50+ scripts, 15-package publish pipeline, changesets, husky, ESLint, lint-staged, examples, docs — all of which we'd then delete. mcp-ts is smaller, team-owned, and already pnpm. Net-cheaper to build up from there.
- **Bun workspaces** — rejected. Not production-ready for RN/Expo; Metro has known resolution issues with bun's linking.
- **Keep yarn 4 at the root** — rejected. Team preference is pnpm, mcp-ts is already pnpm, translating yarn `resolutions` to pnpm `overrides` is a 5-line change.
- **Turborepo from day 1** — deferred. Workspaces alone are enough. Add Turborepo when CI duration actually becomes annoying; it's additive, no cost to defer.
- **Zero-downtime fly cutover via staging app** — rejected. User accepted some mcp-ts downtime. Transform in place; break and fix.
- **Git subtree merge to preserve SDK/app history** — rejected. Clean slate. Record upstream SHA in commit message; use `git format-patch` for future graduation of atomic SDK commits back to public vultisig-sdk.
- **Keep publishing to npm from this repo** — rejected. External consumers use the public vultisig-sdk. This is a private fork for the agent team; YAGNI on publish.
- **Vendor `@vultisig/mpc-wasm` / `mpc-native` / `walletcore-native`** — rejected. Stable, WASM toolchain is gnarly, not on the agent critical path. Keep consuming from npm.
- **Include vultisig-sdk `examples/*`** — rejected. Upstream's concern.
- **Rename `@vultisig/*` package scopes** — rejected. Huge churn across thousands of imports, no benefit since we're not publishing.
- **Collapse `clients/wallet-mcp` and `apps/agent-mcp` into one MCP** — rejected. Different products, different consumers, different scopes (narrow vs broad tool surface), different deployment models (local vs hosted).
- **Drop `clients/cli` and `clients/wallet-mcp`** — rejected. Both are team-owned AI-harness products, come along for the ride.
- **Drop `packages/client-shared`** — rejected. Used by `clients/cli` and `clients/wallet-mcp`. Verify in Phase 4 that it's still imported; if NOT imported, drop at that point.
- **Parallel-run new repo alongside mcp-ts** — rejected. In-place transform is simpler for a branch-based POC.
- **Add Husky pre-commit Biome hook** — deferred. Open question for later.
- **Unify Jest (app) → Vitest (everything else)** — rejected. Vitest + RN babel transform is fragile. Two test runners coexisting is fine.

## Open questions (non-blocking — decide during execution if needed)

1. **Biome rule gaps** — accept the delta from ESLint, or schedule a follow-up lint-hardening pass? Default: accept, flag in MIGRATION.md.
2. **`packages/client-shared` trimming** — Expo app might not use it. Check during Phase 5 dep audit; drop from workspace if not imported by `apps/vultiagent/`.
3. **`fly.toml` location** — stay at root (simpler, no `-c` flag needed) or move to `apps/agent-mcp/fly.toml` (cleaner, but requires `flyctl deploy -c apps/agent-mcp/fly.toml` in CI). Default: stay at root.
4. **Dockerfile location** — same tradeoff. Default: stay at root.
5. **`knip` devDep** — SDK uses it for dead-code detection. Drop or keep? Default: drop, revisit if missed.
6. **React version alignment** — if Phase 6 install surfaces duplicate React, decide between pinning in root `pnpm.overrides` vs adjusting `packages/client-shared` peerDeps. Default: pin via override.
7. **Lockfile conflict resolution** — imported SDK packages have exact-semver pins to each other (`packages/core/*` reference each other with exact versions for `npm pack` off-repo). Once they're `workspace:*` this is moot, but the literals might still be in package.json. Update to `workspace:*` during import.

## Not in scope (explicitly deferred)

- Adding Turborepo / Nx
- Automating graduation of changes back to public vultisig-sdk
- Migrating `@vultisig/walletcore-native` / `mpc-native` / `mpc-wasm` source into the monorepo
- Unifying test runners
- `agent-backend` (Go) repo — stays separate; talks to `apps/agent-mcp` over MCP wire protocol
- Cutover of `main` branch to the new layout (happens at merge time, after team approval)
- Fly.io deployment from the new branch (branch doesn't auto-deploy per CI config)
- GitHub repo rename `vultisig-mcp-ts` → `vultiagent` (post-merge)
- Renaming git remotes on dev machines (post-merge)

## Acceptance criteria

- [ ] Branch `monorepo-poc` created on `mcp-ts`, branched off `refactor/v-pxuw-tool-consolidation`
- [ ] Commit log follows the 10-phase structure (or fewer via merge, never more)
- [ ] `pnpm install` at root succeeds without errors
- [ ] All workspace builds pass: `@vultisig/sdk`, `@vultisig/agent-mcp`, `@vultisig/cli`, `@vultisig/wallet-mcp`, `@vultisig/rujira`
- [ ] All workspace tests pass: same list
- [ ] `cd apps/vultiagent && pnpm typecheck && pnpm test` passes
- [ ] `cd apps/vultiagent && pnpm expo start` boots without errors
- [ ] `docker build -t monorepo-test -f Dockerfile .` succeeds locally
- [ ] `curl localhost:9091/healthz` returns 200 against local `pnpm --filter @vultisig/agent-mcp dev:http`
- [ ] Edit-in-sdk → hot-reload-in-app verified and documented
- [ ] No `eslint-*`, `prettier*`, `@typescript-eslint/*`, `husky`, `lint-staged`, `@changesets/*` deps remain in any package.json
- [ ] No ESLint/Prettier config files remain anywhere
- [ ] No `.changeset/`, `.husky/` directories remain
- [ ] Root `biome.json` exists; per-workspace Biome configs deleted
- [ ] `.npmrc` has `node-linker=hoisted`
- [ ] `MIGRATION.md` at repo root: pitch summary, before/after, smoke verification notes, known gotchas
- [ ] `fly.toml` app name unchanged (`vultisig-mcp-ts`)
- [ ] `.dockerignore` updated (removed `pnpm-workspace.yaml` exclusion, added `apps/vultiagent`)

## References

- Existing SDK: `/home/sean/Repos/vultisig/vultisig-sdk` — yarn@4 workspaces, node-modules linker, 15 packages, publishes via changesets
- Existing MCP: `/home/sean/Repos/vultisig/mcp-ts` — pnpm 10.31.0, file: override to sibling SDK, deploys to fly
- Existing app: `/home/sean/Repos/vultisig/vultiagent-app` — npm, Biome, Expo 55, React 19, metro with symlinks
- SDK CLAUDE.md: `/home/sean/Repos/vultisig/vultisig-sdk/CLAUDE.md` (layout, build commands, conventions)
- mcp-ts CLAUDE.md: `/home/sean/Repos/vultisig/mcp-ts/CLAUDE.md` (tool patterns, default port, fly)
- Expo monorepo guide: https://docs.expo.dev/guides/monorepos/
- pnpm deploy docs: https://pnpm.io/cli/deploy
- Biome: https://biomejs.dev/
- v-pxuw ticket: prior multi-repo refactor (tool consolidation) for format reference

## Notes

**2026-05-04T23:09:20Z**

auto-closed: tracked in PR vultisig/mcp-ts#41 monorepo POC (DRAFT)
