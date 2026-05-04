---
id: v-btxo
status: closed
deps: []
links: []
created: 2026-04-30T04:52:45Z
type: task
priority: 2
assignee: Jibles
---
# vultiagent-app: parseServerTx refactor — dispatch on tx_encoding for execute_* envelope across all chain families

## Problem Statement

The vultiagent-app parser (`parseServerTx` in `src/features/agent/lib/txProposal.ts`) was never updated when mcp-ts migrated swap/send tools to the unified `execute_*` envelope (`{chain, tx_encoding, ...family-specific fields}`). The parser still pattern-matches on legacy field shapes (`swap_tx`, `transactions[]`, top-level `from/to/amount`) and never reads `tx_encoding`. EVM canonical envelopes work by accident via a legacy fallback in `getServerTx`; Solana-canonical is confirmed broken (this manifests as v-uneu); UTXO / Cosmos / Tron / TON / Sui / Cardano are likely broken via the same gap. "Solved" means `parseServerTx` dispatches on `tx_encoding` for canonical envelopes, falls through to legacy pattern-matching when absent, with per-chain handlers covering the 8 actively-emitted encodings, and existing safety carve-outs preserved.

## Research Findings

### Current State
- `parseServerTx` (`vultiagent-app/src/features/agent/lib/txProposal.ts:241`) is a flat ~700-line function that pattern-matches on field shapes. The string `tx_encoding` does not appear once.
- `getServerTx` (line 88) reads `data.swap_tx ?? data.send_tx ?? data.tx`. The execute_* EVM envelope nests its tx under `data.tx`, so EVM-canonical works by accident.
- Solana branches (lines 248, 379, 423) check `data.swap_tx`, `data.tx`, `mint`, top-level `to/amount` — none recognise `{chain, tx_encoding: 'solana-tx', data: <base64>}`. Confirmed broken end-to-end via diagnostic log capture: `topLevelKeys: ["chain", "data", "tx_encoding"]`.
- Sui legacy branch (line 253) requires `swap_tx.data` — does not match canonical `{tx_encoding: 'sui-tx', from, to, amount}`.
- UTXO branch (line 431) requires top-level `from/to/amount/feeRate`. Canonical envelope shape provides these (with `fee_rate` snake-case) plus `tx_encoding`/`memo` — may match accidentally; needs per-chain verification.
- Existing safety carve-outs that must survive the refactor: foreign-marker rejection on Solana (`coin_type`, `jetton_master`), mixed-chain rejection on `transactions[]`, SPL-malformed throw (`TokenPayloadMalformedError`), evm-sequence length cap (`MAX_EVM_SEQUENCE_LEN = 8`).

### Available Tools & Patterns
- `mcp-ts/src/tools/execute/_base.ts` — `TxEncoding` discriminator union, `EvmTxArgs` narrow type, `buildEvmTxArgs` builder. The comments document the wire contract explicitly.
- `mcp-ts/src/tools/execute/_thorchain.ts` — `buildThorchainLpTxArgs` for LP add/remove returns `{...payload, chain: 'THORChain', tx_encoding: 'cosmos-msg', from, to: '', msg_type: 'deposit'}`.
- 4 existing test files for txProposal (`txProposal.test.ts`, `txProposal.lpContract.test.ts`, `txProposal.tokensMvp.test.ts`, `txProposal.xrp.test.ts`) cover legacy paths and act as regression net for the move.
- Diagnostic log added at `signAndBroadcast.ts` near the parse-null branch was used to capture the v-uneu Solana shape; same approach can verify remaining chains during smoke-test.

### Validated Behaviors
mcp-ts emits these `tx_encoding` values from `execute_*` tools (verified by reading every emission site in execute_send.ts, execute_swap.ts, execute_lp_*.ts, execute_contract_call.ts, _base.ts, _thorchain.ts):

- **`evm`** — `_base.ts:buildEvmTxArgs`, used by execute_send, execute_swap (general EVM + native EVM-router), execute_contract_call, execute_lp_*. Shape: `{chain, tx_encoding, chain_id, tx: {to, value, data, gas_limit, max_fee_per_gas, max_priority_fee_per_gas, nonce}}`.
- **`solana-tx`** — execute_send (native) AND execute_swap general (prebuilt). TWO sub-shapes:
  - Native: `{chain, tx_encoding, from, to, amount}`
  - Prebuilt: `{chain, tx_encoding, data: <base64>}`
