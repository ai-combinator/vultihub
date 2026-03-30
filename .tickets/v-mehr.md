---
id: v-mehr
status: open
deps: []
links: [v-hfrl]
created: 2026-03-30T02:52:35Z
type: task
priority: 2
assignee: Jibles
---
# Migrate vultiagent-app from npm to pnpm

## Problem Statement

vultiagent-app uses npm as its package manager with no particular reason — no npm-specific config, no .npmrc, no npm-only features in use. As we move toward a unified SDK monorepo (see v-hfrl), pnpm is the better foundation: proper workspace support, strict dependency resolution (no phantom deps), faster installs, and disk-efficient via content-addressable store. "Solved" means vultiagent-app runs on pnpm with no regressions.

## Research Findings

### Current State
- Package manager: npm, confirmed by package-lock.json (no yarn.lock, pnpm-lock.yaml, or packageManager field)
- No .npmrc configuration
- No npm-specific features used (no overrides, no npm audit in CI)
- 94 dependencies, 13 devDependencies
- Expo SDK 55 + React Native 0.83 + TypeScript strict mode
- Native modules: expo-dkls (custom), vault-sharing (custom), plus standard Expo modules
- Build tooling: Metro bundler, Expo CLI, Detox for E2E
- vultisig-windows uses yarn workspaces; vultiagent-cli uses npm — no consistency across repos today

### Available Tools & Patterns
- pnpm has first-class workspace support via pnpm-workspace.yaml
- Expo and React Native are compatible with pnpm — Expo docs reference pnpm setup
- Metro bundler works with pnpm when node_modules/.pnpm is configured (may need metro.config.js symlink resolution tweak)
- pnpm has a `pnpm import` command that converts package-lock.json → pnpm-lock.yaml

### Validated Behaviors
- Confirmed: no npm-specific config or scripts that would break
- Confirmed: Expo SDK 55 supports pnpm (Expo docs mention it)
- Confirmed: Metro bundler may need `unstable_enablePackageExports` or symlink resolver config for pnpm's strict node_modules structure

## Constraints

- Metro bundler (React Native) uses a non-standard module resolution algorithm — pnpm's strict node_modules layout (symlinked, not flat) can cause "module not found" errors if Metro isn't configured for it
- expo-dkls native module uses standard Expo module resolution — should work, but needs testing
- Detox E2E tests should be verified after migration
- CocoaPods (iOS) resolves node_modules paths — may need node_linker=hoisted or specific pnpm config

## Open Questions

- Should we use `node-linker=hoisted` in .npmrc (pnpm config) for maximum compatibility with RN/Expo, or strict mode with Metro resolver config? Hoisted is safer for initial migration but loses pnpm's strictness benefit.
- Should this migration happen before or after the SDK unification (v-hfrl)? Doing it first means less churn later. Doing it after means we only migrate once when the monorepo structure is final.
- Does the Detox test runner have any npm-specific assumptions?
