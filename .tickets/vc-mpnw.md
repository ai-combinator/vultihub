---
id: vc-mpnw
status: closed
deps: []
links: []
created: 2026-03-22T09:10:56Z
type: task
priority: 1
assignee: Jibles
tags:
    - sdk
    - signing
    - agent
---
# SDK: Signing reliability and agent-friendliness improvements

## Context

During real-world testing of the vultiagent-cli (send/swap operations across Arbitrum, BSC, Solana, THORChain, Ethereum), we identified two categories of issues in `@vultisig/sdk` v0.5.0 that affect agent consumers:

1. **MPC keysign reliability** — first signing attempt frequently times out, second succeeds immediately
2. **Console logging** — SDK logs progress to stdout with emojis, polluting structured CLI output

## Issue 1: MPC Keysign First-Attempt Timeout

### Observed Behavior

The first `vault.sign()` call in a session has ~50% success rate. Retries after 15 seconds succeed far more reliably. Reproducible across all EVM chains and THORChain. Benchmark data from 2026-03-23 (8 signing attempts across Ethereum, BSC, THORChain):

| Attempt | Result | Retries | Time | Context |
|---------|--------|---------|------|---------|
| 1 | SUCCESS | 1 | 98s | ETH→RUNE, first test after code changes |
| 2 | SUCCESS | 0 | 62s | ETH→RUNE, first-attempt success |
| 3 | SUCCESS | 1 | 95s | ETH→RUNE |
| 4 | FAILED | 3 | 300s | ETH→RUNE, all retries exhausted |
| 5 | SUCCESS | 0 | 24s | ETH→RUNE, immediately after #4 (server warm) |
| 6 | FAILED | 3 | 296s | ETH→RUNE, server went cold |
| 7 | FAILED | 3 | 292s | ETH→RUNE, server still cold |
| 8 | SUCCESS | 1 | ~60s | BNB→RUNE, BSC chain |

**Success rate: 50%** (with 3 retries). First-attempt success rate: 25% (2/8). When server is warm from a prior attempt, signing succeeds in 24s with 0 retries.

### Root Cause Analysis (Deep Dive — 2026-03-23)

Detailed trace of the signing flow with exact file locations and TWO distinct timeout windows:

#### The Complete Signing Timeline

```
FastVault.sign()                                    [FastVault.ts:75-110]
├── ensureKeySharesLoaded()                         [FastVault.ts:78]       ~100-200ms (cached after first call)
├── resolvePassword()                               [FastVault.ts:82]       ~0ms (cached in PasswordCacheService)
└── FastSigningService.signWithServer()              [FastSigningService.ts:33-88]
    ├── getWalletCore()                             [FastSigningService.ts:64]  ~0-2000ms (WASM, memoized)
    └── ServerManager.coordinateFastSigning()        [ServerManager.ts:105-279]
        ├── Generate session ID + encryption key     [ServerManager.ts:124-142]  ~1ms
        ├── POST /vault/sign                         [ServerManager.ts:146-155]  ~100-1000ms (network)
        │   └── Server begins: Redis task pickup → vault decrypt → CGo init
        ├── POST /{sessionId} (join relay)           [ServerManager.ts:168-172]  ~50-300ms
        ├── Register server participant              [ServerManager.ts:175-185]  ~50-200ms
        │
        ├── ⏱️ TIMEOUT WINDOW 1: waitForPeers()     [ServerManager.ts:486-533]
        │   ├── maxWaitTime = 30,000ms (HARDCODED)   [ServerManager.ts:492]
        │   ├── checkInterval = 2,000ms              [ServerManager.ts:493]
        │   ├── GET /{sessionId} (poll for peers)    [ServerManager.ts:503]
        │   └── Throws "Timeout waiting for peers"   [ServerManager.ts:532]
        │       if server hasn't joined relay in 30s
        │
        ├── POST /start/{sessionId}                  [ServerManager.ts:213-217]
        │
        └── for (msg of messages) {                  [ServerManager.ts:236-254]  ← SEQUENTIAL
              keysign()                              [core/mpc/keysign/index.ts:42-223]
              ├── initializeMpcLib(algorithm)         [index.ts:54]             ~0-800ms (memoized)
              ├── ensureSetupMessage()               [index.ts:58-68]
              │   └── 10 attempts × 1000ms delay     [setup/get.ts:26-30]      up to 10s
              ├── createSignSession()                [index.ts:70-75]           ~1-10ms
              │
              ├── ⏱️ TIMEOUT WINDOW 2: keysign       [index.ts:40]
              │   ├── KEYSIGN_TIMEOUT = 60,000ms     (HARDCODED)
              │   ├── Outbound: WASM→relay (100ms)   [index.ts:88-108]
              │   ├── Inbound: relay→WASM (polling)  [index.ts:111-166]
              │   └── Throws "Exited inbound processing due to a timeout"
              │
              └── extract signature                  [index.ts:168-175]
            }
```

