---
id: va-yvtt
status: closed
deps: []
links: []
created: 2026-04-07T23:16:50Z
type: task
priority: 2
assignee: Jibles
---
# Remove NativeWind and TrueSheet to fix Android e2e and simplify codebase

## Objective

Remove NativeWind (nativewind / react-native-css-interop) and @lodev09/react-native-true-sheet from the app. Both libraries cause real Android bugs that required architectural workarounds for Detox e2e tests, and analysis shows neither provides meaningful value over vanilla React Native alternatives already used elsewhere in the codebase. After migration, simplify the e2e test infrastructure that was built to work around these issues.

## Context & Findings

### Why NativeWind should go

- **The bug it causes:** `Animated.createAnimatedComponent(Pressable)` combined with NativeWind's `className` prop triggers a React setState-during-render error on Android. NativeWind's CssInterop wraps components in additional views that conflict with Reanimated's animated component wrapping. This is a known NativeWind v4 bug (GitHub issues #1712, #1466, #1522). The error overlay covers the entire screen, making all elements untappable.
- **Current workaround:** PressableScale uses `Animated.View` (layout+animation) with a transparent `Pressable` absoluteFill overlay for touch+testID. This works but is an unusual architecture forced by the CssInterop conflict.
- **Usage is minimal:** 152 `className` occurrences across 19 files. Meanwhile, inline `style` / `StyleSheet.create` is used 658 times across 72 files — the codebase is already 80%+ vanilla styles. Every `className` usage is basic utility classes (`flex-row`, `items-center`, `bg-[#hex]`, `rounded-[12px]`). Zero responsive breakpoints, zero dark mode, zero Tailwind features beyond what inline styles provide.
- **Migration is mechanical:** Each `className` maps 1:1 to inline styles. No logic changes needed.
- **What we gain:** Removal of CssInterop babel plugin, metro config integration, and the nativewind-env.d.ts type shim. PressableScale can go back to a simple `AnimatedPressable` single-element architecture (or even simpler). One fewer native dependency and build complexity layer.

Files with className usage (all 19):
- `src/components/ui/Badge.tsx` (2)
- `src/components/ui/IconCircle.tsx` (2)
- `src/components/ui/PressableScale.tsx` (5)
- `src/components/ui/PrimaryButton.tsx` (3)
- `src/components/ui/StepIndicator.tsx` (2)
- `src/features/agent/components/AgentConfirmationCard.tsx` (18)
- `src/features/agent/components/VaultPicker.tsx` (14)
- `src/features/scheduling/components/ScheduleAddedBanner.tsx` (2)
- `src/features/scheduling/components/ScheduleBadge.tsx` (2)
- `src/features/scheduling/components/ScheduleCard.tsx` (10)
- `src/features/scheduling/components/SuggestionChip.tsx` (6)
- `src/screens/onboarding/import/components/DecryptPasswordSheet.tsx` (10)
- `src/screens/onboarding/import/components/FileDropZone.tsx` (2)
- `src/screens/onboarding/import/components/FileImportArea.tsx` (8)
- `src/screens/onboarding/import/ImportSelectionSheet.tsx` (15)
- `src/screens/onboarding/import/ImportVaultScreen.tsx` (6)
- `src/screens/onboarding/seedphrase/SeedPhraseExplainScreen.tsx` (16)
- `src/screens/onboarding/seedphrase/SeedPhraseScreen.tsx` (14)
- `src/screens/scan/ScanQRScreen.tsx` (15)

### Why TrueSheet should go

- **The bug it causes:** TrueSheetView.kt on Android overrides `getChildCount()`/`getChildAt()` to delegate to its viewController, but the viewController's content is also in a CoordinatorLayout. Detox (and Android accessibility/TalkBack) finds every testID'd element twice through dual traversal paths. This is an unreported bug in the library.
- **Current workaround:** A pnpm native patch on TrueSheetView.kt that returns 0 from getChildCount(). Works but is fragile across version upgrades.
- **Usage is trivial:** Exactly 1 component (`VaultPicker.tsx`) uses `<TrueSheet>`. Features used: `cornerRadius`, `backgroundColor`, `detents` (snap points), `onDidDismiss`, `grabberOptions`.
- **The app already has the replacement:** `ImportSelectionSheet` and `DecryptPasswordSheet` both implement bottom sheets using vanilla `<Modal animationType="slide" transparent>` with identical UX (grabber bar, backdrop dismiss, rounded corners). The pattern is proven and working.
- **What we lose:** iOS native drag-to-dismiss gesture (minor polish). What we gain: no native patch, consistent sheet implementation, one fewer native dependency.

