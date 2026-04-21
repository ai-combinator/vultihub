---
id: v-trcs
status: open
deps: []
links: []
created: 2026-04-17T00:19:41Z
type: task
priority: 0
assignee:
---
# Terra Classic Station-era address snapshot

## Why

Airdrop eligibility splits wallets into two tiers: **Station OG** (used Station during the Terra Classic era, May–Dec 2022) and **Vault OG** (everyone else). To decide which pool a boarding user enters, the backend derives the wallet's Terra Classic address from the imported seed phrase and checks it against a list of every address that touched Station in that window.

That list does not exist anywhere we can query today. Producing it unblocks `/airdrop/register` and the raffle draw — both on the critical path for Day 8 boarding open.

## Goal

A queryable snapshot of every Terra Classic address that interacted with Station between 2022-05-01 and 2022-12-31, available to the backend with a lookup latency low enough to not block registration (PRD target: <5ms per address).

Storage format, extraction implementation, and whether the snapshot is rebuilt or append-only are open — pick what fits. The deliverable is the data being available and the lookup path being live in the backend before Day 8.

## Context — infrastructure blocker

Public Terra Classic RPC nodes are pruned; their earliest retained block is around 28M. Station-era activity sits at blocks ~4.7M–7.5M, which is gone from standard RPCs. An archive node (full historical state) is required to extract.

Three paths worth considering, rough order of effort/cost:

- **Rent a dedicated archive node.** Allnodes offers these at ~$50–100/mo. Fastest if we only need to run the extraction once or twice.
- **Contact `hexxagon_io`.** Reportedly already runs an archive node and may grant access.
- **Self-host.** Cheapest long-term, highest ops burden. Probably overkill for a one-shot extract.

## Out of scope

- The `/airdrop/register` endpoint (separate mission consumes this data)
- Tier assignment logic beyond "did this address touch Station in the window"
- Ongoing/live updates — this is a one-shot historical snapshot

## Definition of done

- Archive node access secured via one of the paths above
- Snapshot produced covering 2022-05-01 → 2022-12-31 Station interactions
- Backend can resolve an arbitrary `terra_address` to a boolean (Station OG or not) at the latency PRD-ELIGIBILITY-TRACKING US-1 specifies
- Extraction is resumable from checkpoint if interrupted mid-run (extract takes hours-to-days)

## References

- Engineering Brief → Eligibility workstream (v-vultiverse deck)
- `PRD-ELIGIBILITY-TRACKING.md` US-0 (extractor), US-1 (loader)
- Pre-Flight Status row "Terra Classic address snapshot?" marked BLOCKED