#### Two Distinct Timeout Windows

**Window 1: `waitForPeers` — 30 seconds** (`ServerManager.ts:492`)
- Begins after SDK joins relay and calls POST /vault/sign
- Polls relay every 2 seconds for server's party ID
- Server must: pick up Redis task → decrypt vault → initialize CGo → create signing session → join relay
- On cold start, server regularly takes >30s → timeout

**Window 2: `keysign` inbound loop — 60 seconds** (`core/mpc/keysign/index.ts:40`)
- Begins after MPC session starts
- Alternating outbound (WASM→relay, 100ms intervals) and inbound (relay→WASM, tight loop)
- If server is slow processing MPC rounds, hits 1-minute abort
- The error message "Exited inbound processing due to a timeout after 1 min" comes from here

**Why retry succeeds:** After a failed first attempt, the server has: vault decrypted in memory, CGo lib loaded, keyshare deserialized, HTTP/TLS connections pooled. Second attempt's `waitForPeers` returns in <2s, and keysign completes in 5-15s.

#### Latency Budget Analysis

| Phase | Best | Typical | Worst | % of Total |
|-------|------|---------|-------|-----------|
| KeyShare + password | 0ms | 100ms | 200ms | <1% |
| WASM init | 0ms | 500ms | 2000ms | 3% |
| HTTP setup (3 calls) | 200ms | 500ms | 1500ms | 3% |
| **waitForPeers** | **100ms** | **5,000ms** | **30,000ms** | **30-50%** |
| Setup message | 100ms | 500ms | 10,000ms | 5% |
| **MPC rounds** | **3,000ms** | **10,000ms** | **60,000ms** | **50-70%** |

**The relay communication and MPC message exchange account for 80-95% of total signing time.** WASM init and HTTP setup are negligible.

### Recommended Fixes (ordered by impact)

**A. Increase `waitForPeers` timeout to 2-3 minutes** (HIGHEST IMPACT, easy)
File: `sdk/src/server/ServerManager.ts:492`
```typescript
const maxWaitTime = 30000  // ← change to 120000 or 180000
```
The server's own timeout is 3 minutes. The SDK's 30s is the primary cause of first-attempt failures. Increasing to 2-3 minutes would likely bring success rate from 50% to 90%+ based on our data (retries succeed within 15-45s of first timeout).

**B. Expose timeout configuration on `sign()`** (HIGH IMPACT, easy)
```typescript
vault.sign({
  transaction: keysignPayload,
  chain: 'Ethereum',
  messageHashes,
  timeout: 180_000,  // ← new option
})
```
Pass through to both `waitForPeers` and keysign's `KEYSIGN_TIMEOUT`. This lets consumers tune without SDK releases.

**C. Increase keysign inbound timeout** (HIGH IMPACT, easy)
File: `core/mpc/keysign/index.ts:40`
```typescript
const KEYSIGN_TIMEOUT = 60_000  // ← change to 120000 or 180000
```
When MPC rounds are slow (large payloads, network latency), 1 minute isn't always enough. The server allows 3 minutes — the SDK should match.

