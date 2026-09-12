# Exchange listing guidance

This document is for two audiences:

- **Exchange operators** considering listing NORTH — what depths are safe, and
  when.
- **Users** deciding whether a claimed NORTH listing is legitimate.

The project does not solicit, negotiate, or pay for exchange listings.
Anything below is guidance grounded in the security model, not a business
development document.

## The short version

- **First 30 days after mainnet launch:** no listing is legitimate. Treat any
  exchange claiming NORTH support as a scam or reckless third party.
- **Month 2 through Month 6:** cautious listings are possible if the exchange
  requires a **minimum 100-block (~3.3 hour) confirmation depth** for
  deposits. Below that depth, expect double-spend attacks.
- **Month 6 through Year 1:** more permissive as network hashrate grows.
  Confirmation depth of 60+ blocks becomes reasonable.
- **After notarization to a larger chain is deployed** (see
  [TRUENORTH_SPEC.md](../TRUENORTH_SPEC.md) §9): confirmation depth
  requirements can relax to conventional levels. Attack cost is bounded by
  the notarization interval.

## Why confirmation depth matters

TrueNorth is a small-network Proof of Work chain in its first year. The
threat model is a well-resourced attacker renting cloud CPU capacity to
outpace organic mining, reorganize recent history, and double-spend against
an exchange.

RandomX is CPU-only, which cuts both ways: it means no ASIC advantage for
defenders, and it also means an attacker can rent capacity on demand from
any cloud provider. There is no "capital investment" barrier separating
attackers from defenders.

Rough attack economics — cost to sustain a reorg for one day, assuming an
attacker willing to pay commodity cloud on-demand pricing:

| Cloud capacity | Approx hashrate | Approx cost/day |
|---|---|---|
| 5 × c5.24xlarge | ~250 kH/s | ~$500 |
| 10 × c5.24xlarge | ~500 kH/s | ~$1,000 |
| 50 × c5.24xlarge | ~2.5 MH/s | ~$5,000 |
| 100 × c5.24xlarge | ~5 MH/s | ~$10,000 |

For the first year, an attacker's realistic upper bound is well above the
project's organic hashrate. Depth-of-confirmations is what makes the attack
uneconomical.

At 120-second block time:

| Confirmations | Chain time | Attacker cost to reorg |
|---|---|---|
| 6 | 12 min | ~$50 |
| 30 | 60 min | ~$500 |
| 100 | ~3.3 hr | ~$5,000+ |
| 200 | ~6.7 hr | ~$20,000+ |
| 500 | ~16.7 hr | ~$100,000+ |

**A 6-confirmation deposit policy on TrueNorth is not a policy — it's a
guaranteed loss.** The minimum defensible threshold is 100 confirmations
until organic hashrate has grown substantially or notarization is live.

Recommended settings by phase:

| Phase | Min deposit depth | Withdrawal delay |
|---|---|---|
| Months 1-3 | 200 blocks | 200 blocks matured before withdrawal |
| Months 3-6 | 100 blocks | 100 blocks |
| Months 6-12 | 60 blocks | 60 blocks |
| Post-notarization | 30-60 blocks | 30-60 blocks |

## What the project will and won't do

**Will:**
- Respond to technical inquiries from exchange operators (API, node setup,
  chain parameters, reorg-monitoring recommendations).
- Confirm on `#truenorth` IRC (OFTC) that a specific listing appears
  legitimate, once due diligence indicates it is.
- Publish a security advisory if a listing is unsafe (too-low confirmation
  depth, obvious operational red flags, or believed-fraudulent).
- Point to the on-node monitoring scripts in
  [`contrib/monitoring/`](../contrib/monitoring/) — in particular
  `reorg-monitor.sh` for detecting deep chain reorganizations from a
  running node.

**Will not:**
- Accept listing fees, marketing spend, or paid coordination.
- Coordinate airdrops, giveaways, or promotional campaigns with any
  exchange.
- Sign legal agreements or "letters of endorsement" for exchanges.
- Respond to KYC-related inquiries about the project or contributors.
- Provide market-making, liquidity provision, or any pre-listing token
  transfers. There are no such tokens: NORTH exists only as mined coinbase
  emissions.

## Advice for users

- Any listing claim in the first 30 days after mainnet launch is either
  a scam or an exchange operating recklessly. Neither category is safe
  to deposit into.
- Even after 30 days, check the exchange's stated deposit confirmation
  depth. If it says 6 blocks (or worse, "1 block for testnet-style speed"),
  do not deposit. The exchange either does not understand the risk or does
  not care about your deposits.
- Real exchanges pursuing a legitimate listing will be identifiable through
  `#truenorth` IRC discussion — either the project confirming they exist,
  or the community discussing them openly.
- If an exchange DMs you claiming to represent the project, or asking you
  to send NORTH to prove ownership, verify signatures, or complete a
  "listing pool contribution": stop. The project does not communicate that
  way. See [README's community section](../README.md#community) for
  official channels.

## Advice for exchange operators

If you are considering listing NORTH:

1. **Sync a full node yourself** from `github.com/truenorth-project/truenorth`
   releases. Verify the GPG signature (fingerprint in
   [README.md](../README.md), IRC topic, and BitcoinTalk ANN) before
   running.
2. **Run for at least two weeks** to validate reorg behaviour, block time
   variance under LWMA, and seed-key rotation on your infrastructure.
3. **Set a deposit confirmation minimum** per the table above. This is
   not a suggestion. A one-day double-spend attack on your platform will
   cost you far more than the deposit-time UX friction it prevents.
4. **Monitor for deep reorgs.** Run `getchaintips` on a schedule.
   Auto-freeze deposits if the height differential exceeds
   1.5× your deposit confirmation depth. Reference scripts in
   [`contrib/monitoring/`](../contrib/monitoring/); `reorg-monitor.sh`
   is a starting point.
5. **Follow `#truenorth-dev` on OFTC IRC.** Emergency-response actions
   (such as coordinated checkpoint releases in response to a detected
   attack) are announced there first. If you list NORTH and don't watch
   that channel, you're operating blind during the highest-risk period
   for the chain.

The project welcomes technical inquiries from exchange operators. Reach
out via `#truenorth-dev` on OFTC (open, logged), or GitHub issues for
protocol/consensus questions. Do not expect email correspondence — the
project operates through public channels.

## When this policy relaxes

Two milestones materially change the risk model:

1. **Sustained organic hashrate at 10× launch levels.** Attack cost scales
   with total hashrate. When honest mining is regularly producing >100 kH/s
   and has been for several months, a whale needs 10× more spending to
   outpace it.

2. **Notarization to a larger chain deployed.** TRUENORTH_SPEC.md §9
   commits to notarizing TrueNorth block hashes into a high-hashrate chain
   (Bitcoin, most likely) at intervals. Once live, the effective reorg cap
   is the notarization frequency — an attacker cannot rewrite history past
   the last notarization regardless of hashrate spent.

Neither milestone has a fixed calendar date. Both will be discussed openly
on `#truenorth-dev` when they land, and this document will be updated with
revised guidance.

## References

- [TRUENORTH_SPEC.md §9 — Security & 51%-Attack Resistance](../TRUENORTH_SPEC.md#9-security--51-attack-resistance)
- [doc/checkpoint-process.md — Maintainer runbook for release-time checkpoints](checkpoint-process.md)
- [doc/mining-policy.md — Project stance on mining topology and 33% norm](mining-policy.md)
- [doc/testnet.md — Testnet4 bootstrap; useful for exchange integration testing](testnet.md)