- **`utxo-psbt`** — execute_send + execute_swap THORChain UTXO source. `{chain, tx_encoding, from, to, amount, memo, fee_rate}` for non-Zcash; Zcash drops `fee_rate` and adds `fee_note: 'ZIP-317 fee computed client-side'`.
- **`cosmos-msg`** — execute_send (`msg_type: 'send'`), execute_swap THORChain cosmos source (`msg_type: 'send'`), execute_lp_add/remove (`msg_type: 'deposit'`). Sub-discriminator: `msg_type`. LP shape spreads payload: `{...payload, chain: 'THORChain', tx_encoding, from, to: '', msg_type: 'deposit'}`.
- **`tron-tx`** — execute_send. `{chain, tx_encoding, owner_address, to_address, amount}` — note `owner_address`/`to_address`, NOT `from`/`to`.
- **`ton-tx`** — execute_send. `{chain, tx_encoding, from, to, amount, seqno, memo?}`.
- **`sui-tx`** — execute_send only (execute_swap Sui not implemented). `{chain, tx_encoding, from, to, amount}`.
- **`cardano-tx`** — execute_send only. `{chain, tx_encoding, from, to, amount, decimals}`.
- **`ripple-tx`** — declared in TxEncoding union but NEVER emitted. execute_send returns textError pointing the LLM at the legacy XRP send builder. The app must preserve its existing legacy XRP branch.

## Constraints
- Legacy paths MUST be preserved verbatim. mcp-ts has not migrated `build_swap_tx`, `build_*_send`, scheduled-proposal spread shapes, top-level `transactions[]` (Morpho prepare_*), SPL-with-mint, etc. These continue to be emitted indefinitely and the parser remains their consumer.
- Safety carve-outs (foreign-marker rejection, mixed-chain rejection, SPL-malformed, evm-sequence length cap) must continue to fail loud with existing error types.
- No consumer churn: `parseServerTx` and `ParsedServerTx` are imported across the agent feature folder; signatures must stay stable.

## Dead Ends
- **Chain-only dispatch (`if (chain === 'Solana') handleSolanaTx(data)`)** — rejected because each chain has multiple kinds determined by present fields, not by chain alone. Solana has 4 kinds (native, prebuilt, SPL, foreign-marker reject), EVM has 3 (single, sequence, with-approval), Cosmos has 6+ (cw20, wasm_execute, wasm_execute_multi, native send, LP add, LP remove). `tx_encoding` is by chain *family* (`evm` covers ~10 chains, `cosmos-msg` covers ~10), so chain-only dispatch wastes the family abstraction.
- **Patching only Solana for v-uneu** — rejected because the same gap exists across 6+ other chain families. Whack-a-mole as users hit each one.
- **Encoding-only dispatch (no chain check)** — insufficient: family-internal kinds need both `chain` AND `tx_encoding` (e.g. Zcash within `utxo-psbt` has a different shape than BTC).

## Open Questions

### File structure
- Per-chain helper files in a new `txProposal/` folder (`evm.ts`, `solana.ts`, etc., one per family + `index.ts` dispatcher), or keep `txProposal.ts` flat with a top-level `tx_encoding` switch and helper functions inline? Per-file is more navigable but adds ~9 files.
- If per-file: where do safety carve-outs live? Inside the chain file (e.g. foreign-marker rejection in `solana.ts`) or a shared `safetyChecks.ts` consumed by each?
- Existing test files are split per-feature (`txProposal.lpContract.test.ts`, `txProposal.tokensMvp.test.ts`, etc.). Do new canonical-envelope tests join `txProposal.test.ts` or get a dedicated `txProposal.canonical.test.ts`?

### Refactor approach
- Big-bang (dispatcher rewrite + 9 chain branches + tests in one PR), or staged (dispatcher + Solana first to unblock v-uneu, then one chain per follow-up PR)?
- Should the dispatcher fall through to legacy ONLY when `tx_encoding` is absent, or also when a present `tx_encoding` value yields `null` from its canonical branch? Latter is forgiving but could mask malformed canonical payloads.

### Coverage / verification
- All 8 emitted encodings smoke-tested manually before merging, or just SOL + ETH + 1 representative per remaining family (BTC for UTXO, THOR for cosmos-msg, etc.)?
- Should mcp-ts add unit tests asserting wire-shape stability per encoding so the parser contract is enforced from both sides?

### Sub-discrimination within encodings
- `solana-tx`: discriminator between native and prebuilt is presence of `data` (string, base64) vs `from/to/amount`. Is this robust, or should mcp-ts add an explicit sub-tag like `solana_kind: 'native' | 'prebuilt'`?
- `cosmos-msg`: only `msg_type: 'send' | 'deposit'` are emitted today. Are other msg_types in flight (cw20 execute, wasm_execute, contract calls)? If so, the current legacy parser branches need to be re-routed under the canonical dispatch when those tools migrate.