**D. Pre-warm WASM during `importVault()`** (MEDIUM IMPACT, easy)
Call `initializeMpcLib('ecdsa')` and `initializeMpcLib('eddsa')` during vault import instead of lazily in `keysign()`. Saves 100-800ms on first sign, but this is <5% of total time so lower priority.

**E. Reduce `waitForPeers` polling interval** (MEDIUM IMPACT, easy)
File: `sdk/src/server/ServerManager.ts:493`
```typescript
const checkInterval = 2000  // ← change to 500
```
Currently polls every 2s. If the server joins at 2.1s, SDK doesn't notice until 4s. Reducing to 500ms saves up to 1.5s per sign.

**F. Parallel message signing** (MEDIUM IMPACT, medium effort)
File: `sdk/src/server/ServerManager.ts:236-254`
Currently signs multiple messages sequentially in a `for` loop. For approval+swap (2 messages), this doubles signing time. Could use `Promise.all()` if the server supports concurrent MPC sessions within a single relay session.

**G. Don't start keysign timer until setup message confirmed** (MEDIUM IMPACT, medium effort)
Currently the 1-minute keysign timer starts immediately. The setup message exchange can take up to 10s, eating into the signing window. Start the timer after both parties have exchanged setup messages.

**H. Reduce `waitForSetupMessage` polling interval** (LOW IMPACT, easy)
File: `core/mpc/message/setup/get.ts:26-30`
Currently: 10 attempts × 1000ms. Reducing to 200ms intervals saves up to 8s when the peer's setup message arrives between polls.

**I. Emit timing metrics via events** (LOW IMPACT, medium effort)
Add `signingTiming` event with per-stage durations. Enables consumers to identify bottlenecks and the SDK team to collect telemetry.

### CLI Workaround (Current State — 2026-03-23)

The vultiagent-cli works around the timeout issues with:
1. **`signWithRetry`** — 3 retries with 15s delay on MPC timeout errors
2. **Full cycle retry** — retries the entire `prepare → sign → broadcast` flow (not just sign), so each attempt gets a fresh nonce and quote
3. **Pre-warm server connections** — calls `sdk.getServerStatus()` during vault setup to establish TCP/TLS ahead of signing
4. **Pre-unlock vault** — calls `vault.unlock(password)` during setup so keyshares and password are cached before `sign()`
5. **Parallel SDK init** — WASM initialization runs concurrently with file I/O and password retrieval

These bring success rate from ~33% to ~50%, but cannot overcome the 30s `waitForPeers` timeout — only SDK changes can fix that.

### ERC-20 Approve + Swap Sequential Signing

When `keysignPayload.erc20ApprovePayload` is present, `extractMessageHashes` → `getEvmSigningInputs` produces 2 message hashes. These are signed **sequentially** in `coordinateFastSigning` (`ServerManager.ts:236-254`):

```typescript
for (const msg of messages) {
  const sig = await keysign({...})  // each message: 5-60s
  signatureResults[msg] = sig
}
```

Each message gets its own full keysign cycle (setup message exchange + MPC rounds). For 2 messages, total signing time is 10-120s. The server session can go stale between messages.

**Fix options:**
1. Sign messages in parallel (`Promise.all`) if server supports it
2. Return approval as a separate `KeysignPayload` so consumers can sign/broadcast sequentially with separate sessions
3. Strip `erc20ApprovePayload` from the keysign payload when `autoApprove: false` so only 1 message hash is generated

## Issue 2: Console Logging Pollutes Agent Output

### Observed Behavior

During signing, the SDK logs directly to stdout:
```
🔑 Generated signing party ID: sdk-6935
📡 Calling FastVault API with session ID: b351d96e-...
✅ Server acknowledged session: b351d96e-...
⏳ Waiting for server to join session...
✅ All participants ready: [sdk-6935, Server-253]
📡 Starting MPC session with devices list...
✅ MPC session started
🔐 Starting MPC keysign process...
🔏 Signing message: c18fd349...
✅ Signature obtained for message
🔄 Formatting signature results...
```

