# Flock

**A Bitcoin Layer 2 on the Nostr relay network — payments without channels, liquidity
management, or single points of failure.**

Status: **DRAFT v0.3 — 2026-06-12** · protocol design only, no implementation yet.

## What this is

Flock is a protocol design for a Lightning successor built from covenant-less Ark
pools, with two structural changes and one infrastructural one:

1. **No single operator.** Each pool is run by a 4-of-7 FROST quorum of independent
   operators. Theft requires 4 colluding; liveness survives 3 failures; checkpointed
   funds survive even total quorum collusion or death.
2. **No liquidity management, ever.** Receivers need only a Nostr identity (npub) —
   no inbound liquidity, no channels, no setup transaction. Payments are instant
   out-of-round cosigns; the capital burden moves to operators and is explicitly
   priced.
3. **Nostr relays are the entire off-chain infrastructure.** Discovery, signing
   ceremonies, payment delivery, encrypted state backup, and watchtower coordination
   are all Nostr events over ordinary, untrusted public relays. There is no operator
   API. Wallets are recoverable from seed plus public pool data alone.

Everything runs on today's Bitcoin (Taproot, MuSig2/FROST, CLTV/CSV, pre-signed
transaction trees, TRUC/P2A). No covenant soft fork is assumed anywhere; unilateral
on-chain exit is always available.

The design is deliberately honest about its costs: operators can see the payment
graph (privacy is below Lightning's), every refresh carries a 0.5–0.9% capital fee
(the price of a ~58-day operator capital lock), freshly received funds are
sender-revocable until their first checkpoint, and a watchtower market is
load-bearing for offline users. The open-problems list is part of the spec, not an
afterthought.

## Reading order

| Document | Contents |
|---|---|
| [whitepaper.md](whitepaper.md) | The design: motivation, goals, architecture, on-chain construction, payment flows, trust model, economics, comparison, open problems |
| [specs/event-kinds.md](specs/event-kinds.md) | NIP-style spec of every Nostr event kind in the protocol |
| [specs/onchain.md](specs/onchain.md) | Tapscripts, transaction templates, the claim ladder, recovery tapleaf, exit packages, timelock ledger |
| [specs/ceremonies.md](specs/ceremonies.md) | Signing ceremonies, nonce discipline, arkoor transfers, watchtower protocol, recovery from seed |
| [analysis/attacks.md](analysis/attacks.md) | The full 42-finding adversarial attack matrix with dispositions |
| [analysis/economics.md](analysis/economics.md) | Operator model, user costs, exit and mass-exit economics |
| [analysis/comparison.md](analysis/comparison.md) | Flock vs Lightning, Ark, Spark, Cashu/Fedimint — including when *not* to use Flock |
| [research/sources.md](research/sources.md) | Research digests with citations, plus the raw design-contest artifacts |

## How this design was produced

The architecture is the survivor of an adversarial process (artifacts in
[research/raw/](research/raw/)): five research deep-dives, three competing
architectures scored by a three-lens judge panel, a 42-finding attack panel against
the winner, a consolidated synthesis, and a 51-finding documentation review whose
adjudicated resolutions are recorded in
[research/AMENDMENTS-v0.3.md](research/AMENDMENTS-v0.3.md). Canonical protocol
constants live in [research/ERRATA.md](research/ERRATA.md).

## Status and next steps

This is a design document set, not audited cryptography and not running code.
The marked open problems — operator payment-graph visibility, uncheckpointed-receipt
data availability, the watchtower market, mempool replacement-cycling — are genuine
and unresolved. Next steps would be: external review, a transaction-level testnet
prototype of one cell ceremony, and measured latency numbers for relay-carried
signing rounds.
