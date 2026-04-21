# v-pxuw — Pre-Emulator Test Plan

Handoff doc for a fresh agent. Ticket: `.tickets/v-pxuw.md`. The four PRs
to exercise:

- vultisig-sdk #284 — `refactor/v-pxuw-tool-consolidation`
- mcp-ts #30 — `refactor/v-pxuw-tool-consolidation`
- agent-backend #128 — `refactor/v-pxuw-tool-consolidation`
- vultiagent-app #164 — `refactor/v-pxuw-tool-consolidation`

All four branches were rebased onto current `main` and force-pushed. The
app PR is the only one that needs the emulator to meaningfully verify
end-to-end; the other three can be exercised without it, and this plan
intentionally defers the emulator to the end.

---

## 0. What the branch does — one paragraph

The `execute_*` surface in mcp-ts (send, swap, contract_call, lp_add,
lp_remove) replaces ~15 per-chain `build_*` tools and the multi-step
swap/send LLM recipes. Backend (agent-backend) consolidates CRUD tools
to one-name-per-domain (`schedule_create`, `vault_coin`, `address_book`,
`credits`, etc.), rewrites the system prompt around a compact routing
table, and fans out balance lookups to mcp-ts for chains not pre-fetched
in SSE context. SDK gains `fiatToAmount` + `normalizeChain` utilities
plus a vault-free prep surface (`prepareSendTxFromKeys` etc.) so mcp-ts
can call SDK logic without holding a vault instance. Goal: compress the
LLM's visible tool surface from ~136 to ~30 always-on to improve
selection accuracy on weak models.

**"Testing worked" means:** (1) the five `execute_*` tools behave as a
drop-in for the old recipes including the C1–C4 rejection paths, (2)
the renamed CRUD tools still route to the right handlers, (3) the
hand-merged code paths (documented below) do what they should under
both launchsurface flag states, (4) historical conversations containing
old tool-part types still render, (5) signing correctness is unaffected.

---

## 1. Hand-resolved merges — target these first

Three rebase conflicts were resolved by judgment, not mechanically.
They need direct test coverage because they're where regressions are
most likely to hide.

### 1a. `execCredits` merge (agent-backend)

**File:** `internal/service/agent/executor.go` (around line 2538 post-rebase)

**What happened:** main added a launchsurface gate
(`if !launchsurface.CreditsEnabled(ctx) { return unlimited_sentinel }`)
to a function called `execCheckCredits(ctx, req)`. v-pxuw renamed the
function to `execCredits(ctx, input, req)` with a `detail` param for
the consolidated `credits(detail?: bool)` tool. We kept v-pxuw's
signature and detail branch, and kept main's launchsurface gate as the
first check inside the function. Test-side we renamed
`TestExecCheckCredits_*` → `TestExecCredits_*` and changed the call
site from `execCheckCredits(ctx, req)` to `execCredits(ctx, nil, req)`.

**Paths to hit:**
1. Flag off, `detail=false` → unlimited sentinel.
2. Flag off, `detail=true` → unlimited sentinel (unchanged; flag wins).
3. Flag on, `detail=false`, creditRepo=nil → error JSON.
4. Flag on, `detail=true`, creditRepo=nil → error JSON.
5. Flag on, `detail=false`, real creditRepo → summary balance only.
6. Flag on, `detail=true`, real creditRepo → summary + `spending_history`.

Existing `credits_flag_test.go` covers paths 1 and 3. **Add cases for
paths 4 and 6 at minimum** — the `detail=true` branch has no coverage
today and it's where our merge risks diverging from intent.

### 1b. Prompt templating dropped (agent-backend)