### Ripple
- `ripple-tx` is declared but never emitted (execute_send rejects with textError). Should the dispatcher have a placeholder branch that throws "not yet wired" if it ever appears, or fall through to legacy silently?

### v-uneu relationship
- Does this ticket subsume v-uneu (close v-uneu when refactor lands), or does v-uneu stay open as the user-facing bug entry tracked separately?

## Notes

**2026-04-30T05:18:18Z**

## Design (approved 2026-04-30)

**Locked decisions:**
- Big-bang single PR — dispatcher + all 8 emitted encodings + tests
- Strict fall-through — `tx_encoding` present but canonical branch returns null ⇒ throw, don't retry as legacy

### Architecture

Split `txProposal.ts` into a `txProposal/` folder. Big-bang adds ~400 lines of canonical handling on top of ~700; per-chain modules keep review hunks small and the dispatcher reads as a switch.

```
src/features/agent/lib/txProposal/
  index.ts          dispatcher + parseServerTx + ParsedServerTx union (re-exports for stable import sites)
  legacy.ts         everything currently in txProposal.ts that doesn't read tx_encoding (verbatim move)
  evm.ts            tx_encoding: 'evm' — single, with-approval, sequence (cap + mixed-chain reject stay here)
  solana.ts         tx_encoding: 'solana-tx' — native vs prebuilt sub-discrim by field presence; foreign-marker reject stays here
  utxo.ts           tx_encoding: 'utxo-psbt' — Zcash sub-shape (no fee_rate, has fee_note)
  cosmos.ts         tx_encoding: 'cosmos-msg' — sub-discrim on msg_type (send | deposit)
  tron.ts           tx_encoding: 'tron-tx' — owner_address/to_address mapping
  ton.ts            tx_encoding: 'ton-tx' — seqno + optional memo
  sui.ts            tx_encoding: 'sui-tx' — native send only (prebuilt stays in legacy)
  cardano.ts        tx_encoding: 'cardano-tx'
  ripple.ts         tx_encoding: 'ripple-tx' — placeholder that throws "not yet wired"
  shared.ts         readString/readNumber/resolveChainId helpers
```

`txProposal.ts` becomes a 1-line re-export stub so every existing consumer import keeps working unchanged.

### Dispatch flow (`index.ts`)

```
parseServerTx(data):
  1. resolvePendingTxChain(data) → chain | null. If null, return null.
  2. If data.tx_encoding is a string:
       switch (data.tx_encoding):
         'evm'        → parseEvmCanonical(data, chain)
         'solana-tx'  → parseSolanaCanonical(data, chain)
         'utxo-psbt'  → parseUtxoCanonical(data, chain)
         'cosmos-msg' → parseCosmosCanonical(data, chain)
         'tron-tx'    → parseTronCanonical(data, chain)
         'ton-tx'     → parseTonCanonical(data, chain)
         'sui-tx'     → parseSuiCanonical(data, chain)
         'cardano-tx' → parseCardanoCanonical(data, chain)
         'ripple-tx'  → throw 'ripple-tx canonical envelope not yet wired'
         default      → throw `unknown tx_encoding: ${data.tx_encoding}`
     Each canonical parser throws on a structurally-malformed envelope (strict fall-through).
     If chain doesn't match encoding family (e.g. tx_encoding 'evm' on Solana), throw.
  3. Else (no tx_encoding) → parseLegacy(data, chain) — verbatim current logic.
```

### Sub-discrimination

- **`solana-tx`**: `data: string` present ⇒ `kind: 'solana-prebuilt'`; `from/to/amount` present ⇒ `kind: 'solana'`. Both or neither ⇒ throw. No mcp-ts wire-shape change required.
- **`cosmos-msg`**: `msg_type: 'send'` ⇒ `kind: 'cosmos'` (existing); `msg_type: 'deposit'` ⇒ `kind: 'thorchain-deposit'` (NEW variant for LP add/remove). Unknown msg_type ⇒ throw.
- **`utxo-psbt`**: branch on `chain === 'Zcash'` for fee handling. All other UTXO chains use `fee_rate`.

### Safety carve-outs

Stay co-located with their chain — NOT extracted to a shared module. Foreign-marker rejection is Solana-specific, evm-sequence cap + mixed-chain reject are EVM-specific, SPL malformed throw is Solana-specific. A `safetyChecks.ts` would become a junk drawer.

### ParsedServerTx union additions

Only one new variant needed: `thorchain-deposit` for `cosmos-msg + msg_type: 'deposit'` (LP add/remove). All other canonical encodings parse into existing variants. `dispatchParsedTx` only changes to add the `thorchain-deposit` case.

