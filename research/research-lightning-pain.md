# Lightning Network: Structural Problems & Mitigations (as of June 2026)

## 1. Inbound liquidity

- **Mechanics**: a freshly opened channel puts 100% of funds on the opener's side; you can send but cannot receive a single sat until a counterparty commits capital toward you (open a channel to you, or you spend first). Receiving capacity is someone else's locked bitcoin ([lightning.engineering](https://docs.lightning.engineering/the-lightning-network/liquidity/how-to-get-inbound-capacity-on-the-lightning-network), [bitcoin.design](https://bitcoin.design/guide/how-it-works/liquidity/)).
- **Why it persists**: it is architectural, not a bug — capital must be pre-allocated per-edge, so every new receiver faces a cold-start, and every received payment depletes the same inbound it used. Rebalancing is a continuous, paid, operational task ([Plaitr 2026 reality check](https://www.plaitr.com/blog/lightning-network-for-merchant-payments-a-2026-reality-check), [Spark](https://www.spark.money/research/lightning-network-liquidity-explained)).
- **Mitigations** (all cost fees or trust): buying inbound (Lightning Pool, Magma, LSP channel-opens), submarine swaps (Loop/Boltz), JIT channels. None remove the underlying capital-lockup requirement.

## 2. LSP specs, liquidity ads, splicing — actual deployment

- **LSPS0/1/2** (transport, channel purchase, JIT channels) were finalized as **bLIP-50/51/52**; the standalone LSP-spec repo was archived Jan 2025 in favor of the bLIP process ([github.com/BitcoinAndLightningLayerSpecs/lsp](https://github.com/BitcoinAndLightningLayerSpecs/lsp)). Core Lightning shipped early LSPS support in v25.05 (June 2025) ([Blockstream Q2 2025](https://blog.blockstream.com/blockstream-quarterly-update-q2-2025/)); LDK Node and a handful of LSPs (Breez, Olympus) implement bLIP-52. Real, but a thin market — Phoenix notably uses ACINQ's own protocol, not LSPS.
- **Liquidity ads**: experimental in Core Lightning since ~2021 (older material); Eclair v0.11 implemented liquidity ads + on-the-fly funding, in production in Phoenix ([nobsbitcoin](https://www.nobsbitcoin.com/eclair-v0-11-0/)). A cross-implementation open liquidity market never materialized; it functions mainly as ACINQ↔Phoenix plumbing.
- **Splicing**: Eclair shipped it on public channels (Feb 2025) and taproot channels (Aug 2025); CLN finalized interop in v25.05; LDK 0.2 added experimental splicing (Dec 2025). The **spec PR remained unmerged at end-2025** and **LND still lacks splicing entirely** ([Optech 2025 year-in-review](https://bitcoinops.org/en/newsletters/2025/12/19/), [Optech splicing topic](https://bitcoinops.org/en/topics/splicing/)).

## 3. Always-online requirement

- **Root cause**: receiving requires the recipient online to provide an invoice/preimage and co-sign the new commitment state; mobile OSes suspend background apps.
- **Watchtowers** guard against *breach* while offline, but do **not** enable receiving; third-party watchtower services remain sparse with no sustainable fee model (claims rest largely on pre-2024 material) ([CCN](https://www.ccn.com/education/crypto/watchtowers-in-bitcoin/)).
- **Async payments**: BOLT12 *static invoices* + an always-online server fetching invoices for offline receivers. LDK shipped client (#3618) and server (#3628) logic in 2025; CLN v25.12 serves/pays static invoices; the bolts spec PR ([#1149](https://github.com/lightning/bolts/pull/1149)) is still unmerged ([Optech async-payments](https://bitcoinops.org/en/topics/async-payments/)).
- **Trampoline/LSP-held HTLCs**: Eclair holds payments at a trampoline node and wakes the mobile peer via push notification (#2865) — Phoenix/Breez production behavior. This works but reintroduces trust in the LSP, fails when notifications are blocked, and risks HTLC timeouts.
- **Net**: in 2026, noncustodial mobile receive still means "your LSP holds the payment and hopes your phone wakes up."

## 4. Force-close costs, toxic state, SCB failure modes

- **Force-close cost**: unilateral close broadcasts a pre-signed commitment plus a second-stage transaction *per in-flight HTLC*; at 50 sat/vB with full 483-HTLC slots this can exceed 1M sats ([Voltage](https://voltage.cloud/blog/lightning-node-management-handling-high-on-chain-fees)). Funds are then timelocked days–weeks; fee spikes make small channels uneconomical to close at all.
- **Toxic state**: any stale copy of channel state is dangerous — broadcasting an old commitment triggers the penalty and the peer takes the entire channel. Continuous state backup is therefore unsafe by design.
- **Static channel backups** don't restore channels; they ask peers (via `option_data_loss_protect`) to force-close at the latest state. Recovery depends on the peer being **honest, reachable, and running** — funds are stranded if the peer is gone — and takes 1–2 weeks ([lnd safety.md](https://github.com/lightningnetwork/lnd/blob/master/docs/safety.md), [lightning.engineering disaster recovery](https://docs.lightning.engineering/lightning-network-tools/lnd/disaster-recovery)).

## 5. Jamming and griefing

- **Channel jamming**: attackers occupy HTLC slots (483 max) or liquidity with payments they never settle; failed HTLCs cost nothing. The **hybrid mitigation** (local reputation + accountable signal, bLIP-04, plus unconditional fees) showed 2025 simulation results judged "sufficient for implementation"; LDK runs a read-only logging deployment — but it is **not yet active network-wide** ([bolts #1218](https://github.com/lightning/bolts/issues/1218), [Delving Bitcoin](https://delvingbitcoin.org/t/hybrid-jamming-mitigation-results-and-updates/1147), [Optech](https://bitcoinops.org/en/topics/channel-jamming-attacks/)).
- **Replacement cycling** (Riard, 2023): mempool-replacement games to steal in-flight HTLCs. Implementations shipped partial mitigations (aggressive rebroadcast/fee-bumping); a full fix needs script/consensus changes; no observed in-the-wild exploitation (status rests on 2023–24 reporting) ([Bitcoin Magazine postmortem](https://bitcoinmagazine.com/technical/postmortem-on-the-lightning-replacement-cycling-attack)).
- Also: balance probing, dust-HTLC griefing, mass-exit/flood-and-loot scenarios under fee spikes ([arXiv mass exit](https://arxiv.org/pdf/2208.01908)).

## 6. Centralization & network trends 2024–2026

- **Public capacity**: ~5,400 BTC (late 2023) → ~3,850–4,200 BTC (Aug 2025), a 20–30% decline; channels ~41,700 (Aug 2025) vs >80,000 peak (mid-2023); ~16,000 nodes early 2025, declining since 2022 ([CryptoSlate](https://cryptoslate.com/why-lightning-network-capacity-declining-20-in-2025-is-not-as-bad-as-it-sounds/), [Bitcoin.com News](https://news.bitcoin.com/data-shows-sustained-slide-in-lightning-network-capacity-channels-through-2025/), [Bitcoin Visuals](https://bitcoinvisuals.com/lightning)).
- **Concentration**: top-10 operators held ~2,389 BTC ≈ **62% of public capacity** (Aug 2025); explicit hub-and-spoke topology around exchanges (Bitfinex, Kraken) and LSPs; node hosting concentrated on AWS/GCP/Hetzner ([Bitcoin.com News](https://news.bitcoin.com/data-shows-sustained-slide-in-lightning-network-capacity-channels-through-2025/), [VU centrality study, 2018–2025 snapshots](https://epublications.vu.lt/object/elaba:252675111/252675111.pdf)).
- **Counterpoint**: usage grew — Coinbase routed ~15% of BTC withdrawals via LN (mid-2025); CoinGate LN share >16% (2024); claimed 266% YoY public-volume growth (single-source, lower confidence) ([CryptoSlate](https://cryptoslate.com/why-lightning-network-capacity-declining-20-in-2025-is-not-as-bad-as-it-sounds/), [AInvest](https://www.ainvest.com/news/lightning-network-volume-surges-flow-analysis-igaming-payouts-2604/)). Much activity moved to custodial/private channels, invisible to public metrics — capacity decline ≠ usage decline, but it does mean the *noncustodial edge* is shrinking.

---

## (a) Failure modes a successor MUST solve

- [ ] Receive without pre-provisioned per-edge inbound capital or third-party channel opens
- [ ] Fully offline/async receive without an LSP holding funds or trusting push notifications
- [ ] No toxic state: stale backups must be safe; recovery must not depend on a counterparty's honesty/liveness
- [ ] Unilateral exit cost bounded and independent of in-flight payment count and fee spikes
- [ ] No free resource-exhaustion (jamming) — attackers must pay for held liquidity/slots by construction
- [ ] No mempool-games on safety (replacement-cycling-class attacks)
- [ ] Self-custody viable on mobile without consolidating onto ~10 hubs holding 60%+ of capacity
- [ ] Capital efficiency: liquidity reusable across counterparties, not locked per-channel

## (b) Lightning strengths a successor must NOT regress

- [ ] Sub-second, irreversible settlement finality for the payee
- [ ] Sub-cent fees at small amounts; fees independent of L1 congestion for routine payments
- [ ] Self-custodial option with unilateral exit to L1 (no permissioned operator can freeze funds)
- [ ] BOLT11/BOLT12/LNURL/BIP-353 interop — accepted across exchanges (Coinbase, Kraken, Cash App), wallets, and merchant processors
- [ ] Network effects: existing routing graph, LSP tooling (bLIP-50/51/52), and the four-implementation ecosystem (LND/CLN/Eclair/LDK)
- [ ] Payer/payee privacy at least at Lightning's level (onion routing, blinded paths)
- [ ] Amount granularity to ~msat and proof-of-payment (preimage/PoP)