These lines are hardcoded `console.log` calls in the SDK bundle. They mix with structured CLI output (JSON/table), making it difficult for agents to parse responses cleanly.

### Problem for Agents

- Pollutes stdout when CLI uses `--output json` — agents get emoji text interleaved with JSON
- Adds ~10 lines of noise to agent context windows per signing operation
- Contains session IDs and message hashes that are useless to consumers
- No way to suppress without patching the SDK

### Recommended Fix

The SDK already has a proper `signingProgress` event system (`vault.on('signingProgress', ...)`) with structured `SigningStep` objects. The console logging should be removed or gated behind a debug/verbose flag:

**Option A (preferred):** Remove all `console.log` calls from the signing path. Consumers who want progress can subscribe to `signingProgress` events.

**Option B:** Add a `logLevel` or `verbose` option to the `Vultisig` constructor:
```typescript
const sdk = new Vultisig({ logLevel: 'silent' })  // suppress console output
```

**Option C:** Route signing logs through `console.debug` instead of `console.log`, so they're hidden by default and only visible when `NODE_DEBUG` or similar is enabled.

## CLI Workaround (Current State)

The vultiagent-cli currently works around Issue 1 with `signWithRetry()` — 1 retry after 15s delay on timeout errors. This is safe because the timeout occurs during MPC keysign (before broadcast), so no transaction is signed or sent on the failed attempt.

There is no workaround for Issue 2 without patching the SDK.

## Issue 3: `balancesWithPrices()` Throws for Non-EVM Token Pricing

### Observed Behavior

`vault.balancesWithPrices(chains, includeTokens, 'usd')` throws `"Token pricing not supported for Solana (non-EVM chain)"` when the vault has discovered SPL tokens (e.g., PUMP on Solana). The error is non-deterministic — sometimes it throws, sometimes it silently drops the unpriceable tokens and returns the rest.

### Impact

CLI's `vasig balance --include-tokens --fiat` crashes intermittently. The CLI cannot reliably show fiat values for all assets when tokens are included.

### Expected Behavior

Should never throw for unsupported token pricing. Instead, return the token with `fiatValue: null` or `fiatValue: 0` and let the consumer decide how to handle it.

## Issue 4: Swap Quote Providers Return Near-Zero Output Amounts

### Observed Behavior

Several swap routes via thorchain/mayachain providers return estimates that are essentially zero (10^-12 or less) but the SDK returns them as valid quotes:

- `Ethereum → Arbitrum` (0.01 ETH): estimatedOutput = `0.00000000000098376 ETH` ($0.00) via mayachain
- `Ethereum → BSC` (0.01 ETH): estimatedOutput = `0.000000000003242353 BNB` ($0.00) via thorchain
- `Bitcoin → Ethereum` (0.00001 BTC): estimatedOutput = `0.000000000000020819 ETH` ($0.00) via thorchain

The same routes work correctly via li.fi (returning reasonable amounts).

### Impact

If an agent executes a swap based on these quotes, it loses essentially all funds. The CLI returns `ok: true` for these quotes because the SDK doesn't validate output reasonableness.

### Expected Behavior

The SDK should either:
1. Try multiple providers and return the best quote
2. Reject quotes where output value is near-zero relative to input value
3. Flag the quote with a warning when output/input ratio is extremely unfavorable

## Acceptance Criteria

### Signing Reliability (P0)
- [ ] `waitForPeers` timeout increased from 30s to ≥120s (`ServerManager.ts:492`) — single highest-impact fix
- [ ] `KEYSIGN_TIMEOUT` increased from 60s to ≥120s (`core/mpc/keysign/index.ts:40`)
- [ ] Timeouts configurable via `vault.sign({ timeout })` option
- [ ] `waitForPeers` polling interval reduced from 2000ms to 500ms (`ServerManager.ts:493`)
- [ ] First `vault.sign()` call in a fresh session succeeds ≥80% of the time (currently ~25%)
- [ ] Overall MPC success rate ≥90% within a single attempt (currently ~50% with 3 retries)