### Testing

- New file `txProposal.canonical.test.ts` — one happy-path fixture per encoding (8 cases) plus structural-error throws (mismatched chain/encoding, malformed solana-tx, unknown tx_encoding, ripple-tx placeholder).
- Existing 4 test files unchanged — cover legacy paths, act as regression net for the verbatim move into `legacy.ts`.
- On-device smoke pre-merge: SOL native send (closes v-uneu), SOL prebuilt swap, ETH send, BTC send (UTXO rep), THOR LP add (cosmos-msg + deposit rep).
- Tron/TON/Sui/Cardano: unit-fixture verified, not on-device. Pre-launch user volume on these is ~0 and wire shape is documented in mcp-ts emission sites. Diagnostic-log capture pattern (per v-uneu) handles any post-merge surprises.

### Tasks

1. **Scaffold folder + verbatim legacy move** — create `txProposal/`, move current `txProposal.ts` body into `legacy.ts` (no logic change), make `txProposal.ts` a re-export stub, extract `readString`/`readNumber`/`resolveChainId` to `shared.ts`. Existing 4 test files pass unchanged. Independent of all other tasks.
2. **Dispatcher + tx_encoding switch** — `txProposal/index.ts` exports new `parseServerTx`, branches on `tx_encoding`, falls through to `legacy.parseServerTx`. Stub each chain parser to throw `not implemented`. Wire unknown-encoding + mismatched-chain throws. Depends on 1.
3. **EVM canonical** — `evm.ts` handles `tx_encoding: 'evm'`. Reads `data.tx`, normalizes snake/camel gas fields, applies `MAX_EVM_SEQUENCE_LEN` cap if `transactions[]` present, mixed-chain reject. Depends on 2.
4. **Solana canonical** — `solana.ts` handles `tx_encoding: 'solana-tx'` with native/prebuilt sub-discrim. Foreign-marker reject retained as separate guard for legacy Solana payloads. Depends on 2. **Closes v-uneu.**
5. **UTXO canonical** — `utxo.ts` handles `tx_encoding: 'utxo-psbt'`, including Zcash fee carve-out. Depends on 2.
6. **Cosmos canonical + thorchain-deposit variant** — `cosmos.ts` handles `tx_encoding: 'cosmos-msg'`, sub-discrim on `msg_type`. Adds `thorchain-deposit` to `ParsedServerTx` union and `dispatchParsedTx` switch (only dispatch change). Depends on 2.
7. **Tron / TON / Sui / Cardano canonical** — four small files mapping onto existing `ParsedServerTx` shapes; no new variants. Depends on 2.
8. **Ripple placeholder** — `ripple.ts` throws 'not yet wired'. Trivial; bundles with task 7. Depends on 2.
9. **Canonical test file** — `txProposal.canonical.test.ts` with 8 happy-path + 4-5 structural-throw fixtures. Depends on 3–8.
10. **On-device smoke + PR** — run SOL native, SOL prebuilt, ETH send, BTC send, THOR LP. Capture receipts. Open PR tagged `[app/agent] parseServerTx canonical envelope dispatch`, body lists all 9 encodings, closes v-uneu.

Tasks 3–8 are independent once 2 lands: 1 → 2 → (3,4,5,6,7,8 in parallel) → 9 → 10.

**2026-04-30T05:44:57Z**

Task 1 complete: scaffolded txProposal/ folder, verbatim legacy move (1185 lines), shared.ts helpers extracted. All 1724 tests + lint + typecheck pass. Commit 21c99aae. Note: dispatchParsedTx lives in signAndBroadcast.ts, not txProposal — task 6 does not need to re-export it.

**2026-04-30T05:50:14Z**

Task 2 complete: dispatcher + 9 chain stubs + dispatch tests. Family validation in dispatcher via chainRegistry.kind. 1731 tests pass. Commit 1b6adc9e.

**2026-04-30T06:04:22Z**

Tasks 3-8 complete (parallel). Branch state: 1792 tests pass, lint+typecheck clean. Commits: 21c99aae (task1), 1b6adc9e (task2), 699aaf6e (task4 sol), 3475484e (task5 utxo), 0a78af26 (task7+8 - mistitled cosmos), c935a585 (task3 evm), 9a40316b (task6 cosmos). Note: 0a78af26 has cosmos title but contains tasks 7+8 content due to parallel git race; will note in PR body.

**2026-04-30T06:19:14Z**

PR opened: https://github.com/vultisig/vultiagent-app/pull/320 — closes v-uneu. Final state: 9 commits, 21 files, 1813 tests passing, lint+typecheck clean.