**File:** `internal/service/agent/prompt.go` (took v-pxuw's version entirely)

**What happened:** main's PR #123 (`launch-surface gating via PostHog
flag`) added four template tokens to the system prompt: `{{CHAINS}}`,
`{{SWAP_PROVIDERS}}`, `{{SWAP_PROVIDER_EXAMPLES}}`,
`{{SWAP_PROVIDER_EXAMPLE}}`, plus helper functions `chainsForPrompt`,
`swapProvidersForPrompt`, etc., plus `prompt_launchsurface_test.go`
asserting the substitution works. v-pxuw deleted the verbose send/swap
recipes these tokens lived in, in favor of a compact `## Tool Routing`
table. We kept v-pxuw's entire `prompt.go` (helpers + tokens gone) and
**deleted `prompt_launchsurface_test.go`** because it tested a feature
we intentionally removed. Also renamed a call site in `agent.go`:
`BuildStablePromptPrefixWithFlags(plugins, ctx)` →
`BuildStablePromptPrefix(plugins)` (the `...WithFlags` wrapper no
longer exists).

**Rationale:** launchsurface's goal (LLM can't call flagged-off
capabilities) is already enforced by two server-side layers —
`enforceLaunchFlags` in `executor.go` rejects calls targeting gated
chains at dispatch time, and `tool_filter.go` drops flagged-off tools
from the list the LLM ever sees. Prompt-side templating was
belt-and-suspenders at the wrong layer.

**Paths to hit:**
1. `POSTHOG_ENVIRONMENT=dev` (short-circuits to EverythingOn) → full
   tool set visible in MCP `tools/list`.
2. `POSTHOG_ENVIRONMENT=prod` with `AgentChainTron=false` → user asks
   "send 1 TRX"; LLM picks `execute_send`, backend rejects via
   `enforceLaunchFlags` with a readable error; LLM surfaces "Tron
   isn't enabled" to user. **One extra round-trip vs. pre-rebase, but
   correct end state.**
3. `POSTHOG_ENVIRONMENT=prod` with `AgentSwapKyber=false` →
   `execute_swap` result never attributes output to Kyber (because
   Kyber is filtered server-side). Prompt never mentioned providers
   anyway.

### 1c. `StripContextByFlags` × `resolveTools` collision (agent-backend)

**File:** `internal/service/agent/agent.go` (lines ~542 and ~1066 before rebase)

**What happened:** main added two lines at both the non-stream and
stream conversation paths:
```go
fullCtx = StripContextByFlags(ctx, fullCtx)
resolvedTools := s.resolveTools(ctx, convID, req.Tools)
```
v-pxuw removed `SendMessageRequest.Tools` entirely (client no longer
plumbs tools through the request). We **kept** `StripContextByFlags`
and **dropped** the `resolveTools` line; the field and the function
it called no longer exist in v-pxuw's tree.

**Paths to hit:**
1. Context stripping: with flags off for a chain, request a portfolio
   and assert that chain's balance/addresses don't appear in the
   assembled context.
2. No request.Tools: POST a SendMessage with no `tools` field and
   confirm the conversation proceeds (it should — the field was
   removed everywhere).

---

## 2. Environment — already set up

### Built / installed

