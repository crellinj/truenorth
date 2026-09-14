# Notarization to Bitcoin — design space

**Status:** Design document. No implementation yet. No committed timeline
for building or activation. This document exists so future work has a
clear starting point.

TRUENORTH_SPEC.md §9 commits to *"notarization to a larger chain
(delayed-PoW style)"* as a planned defense against long-range reorg
attacks. This doc lays out the design space, cost model, and open
questions. It does not pick answers.

## What notarization is (and isn't)

**Notarization** — periodic commitment of TrueNorth block hashes into a
higher-hashrate chain (Bitcoin, most likely), used as an immutable
anchor. Once a TrueNorth block hash has been committed into Bitcoin and
buried under enough Bitcoin confirmations, any attempt to reorg the
TrueNorth chain past that point can be rejected by consensus rules,
regardless of PoW.

**Notarization is not:**
- A payment channel or L2 throughput mechanism
- A sidechain or bridging system
- A replacement for the 72-block reorg cap
- A way to speed up TrueNorth transactions

## Why it matters

TrueNorth's early-year security relies on a stack of defenses (see
TRUENORTH_SPEC.md §9):

1. RandomX PoW with RandomNorth variant (raises attacker cost)
2. LWMA-1 per-block difficulty (defeats hash-and-run)
3. 72-block hard reorg cap (bounds attack window)
4. Release-time checkpoints (immutable historic anchors)
5. Notarization (this document)
6. Emergency-response protocol (post-attack recovery)

The 72-block reorg cap is the load-bearing defense at v1 launch — it
means no attack can rewrite more than ~2.4 hours of chain history no
matter how much hashpower the attacker rents. Checkpoints anchor
historical blocks at each release.

**What notarization adds:** the gap between "last release-time
checkpoint" and "current 72-block reorg cap window" is currently
undefended in real-time terms. If a release ships a checkpoint at
height H and current tip is H+50,000, the 72-block cap only protects
the last 72 blocks. Blocks between H+72 and H+50,000-72 are protected
only by cumulative PoW — which a well-funded whale could theoretically
overwhelm on a small chain.

Notarization fills this gap by anchoring current chain state into
Bitcoin at regular intervals. The effective reorg cap becomes
**"whichever comes first: 72 blocks, or the last notarized block."**

## Core mechanism

Regardless of specifics, the shape is:

1. **Poster** — a service that:
   - Reads the current TrueNorth chain tip periodically
   - Constructs a Bitcoin transaction containing a commitment
     (the TrueNorth block hash, or a merkle root of recent hashes)
   - Broadcasts the transaction to Bitcoin
2. **Bitcoin confirmation** — the notarization commitment gets buried
   under Bitcoin PoW over several Bitcoin blocks
3. **TrueNorth verification** — TrueNorth nodes verify that:
   - The commitment exists in a Bitcoin block
   - That Bitcoin block has sufficient Bitcoin PoW behind it
   - The committed TrueNorth block hash matches the local TrueNorth
     chain state
4. **Consensus enforcement** — TrueNorth nodes reject any reorg attempt
   that would replace a notarized block

## Design choices

Four decisions determine the shape of any implementation. This doc
enumerates the tradeoffs; specific answers wait until implementation
work is scheduled.

### 1. Notarization frequency

How often is a commitment posted?

| Frequency | Approx TrueNorth height coverage | Notes |
|---|---|---|
| Every 100 blocks | ~3.3 hours between commitments | Tightest anchor; highest Bitcoin fee cost |
| Every 500 blocks | ~16.7 hours | Moderate |
| Every 1,000 blocks | ~33 hours (daily-ish) | Reasonable balance |
| Every 5,000 blocks | ~7 days (weekly) | Cheap; loose |
| Every 50,000 blocks | ~70 days (post-year-3 checkpoint cadence) | Very cheap; effectively parity with checkpoints |

**Tradeoff**: tight anchor = short window an attacker could target with a
long-range reorg attempt, at higher operating cost. Loose anchor = cheap
to run but longer undefended-window between checkpoints and
notarizations.

**Consideration**: the anchor doesn't need to match the block cadence.
A time-based schedule (post every 6 hours) is also viable and simpler
to reason about operationally.

### 2. Poster architecture

Who runs the poster? Three archetypes:

**Single-poster (maintainer-run):**
- Simplest. One entity holds a Bitcoin wallet, runs the poster daemon,
  pays the fees.
- Single point of failure. If the poster stops running, notarizations
  stop, and the effective reorg cap shrinks to 72 blocks until the
  poster resumes.
- Single point of trust. The poster could be coerced or compromised
  into publishing false commitments (though the consensus rule requires
  the commitment to MATCH the actual chain — a wrong commitment doesn't
  reorg the chain, it just doesn't help).
- Right choice for early years.

**Federated poster set:**
- A committee of N operators, threshold M-of-N required for a valid
  commitment (multi-sig Bitcoin transaction).
- No single point of failure.
- More complex operations, needs coordination protocol.
- Right choice once the project has more established operators.

**Anyone-can-post (permissionless):**
- Any operator can post commitments; consensus takes the earliest valid
  one for each notarization epoch.
- No trust concentration.
- Fee costs are competitive/distributed but not aggregated (each poster
  pays for their own attempts, first to succeed "wins" from a consensus
  perspective).
- Interesting long-term direction; hard to reason about at launch.

### 3. Node verification model

How do TrueNorth nodes verify that a Bitcoin commitment actually exists
and is buried under sufficient PoW?

**Full Bitcoin node dependency:**
- Every TrueNorth node runs a Bitcoin full node in parallel.
- Strongest security; no trust in third parties.
- Massive resource cost — Bitcoin's chain is ~600 GB, TrueNorth nodes
  today are ~1 GB. Multiplies operator storage 600×.
- Not viable for a mainnet requirement.

**Bitcoin SPV (light client) integration:**
- TrueNorth nodes include an SPV Bitcoin client that verifies block
  headers only (~50 MB, not full chain).
- Medium security. SPV is vulnerable to some attacks but sufficient for
  notarization verification.
- Medium implementation complexity. TrueNorth's codebase would need to
  incorporate Bitcoin network peer discovery and header sync.
- Practical for a mainnet feature.

**Maintainer-signed assertions:**
- The poster (or maintainer) periodically publishes a
  cryptographically-signed statement: "TrueNorth block hash X was
  notarized at Bitcoin height Y at time Z."
- TrueNorth nodes verify the signature against the maintainer's public
  key; consensus rejects reorgs past X.
- Lightest security — reduces to "trust the maintainer's signing key."
- Very low implementation complexity.
- Sits somewhere between "actual notarization" and "release-time
  checkpoints." Useful bridge if actual SPV integration is deferred.

**Hybrid:**
- Node either has an SPV client OR trusts maintainer assertions,
  chosen per-node.
- SPV nodes get full security; assertion-trusting nodes get maintainer-
  trust security.
- Best of both, most complex.

### 4. Funding model for Bitcoin fees

Bitcoin transactions cost real money. See "Cost estimate" below.

Options:

**Maintainer pays out of pocket:**
- Simplest. Septentrion or whoever runs the poster funds it.
- Not sustainable long-term if the project doesn't generate revenue.

**Coinbase contribution:**
- A configurable fraction of block reward gets redirected to a
  notarization fund address.
- Aligns cost with the network being defended.
- Requires consensus change (soft fork). Ideally activated at genesis
  or as part of a planned soft fork.
- Cleanest long-term solution.

**Community donation:**
- Voluntary contributions to a notarization treasury address.
- No consensus change needed.
- Uncertain funding stream.

**No funding (defer):**
- Don't build until sustainable funding is arranged.
- Realistic if notarization is a year-2+ effort.

### 5. Activation approach

If TrueNorth nodes need to enforce notarization commitments as a
consensus rule, this is a network-wide change:

**Hard fork:**
- Old nodes reject the new rule; new nodes require it.
- Requires all operators to upgrade before the fork height.
- Clean semantics.

**Soft fork:**
- Old nodes still accept the chain but don't enforce the rule.
- Only miners need to upgrade to produce blocks.
- Notarization rule effectively becomes enforcement-by-majority-miners.
- Simpler activation logistics.

**Optional per-node (no consensus change):**
- Nodes optionally enforce notarization based on local config.
- No fork. Notarization becomes a defense-in-depth option rather than
  a consensus rule.
- Weakest — one non-enforcing node could still be reorged.

## Cost estimate

Bitcoin transaction fees vary. Rough numbers:

| BTC fee tier | Typical tx fee (USD) | Cost at various cadences |
|---|---|---|
| Low congestion (~1 sat/vB) | ~$0.30 | Daily: $110/yr; weekly: $16/yr |
| Medium (~10 sat/vB) | ~$3 | Daily: $1,100/yr; weekly: $156/yr |
| High (~50 sat/vB) | ~$15 | Daily: $5,500/yr; weekly: $780/yr |
| Extreme (~200 sat/vB) | ~$60 | Daily: $22,000/yr; weekly: $3,100/yr |

Assumptions: single-input single-OP_RETURN-output transaction, ~200
vbytes. Actual sizes vary with wallet implementation.

**Realistic budget planning: ~$500-2,000 USD per year for weekly
notarization at typical BTC fee tiers.** Higher if the project chooses
daily or tighter cadence.

**In BTC terms**: at $50,000 BTC/USD, weekly notarization is ~0.01
BTC/year. A modest BTC reserve of 0.1 BTC funds ~10 years of weekly
notarization at typical fees.

## Threat model

**What notarization defends against:**

- **Long-range reorg attacks** where an attacker with sustained >50%
  hashrate builds a private chain and publishes it. Notarization
  prevents reorg past the last committed block.
- **Time-warp attacks** past a notarized point.
- **Nothing-at-stake / stake-based attacks** — N/A for a PoW chain,
  mentioned for completeness.

**What notarization does NOT defend against:**

- **Attacks within the reorg window** (between the last notarization
  and the current tip). Up to the notarization frequency, a 51%+
  attacker could still reorg — bounded by the 72-block cap.
- **Consensus bugs on the TrueNorth side**. Notarization commits a
  hash; if the chain state producing that hash was already invalid,
  notarization doesn't rescue anything.
- **Bitcoin-side attacks**. If Bitcoin itself gets attacked or forks,
  the anchor becomes ambiguous. Practically not a concern for the
  foreseeable future.
- **Poster compromise leading to withheld commitments**. If the poster
  stops posting or is compelled to stop, the effective reorg cap
  shrinks to the 72-block cap. Not fatal, but reduces security.

## Interaction with existing defenses

- **72-block reorg cap** stays in place regardless. Notarization is
  additive, not a replacement.
- **Release-time checkpoints** remain the mechanism for anchoring
  historical blocks at every release. Notarization anchors the *gap*
  between checkpoints.
- **`nMinimumChainWork` and `defaultAssumeValid`** should be updated
  alongside notarization implementations, similar to checkpoint
  updates today.
- **Emergency-response protocol** (post-attack coordinated checkpoint
  releases) remains the ultimate recovery mechanism if notarization
  and other defenses fail. Notarization makes the emergency-response
  scenario much less likely to be needed.

## Open questions to resolve at implementation time

1. **Which specific frequency** (100 / 500 / 1,000 / 5,000 / 50,000
   blocks, or time-based)?
2. **Poster architecture** — single-poster acceptable for launch or
   push directly to federated?
3. **Node verification** — SPV integration effort worth it for launch,
   or maintainer-signed assertions as bridge?
4. **Funding model** — coinbase contribution requires soft fork
   planning; decide before or after other soft fork batching?
5. **Activation approach** — hard vs soft fork, block-height or
   flag-day activation?
6. **What happens if a notarization commitment gets orphaned on the
   Bitcoin side** (rare but possible)? Retry logic? Grace period?
7. **How is the poster's Bitcoin key managed** — hot wallet, cold
   wallet, hardware security module?
8. **What operational monitoring is needed** — alerts when
   notarization delays exceed a threshold?

## References

- [`TRUENORTH_SPEC.md`](../TRUENORTH_SPEC.md#9-security--51-attack-resistance) §9
- [`doc/checkpoint-process.md`](checkpoint-process.md) — related historic-anchor mechanism
- [`doc/exchange-listing-guidance.md`](exchange-listing-guidance.md) — attack economics context
- Komodo delayed-PoW — prior art in altcoin notarization to Bitcoin
- Veriblock — commercial notarization service (proof-of-proof to Bitcoin)

## Implementation status

**Not implemented. No committed timeline.** Future maintainers or a
future project decision-point make the actual build calls. This
document exists to give that future work a clear starting point.