### E2E test simplification after migration

With NativeWind and TrueSheet removed:
- **PressableScale** can be simplified back to a single `AnimatedPressable` element (no absoluteFill overlay needed since there's no CssInterop conflict without className)
- **TrueSheet pnpm patch** can be deleted entirely
- **`dismissLogBox()`** calls may still be needed (LogBox is a React Native dev-mode thing, not library-specific), but the warnings that triggered LogBox may be reduced
- The `.atIndex(0)` workaround was already removed, and should stay removed — verify it's not needed after migration
- **`e2e/__tests__/importVault.test.js`** unit test for `isAndroid()` helper should still work with its dedicated jest config
- Run the full Android e2e test suite to verify all button taps, navigation, and sheet interactions work without workarounds

### Rejected approaches

- **Keeping NativeWind with workarounds:** The absoluteFill Pressable overlay works but has a caveat — nested interactive children inside PressableScale won't be tappable. This is acceptable today but a lurking issue.
- **Replacing TrueSheet with @gorhom/bottom-sheet:** Adding a dependency to replace a dependency. Not worth it for a single usage when vanilla Modal works.
- **Removing Reanimated:** Deeply integrated across 16 files with features that have no vanilla equivalent (gesture-driven animations, interpolateColor, layout animations, 20-bar concurrent waveform). Keep it.

## Files

### NativeWind removal
- All 19 files listed above — convert className to inline style
- `babel.config.js` — remove nativewind preset/plugin
- `metro.config.js` — remove nativewind/metro integration if present
- `tailwind.config.js` — delete
- `nativewind-env.d.ts` — delete
- `package.json` — remove nativewind, react-native-css-interop dependencies

### TrueSheet removal
- `src/features/agent/components/VaultPicker.tsx` — replace TrueSheet with Modal (follow ImportSelectionSheet pattern)
- `patches/@lodev09__react-native-true-sheet@3.10.0.patch` — delete
- `package.json` — remove @lodev09/react-native-true-sheet dependency

### PressableScale simplification (after NativeWind removal)
- `src/components/ui/PressableScale.tsx` — simplify back to single AnimatedPressable (no className, no absoluteFill overlay)
- `src/components/ui/PrimaryButton.tsx` — restore z-[1] if needed, or simplify since no NativeWind

### E2E test verification
- Run full Android e2e suite: `DETOX_CONFIGURATION=android.emu.debug npx detox test --configuration android.emu.debug`
- Run iOS e2e suite: `npx detox test --configuration ios.sim.debug`
- Verify no `.atIndex(0)`, no try/catch tap fallbacks, no text-based tap workarounds remain

## Acceptance Criteria

- [ ] All `className` props replaced with inline styles across 19 files
- [ ] NativeWind, react-native-css-interop removed from dependencies and build config
- [ ] tailwind.config.js and nativewind-env.d.ts deleted
- [ ] TrueSheet replaced with Modal in VaultPicker.tsx (following ImportSelectionSheet pattern)
- [ ] @lodev09/react-native-true-sheet removed from dependencies
- [ ] TrueSheet pnpm patch file deleted
- [ ] PressableScale simplified to single-element architecture (no absoluteFill overlay)
- [ ] No `className` prop on PressableScale or any component
- [ ] Android e2e tests pass without `.atIndex(0)`, try/catch fallbacks, or text-based tap workarounds
- [ ] iOS e2e tests still pass
- [ ] App renders correctly on both platforms (manual spot-check onboarding flow, vault picker, import sheets)
- [ ] TypeScript compiles without errors

## Gotchas

- PressableScale can ONLY go back to AnimatedPressable after NativeWind is removed — the CssInterop conflict is specifically with className on animated components
- VaultPicker uses TrueSheet's imperative `present()`/`dismiss()` API — replace with `useState(visible)` + `useImperativeHandle` like ImportSelectionSheet does
- Some className values use Tailwind arbitrary notation like `bg-[#061B3A]` — convert to `backgroundColor: '#061B3A'`
- The `buffer` and `react-native-css-interop` dependencies added to package.json for pnpm hoisting — `react-native-css-interop` can be removed entirely; check if `buffer` is still needed independently
- LogBox.ignoreAllLogs() in index.ts and dismissLogBox() in test helpers should stay — those address a React Native dev-mode issue, not a library issue

## Notes

**2026-04-08T00:31:42Z**

All tasks complete: NativeWind removed (19 files migrated to inline styles), TrueSheet replaced with Modal in VaultPicker, PressableScale simplified to single AnimatedPressable, build config cleaned up, all config/patch files deleted. TypeScript compiles, lint passes, 302/302 tests pass.