| Thing | State |
|---|---|
| SDK shared + bundles | Built (`yarn build:shared && yarn build:sdk`) |
| CLI | Built + globally linked (`vsig` / `vultisig` v0.15.2 on PATH) |
| mcp-ts SDK link | pnpm override → `file:../vultisig-sdk/packages/sdk` (picks up today's build) |
| mcp-ts deps | installed |
| agent-backend `.env` | all required vars set (JWT_SECRET, AUTH_CACHE_KEY_SECRET, PROTOCOL_STATE_ENCRYPTION_KEY, AI_API_KEY via OpenRouter, `MCP_SERVER_URL=http://localhost:9091`, `DEV_SKIP_AUTH=true`, `POSTHOG_ENVIRONMENT=dev`) |
| agent-backend build | `go build ./...` + `go vet ./...` clean post-rebase |
| mcp-ts `.env` | port 9091, upstream Nansen/Etherscan keys empty (optional) |
| Postgres | `vulti-postgres` container up |
| Redis | `vulti-redis` container up |
| DB migrations | `goose: no migrations to run. current version: 20260410000001` |
| Test vault | `~/Downloads/Test Vault-36f2-share1of2.vult` (fast vault share 1 of 2) |
| Vault password | `~/.secrets` exports `VAULT_PASSWORD` + `VAULT_PASSWORDS="Test Vault-36f2:..."`, sourced from `~/.zshrc` |

### Services that need starting

The backend + MCP server aren't running yet. Start them in two terminals:

```bash
# Terminal 1 — mcp-ts (must start first; backend connects at boot)
cd /home/sean/Repos/vultisig/mcp-ts && pnpm dev:http
# listens on http://localhost:9091

# Terminal 2 — agent-backend
cd /home/sean/Repos/vultisig/agent-backend && go run ./cmd/server
# listens on http://localhost:8084
```

### CLI ready to attach to the test vault

```bash
source ~/.secrets   # if shell was opened before ~/.secrets was edited
vsig import "/home/sean/Downloads/Test Vault-36f2-share1of2.vult"
# password auto-resolved from VAULT_PASSWORDS

vsig balance ethereum    # sanity: reads from chain directly, no backend needed
```

---

## 3. Out of scope — don't chase these

- **Emulator / iOS / Android** — vultiagent-app (#164) changes are
  deferred. Stepper UI transitions, card rendering, historical
  conversation UI replay, device MPC signing round-trip are all
  emulator-only.
- **Full secure-vault keygen** — we only have a fast-vault share
  locally; don't try to run `create secure` or secure signing.
- **Broadcasting real transactions** — prep-only smoke is fine. If you
  need to exercise signing, use a testnet or fork, not mainnet.
- **Plugin verifier flows** — `VERIFIER_URL` isn't running; plugin
  skills will be empty. Not relevant to v-pxuw.

---

## 4. Test tiers — run in order, stop at each to report

Each tier gets progressively more expensive. If a tier fails, report
before moving on — upstream breakage usually invalidates downstream
assumptions.

### Tier A — per-repo automated tests (no services needed)

Fastest signal. Run all four in parallel; ~5 min total.

```bash
# SDK (the full suite; tests the vault-free prep surface + utilities)
cd /home/sean/Repos/vultisig/vultisig-sdk
yarn test
# Expect all green. Ticket notes cite 1067 tests pre-rebase; post-rebase
# should match or exceed. If any prep/contractCall or walletCore-related
# tests fail, that's the SDK hand-merge from earlier in the session.

# mcp-ts
cd /home/sean/Repos/vultisig/mcp-ts
pnpm typecheck && pnpm test
# Ticket notes: 252 tests. The execute_* tests + chain-family helpers
# + cosmos-chains registry are the net-new surface.

# agent-backend — highest priority post-rebase
cd /home/sean/Repos/vultisig/agent-backend
go test ./...
# Then specifically:
go test ./internal/service/agent/ -run TestExecCredits -v
# This validates the hand-merged execCredits path from §1a.

# vultiagent-app (tests don't need the emulator)
cd /home/sean/Repos/vultisig/vultiagent-app
npx jest
# Ticket notes: 507 tests. Registry + card tests cover the new surface.
```

**Pass criteria:** all green across the four. Flag any red test in
SDK prep/walletCore, mcp-ts execute_*, or agent-backend agent/executor
paths as P0 — those are the surfaces we hand-merged.

**Add missing tests:** before moving on, add the two uncovered
`execCredits` cases from §1a (paths 4 and 6). They should take ~20
minutes.

### Tier B — mcp-ts HTTP smoke via JSON-RPC

Start mcp-ts (`pnpm dev:http`). No backend needed. Hit each
`execute_*` tool with representative inputs and assert the wire
contract. Curl + jq suffices.

```bash
MCP=http://localhost:9091/mcp

# B1. Tool list — confirm the consolidation surface is present.
curl -s $MCP -X POST -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  | jq '.result.tools | map(.name) | sort' > /tmp/tools.json
# Expect: execute_send, execute_swap, execute_contract_call,
#   execute_lp_add, execute_lp_remove present.
# Expect: defi_prices, defi_yields, defi_protocols, defi_stablecoins,
#   defi_dex_volumes present (consolidated from 9).
# Expect: uniswap_pool_info, uniswap_position_info, uniswap_tick_math.
# Expect: polymarket_search, polymarket_market, polymarket_my.
# Expect: old build_*_send (per-chain) NOT in default list
#   (gated behind "custom_send" category).

# B2. execute_send — EVM native happy path (prep only, no signing).
# Fill <addr> with the Test Vault's Ethereum address: `vsig address ethereum`
curl -s $MCP -X POST -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
    "name":"execute_send",
    "arguments":{
      "chain":"Ethereum",
      "amount":"0.001",
      "recipient":"0x000000000000000000000000000000000000dEaD",
      "sender":"<addr>"
    }
  }}' | jq '.result.content[0].text | fromjson'
# Expect: tx_args with tx_encoding="evm", nonce/gas fields populated
# server-side (execute_send now resolves gas internally — this is the
# new behavior from the final mcp-ts commit).

# B3. C2 regression — SPL token send must reject with escape hatch.
curl -s $MCP -X POST -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
    "name":"execute_send",
    "arguments":{
      "chain":"Solana",
      "token":"EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "amount":"1",
      "recipient":"11111111111111111111111111111111",
      "sender":"11111111111111111111111111111112"
    }
  }}' | jq
# Expect: error mentioning non-native token on Solana is unsupported
# on execute_send; redirect to build_solana_tx or similar.

# B4. C3 regression — Ripple send must reject (no signing_pub_key).
curl -s $MCP -X POST -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
    "name":"execute_send",
    "arguments":{
      "chain":"Ripple",
      "amount":"1",
      "recipient":"rN7n7otQDd6FczFgLdSqtcsAUxDkw6fzRH",
      "sender":"rPEPPER7kfTD9w2To4CQk6UCfuHM9c6GDY"
    }
  }}' | jq
# Expect: explicit rejection pointing at build_ripple_send.

# B5. C4 regression — UTXO swap must emit fee_rate.
curl -s $MCP -X POST -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
    "name":"execute_swap",
    "arguments":{
      "sell_token":"BTC",
      "buy_token":"ETH",
      "amount":"0.001",
      "sender":"bc1q...",
      "destination":"0x..."
    }
  }}' | jq '.result.content[0].text | fromjson'
# Expect: approvalTxArgs OR txArgs has `fee_rate` for the UTXO side.

# B6. C1 regression — LP add/remove must include `kind` discriminator.
curl -s $MCP -X POST -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
    "name":"execute_lp_add",
    "arguments":{"pool":"BTC.BTC","amount_rune":"10","sender":"thor1..."}
  }}' | jq '.result.content[0].text | fromjson.kind'
# Expect: "lp_add" (or equivalent discriminator).

# B7. parseAmount — natural-language amount strings should round-trip.
# Submit "$50", "max", "50%", "10 USDC" and assert amountResolution
# either succeeds or emits an LLM-readable error message.
# (Pick any execute_send invocation varying just `amount`.)
```

**Pass criteria:** B1 shows the right surface; B3/B4/B5/B6 reject or
emit the right discriminator; B2 shows server-side gas resolution
(nonce present even without prior evm_tx_info call). Each failure is
a P0 since these are wallet-grade correctness checks.

### Tier C — agent-backend integration (backend + mcp-ts, no LLM)

Start both services. Exercise the wire contract between backend and
mcp-ts using direct HTTP to the backend. Good signal on: tool
classification, tool filtering, client-side action prep, balance
fan-out, SSE event shapes.

```bash
API=http://localhost:8084

# C1. Health.
curl -s $API/healthz

# C2. Auth with the test vault (requires signing a challenge). Easier
# path: DEV_SKIP_AUTH=true is set in .env, so the middleware accepts
# any JWT for dev. Issue a throwaway one:
export JWT=$(
  curl -s $API/auth/token -XPOST -H 'content-type: application/json' \
    -d '{"public_key":"<vault-ecdsa-pubkey-hex>","signature":"<signed-challenge>","timestamp":'$(date +%s)'}' \
    | jq -r .token
)
# If DEV_SKIP_AUTH works as labeled, you can skip signing and POST
# a fake token; check `internal/auth/jwt.go` for the dev-mode path.

# C3. Start a conversation, send a balance question, verify tool
# classification shows `get_balances` getting called with chain
# fan-out to mcp-ts for any chain not in the prefetched SSE context.
CONV=$(curl -s $API/agent/conversations -XPOST \
  -H "Authorization: Bearer $JWT" -H 'content-type: application/json' \
  -d '{"title":"test"}' | jq -r .id)

curl -s $API/agent/conversations/$CONV/messages -XPOST \
  -H "Authorization: Bearer $JWT" -H 'content-type: application/json' \
  -d '{
    "public_key":"<vault-ecdsa-pubkey-hex>",
    "content":"what'\''s my ETH and SOL balance?",
    "context":{"balances":[{"chain":"Ethereum","amount":"0.1"}]}
  }' \
  | tee /tmp/send.json | jq '.message.content, .tool_calls'

# Expect: server fans out to mcp-ts for Solana balance (not prefetched),
# response mentions both values. Also confirms §1c Step 2: the
# SendMessageRequest with no `tools` field is accepted.

# C4. Client-side action tool bridge (JSON response path).
curl -s $API/agent/conversations/$CONV/messages -XPOST \
  -H "Authorization: Bearer $JWT" -H 'content-type: application/json' \
  -d '{
    "public_key":"<vault-ecdsa-pubkey-hex>",
    "content":"add USDC to my vault"
  }' | jq '.tool_calls'
# Expect: a client-side tool_call for vault_coin with
# action="add", coins=[...] — this is the JSON-response bridge we
# added in the final agent-backend commit.

# C5. Historical conversation replay (§ covers convert.go C6).
# If the DB has pre-rename conversations (old tool part types like
# `schedule_task`, `add_coin`, etc.), fetch one via
# GET /agent/conversations/:id and assert the part types are rewritten
# to the new names on read (the replay happens in
# internal/storage/postgres/convert.go).
curl -s $API/agent/conversations/<old-conv-id> \
  -H "Authorization: Bearer $JWT" | jq '.messages[].parts[].type'
# Expect: no stale `schedule_task` / `add_coin` / etc. — they should
# be rewritten to `schedule_create` / `vault_coin` etc.

# C6. Credits — §1a paths 1-4 against the live handler.
# Path 1: DEV short-circuits launchsurface.CreditsEnabled to true
# (everything-on). Flip POSTHOG_ENVIRONMENT=prod in .env, restart
# the backend, then call credits. With no real PostHog flag in
# PH_API_KEY='', the default fallback path should engage.
```

**Pass criteria:** C3 shows mcp-ts fan-out; C4 returns client-side
tool_calls in the non-stream JSON response (this is net-new in the
final agent-backend commit); C5 replays renamed part types; credits
respects the flag.

### Tier D — full stack via CLI (end-to-end, no emulator)

The CLI talks to the backend over HTTP and drives the same code paths
the mobile app does (auth → conversation → SSE → tool calls). This is
the closest we get to emulator coverage without booting it.

```bash
VULTISIG_AGENT_URL=http://localhost:8084 vsig agent ask "what's my ETH balance"
# Expect: answer mentions real ETH amount. Exercises auth flow,
# prompt assembly, tool selection (`get_balances`), SSE streaming.

VULTISIG_AGENT_URL=http://localhost:8084 vsig agent ask "list my BTC and DOGE balances"
# Multi-chain query — exercises balance fan-out for chains not in
# prefetched SSE context.

VULTISIG_AGENT_URL=http://localhost:8084 vsig agent ask "send 0.0001 ETH to 0x000000000000000000000000000000000000dEaD"
# Prep-only; the CLI should show a swap/send confirmation card
# mediated by the Execution.Stepper-equivalent client logic.
# Cancel before signing to avoid burning testnet/real funds.

VULTISIG_AGENT_URL=http://localhost:8084 vsig agent ask "swap 0.0001 ETH to USDC on Arbitrum"
# Exercises execute_swap. Same — cancel before signing.

VULTISIG_AGENT_URL=http://localhost:8084 vsig agent ask "what's my credit balance"
# Exercises the hand-merged execCredits path — matches §1a.

VULTISIG_AGENT_URL=http://localhost:8084 vsig agent ask "add BONK to my vault"
# Exercises the client-side action bridge (vault_coin tool). The CLI
# should receive a client-side tool_call and prompt for confirmation.
```

Run these interactively too (`vsig agent` without `ask`) — the TUI
path uses a slightly different client code path in the CLI.

**Pass criteria:** no stalls, no hung SSE, no `[building...]`
placeholders leaking through, the send/swap confirmation emits the
structured bullet template the app expects.

### Tier E — LLM tool-selection eval (optional, requires a harness)

The whole consolidation's justification is selection-accuracy gain on
weak models. Without an eval set you can't actually measure this — but
if the team has one (or is willing to write ~20 canned utterances →
expected first-tool-call pairs), this is the highest-signal single
test you can run pre-emulator.

Minimum viable harness:
```
- "send 0.1 ETH to alice" → execute_send
- "swap 10 USDC to ETH" → execute_swap
- "call balanceOf(0xabc) on 0xdef" → execute_contract_call
- "add 100 RUNE to the BTC pool" → execute_lp_add
- "remove half my BTC LP" → execute_lp_remove
- "what's my portfolio" → get_portfolio
- "what's the TVL of Aave" → defi_protocols
- "price of ETH" → get_market_price
- "add DOT to my vault" → vault_coin
- "cancel my swap" → schedule_cancel  (or protocol_state depending on context)
- (+ the rest)
```
Drive via Tier D commands, capture the first `tool_call` emitted,
compute match rate against expected. v-pxuw wins if this is >=80%.

Check `agent-backend/scripts/benchmark-*` — the ticket notes said the
old benchmark testdata was deleted as regeneratable; the scripts may
still exist and be adaptable.

---

## 5. Reporting format

Report a single markdown table per tier:

| Tier.Step | Result | Notes | Priority |
|---|---|---|---|
| A.SDK | ✅ 1067/1067 | — | — |
| A.mcp-ts | ❌ 2 fail | `execute-swap.test.ts:120` amount resolution | P0 |
| B.1 tools/list | ✅ | 30 always-on + ~110 gated | — |
| B.3 SPL reject | ❌ | accepts SPL send instead of rejecting | P0 |
| ... | | | |

Priority flags:
- **P0** — wallet-grade correctness (money-at-risk, signing incorrectness, crashes).
- **P1** — flow-broken (a documented v-pxuw feature doesn't work).
- **P2** — UX-degraded (LLM stalls, wasted round-trips, confusing errors).
- **P3** — cosmetic (log noise, doc drift).

At the end, include:
- Count of P0/P1 issues with one-line summaries.
- Whether the branch is ship-ready on this surface.
- List of un-coverable-without-emulator paths (we know these exist;
  the purpose is to enumerate them so the emulator pass is scoped).

---

## 6. Known non-obvious things

- Agent-backend `DEV_SKIP_AUTH=true` is set; don't expect auth errors
  to behave like prod. Flip to `false` explicitly if you want to test
  the JWT `exp` enforcement added in the final agent-backend commit
  (`internal/auth/jwt.go`).

- `POSTHOG_ENVIRONMENT=dev` short-circuits launchsurface to
  EverythingOn. To exercise the gating paths (§1a paths involving
  disabled flags), flip to `prod` and either configure a real PostHog
  key or rely on the default fallback payload
  (`internal/launchsurface/flags.go`).

- mcp-ts consumes the local SDK via pnpm override (`file:../vultisig-sdk/packages/sdk`).
  If you rebuild the SDK (`yarn build:shared && yarn build:sdk`),
  mcp-ts picks it up on the next restart — no `pnpm install`
  needed unless you change the override.

- The test vault is a fast-vault **share 1 of 2**. You have the key
  share for auth but signing requires VultiServer cooperation via the
  default `VULTISIG_SERVER_URL=https://api.vultisig.com/vault`. That's
  fine for reads and auth; don't try to sign secure-vault flows
  locally.

- `~/.secrets` is sourced by `~/.zshrc`, so in a fresh terminal
  `VAULT_PASSWORD` / `VAULT_PASSWORDS` are already set. In a shell
  opened before the secret was added, run `source ~/.secrets`.

- If you see `build_evm_tx` / `abi_encode` / `convert_amount` in the
  default MCP `tools/list`, the `_meta.categories` gating broke
  somewhere. They should be gated behind `contract` / `raw` keywords
  per the ticket's §4 design.

---

## 7. When you're done

Leave a `tk note v-pxuw "..."` on the ticket with the final report
table + a link to this file. If any P0 surfaced, also leave one-line
notes on each of the four PRs (#284, #30, #128, #164) with the
specific regression + repro command.