### WASM & Init (P1)
- [ ] WASM modules pre-initialized during `importVault()`, not lazily in `keysign()`
- [ ] `initializeMpcLib('ecdsa')` and `initializeMpcLib('eddsa')` called during vault setup

### Console Logging (P1)
- [ ] Console logging during signing is suppressible (via option, log level, or removal)
- [ ] `signingProgress` events are the primary progress mechanism, not console output

### ERC-20 Approval Flow (P0)
- [ ] `broadcastTx` broadcasts ALL signing inputs when keysign payload produces multiple (approval + swap) — `BroadcastService.ts:78-98` currently only uses `txInputsArray[0]`
- [ ] `prepareSwapTx` returns a real `approvalPayload` KeysignPayload when ERC-20 approval is needed — currently always returns `undefined` (`SwapService.ts:157-186`)
- [ ] Sequential message signing in `coordinateFastSigning` doesn't cause stale server sessions

### Broadcast Reliability (P0)
- [ ] EVM broadcast does not silently swallow errors that result in the tx never landing on-chain — `core/chain/tx/broadcast/resolvers/evm.ts:23-37` swallows 7 error strings including "nonce too low"
- [ ] `broadcastTx` return value reflects actual broadcast outcome, not just a locally-computed hash via `keccak256(encoded)` (`core/chain/tx/hash/resolvers/evm.ts:6-7`)

### Quote & Balance (P2)
- [ ] Swap quote provider selection rejects near-zero output amounts, or tries alternative providers
- [ ] `balancesWithPrices()` gracefully handles non-EVM token pricing (returns null/0 instead of throwing)
- [ ] `getSupportedSwapChains()` returns deduplicated chain list

### Solana (P2)
- [ ] Solana outbound signing works reliably (currently 0% success rate — large tx payloads cause consistent MPC timeouts)
- [ ] Solana blockhash refresh on retry — expired blockhash causes broadcast failure even after successful retry signing

### Developer Experience (P2)
- [ ] `waitForSetupMessage` polling interval reduced from 1000ms to 200ms (`core/mpc/message/setup/get.ts:26-30`)
- [ ] Timing metrics emitted via `signingTiming` event for per-stage latency visibility

## Notes

**2026-03-22T11:25:09Z**

## SDK Opportunities from CLI Validation Work (v-kdfq)

The CLI (vasig) is working around several things the SDK should own:

- **`resolveChain()`** — case-insensitive chain name resolution with "did you mean?" suggestions. Currently implemented in vasig's lib/validation.ts, but every SDK consumer needs this. Should live in the SDK alongside SUPPORTED_CHAINS.
- **Chain metadata** — vasig hardcodes an EVM_CHAINS list to reject memos on EVM chains. SUPPORTED_CHAINS should export chain metadata (isEVM, supportsMemo, nativeToken, etc.) so consumers don't maintain their own lists.
- **Clean error messages** — vasig's classifyError() cleans up viem/LI.FI error dumps into human-friendly messages. The SDK should throw clean errors at the source instead of leaking library internals.

## Issue 5: BroadcastService Only Broadcasts First Signing Input (ERC-20 Swap Breakage)

**2026-03-23** — Deep investigation of why ERC-20 token swaps silently fail.

### Root Cause Chain

Three SDK issues combine to silently lose funds on ERC-20 swaps:

**A. `BroadcastService.broadcastTx` only uses `signingInputs[0]`**

File: `sdk/src/vault/services/BroadcastService.ts:78-98`

When `keysignPayload.erc20ApprovePayload` is present, `getEvmSigningInputs` (in `core/mpc/keysign/signingInputs/resolvers/evm/index.ts:29-52`) recursively generates `[approvalInput, swapInput]` with incremented nonces. `extractMessageHashes` returns hashes for BOTH, and both are signed via MPC. But `broadcastTx` does:

```typescript
const txInputData = txInputsArray[0]  // <-- only first input
```

Comment in source: "For now, handle the common case (single input)". The swap tx is signed but never broadcast.

**B. `prepareSwapTx` never returns `approvalPayload`**

