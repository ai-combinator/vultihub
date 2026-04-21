# Vulti Agent Marketing Playbook

For engineers. No fluff, no content calendars, no approval flows.

---

## 1. Go to Market: Station Rebrand

Station becomes Vulti Agent. We rename in-place — no new app store listing.

**Why rename in-place:** Trust Wallet (originally Trust) and MEW (MyEtherWallet) both did this. You keep ratings, reviews, download count, and search ranking. A new listing starts at zero.

### Staged rollout

1. **Hype (1-2 weeks before):** Push an update to Station with a teaser screen on launch. Something like "Something new is coming" with the Vulti Agent logo. This does two things: validates the update pipeline works, and seeds curiosity with existing users.
2. **Tease on X:** @vultisig posts a single cryptic visual — the Station icon morphing into Vulti Agent. No explanation. Let people speculate for 3-5 days.
3. **Announce:** @vultisig drops a thread. What it is, why it exists, link to the update. Engineers quote-tweet with their own angle.
4. **Ship:** App store update goes live same day as the announcement thread. "Formerly Station" stays in the app subtitle for 2-3 months, then drop it.

### KOL seeding

Target 15-20 mid-tier crypto influencers (10K-100K followers) who actually try products. These are people who post real screen recordings, not "excited to announce my partnership with" types.

- Find them by searching for tweets with screenshots of other wallets (Rabby, Phantom, Rainbow)
- DM them a TestFlight/APK link 48 hours before the public announcement
- No payment, no contracts. If they like it, they post. If they don't, you learn something
- One person owns this list and the DMs — don't spray from multiple accounts

### Early Supporter badge

First-week updaters get an on-chain attestation (EAS on Base or similar). Costs us gas, costs them nothing. This becomes a social signal — people screenshot badges and post them. It also builds a list of engaged early users for future beta access.

---

## 2. The AI Demo Is the Marketing

Nobody else has a natural language wallet. That is the entire marketing strategy. Show it working.

### Why this works

ChatGPT didn't grow through ads. It grew because people shared screenshots of conversations. Same principle: when someone sees a text prompt execute a swap or check a portfolio, the reaction is "wait, I can do that?" That reaction is distribution.

### Short-form demo content

- **Side-by-side comparisons:** Split screen. Left: the 7-tap flow to swap on MetaMask. Right: "swap 0.5 ETH to USDC" in Vulti Agent. No voiceover needed.
- **Screen recordings:** 30-60 seconds max. Real tasks, real chains, real results. Post natively to X (not YouTube links — algorithm buries external links).
- **Before/after:** Show the same task in a traditional wallet vs. Vulti Agent. Time it. The delta speaks for itself.

Anyone on the team can make these. QuickTime screen record on macOS, trim in iMovie, post. No production value needed — raw is better in crypto.

### MPC security narrative

"No seed phrase" is a massive differentiator. Fireblocks built a billion-dollar institutional business on this single message. We have the same tech in a consumer product.

Frame it simply:
- Traditional wallets: one key, one point of failure, write 12 words on paper
- Vulti Agent: MPC splits the key across devices, no single point of compromise, nothing to write down

This matters for the AI angle too: you can give an AI agent signing capability without ever exposing a private key. That's a story nobody else can tell right now.

### Ecosystem visibility

Low-effort, high-leverage moves:
- **DeFi Llama:** Get listed under wallets. It's a PR to their GitHub repo.
- **Chain ecosystem pages:** Every L2 maintains a list of supported wallets. Submit to Arbitrum, Base, Optimism, etc.
- **WalletConnect:** Already integrated with Vultisig. Make sure the listing is updated with Vulti Agent branding.
- **MCP server as distribution:** The MCP server means any AI agent framework (LangChain, CrewAI, AutoGPT) can plug into Vultisig for wallet operations. Post in those communities. Target AI agent builders, not just crypto users.

---

## 3. Builder-Led Marketing: The Engineer's Guide

In crypto, technical depth equals legitimacy. Polished marketing is a scam signal. Monad, Eigenlayer, and Uniswap all grew on the backs of engineers posting about what they were building. Not a marketing team.

### Setup (one-time, 10 minutes)

- Put "Building @vultisig" in your X bio. Vulti Agent is a sub-product. One brand, one funnel. Nobody outside the team needs to know the org chart.
- Pin a tweet explaining what Vulti Agent is in your own words. One tweet or a short thread. This is the thing people see when they click your profile after you post something interesting.

### What to post

Only post when you hit something genuinely interesting:
- A cool bug and how you fixed it
- An architecture decision and why you made it
- A before/after of a feature
- A demo of something working for the first time
- A hot take on wallet UX, MPC, or AI agents that you actually believe

No schedule. No quota. Some weeks you'll post three times, some weeks zero. Both are fine.

### How to post

- Tag @vultisig at the end of threads, not in every tweet. Constant tagging looks desperate.
- Use screenshots, screen recordings, or code snippets. Visual content gets 2-5x the engagement of text-only.
- Threads > single tweets for anything that needs context. But keep threads to 3-5 tweets max.

### The @vultisig account

- Posts ONE shipping thread per week. Friday. Takes 30 minutes to write. Screenshots, clips, what shipped, what's next.
- Amplifies engineer posts via quote-tweet with added context ("Our engineer just solved X — here's why that matters for Y"). Quote-tweets get 2-3x the reach of a plain retweet.
- That's it. The main account is a megaphone for the team's work, not a content machine.

### What NOT to do

- Don't create a @vultiagent account. One brand, one audience, one account.
- Don't set up content calendars or posting schedules.
- Don't do daily posting for the sake of consistency.
- Don't require approval flows for personal posts. You're adults.
- Don't post "GM" or engagement bait. Instant credibility loss.

---

## 4. What "Good" Looks Like (3 months)

### Per engineer
- 500-2,000 engaged followers who associate you with Vultisig
- A few posts that got meaningful replies from people you don't know
- At least one "how does this work?" DM from another builder

### @vultisig
- 3,000-5,000 engaged followers (not bought — engagement rate matters more than count)
- 2-3 threads that broke 50K impressions
- Inbound partnership or integration DMs from X (this is the real signal)

### The metric that matters

Engagement rate and inbound DMs. Not follower count.

If people are replying, quoting, and DMing about integrations, the strategy is working. If follower count is climbing but nobody's replying, something is wrong — you're attracting bots or the content isn't resonating.

Track inbound DMs in a shared channel so the whole team has visibility. That's your only "marketing dashboard."