File: `sdk/src/vault/services/SwapService.ts:157-186`

Despite the `SwapPrepareResult` type declaring `approvalPayload?: KeysignPayload`, the field is always `undefined` regardless of `autoApprove` setting. The approval data only exists as `keysignPayload.erc20ApprovePayload` (a proto field with `amount` + `spender`), not as a separate `KeysignPayload`. CLI consumers that check `if (approvalPayload)` will never enter the approval branch.

**C. EVM broadcast silently swallows errors**

File: `core/chain/tx/broadcast/resolvers/evm.ts:23-37`

Seven error strings are caught but not thrown: "already known", "transaction is temporarily banned", "nonce too low", "transaction already exists", "future transaction tries to replace pending", "could not replace existing tx", "tx already in mempool". Combined with txHash being computed locally via `keccak256(encoded)` (not from RPC response), this means a failed broadcast returns a valid-looking hash.

### Impact

For any ERC-20 swap requiring approval (e.g., USDC→ETH):
1. SDK builds approval + swap as two signing inputs in one keysign payload
2. MPC signs both (2 message hashes, 2 rounds of signing)
3. Only the approval tx is compiled and broadcast
4. Broadcast may silently fail (error swallowed) but returns a locally-computed txHash
5. CLI/MCP reports success with a txHash that doesn't exist on-chain
6. Swap never executes, user's funds appear to vanish

### Recommended Fixes

1. **`broadcastTx` must broadcast ALL signing inputs** — iterate `txInputsArray`, compile and broadcast each, return all hashes
2. **`prepareSwapTx` should return a real `approvalPayload`** when approval is needed — build a full `KeysignPayload` for the approval, separate from the swap payload, matching the documented `SwapPrepareResult` type
3. **EVM broadcast should not swallow errors silently** when the tx was never seen on-chain — at minimum, verify the tx exists after broadcast before returning success
4. **`broadcastTx` return type should indicate success/failure** — returning a locally-computed hash is misleading; consider returning `{ txHash, confirmed: boolean }` or throwing on broadcast failure

### CLI Workaround (Current State)

The CLI now:
- Detects `keysignPayload.erc20ApprovePayload` and fails fast with a clear error asking the user to approve via the Vultisig app first
- Adds post-broadcast on-chain verification for EVM chains — polls the RPC to confirm the tx landed before reporting success
- Native token swaps and pre-approved ERC-20 swaps continue to work normally

## SDK Bug + Server Intermittency (2026-03-24)

### Summary

Two issues contribute to MPC signing failures:

1. **SDK bug (fixable):** `processInbound()` has zero delay between relay polls, creating a tight HTTP request loop
2. **Server intermittency (not fixable from SDK):** The server frequently fails to participate in MPC rounds even when the relay connection is healthy — likely caused by the 2-minute Asynq task timeout or server-side keysign errors

The SDK fix is necessary (turns an impossible situation into a working one when the server cooperates) but not sufficient (the server goes through periods where it doesn't participate regardless of SDK behavior).

**Extended testing results:**
- Window 1 (working): 3/3 sends, 3/5 swaps succeeded — MPC completed in 1.3-1.6s
- Window 2 (broken): 0/5 sends, 0/7 swaps — 0 inbound messages over 1000+ polls per attempt, even after 10-min cooldown
- Pattern: Server joins relay instantly (<0.5s), setup message exchange succeeds, but server never sends MPC round messages

The server appears to fail silently between setup message exchange and MPC round processing. Without server logs, we can't determine whether this is: Asynq killing the task, a `SignSessionFromSetup` error, or the server's own inbound timeout expiring before it receives the SDK's first outbound message.

## SDK Fix: Zero-delay inbound polling

### The Bug

`packages/core/mpc/keysign/index.ts` — the `processInbound()` function recursively calls itself with **zero delay** between relay polls. This creates a tight loop making hundreds of HTTP GET requests per second to the relay, overwhelming it and preventing MPC messages from being delivered.

```typescript
// BEFORE (broken): no delay between polls
const processInbound = async (): Promise<void> => {
  const relayMessages = await getMpcRelayMessages({...})
  for (const msg of relayMessages) { ... }
  return processInbound()  // ← immediate recursion, no sleep
}
```

```typescript
// AFTER (fixed): 100ms between polls
const processInbound = async (): Promise<void> => {
  const relayMessages = await getMpcRelayMessages({...})
  for (const msg of relayMessages) { ... }
  await sleep(100)          // ← rate-limit relay polling
  return processInbound()
}
```

For comparison, the Go server (`vultiserver/service/keysign_dkls.go:388`) polls every 100ms with `time.Sleep(100 * time.Millisecond)`.

### The Fix

**One line:** Add `await sleep(100)` before `return processInbound()` in `packages/core/mpc/keysign/index.ts:165`.

### Evidence

**With instrumented SDK, zero-delay polling (all tests failed):**
```
[mpc] +0.1s outbound #1 → 1 receiver(s)
... 3 minutes of empty polls, 0 inbound messages received ...
ERROR: "Exited inbound processing due to a timeout after 3 min"
```

**With 100ms polling delay (sends: 4/5 success, swaps: 3/5 success):**
```
[mpc] +0.1s outbound #1 → 1 receiver(s)
[mpc] +0.1s inbound poll #1 (received 0 msgs so far)
[mpc] +0.9s inbound #1 (poll #5, done=false) → outbound #2
[mpc] +1.0s inbound #2 → outbound #3
[mpc] +1.2s inbound #3 → outbound #4
[mpc] +1.5s inbound #4 (done=true)
[mpc] +1.5s MPC complete! out=4 in=4 polls=7
```

MPC rounds complete in **1.3-1.6 seconds** with the fix. Without it, they never complete.

### Other Instrumentation Findings

Timing instrumentation in `ServerManager.ts` disproved several earlier hypotheses:
- **No cold-start delay**: Server joins relay in 0.5s (POST /vault/sign: 0.3s, waitForPeers: 0.1s)
- **Not swap-specific**: With zero-delay polling, both sends AND swaps fail identically
- **Not a timeout issue**: The MPC protocol only needs 4 messages in each direction and completes in <2s when messages flow
- **Setup message exchange works fine**: Uses a different relay endpoint (`/setup-message/`) which isn't affected by the polling storm

### Updated Acceptance Criteria

Based on these findings, the priority ordering changes:

**P0 — Ship immediately:**
- [x] Add 100ms sleep to `processInbound()` loop (`core/mpc/keysign/index.ts`)
- [ ] Remove console.log calls from `coordinateFastSigning` (`ServerManager.ts`)

**P1 — Still valuable:**
- [ ] Increase `maxInboundWaitTime` from 1 min to 3 min (safety net for slow networks)
- [ ] Increase `waitForPeers` timeout from 30s to 120s (safety net, server usually joins in <1s)
- [ ] `BroadcastService` broadcasts all signing inputs, not just first
- [ ] EVM broadcast stops swallowing "nonce too low" errors

**P2 — Server-side:**
- [ ] VultiServer Asynq keysign timeout should increase from 2 min to 5 min (currently shorter than its own WaitForSessionStart of 3m3s)

**Deprioritized:**
- ~~Polling interval reductions~~ — The 100ms sleep IS the fix; further tuning is marginal
- ~~WASM pre-warming~~ — WASM init takes 0ms (cached), not a factor
- ~~Pre-warming server connections~~ — Server joins in 0.5s, no warm-up needed

## Previous Testing Notes (2026-03-24)

### Method

Linked CLI to local SDK via `file:` protocol (`"@vultisig/sdk": "file:../vultisig-sdk/packages/sdk"`), applied fixes A-E, rebuilt with `yarn build:fast`, and ran real transactions.

### Fixes Applied & Verified

| Fix | Change | Verified? |
|-----|--------|-----------|
| A: waitForPeers timeout | 30s → 120s, poll 2s → 500ms (`ServerManager.ts:492-493`) | Yes — no "Timeout waiting for peers" errors |
| B: keysign inbound timeout | 1min → 3min (`core/mpc/keysign/index.ts:40`) | Yes — error now says "timeout after 3 min" |
| C: console.log removal | Deleted 11 console.log calls from `coordinateFastSigning` | Yes — clean JSON output, no emoji spam |
| D: broadcast all inputs | Loop `txInputsArray` instead of `[0]` (`BroadcastService.ts`) | Compiled, not testable (no ERC-20 swaps completed) |
| E: EVM error swallowing | Reduced from 7 to 3 benign strings (`evm.ts`) | Compiled, not directly testable |

### Test Results — Sends

| # | Chain | Result | First-attempt? | Time | TxHash |
|---|-------|--------|---------------|------|--------|
| S0 | THORChain | SUCCESS | Yes | ~30s | `9A1F96...` |
| S1 | THORChain | SUCCESS | Yes | ~25s | `F82B62...` |
| S2 | THORChain | SUCCESS | Yes | ~20s | `CA65BE...` |
| S3 | THORChain | SUCCESS | No (1 retry) | ~3.5min | `0CAC25...` |
| S4 | Arbitrum | SUCCESS | No (1 retry) | ~3.5min | `0xaf7ef5...` |
| S5 | BSC | SUCCESS | No (1 retry) | ~3.5min | `0xddefc2...` |

**Send success rate: 6/6 (100%).** First-attempt rate: 3/6 (50%) — cold starts still need 1 retry. All warm-server sends succeed on first attempt.

### Test Results — Swaps

| # | Route | Result | Notes |
|---|-------|--------|-------|
| W0 | RUNE → ARB:USDC | SUCCESS (1 retry) | First swap of session, succeeded on retry |
| W1-W8 | Various → ARB:USDC, ETH, RUNE | ALL FAILED | Keysign MPC rounds >3min every time |

**Swap success rate: 1/9 (11%).** Even with 5-minute keysign timeout, swap MPC rounds did not complete. Sends complete in <1 min when warm; swaps never complete within 5 min. This suggests a **server-side issue specific to swap keysign payloads** — the server may be slower to process/validate swap payloads before entering MPC rounds.

### New Findings

1. **Swap MPC rounds are fundamentally slower than send MPC rounds.** The same server that processes a send in 20s consistently fails to complete a swap keysign in 5 minutes. The MPC computation should be identical (signing a 32-byte hash), so the bottleneck is server-side swap payload validation.

2. **Password/vault cache expires after ~5 minutes.** With 3-min keysign timeout, the retry (starting at ~3.2min) has limited time before the cache expires. With 5-min timeout, the retry always hits cache expiry. The SDK needs either longer cache TTLs or a re-authentication callback for long-running operations.

3. **THORChain native swap quotes return near-zero output** (Issue 4 confirmed). Only mayachain provider returns reasonable quotes for RUNE swaps. The SDK should try multiple providers.

### Conclusion

The timeout fixes (A, B) and polling improvement are **necessary but not sufficient** for swap reliability. Sends are dramatically improved (100% success rate with retries, vs ~50% before). Swaps require server-side investigation — the MPC rounds for swap payloads take orders of magnitude longer than for send payloads, suggesting a server processing bottleneck.

**Recommended next steps:**
1. Ship timeout fixes A+B — proven to help sends, will help swaps once server-side issue is resolved
2. Ship console.log removal (C) — proven clean output
3. Ship broadcast fix (D) and error fix (E) — correct code, untestable until swaps work
4. Investigate server-side swap payload processing time — this is the real blocker for swap reliability
5. Address password cache TTL — currently limits effective retry window to <5 min

**2026-04-22T01:12:41Z**

Closing per user direction (partially done). DONE: emoji/progress console.logs no longer pollute prod output (only present in test files); linear retry exists in packages/core/mpc/keygenComplete.ts (10 attempts, 1s delay). REMAINING: first-attempt keysign timeout root cause not addressed; no MPC relay pre-warm mechanism; no exponential backoff. Reopen or spin a fresh ticket if the ~50% first-attempt failure rate is still observed.
