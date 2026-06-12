# Flock: A Bitcoin Layer 2 on the Nostr Relay Network

### Payments without channels, liquidity management, or single points of failure

Flock protocol - DRAFT v0.3 - 2026-06-12

## Abstract

Flock is a Bitcoin Layer-2 payment protocol designed to succeed Lightning. It runs covenant-less Ark pools operated by a 4-of-7 FROST quorum, with all off-chain coordination, payment messaging, encrypted state backup, and watchtower traffic carried as Nostr events over untrusted public relays. Users receive instantly to an npub with zero on-chain setup, no channels, and no liquidity management; the capital burden moves to operators and is explicitly priced. Every quorum-gated output uses a tapscript requiring two independent BIP-340 signatures, so the quorum signs only plain, shipped FROST-Schnorr and never participates in MuSig2. Refresh atomicity is enforced by Ark-style connector outputs; stale-exit defense by a decrementing relative-timelock claim ladder that is valid even under a dead quorum; and dormant funds degrade to tower-protected self-custody through a rotating CLTV recovery tapleaf plus standing watchtower deposits, enabling recovery from the seed plus surviving public round data. The protocol requires no soft fork. Residual trust is explicit and bounded: uncheckpointed out-of-round receipts are trusted (sender plus 4-of-7 collusion once accepted; sender alone against a receiver who stays offline past the claim-ladder differential with no tower acting) until the next checkpoint — a round refresh, available hourly; the quorum's per-cosign transfer log makes equivocation provable in the interim. Exposure is capped by a wallet value-at-risk policy on the aggregate uncheckpointed balance, enforceable only while the wallet is online; offline exposure is bounded by the acceptance rule and an optional quorum-side uncheckpointed ceiling. Checkpointed funds are theft-proof while the owner or its towers act within stated deadlines, with unilateral exit always available. Open problems — operator payment-graph visibility, data availability of uncheckpointed receipts, mempool replacement cycling, and a structural carrying cost on hot balances — are conceded, not glossed.

## 1. Introduction and motivation

Lightning proved that off-chain bitcoin payments work: sub-second finality, sub-cent fees, self-custody with unilateral exit, and broad ecosystem adoption. A decade in, it has also demonstrated two failures that are structural — properties of the channel construction itself, not of any implementation — and that a successor must remove rather than mitigate.

### 1.1 Failure one: per-edge liquidity management

A Lightning channel pre-allocates capital to one edge of a graph. A freshly opened channel puts 100% of its funds on the opener's side: the opener can send but cannot receive a single satoshi until a counterparty commits capital toward them [1][2]. Receiving capacity is, by construction, someone else's locked bitcoin. This is not a bug to be engineered away; it is the architecture. Every new receiver faces a cold-start problem, every received payment depletes the inbound capacity it used, and rebalancing is a continuous, paid, operational task [3][4].

A mitigation industry exists, and its shape is the diagnosis. Inbound can be bought (Lightning Pool, Magma, LSP channel-opens), swapped (Loop, Boltz), or provisioned just-in-time by an LSP. The LSP specifications LSPS0/1/2 were finalized as bLIP-50/51/52 and the standalone spec repository was archived in January 2025 [5]; implementations exist in Core Lightning, LDK Node, Breez, and Olympus — but the market is thin, and the largest mobile wallet (Phoenix) uses ACINQ's own protocol instead. Liquidity ads never produced a cross-implementation open market; in production they function mainly as ACINQ-to-Phoenix plumbing [6]. Splicing — the ability to resize channels without closing them — shipped in Eclair and CLN, but the spec PR remained unmerged at end-2025 and LND still lacks it entirely [6][7]. None of these efforts remove the underlying requirement: capital locked per edge, managed forever.

### 1.2 Failure two: single-node points of failure

Every Lightning payment and every Lightning balance depends on the health, honesty, and liveness of specific named nodes.

- **Receiving requires being online.** The recipient must provide an invoice and co-sign the new commitment state; mobile operating systems suspend background apps. The deployed workaround is an LSP that holds the in-flight HTLC and wakes the phone with a push notification — production behavior in Phoenix and Breez. This reintroduces trust in the LSP, fails when notifications are blocked, and risks HTLC timeouts. BOLT12 async payments remain an unmerged spec PR [8][9]; watchtowers guard against breach but do not enable receiving, and third-party watchtower services remain sparse with no sustainable fee model [10]. As of 2026, noncustodial mobile receive still means "your LSP holds the payment and hopes your phone wakes up."
- **Channel state is toxic.** Broadcasting a stale commitment costs the entire channel via the penalty mechanism, so continuous state backup is unsafe by design. Static channel backups do not restore channels; they ask the peer to force-close at the latest state, so recovery depends on that peer being honest, reachable, and still running — funds are stranded if the peer is gone [12][13].
- **Exit cost scales with in-flight state.** A unilateral close broadcasts a pre-signed commitment plus a second-stage transaction per in-flight HTLC; at 50 sat/vB with full HTLC slots this can exceed one million sats, and fee spikes make small channels uneconomical to close at all [11]. (For fairness: a Flock worst-case first exit at the same 50 sat/vB is ~80k sats — the same-feerate comparison is tabulated in §5.6. The structural point is that Flock's worst case is bounded by tree depth while Lightning's scales with in-flight HTLC count.)
- **The graph has consolidated into hubs.** Public capacity fell from roughly 5,400 BTC in late 2023 to roughly 3,850–4,200 BTC by August 2025 (a 20–30% decline); channel count roughly halved from its 2023 peak; node count has declined since 2022 [17][18]. The top ten operators held about 62% of public capacity in August 2025, in an explicit hub-and-spoke topology around exchanges and LSPs, with node hosting concentrated on a handful of cloud providers [18]. Usage grew over the same period — Coinbase routed ~15% of BTC withdrawals over Lightning in mid-2025 [17] — but much of it moved to custodial and private channels. Capacity decline is not usage decline, but it does mean the *noncustodial* edge of the network is shrinking: ordinary users increasingly interact with Lightning through a single custodian or a single LSP, which is precisely the single point of failure the system was built to avoid.

Add the open attack surface — channel jamming (mitigation simulated but not active network-wide [14]), replacement cycling against in-flight HTLCs (partial mitigations only; a full fix needs consensus changes [15]), and mass-exit dynamics under fee spikes [16] — and the conclusion is that Lightning's pain is concentrated exactly where its architecture concentrates obligations: on individual nodes and individual edges.

### 1.3 What a successor must do

From the failure analysis, a successor must: let anyone receive without pre-provisioned inbound capital or third-party channel opens; support fully offline receive without an LSP holding funds; eliminate toxic state (stale backups must be safe, and recovery must not depend on a counterparty's honesty or liveness); bound unilateral exit cost independently of in-flight payment count; make resource-exhaustion attacks cost the attacker something by construction; keep safety off the mempool-games attack surface as far as possible; and remain self-custodial on mobile without consolidating onto a few hubs.

It must not regress what Lightning got right: sub-second irreversible finality for the payee; fees independent of L1 congestion for routine payments; a self-custodial option with unilateral exit that no operator can freeze; interoperability with the deployed Lightning ecosystem (BOLT11/BOLT12); and proof of payment.

One bar on that list is consciously missed, and it should be said plainly: Flock's arkoor finality is weaker than Lightning's payee finality. An arkoor receipt is fast (a ~1–2 s target, pending measurement over relay+Tor paths) but remains revocable — by sender plus 4-of-7 collusion once accepted, and by the sender alone against a receiver who stays offline past the claim-ladder differential with no tower acting — until the next checkpoint extinguishes the trust (§8.2). The design accepts this regression because it is exactly what buys the two properties Lightning structurally cannot provide: offline receive with zero receiver liveness and zero inbound liquidity. Merchants who need Lightning-grade finality must require checkpoint inclusion or verify online before treating a payment as accepted (§7.2).

Flock's thesis is that both structural failures share one root — obligations pinned to specific nodes and edges — and that the fix is to move the obligations to (a) a threshold quorum that cannot unilaterally steal, and (b) a commodity broadcast layer that cannot equivocate undetected. Covenant-less Ark pools supply the first; the public Nostr relay network supplies the second. Neither piece is exotic: Ark pools run on Bitcoin mainnet today [26], and every coordination primitive Flock needs — encrypted RPC over relays, threshold signing over relays, bearer-asset data availability on relays — has shipped in production Nostr systems [37][38][40]. Flock's contribution is the composition, and the removal of every mechanism in that composition that required unproven cryptography or unbounded trust.

## 2. Design goals and non-goals

### 2.1 Requirements

**R1 — Sustainable economics.** Operators must earn a margin above cost of capital without custody or token games, and the user-side fee burden must be competitive with Lightning's channel and rebalancing amortization. Existing Lightning LSPs must be able to migrate to operating pools with the same balance sheet and better capital efficiency (no per-edge lockup).

**R2 — Nostr-only infrastructure, relays untrusted.** All off-chain traffic — coordination ceremonies, payment delivery, state backup, watchtower signaling, discovery — is signed Nostr events on public relays. No proprietary API, no operator-hosted endpoint, no HTTP service may be load-bearing for safety. Relays are assumed to drop, delay, reorder, and censor; the protocol must fail closed when they do. Scope, stated honestly: no operator endpoint is load-bearing for *checkpointed* funds; uncheckpointed receipts additionally depend on relay retention — including the operators' paid archival relays, which speak plain NIP-01 with long retention and are listed in the pool advertisement (§11, problem 2).

**R3 — No single point of failure.** No single operator, relay, or server whose failure halts payments or endangers funds. Operator failure below the liveness threshold must change nothing; failure beyond it must degrade to exit-only mode, never to loss.

**R4 — Zero user liquidity management.** No inbound capacity, no channels, no rebalancing, no splicing, nothing to manage. The capital burden moves to operators and is explicitly priced.

**R5 — Today's Bitcoin only.** No CTV, CSFS, or any other soft fork is assumed anywhere. Mempool policy reliance is limited to what Bitcoin Core ships today (TRUC/v3 transactions, P2A anchors, 1-parent-1-child package relay) [51].

**R6 — Trust-minimized unilateral exit.** Every user holds a complete pre-signed exit package at all times. Exit requires no operator cooperation, survives a dead quorum, and is hardened against pinning.

**R7 — npub-addressed payments.** A recipient needs only a Nostr keypair: zero on-chain setup, zero prior footprint, receiving while offline, including the very first payment they ever receive. Discovery is specified, not assumed: a receiver may publish a kind-38407 payment descriptor (stealth base key, preferred pools, relays); when neither a 38407 nor a kind-10002 relay list exists, senders set the stealth base key to the receiver's Nostr pubkey and publish to the pool relay set (mandatory in all cases). First-contact scan cost grows with pool count — a conceded trade-off.

**R8 — Lightning interoperability.** Pay any BOLT11/BOLT12 invoice and be payable from any Lightning wallet from day one, via swap gateways. Zap (NIP-57) compatibility so existing Nostr clients work unchanged.

### 2.2 Non-goals

- **Privacy better than Lightning.** Conceded: the quorum sees amounts and payment links at cosign time (§11, problem 1). Flock's privacy target is "no worse than a single Ark or Spark operator, and nothing sensitive in cleartext on relays" — below Lightning's onion-routed payer privacy.
- **Sub-dust micropayments as on-tree outputs.** The minimum leaf denomination is 10,000 sats (≈3× the ~310 vB marginal exit cost at 10 sat/vB — and underwater at 50 sat/vB; §5.6, §11 problem 4); smaller value rides as uncheckpointed change with bounded trust. Streaming-sats use cases belong on ecash or aggregated layers above Flock.
- **Covenant dependence.** Designs that become dramatically simpler under CTV/CSFS are catalogued as future work (§12) and deliberately excluded from the normative protocol.

## 3. Related work

### 3.1 Ark and clArk

Ark batches users into periodic on-chain rounds funding pre-signed vTXO trees; payments can occur out-of-round ("arkoor") by having sender and operator cosign a new transaction off an existing leaf, instantly and without consuming operator liquidity [19][23]. Covenant-less Ark (clArk) replaces the covenant with an n-of-n pre-signed pseudo-covenant: the operator plus round participants MuSig2-cosign every tree transaction and then delete their keys; security holds if at least one cosigner deletes [21]. Two production systems reached mainnet: Second's Bark (June 2026) and Ark Labs' Arkade (October 2025) [26][20].

Ark gets the core economics right — receive with zero setup, capital fronted by the operator and recycled at expiry, exit always available. Its open problems are equally clear: a single operator is a SPOF for rounds, cosigning, and liquidity [27]; clArk rounds fail and restart if any cosigner stalls, which is why rounds are restricted to self-refreshes [22]; users who miss every round for ~30 days can be swept by the operator [20]; and arkoor receivers trust sender-plus-operator non-collusion until they refresh [23].

**What Flock inherits from Ark:** vTXOs, hourly rounds funding pre-signed radix-4 trees, ~30-day expiry with refresh, arkoor out-of-round payments, connector-based forfeit atomicity [24], boarding outputs, and the rounds-are-refreshes-only restriction that kills the receiver-DoS problem.

**What Flock changes:** the single operator becomes a 4-of-7 FROST quorum that signs only plain Schnorr (never MuSig2); the operator's gRPC API becomes public Nostr relays end to end; the expiry sweep stops being confiscatory in all but the worst case (a rotating recovery tapleaf plus standing tower deposits return dormant funds to self-custody before the quorum's sweep matures; confiscation requires owner and all towers absent for ~26 days past recovery maturity, §5.5); stale-exit defense becomes a consensus-enforced decrementing claim ladder instead of a reactive contest window; Bark's hash-lock refresh atomicity [25] is rejected in favor of connectors (§5.3); one big round ceremony becomes 16 independently funded 64-leaf cells, one cell transaction each; arkoor receipts gain a quorum-signed per-cosign Merkle transfer log with fraud-provable equivocation; and offline users are protected by sender- and quorum-funded, conflict-triggered watchtower height-deposits rather than trust-downgraded delegation.

### 3.2 Statechains and Spark

Statechains transfer a UTXO off-chain by handing over a key, co-signed by an operator that promises to forget the previous owner's share [29]. Mercury Layer shipped blind co-signing and decrementing-timelock exits, then died: an independent audit found two instant-theft vulnerabilities, and CommerceBlock shut down in 2025 [30]. Spark (Lightspark) revived the construction as a t-of-n FROST federation with additive key tweaks, leaf denominations, and offline Lightning receive via operator-held preimage shards [31][36]. Spark demonstrates real demand for "receive to a key, no channels" — and also the cost of unverifiable promises: key-share deletion cannot be proven, so its "non-custodial" label is contested [32][33][34]; as of 2025–26 it ran with only two operators, both of whom see all transactions [35].

Flock takes from this family the decrementing-timelock exit ordering (as the claim ladder) and the threshold-federation framing, but rejects the key-deletion trust root for *transfers*: a Flock arkoor transfer is a new pre-signed Bitcoin transaction chain, not a key handover, and after checkpointing, theft requires forging a Bitcoin signature, not merely remembering a share. The only deletion assumption in Flock is the clArk one — each user deletes their *own* ephemeral tree key, so each user's safety rests on their own deletion, not an operator's.

### 3.3 Cashu and Fedimint

Chaumian ecash mints offer the best privacy (blind signatures) and the best UX floor (hold for free, receive offline) at the price of full custody: no unilateral exit, no proof of reserves that prevents a mint from refusing redemption. Fedimint federates the custodian but does not change the category. Flock's economics must compete with "free" custody while offering what mints cannot: consensus-enforced exit. Flock's dormancy treatment (§5.5) — no carrying fee, never confiscated while towers function, but one automatic on-chain exit to self-custody ~32 days after the last checkpoint, at the owner's expense — exists because the contest's economics judge found that a protocol charging idle users carrying fees loses to Cashu for ordinary users. The honest residue: Flock still exits the absentee on-chain (it does not confiscate), so for hold-forever-free UX the mint keeps the edge; savings-profile pools can stretch expiry and `sweep_delay` to lengthen the dormancy horizon.

### 3.4 Lightning LSPs

The LSP ecosystem (bLIP-50/51/52 [5]) is prior art for the *business* of fronting capital to payment users. Flock's R1 explicitly frames pool operation as the LSP pivot: the same balance sheet that today funds JIT channels can fund pool rounds, with no per-edge lockup and no path liquidity risk. What LSPs cannot do inside Lightning — serve offline receivers noncustodially — is exactly what the quorum-plus-relays construction provides.

### 3.5 Nostr prior art

Every building block Flock needs has shipped on relays, separately:

- **NIP-47 Nostr Wallet Connect** — encrypted request/response RPC over relays controlling real money, the de-facto standard for app-wallet links [37]. Proof that interactive payment protocols survive relay transport.
- **NIP-57 zaps** — Lightning receipts as signed events; ubiquitous since 2023. Flock emits kind-9735 receipts for interop.
- **NIP-60/61 Cashu wallets and nutzaps** — wallet state and bearer tokens *stored on relays*, accepting that losing events means losing money [38][39]. The strongest precedent for relays as a bearer-asset data-availability layer; Flock hardens the same idea with read-back replication verification, operator-run archival relays, and an on-chain recovery fallback.
- **FROSTR/bifrost** — t-of-n FROST threshold Schnorr with relays as the signing-round transport, producing standard BIP-340 signatures; actively developed and funded [40][41]. This is the shipped, FROST-only construction Flock's quorum runs — and deliberately the *only* threshold cryptography Flock uses (§5.1).
- **joinstr, Mostro, NIP-88 DLCs** — coinjoin coordination, non-custodial exchange escrow, and contract negotiation over relays [42][43]: multi-party adversarial ceremonies over lossy transport are practical with session discipline and paid/private relays.
- **OpenArk / "NIP-150"** — an early community spec for Ark-over-Nostr-transport, published with no live implementation [44]. Conceptual prior art: it standardizes the transport for a *single-operator* Ark, while Flock removes the single operator and makes relays carry safety-critical state under explicit adversarial assumptions.

Relay-network measurements ground R2's feasibility: posts replicate on ~35 relays on average and >90% remain retrievable with the top-30 relays down [45] — but no relay guarantees retention, most run at a loss [46], and writes are silently dropped under spam pressure. Flock therefore treats relays as lossy broadcast plus *auditable* storage (read-back verification over disjoint connections, fail-closed reads) with the L1 recovery tapleaf as the no-data fallback.

## 4. Architecture

### 4.1 Roles

- **Operators O1..O7** — independent organizations on diverse ASNs and jurisdictions, reachable as hidden services. They hold FROST shares of the group key **Q** with threshold **k=4 of n=7**: theft requires 4 colluding operators; liveness survives 3 simultaneous failures. (Why 4-of-7: theft needs a majority of a jurisdiction-diverse 7-member set while tolerating 3 outages; a wider quorum slows every FROST cosign over Tor hidden services.) Key generation is a PedPoP DKG at pool genesis, with monthly proactive resharing over signed, epoch-tagged transcripts. Operators publish per-operator heartbeats and signed key-deletion attestations (kind 31401). These attestations carry **reputational and legal weight only**: FROST signatures do not attribute which share subset signed and resharing leaves Q unchanged, so share retention remains cryptographically unverifiable — the same standard this paper applies to Spark (§3.2, §11 problem 6). "Provable fraud" in Flock is reserved for what is genuinely cryptographic: transfer-log equivocation (two quorum-signed roots at one sequence) and nullifier collisions (§6.1).
- **Relays** — at least 8 per pool, of which at least 3 must be operator-independent (sized against measured relay churn [45] so disjoint read sets stay available, and so the quorum can never control a user's entire relay view). They are untrusted transport and best-effort storage: they cannot forge events (everything is signed), but they can drop, delay, reorder, and censor. Operators additionally run paid **archival relays** — plain NIP-01, long retention, listed in the pool advertisement — and clients carry a hard-coded, diverse bootstrap relay set.
- **Wallets** — hold a single seed, from which both the Nostr key and a Bitcoin xprv derive. Otherwise stateless: all state is reconstructible from relays (including the archival relays), or — worst case — from the seed plus any surviving copy of the public round data (§7.5). Wallets enforce finality and value-at-risk policies client-side; the protocol never asks a user to trust a number the wallet did not verify.
- **Watchtowers** — permissionless and keyless. They execute *height-deposits* that are both height- and conflict-triggered: "broadcast pre-signed package P at height H unless superseded, or immediately upon any conflicting spend of P's ancestor outpoints" — and fee-bump anyone's exit via P2A anchors. They never hold user keys and cannot redirect funds; the worst a malicious tower can do is fail to act, which is why deposits are redundant and sharded, and why tower payment is contingent: every deposited package carries pre-signed **escalating bump children** (canonical ladder 10 / 50 / 200 sat/vB, spending the owner's dedicated fee UTXOs), each variant carrying a ~2,000-sat fee output collected only by a tower that actually acts — the tower selects the lowest variant that clears, so its out-of-pocket cost is ≈0 (§7.1).
- **LN gateways** — vHTLC/PTLC swap providers bridging Flock vTXOs to BOLT11/BOLT12 Lightning in both directions. Existing LSPs can pivot here with the same capital and no per-edge lockup.

```
                          BITCOIN L1
   cell txs (≤16/round, hourly: root + connectors each)  forfeits/claims/sweeps
            ▲                                   ▲
            │   FROST 4-of-7 quorum Q — plain BIP-340 sigs only
            └───────────────┬───────────────────┘
                            │ ALL traffic = Nostr events
 ┌──────────────────────────┴───────────────────────────────────┐
 │  PUBLIC NOSTR RELAYS (untrusted; ≥8/pool, ≥3 non-operator)   │
 │  pool ads · cell ceremonies · arkoor gift wraps · Merkle     │
 │  transfer log · vaults · height-deposits · service offers    │
 └──┬───────────────┬──────────────────┬───────────────┬────────┘
  Alice           Bob (OFFLINE        Watchtowers     LN Gateways
  (sender)        receiver: npub only) (keyless CPFP + (vHTLC ↔ BOLT11/12)
                                       height-deposits)
```

### 4.2 Pools, rounds, cells

A **pool** is one quorum's recycling capital base plus its relay set, advertised in a quorum-signed kind-38400 event. Every hour, the quorum runs a **round**: a batch of up to **16 independent on-chain cell transactions**, each funding one **64-leaf pre-signed tree** — a "cell", radix-4, depth 3. Each cell tx's inputs are quorum capital plus that cell's boarding outputs; its outputs are one cell root, up to 64 dust-sized **per-user connector outputs**, and quorum change. Each cell's leaves are vTXOs expiring at **T_exp = cell-tx height + 4320 blocks (~30 days)**. Cell-tx fee-bumping is **CPFP-only** (spending Q's change); RBF is forbidden, because any txid change voids that cell's pre-signed graph.

Cells are the liveness unit. clArk's single big ceremony fails entirely if any one cosigner stalls [22]; the contest's judges flagged a 1,024-cosigner hourly ceremony over lossy relays as a load-bearing liveness assumption, and the v0.2 review showed that cells sharing one funding transaction made cell isolation fictional — any rebuild changed the shared txid and invalidated every other cell's pre-signed graph. In v0.3 each cell has its own funding transaction, so one aborter delays one 64-participant cell by one rebuild attempt and a rebuild re-signs that one cell tx only, leaving the other cells untouched: participant sets are finalized *after* the nonce round (non-responders are excluded and the cell rebuilt within the same attempt), intents carry forfeitable bonds (1,000 sats canonical default, §8.2) so aborting after nonces costs money, and ceremony phases run on 120-second adaptive timers (sized for Tor hidden-service round trips with margin).

Rounds serve **only refreshes and boarding** — every round participant is paying themselves — so the clArk receiver-DoS problem (a receiver with nothing at stake stalling everyone [22]) cannot arise.

### 4.3 Arkoor payments

Routine payments do not wait for a round. An **arkoor** (out-of-round) payment is a pre-signed transaction spending the sender's vTXO leaf to the receiver, cosigned by the sender (cooperative tapscript path) and by Q (plain FROST) over a gift-wrapped kind-24405 request/response exchange — there is no MuSig2 between sender and quorum. It is fast (a ~1–2 s target, pending measurement over relay+Tor paths), requires no receiver liveness, and consumes no operator capital. The receiver gets a package containing the full transaction chain back to a confirmed cell transaction, the quorum's cosignature, a Q-cosigned *claim* transaction slotted into the claim ladder (§5.4), and a Merkle inclusion proof against the quorum's per-cosign transfer log (§6.1). An uncheckpointed receipt is not **accepted** until the receiver's fail-closed verification passes (§7.2); a sender clawing back value before acceptance is a failed payment, not theft of accepted funds. After acceptance and until the receiver checkpoints (participates in a round, or is auto-checkpointed once the aggregate uncheckpointed balance crosses the value-at-risk cap — enforceable only while online, §8.2), the funds are trusted in a precisely bounded sense: stealing them requires the *sender and 4 of 7 operators* to collude, or the sender alone if the receiver stays offline past the ladder differential and no tower acts (§8.1, §8.2).

The velocity bound is explicit: arkoor chains are capped at **4 hops**, and each checkpoint resets the chain. A hop-4 holder cannot re-spend until they checkpoint at the next hourly round — so high-velocity users and merchants *do* periodically wait for rounds, and pay the refresh fee when they do. Round checkpoint capacity is 16 cells × 64 leaves = **1,024 leaves per hour per pool** (~2.4 checkpoint opportunities per user per day in a 10k-user pool). If auto-checkpoint demand (VaR-cap crossings plus hop-4 resets) exceeds that, refreshes queue to later rounds. For deadline-driven refreshes this is a throughput ceiling, not a safety failure — deadlines are measured in weeks — but checkpoint-queue congestion proportionally extends above-cap residual-trust exposure, which is why VaR-triggered refreshes take priority in Q's seat assignment (§7.2, §7.3).

## 5. On-chain construction

This section summarizes the transaction model; the normative script-level detail, transaction templates, and weight accounting live in [specs/onchain.md](specs/onchain.md).

### 5.1 Two-signature tapscripts — and why

Every quorum-gated spend in Flock uses a tapscript requiring **two independent BIP-340 signatures**:

```
<X> OP_CHECKSIGVERIFY <Q> OP_CHECKSIG
```

where `X` is a user key or a MuSig2 aggregate of a user cohort, and `Q` is the FROST group key. Users and cohorts sign standard MuSig2 (BIP-327 [47]) or single-key Schnorr; Q signs the same digest independently via plain FROST (RFC 9591 class, as shipped by ZF FROST and FROSTR/bifrost [49][41]). There is **no nonce coupling between the two protocols anywhere in the system**.

This is the single most consequential change from the contest-winning design, and it exists because of a fatal flaw the judges found: the original design made the FROST group act as *one MuSig2 cosigner*, requiring threshold generation of MuSig2's two-point nonces and binding-factor partials — a composition with no proven, shipped construction; the adversarial panel further showed the composition can leak the group key under a split relay view. If that construction failed, the entire pool's funds were compromised at once. v0.2 removed the composition rather than auditing it: Q now signs only the plain FROST-Schnorr that FROSTR-class software already produces over relays, and MuSig2 runs only among user cohorts. The price is a script-path spend instead of a keyspend — roughly +70 vB of witness per input — paid on exits and forfeits only. Trading dozens of vbytes for the removal of an unproven cryptographic assumption from under the whole pool is the kind of trade this design makes everywhere.

The companion rule is **nonce discipline**, which exists because the panel's top-ranked attack extracted private keys and FROST shares from deterministic nonce re-derivation (induce a crash, replay the session against a different nonce set, recover the key from two partials [48]). All MuSig2 and FROST round-1 nonces are CSPRNG-fresh, RAM-only, never persisted, never re-derived; one partial signature per secnonce, ever; any crash, timeout, or duplicate aborts the session, which restarts with fresh nonces; sessions are idempotent via session identifiers, never via nonce reuse. A restored or cloned device cannot reuse a nonce because nonces are never derivable. All signers cross-verify a canonical session transcript (full pubnonce set, signer set, message) before emitting partials.

### 5.2 Cell trees

Cell root and interior node outputs use a NUMS (unspendable) internal key with two tapscripts:

- **Tree path:** `<AggU_branch> CHECKSIGVERIFY <Q> CHECKSIG`, where `AggU_branch` is the MuSig2 aggregate of the users below that branch. Each user cosigns only the transactions on their own root-to-leaf path (depth 3), then deletes the ephemeral signing key. Because the user is a cosigner on every node above their own leaf, *their own deletion alone* makes their path's pseudo-covenant binding — the trust root for tree integrity is each user's own behavior, not an operator promise.
- **Node sweep:** `<T_exp+4032> CLTV DROP <Q> CHECKSIG` — late garbage collection only (the sweep delay is a pool policy parameter in 38400, canonical default 4032), ~4 weeks after expiry, and strictly after every user-favoring path has matured. Note the cell *root* carries this sweep leaf too, which is what makes the dormancy race of §5.5 real.

### 5.3 Leaves, forfeits, connectors

A **vTXO leaf** output (NUMS internal key) carries three tapscripts:

1. **Cooperative:** `<U> CHECKSIGVERIFY <Q> CHECKSIG` — used for arkoor transfers, forfeits, offboards, and pre-signing claims.
2. **Recovery:** `<T_exp+288> CLTV DROP <U_ckpt> CHECKSIG` — pays the *checkpointed* owner's seed-derivable key (§5.5).
3. **Leaf sweep:** `<T_exp+4032> CLTV DROP <Q> CHECKSIG`.

There is deliberately **no bare CSV exit script**: an "owner can exit after a delay" path would let stale prior owners bypass the claim ladder's ordering.

**Refresh atomicity** is the connector construction [24]. When a user refreshes, they pre-sign (in-ceremony) a **forfeit transaction** whose inputs are (a) their old leaf via the cooperative path and (b) their per-user **connector output of the new cell transaction**. The forfeit is consensus-invalid unless the cell transaction that actually contains the user's verified new leaf confirms, because the connector outpoint commits to that exact cell txid. If the quorum mutates the cell transaction after collecting forfeits, every forfeit dies with the original txid and every old vTXO remains intact. Forfeits have nSequence 0, pay Q, and are broadcastable by **anyone** — watchtowers hold copies, which include the full ancestor arkoor chain and per-transaction CPFP children (TRUC 1p1c means the chain confirms serially). Once broadcast, a forfeit beats every laddered claim (minimum 144 blocks) even if Q is dead — but *someone must broadcast it* within the **forfeit stale window**, which is 144–720 blocks depending on the stale owner's hop (§11, problem 9). This window is distinct from the ladder defense deadline of §5.4.

This replaces the hash-lock atomicity used in Bark [25] and asserted in the original contest design. The attack panel rated the hash-lock break critical twice over: a script-level flaw in the asserted construction enabled refresh theft and mass forfeit, and even a correct hash-lock gives the operator a preimage to withhold, freezing refreshes. Connectors have neither failure mode: there is no reveal step to withhold, and the binding is transaction-graph structure that Bitcoin consensus checks. The user's residual obligation is honest L1 verification — confirming that their cell transaction's output script matches their local recomputation, after that cell's set is final, before treating the refresh as final.

**Boarding outputs** (`<U> CHECKSIGVERIFY <Q> CHECKSIG` plus a `288 CSV <U>` refund, ~2 days) bring on-chain funds into a pool; they are converted in the same round that confirms them when possible, and the short refund window bounds the stranding exposure if the quorum stalls after the deposit confirms.

### 5.4 The claim ladder

Every owner of a vTXO holds a pre-signed **claim transaction** spending the leaf's cooperative path to their own key, with a consensus-enforced BIP-68 relative timelock:

| Owner | nSequence (blocks) |
|---|---|
| Checkpointed owner (hop 0) | 720 |
| Arkoor hop 1 | 576 |
| Arkoor hop 2 | 432 |
| Arkoor hop 3 | 288 |
| Arkoor hop 4 (cap) | 144 |

Each arkoor hop decrements by 144; the chain is capped at 4 hops and the ladder resets to hop 0 at every checkpoint. Once a leaf is on-chain, the **current owner's claim matures strictly before any prior owner's** — stale-exit defense is consensus ordering, not a mempool race, and it requires no live quorum. The scope must be stated precisely: the ladder orders *maturity*; it does not extinguish stale claims. A prior owner acting alone can unroll early and simply outwait the current owner's head start (144 blocks per hop). An awake owner wins trivially; a sleeping receiver is covered only if their conflict-triggered height-deposits (§7.1 step 5, §8.2) broadcast the defending chain and claim before the attacker's claim matures — the **ladder defense deadline**, 288–720 blocks after the unroll depending on the attacking hop (the current owner's head-start differential is 144–576 blocks). This is distinct from the 144–720-block forfeit stale window of §5.3, which governs refresh-stale leaves. Q cosigns the claim atomically with each arkoor transfer; a transfer without its claim is invalid and receivers reject it.

The ladder exists because the panel proved the previous design's reactive defense fatal: a 144-block contest window in which *someone* had to broadcast a forfeit meant that a dead quorum made stale exits uncontestable, robbing every arkoor recipient. The decrementing-timelock ordering is Spark/Mercury-precedented [31][30] and was grafted from the contest's statechain design, where the judges identified it as consensus-enforced ordering that works even when every operator is gone.

### 5.5 The rotating recovery tapleaf

The recovery tapscript (`<T_exp+288> CLTV DROP <U_ckpt> CHECKSIG`) is rotated to the current owner's seed-derivable key at every checkpoint. It does two jobs:

- **Tower-protected dormancy.** A user who stops refreshing pays no carrying fee and is not confiscated while towers function. At T_exp+288 — 3,744 blocks (~26 days) *before* the quorum's earliest sweep at T_exp+4032 becomes valid — their leaf becomes spendable by a plain key their seed derives. But the sweep-vs-recovery race is real: the cell root carries its own sweep leaf, so Q can sweep a never-unrolled cell **at its root** without the leaf-level recovery tapleafs ever materializing. Dormancy is therefore protected, not unconditional. The protection is normative: at **every checkpoint**, the wallet pre-signs its recovery claim (recovery tapleaf to a seed-derived cold key) and lodges {unroll branch + recovery claim + escalating bump children (10/50/200 sat/vB, §7.1)} as a self-funded kind-38404 height-deposit with `trigger_height = T_exp+288`, replicated to at least 3 towers. The guarantee, stated exactly: *dormant funds pay no carrying fees, need no owner liveness, and are never confiscated while towers function; ~32 days after the last checkpoint (T_exp+288), the standing deposit automatically exits them to on-chain self-custody at the owner's expense — ≈3.1k–16k sats at 10 sat/vB (5× at 50) plus the ~2,000-sat contingent tower fee. Absolute worst case — the owner stays absent AND no tower acts within the 3,744-block (~26-day) window — the quorum sweep at T_exp+4032 confiscates.* The floor consequence is conceded: a dormant leaf near the 10,000-sat minimum is substantially consumed by its own exit; savings-profile pools can stretch expiry and `sweep_delay` to lengthen the dormancy horizon. This mechanism exists because the economics judge scored the original design's idle-funds story adoption-fatal: ~0.4%/month carrying cost on the whole balance plus a confiscatory or owner-hostile exit after ~25 days idle loses to every custodial alternative — v0.3 deletes the carrying fee and the confiscation, and concedes the automatic exit itself as dormancy's one residual cost — and because the v0.2 review showed the v0.2 claim "Q must unroll to sweep" was simply false.
- **Seed-plus-public-data recovery.** If every vault copy on every relay is gone, checkpointed funds remain recoverable from the **seed plus any surviving copy of the public round data** — kind-38402 round results, towers, archival relays, any cell peer: public, replicated, non-secret data, but a nonzero availability assumption (the full seed-only scan is specified in [specs/onchain.md](specs/onchain.md)). The standing deposit's unroll branch materializes the leaf, and the recovery leaf then pays U_ckpt. This is Flock's answer to Lightning's toxic-state and dead-peer recovery failures.

The conceded edges: the recovery leaf names the *checkpointed* owner, so uncheckpointed arkoor value whose holder stays offline past T_exp+288 is claimable by the prior checkpointed owner — the receive ban near expiry and the value-at-risk cap bound this window (§8.1) — and the dormancy guarantee itself now rests on the tower market (§11, problem 10).

### 5.6 Fees, pinning, and exit costs

All pre-signed transactions — tree nodes, claims, forfeits — are nVersion=3 (TRUC), zero-fee, with a P2A anchor output [50][51]; the cell transactions themselves are nVersion=2, fee-paying, bumped by CPFP on the quorum change output only (never RBF — any txid change voids the pre-signed graph, §4.2). The exiter pre-signs their own minimum-feerate CPFP child at package creation — the anchor is never a free slot for an adversary — and broadcasts 1-parent-1-child packages via `submitpackage`. CPFP is funded from dedicated UTXOs that cannot be conflicted. Anyone, including keyless watchtowers, can fee-bump via the anchor; packages deposited with towers additionally carry pre-signed escalating bump children at 10 / 50 / 200 sat/vB so an acting tower never bumps out of pocket (§7.1).

Exit cost at 10 sat/vB (script-path inputs ≈ 126 vB): a depth-3 unroll is 3 node transactions ≈ 321 vB each, plus a claim ≈ 193 vB, plus CPFP children ≈ 460 vB — **first exiter ≈ 1.6 kvB (~16k sats)**; if the path is already unrolled, **≈ 310 vB (~3.1k sats, including the exiter's own CPFP child)**. Each arkoor hop in an exit chain adds ≈ 235 vB (hops are capped at 4). A full 1,024-leaf pool exit is ~336 node transactions plus 1,024 claims, and **each TRUC parent needs its own ≈115 vB CPFP child** — batch-bumping is impossible under TRUC/ephemeral-dust rules (1p1c only) — so the total is about **0.46 blocks per pool round, ≈451 vB per user amortized**; 100k users across a pool ladder is **~46 blocks**, spread across hourly-staggered T_exp rungs with a 1008-block deadline margin. Mass exit is expensive and slow but safe: pre-signed, pinning-hardened, counterparty-free, and ladder-ordered.

Exit costs compared at the **same feerates** (typical and worst case for each system):

| Scenario | 10 sat/vB | 50 sat/vB |
|---|---|---|
| Flock, already-unrolled path (typical) | ~3.1k sats (≈310 vB) | ~15.5k sats |
| Flock, first exiter, depth-3 unroll (worst) | ~16k sats (≈1.6 kvB) | ~80k sats |
| Lightning, quiet-channel force close (typical) | ~3k sats (≈300 vB) | ~15k sats |
| Lightning, full HTLC slots (worst) | ~200k+ sats (≈20+ kvB) | >1M sats [11] |

Typical costs are comparable; the structural difference is the worst case, which in Flock is bounded by tree depth and in Lightning scales with in-flight HTLC count.

## 6. Nostr protocol surface

Every byte of off-chain Flock traffic is a signed Nostr event. The full NIP-style specification — tag schemas, validation rules, retention requirements — is [specs/event-kinds.md](specs/event-kinds.md). Kind numbers are placeholders pending NIP assignment.

### 6.1 Event kinds

| Kind | Class | Purpose |
|---|---|---|
| 38400 | addressable | Pool advertisement: Q key, operator npubs, k/n, fees, relay set, T_exp ladder, `sweep_delay`, `intent_bond_sats`, `archive_relays`, optional `uncheckpointed_ceiling` (quorum-signed) |
| 38401 | addressable | Round/cell announcement + phase deadlines |
| 31401 | addressable | Operator heartbeat + key-deletion attestation (reputational/legal weight only) |
| 24400 | ephemeral | Refresh intent (bonded, PoW-capped, gift-wrapped to Q) |
| 24401 | ephemeral | Cell proposal: cell-tx template, pre-signed txs, canonical participant-set commitment, session-id (NIP-44 to cell members) |
| 24402 | ephemeral | MuSig2 pubnonces (cohort sessions only; BIP-373 fields, session-bound) |
| 24403 | ephemeral | MuSig2 partial signatures (cohort sessions only; session-bound) |
| 24404 | ephemeral | Intra-quorum FROST / DKG / reshare traffic (epoch-tagged, ACKed) |
| 24405 | ephemeral | Arkoor cosign request/response (gift-wrapped; no MuSig2) |
| 24406 | ephemeral (inner) | Arkoor payment package, carried in a kind-1059 gift wrap |
| 38402 | addressable | Round result: all cell txids + per-cell **inline** tree commitments (external mirrors optional, never the sole copy) |
| 30178 | addressable | **Transfer log**: quorum-signed Merkle roots over (nullifier, commitment) pairs, per-cosign, strictly increasing sequence |
| 37400 | addressable | Wallet vault: self-encrypted, unlinkable author key, padded to size buckets |
| 38404 | addressable | Watchtower height-deposit: height- **and conflict-**triggered, with contingent tower fee output |
| 8405 | regular | Tower alert: conflict / sweep anomaly / broadcast done |
| 38406 | addressable | Service offer book: gateways **and towers** |
| 38407 | addressable | Flock payment descriptor: stealth base key B, preferred pools, relays |
| 9735 | regular | Zap receipt (NIP-57 interop) |

Conventions: NIP-01 event semantics; NIP-44 encryption for everything point-to-point; NIP-59 gift wrap for payment delivery (one-time keys, randomized timestamps [52]); NIP-65 outbox routing plus the kind-38407 payment descriptor for recipient discovery (fallback when neither exists: B = the receiver's Nostr pubkey, delivery to the pool relay set); NIP-13 proof-of-work ≥ 20 bits (relay/Q demand capped at 20) as the spam gate on intents and wraps [53] — roughly a second of commodity-CPU work: a rate limiter, not a GPU-proof barrier.

Terminology, fixed here and used throughout: a **checkpoint** is a round refresh — it extinguishes residual trust and is available hourly. The **transfer log (kind 30178)** is the per-cosign anti-equivocation spine: Q publishes one root per arkoor cosign batch, with a strictly increasing sequence number, so receivers always verify against a fresh root, never an hour-old one (the ≥ k-relay replication requirement applies per root). Each root commits to (nullifier, commitment) pairs with `nullifier = SHA256("FlockNullifier/v0.3" ‖ outpoint)` — derived from the outpoint alone, so any double cosign of the same outpoint produces an identical nullifier under two sequence numbers: a mechanically detectable collision. The concession: detection requires scanning the log — towers and auditors do this, and receivers check their own outpoints — so full-log availability is part of open problem 2. Roots contain no cleartext amounts and must be replicated on at least k relays with read-back verification. Two quorum-signed roots at one sequence number constitute a portable, self-contained fraud proof — any holder can prove equivocation to the world.

### 6.2 Why untrusted relays are survivable

Relays in Flock can drop, delay, reorder, withhold, and censor — and the design budget assumes they will, because most relays are unaccountable volunteers [46]. Three properties make this survivable:

1. **Relays cannot forge or alter anything.** Every event is signed by a user, an operator, or the quorum; ceremonies additionally sign canonical session transcripts. Relay misbehavior is therefore limited to *absence*, never to false state.
2. **Absence fails closed.** A receiver does not display a payment as final until it has verified the signature chain to a confirmed cell transaction, fetched the current 30178 roots from disjoint relay sets including operator-independent relays, verified Merkle inclusion, and verified replication by read-back from ≥ k mirrors. Missing data means "not final," never "assume fine." Liveness can be attacked (a withholding relay set delays finality); safety holds **for checkpointed funds whose owner or towers act within stated deadlines**. The deadline-driven exceptions belong inline, not in a footnote: a held forfeit must be broadcast within its 144–720-block forfeit stale window (§5.3, §11 problem 9); a dormant wallet's recovery must beat the quorum sweep inside the 3,744-block window (§5.5); and an offline receiver's defense depends on its height-deposits executing within the 288–720-block ladder defense deadline (§5.4, §8.2).
3. **The no-data fallback is on-chain.** If relays purge everything and the archival relays are denied, checkpointed funds remain recoverable from the seed plus any surviving copy of the public round data via the recovery tapleaf (§5.5, §7.5), and pre-signed exits held locally or by towers remain valid. The relay layer carries convenience and latency; Bitcoin carries the last-resort guarantees.

The honest qualifier: property 3 does not cover *uncheckpointed* arkoor receipts, whose exit data is not seed-derivable. That residual data-availability dependence is open problem 2 (§11) and is bounded — not eliminated — by read-back replication verification, mandatory local persistence, the operators' paid archival relays (plain NIP-01, long retention, listed in the 38400 pool ad), and auto-checkpointing above the value-at-risk cap.

## 7. Payment flows

Walk-throughs only; ceremony-level normative detail (timers, abort rules, transcript formats) is in [specs/ceremonies.md](specs/ceremonies.md).

### 7.1 Send (Alice → Bob; ~1–2 s target, pending measurement over relay+Tor paths)

1. Alice derives a per-payment stealth key for Bob: `P_B = B + H(ECDH(a,B) ‖ cell_txid ‖ counter)·G`, where B comes from Bob's kind-38407 payment descriptor (fallback: B = Bob's Nostr pubkey, §2.1 R7). The per-payment tweak prevents address-reuse linkage on the public log.
2. She builds the arkoor transaction: payment vTXO to P_B, self-made change back to herself, spending her leaf's cooperative path, and signs it.
3. Over a gift-wrapped kind-24405 exchange, Q verifies there is no nullifier conflict for Alice's leaf outpoint, increments the sequence, and cosigns **both** the transfer and Bob's laddered claim transaction (one hop deeper than Alice's, §5.4). It publishes the updated 30178 transfer-log root — one root per cosign, so Bob's inclusion proof is always against a fresh root.
4. Alice gift-wraps the package — transaction chain to a confirmed cell transaction, signatures, Bob's Q-cosigned claim, Merkle inclusion proof — to the pool relays (mandatory) plus Bob's read relays (NIP-65 or 38407) when known.
5. Alice files a **funded kind-38404 height-deposit** for Bob's exit package, naming specific towers. The deposit is height- *and conflict-*triggered: each tower broadcasts at `trigger_height` or immediately upon detecting any spend touching the package's ancestor outpoints. Payment is contingent: the package carries pre-signed **escalating bump children** (10 / 50 / 200 sat/vB, spending the depositor's dedicated fee UTXOs), each variant carrying a ~2,000-sat fee output to the named tower's per-request key — the tower selects the lowest variant that clears, so its out-of-pocket cost is ≈0; whichever tower completes the broadcast collects, at most one fee is ever paid, and nothing is paid if no tower ever needs to act. The residual: feerates above the 200 sat/vB pre-signed ceiling stall the package, which folds into the fee-spike problem (§11, problem 3). This is a mandatory part of send, not a courtesy: it is what protects a receiver who never comes online (§8.1). Conceded plainly: the rule is unenforceable against a sender who *is* the attacker and a receiver who never comes online — which is why Q also lodges a deposit for every arkoor cosign it performs, and why an uncheckpointed receipt is not accepted until the receiver's own verification passes (§7.2, §8.2).
6. Alice verifies replication by **read-back over disjoint connections** from ≥ k relays (NIP-01 OK responses prove nothing; relay-signed receipts are an optional extra, not the completion condition). Proof of payment is the Merkle inclusion proof under the quorum-signed root.

### 7.2 Receive while offline

Bob needs only an npub — no on-chain setup, no liquidity, no LSP (R4, R7). While he sleeps, the gift wrap waits on relays — including the operators' paid archival relays — and the sender- and quorum-funded, conflict-triggered height-deposits already guard the funds against both expiry and a hostile early unroll. On waking, Bob:

1. Fetches his gift wraps.
2. Verifies the signature chain down to a confirmed cell transaction, checking L1 via compact filters or a full node (block explorers only ever as additional cross-checks, never the sole L1 view).
3. Fetches 30178 roots from disjoint relay sets, including operator-independent relays, **failing closed** if they are absent.
4. Verifies Merkle inclusion and ≥ k-mirror replication by read-back.
5. Persists the package locally (mandatory).
6. If the accepted receipt pushes his **aggregate uncheckpointed balance** over the **value-at-risk cap** (default 50,000 sats — low enough that the worst eclipse-or-collusion prize stays small against the cost of mounting it, high enough that everyday receipts don't force a round seat), the wallet auto-checkpoints at the next available round seat — VaR-triggered refreshes take priority in Q's seat assignment (§4.3); below the cap, the balance displays as "uncheckpointed" until refreshed. The cap is aggregate, not per receipt: structuring many sub-cap receipts does not evade it.

Until these checks pass, the receipt is **not accepted** — a sender clawing back value before acceptance is a failed payment, not theft of accepted funds. Merchants requiring finality MUST require checkpoint inclusion or verify online, including verifying that the height-deposits exist. The VaR cap itself is enforceable only while online; worst-case offline exposure is bounded by this acceptance rule, and Q MAY enforce an optional `uncheckpointed_ceiling` (38400), refusing cosigns beyond a per-recipient uncheckpointed total (a privacy trade-off in name only: Q already sees the graph).

Receives are **refused** against cells with fewer than 2016 blocks (~2 weeks) to T_exp — the sender must refresh first. This bounds the recovery-leaf stale-owner window (§5.5, §8.1).

### 7.3 Refresh / checkpoint

Hourly opportunity; wallet deadline T_exp − 1008 (~1 week of margin).

1. Bob publishes a refresh intent (kind 24400): ownership proof, the intent bond (default 1,000 sats — a **ledger debit, netted at next service**: on post-nonce abort Q records the slash in the transfer log and deducts it from Bob's next successful refresh output or arkoor change, refusing service until netted; as backstop the intent carries a pre-signed **lien transaction** spending the refreshing leaf's cooperative path — value minus bond back to Bob, bond to Q — valid on any future unroll of that leaf, though netting is the economic collection path), PoW ≥ 20 bits (relay/Q demand capped at 20), gift-wrapped to Q.
2. Q assigns him to a 64-leaf cell and publishes the cell proposal — including that cell's funding-transaction template — with a canonical participant-set commitment.
3. Bob verifies his own new leaf and path and the participant-set commitment, then submits MuSig2 pubnonces.
4. The participant set is **finalized post-nonce**: non-responders are excluded and the cell rebuilt in the same attempt — re-signing that one cell tx only (§4.2). Aborting after nonces forfeits the intent bond.
5. Partials are exchanged under 120 s adaptive phase timers; everyone signs the canonical transcript.
6. Bob signs the forfeit spending his old leaf plus his own connector of the proposed cell txid.
7. Q broadcasts the cell transactions (fee-bumped by CPFP on the change output only — never RBF). Confirmation of Bob's cell tx simultaneously activates the new leaf and makes the forfeit valid (connector atomicity); a mutated cell transaction invalidates the forfeit, leaving the old vTXO intact. Bob verifies his cell txid on L1 after his cell's SET-FINAL.
8. Result: fresh 30-day clock, ladder reset to hop 0, recovery tapleaf rotated to Bob, standing dormancy deposit (unroll branch + recovery claim) lodged with ≥3 towers (§5.5), zero residual trust on the refreshed value.

### 7.4 Unilateral exit

1. Broadcast the pre-signed path transactions (3 node transactions for a depth-3 cell) as 1p1c TRUC packages with the pre-signed minimum-fee children; fee-bump via P2A — anyone, including keyless towers, can fund this.
2. After the leaf is on-chain, wait out the claim's nSequence (144–720 blocks depending on hop), then broadcast the claim.
3. No operator cooperation is needed at any step. Stale prior owners are consensus-blocked by their later-maturing claims; a live Q — or any tower holding a forfeit — beats refresh-stale leaves immediately (forfeits are nSequence 0).

### 7.5 Recover from the seed (plus surviving public round data)

1. Seed → Nostr key + Bitcoin xprv.
2. Fetch the kind-10002 relay list and the kind-37400 vault (authored under a seed-derived unlinkable key) from bootstrap relays; fall back to the operators' archival relays (ciphertext, useless to operators).
3. Rebuild the vTXO set; cross-check against 30178 roots and L1 (compact filters or a full node; explorers as cross-checks only).
4. If *all* vault copies are gone: checkpointed funds are still recoverable from the **seed plus any surviving copy of the public round data** — crawl 38400 pool ads from bootstrap relays, fetch 38402 round results, confirm cell txids on L1, rebuild the taptrees from the inline tree commitments, and match leaf scriptPubKeys against seed-derived keys (the normative scan is in [specs/onchain.md](specs/onchain.md)). That data is public, replicated, and non-secret, but its survival somewhere — relays, archival relays, towers, any cell peer — is a nonzero availability assumption. The recovery leaf pays the seed-derived key after T_exp+288; the owner or a deposit tower must act within the 3,744-block (~26-day) window before the quorum sweep at T_exp+4032, which the standing dormancy deposits lodged at every checkpoint make automatic (§5.5).
5. Stale vaults are safe: exiting an already-forfeited leaf only loses an already-spent coin. There is no toxic state anywhere in the protocol.

Conceded: uncheckpointed arkoor receipts are not seed-recoverable (§11, problem 2).

## 8. Trust model and security

The full adversarial matrix, with per-attack mitigations and residual risk, is [analysis/attacks.md](analysis/attacks.md); the raw artifacts of the original 42-finding panel and the 51-finding v0.2 verification review live in [research/raw/](research/raw/), with the adjudicated resolutions in [research/AMENDMENTS-v0.3.md](research/AMENDMENTS-v0.3.md). This section states the trust model exactly and explains *why each headline mechanism exists* — every one of them is a response to a specific flaw one of those reviews found.

### 8.1 What the quorum can and cannot do

**Q can:**

- **Censor** — refuse to cosign payments or refreshes. The remedy is unilateral exit; funds are never frozen, and refusal near expiry still loses to the recovery tapleaf and the standing tower deposits (§5.5).
- **See the payment graph** — amounts and links at cosign time. Conceded (§11, problem 1).
- **With 4-of-7 collusion plus the sender: double-spend uncheckpointed arkoor receipts** — bounded below. (A prior owner acting *alone* can also claw back an uncheckpointed receipt from a receiver who stays offline past the ladder differential while no tower acts — §5.4, §8.2. That is not a quorum power, but it belongs in the same honest accounting.)
- **Slash an intent bond** — unilaterally, on post-nonce abort; exposure is bounded by the bond (§8.2).

**Q cannot:**

- Confiscate or reverse checkpointed vTXOs whose owner or towers act within the stated deadlines — the user cosigned and key-deleted their own tree path, so even full 7-of-7 collusion cannot rewrite it. The one deadline-driven exception is below.
- Mint state — every 30178 root is checkable against L1-confirmed cell transactions, and equivocation is a portable fraud proof.
- Sweep before T_exp+4032 — 3,744 blocks (~26 days) after the recovery leaf matures at T_exp+288. The sweep-vs-recovery race is real: Q can sweep a never-unrolled cell at its root, so if the owner stays absent *and* none of the ≥3 standing deposit towers acts within that window, the sweep confiscates (§5.5). That is the worst case, stated without softening.
- Beat the claim ladder — ordering is BIP-68 consensus, not policy.

**Relays can:** drop, delay, and reorder (sessions abort safely; finality fails closed), read metadata (bounded by stealth keys, gift wraps, padded vaults, Tor), and spam-gate (PoW demand capped, reads verified by read-back). They cannot forge, alter, or steal.

**Watchtowers can:** fail to act (mitigated by redundancy, sharding, conflict triggers, and contingent fees funded by sender, quorum, and owner deposits). They hold no keys and cannot redirect anything.

### 8.2 Residual-trust time bounds

| Exposure | Trust required | Bound |
|---|---|---|
| Uncheckpointed arkoor receipt | Revocable by the **sender alone** if the receiver remains offline beyond the ladder differential after a hostile unroll and no tower acts; revocable by **sender + 4-of-7 collusion** otherwise; extinguished at checkpoint | A receipt is not *accepted* until the receiver's fail-closed verification passes (§7.2) — clawback before acceptance is a failed payment. After acceptance: auto-checkpoint at the next available round seat once the **aggregate** uncheckpointed balance crosses the 50,000-sat VaR cap (VaR-triggered refreshes take seat priority; enforceable only while online; offline exposure bounded by the acceptance rule and the optional `uncheckpointed_ceiling`); hard bound T_exp (receives banned in the final 2016 blocks). Equivocation (two signed 30178 roots at one seq) is portable fraud evidence. |
| Offline safety | No third party is needed before **T_exp − 720**; at T_exp − 720 the height-deposit tower must act | Conflict-triggered deposits (sender-filed, quorum-filed per cosign, and the owner's standing deposits) defend against a hostile early unroll within the **ladder defense deadline — 288–720 blocks after the unroll, depending on the attacking hop** (the current owner's head-start differential is 144–576 blocks; the separate forfeit stale window for refresh-stale leaves is 144–720, §5.3); height triggers exit the funds toward expiry, and checkpointed value rides to the recovery leaf at T_exp+288. If owner and all towers fail to act, the stale claim — or, after T_exp+4032, the sweep — wins. |
| Recovery-leaf stale window | Prior checkpointed owner must not claim | Uncheckpointed value whose holder stays offline past T_exp+288 is claimable by the prior owner — bounded by the VaR cap and the receive-near-expiry ban. |
| Intent bond | Q must not slash a bonded intent dishonestly | ≤ `intent_bond_sats` (default 1,000) per refresh intent; slashing is unilateral, valid only post-nonce, collected as a ledger debit netted at next service (pre-signed lien on the refreshing leaf as backstop, §7.3); boarding users are exempt (PoW-only). |
| LN gateway swaps | Standard HTLC exposure | 144-block CLTV delta. |

Checkpointed funds carry **zero counterparty trust while the owner or its towers act within stated deadlines**: theft-proof under full quorum collusion, exit always available, recovery from the seed plus public round data as the floor. The deadline-driven exceptions — held-forfeit broadcast (problem 9), the recovery-vs-sweep window (§5.5), and height-deposit execution (problem 10) — are the honest residue.

### 8.3 Design rationale: every mechanism has a scar

Flock is the output of an adversarial contest (three competing designs, three judging lenses, a 42-finding attack panel) followed by a 51-finding verification review of the v0.2 drafts ([research/raw/](research/raw/), [research/AMENDMENTS-v0.3.md](research/AMENDMENTS-v0.3.md)). Each headline mechanism replaced a naive version that was demonstrated broken:

| Mechanism (current) | What it replaced | The flaw that killed the naive version |
|---|---|---|
| Two-sig tapscripts; Q signs plain FROST only | FROST group acting as one MuSig2 cosigner | Judges: no proven, shipped construction for threshold MuSig2 nonces; panel: composition leaks the group key under a split relay view — a single failure compromising the entire pool |
| CSPRNG-only, RAM-only nonces; abort-and-restart | Deterministic session-keyed nonce derivation | Panel's #1 critical: induced re-signing across crash/retry yields two partials under one nonce — algebraic extraction of user keys and FROST shares, escalating to whole-pool theft [48] |
| Connector-bound forfeits | Hash-locked forfeit/new-leaf atomicity | Panel: the hash-lock was asserted, not specified, and broke at script level (refresh theft, mass forfeit); even correct, the preimage is a withholdable freeze lever. Connectors leave nothing to withhold |
| Decrementing-nSequence claim ladder | Reactive 144-block forfeit contest | Panel: a dead quorum made stale exits uncontestable, robbing every arkoor recipient; the contest window also sat on the TRUC-pinning/replacement-cycling attack surface. The ladder is race-free consensus ordering, valid with zero live operators |
| Rotating recovery tapleaf + tower-protected dormancy (standing deposits) | Expiry sweep of unrefreshed funds; carrying fees on idle balances; v0.2's "zero-cost, never-swept" dormancy | Panel: refresh censorship plus tower suppression became outright confiscation at T_exp. Economics judge: ~0.4%/month on whole balances plus forced exit after ~25 days idle is adoption-fatal versus Cashu and custodial Lightning. v0.2 review: "Q must unroll to sweep" was false — the root sweep leaf makes the recovery race real, so dormancy needs standing tower deposits |
| 16 × 64-leaf cells, each with its **own funding transaction**, post-nonce finalization, intent bonds | One 1,024-cosigner hourly ceremony; v0.2's single shared round tx | Judges: load-bearing liveness assumption over lossy relays; panel: a single aborter could collapse the round for everyone, hourly and free. v0.2 review: cells sharing one funding tx made isolation fictional — any rebuild changed the shared txid and voided every cell's pre-signed graph. Now one aborter delays one cell by one rebuild of one cell tx and forfeits a bond |
| Delegated refresh **deleted**; sender-funded height-deposits + recovery leaf | Pre-authorized operator-side refresh for offline users | Panel: the quorum could re-sign a sock-puppet branch under the delegation and steal the balance — a trust downgrade masquerading as a convenience. The replacement protects offline users without giving anyone signing authority |
| Per-cosign Merkle transfer log (30178) + outpoint-only nullifiers + fail-closed disjoint-relay reads + VaR cap | Flat per-vTXO sequence log, best-effort conflict checks; v0.2's hourly roots and `H(outpoint‖seq)` nullifier | Panel: an eclipsed receiver could be double-spent with an unactionable fraud proof. v0.2 review: seq-salted nullifiers made cross-seq double-spends unlinkable, and "hourly roots" contradicted the in-send publication. Now: per-cosign roots, collision-visible nullifiers, inclusion proofs, ≥ k-relay read-back, portable equivocation proofs, and a hard cap on what an eclipse can be worth |
| Receive ban < 2016 blocks to T_exp | Receives until expiry | Panel: the recovery tapleaf names the checkpointed owner, so late uncheckpointed receives handed a theft window to stale prior owners. The ban plus auto-checkpointing shrinks that window to sub-cap value of long-offline users |
| Mandatory sender-funded 38404 at send, conflict-triggered, plus quorum per-cosign deposits | Receiver-filed, height-only watch requests | Panel: a never-yet-online receiver has, by definition, never filed a watch request — confiscation at T_exp by default. v0.2 review: height-only triggers slept through a hostile early unroll, and the "mandatory" sender deposit is filed by the attacker — hence conflict triggers and Q's own deposits |

### 8.4 Attack-surface summary

Across the reviews, the residual risks that remain on the **mitigated** findings cluster into four families. This list covers the mitigated surface only; the conceded and structural residuals — operator graph visibility, uncheckpointed data availability, the recovery-vs-sweep window, quorum liveness, the stale-recovery window — live in §11:

1. **Implementation-discipline risks.** Nonce hygiene and adaptor-signature handling are spec-level bans with test vectors, but the failure class (silent, catastrophic key leakage) demands conservative engineering; there is no protocol-level fix for a wallet that re-signs when it should abort.
2. **Network-position risks.** A global eclipse of a target's relay *and* L1 view can delay finality detection or hide conflicts; mitigations (disjoint relay sets, operator-independent attestation, compact filters or a full node with explorers as cross-checks, fail-closed reads) raise the bar to ISP/state level, and the VaR cap bounds the prize.
3. **Mempool risks.** Replacement cycling has no complete mempool-level fix [15]; Flock keeps safety-critical deadlines 1008+ blocks wide, pre-signs its own CPFP children, uses dedicated fee UTXOs, and accepts that a resourced adversary can *delay and tax* deadline-adjacent exits — never redirect them.
4. **Economic-margin risks.** Value near its ~310 vB exit cost is strandable under fee spikes; intent bonds tax occasionally-unlucky honest users; the watchtower fee market is unproven. These are priced and bounded, not solved.

## 9. Economics summary

Full model in [analysis/economics.md](analysis/economics.md).

**Operators** earn: refresh fees of **0.5–0.9% of refreshed value per refresh**, priced on capital-lock duration (not remaining lifetime — pricing by lifetime hands operators a churn-amplification lever); arkoor cosign fees of 1–3 sats + 0.05% (near-pure margin); gateway swap fees of 0.1–0.5%; and paid relay/vault-pinning subscriptions (~1–2k sats/month).

The float arithmetic, inline, because the v0.2 draft's numbers did not close — twice. Capital fronted for a cell is recovered only at the **root sweep**, not at expiry: each sat of tree capital is locked for `(4320 + sweep_delay)/144` days — with the canonical `sweep_delay` of 4032, **~58 days** — so the break-even refresh fee is APY × (4320 + sweep_delay)/52560 ≈ **0.8% per refresh** at 5% APY. The fee is **per refresh**, not per month, because float scales with refresh cadence: boarding-funded leaves cost Q ≈ nothing (the user's own UTXO funds the cell input), so Q's float comes from **refresh churn** — float ≈ hot_balance × 58 d / refresh_cycle. And the protocol's own rules set the cadence: the refresh deadline (T_exp − 1008, ~23 days), the ~16-day receive ban, and hop-4 velocity force cycles well short of the 30-day expiry, which is why the earlier "%/month on a 30-day cycle" framing understated active users' cost. The canonical worked example — computed in [analysis/economics.md](analysis/economics.md) §1.3 and quoted here unchanged: **10k users at 100k-sat average balances (10 BTC custodied), 50% hot, hot balances on a deadline-driven 23-day cycle ⇒ float ≈ 5 × 58/23 ≈ 12.6 BTC; capital cost ≈ 12.6 × 5%/12 ≈ 0.052 BTC/month; refresh revenue at 0.8% per cycle ≈ 0.052 BTC/month — the refresh line is sold ≈ at cost, and the margin lives in arkoor flow fees, gateway swaps, and pinning subscriptions modeled at ~30% adoption** (the full stack table is in economics.md §1.3) — a thin but positive margin, stated honestly, not a profit engine. A once-a-cycle casual user pays ≈0.5–0.9% per ~23-day cycle; high-velocity users pay proportionally more (§11, problem 7). The structural trade-off is explicit: a shorter `sweep_delay` cheapens refreshes but shrinks the dormancy grace window (§5.5). The honest framing: this is a capital-intensive, LSP-like business whose headline product is sold roughly at cost — which is the point of R1. Existing Lightning LSPs can migrate with the same balance sheet and strictly better capital efficiency, since nothing is locked per edge.

**Users** pay: nothing to receive; 1–3 sats + 0.05% to send, plus a **contingent** tower-deposit allocation (~2,000 sats riding each pre-signed bump child, collected only if a tower ever has to act — at most once, and never on the happy path; a budget line, not a per-send fee); **0.5–0.9% of refreshed value per refresh** on hot balances — ≈0.5–0.9% per ~23-day cycle for a once-a-cycle user, proportionally more at higher velocity — the substitute for Lightning's channel-open/rebalance amortization, and competitive with it for *casual-velocity payment balances*. **Dormant balances pay no carrying fee**: skip refreshes entirely and degrade to tower-protected self-custody via the recovery tapleaf and the deposits lodged at the last checkpoint (§5.5). They are not confiscated — but ~32 days after the last checkpoint the standing deposit automatically exits them on-chain at the owner's expense (≈3.1k–16k sats at 10 sat/vB plus the ~2,000-sat contingent tower fee); no monthly-app-open requirement, and confiscation only in the worst case where owner and all towers fail to act for ~26 days after recovery maturity. Savings should go dormant — or sit in savings-profile pools with stretched expiry and `sweep_delay` — since the carrying cost applies only to payments float. Exits: ~16k sats for a first exiter at 10 sat/vB, ~3.1k sats on an already-unrolled path (§5.6).

The structural 0.5–0.9% per refresh on hot balances is intrinsic to covenant-less Ark capital fronting and is conceded as open problem 7 — competitive for casual payment balances, expensive at merchant velocity (LN-gateway sweeps are the relief valve), and wrong for savings, which go dormant (no carrying fee; one automatic exit) instead.

## 10. Comparison summary

Full analysis in [analysis/comparison.md](analysis/comparison.md).

| Dimension | Flock v0.3 | Lightning | Ark (single op) | Spark | Cashu/Fedimint |
|---|---|---|---|---|---|
| Operator trust | Uncheckpointed window: sender alone vs offline receiver (tower-mitigated), else sender + 4-of-7; extinguished at checkpoint; dormant-fund sweep only if owner and all towers absent ~26 d past recovery maturity | None (counterparty risk) | 1 op, same window | k-of-n, unprovable key deletion | Full custody (blind) |
| Unilateral exit | Always; pre-signed, ladder-ordered | Yes (needs channel state) | Yes | Yes (timelock ladder) | No |
| New-user receive | Zero setup, offline OK | Needs inbound liquidity | Zero setup | Zero setup | Zero setup |
| Receiver liveness | None (conflict-triggered towers + recovery leaf) | Online or LSP-held | Round participation | None | None |
| Seed-only recovery | Checkpointed: seed + public round data; receipts: no | No (toxic state) | No | Partial | Backup-dependent |
| Idle-funds cost | No carrying fee; one automatic on-chain exit (~5–18k sats) if absent >~32 d | Channel reserve/mgmt | ~0.5%/mo or sweep | Low | Custodial risk |
| On-chain cost/user | ~451 vB amortized (mass exit); ~310 vB marginal | 2 txs/channel | Similar to Flock | Low | ~0 |
| Privacy | Q sees graph; public sees commitments | Onion-routed (better) | Op sees graph | Op sees graph | Best (blind sigs) |
| Infra dependency | Nostr relays (incl. operator archival) + tower market | P2P gossip + watchtowers | Operator API | Operator API | Mint/federation API |

Flock's distinct cell in this table is the combination: no custody, no channels, no per-edge capital, no proprietary endpoint — at the cost of the bounded arkoor trust window and below-Lightning privacy.

## 11. Open problems

Stated plainly. None of these is hidden in a footnote, and several are structural rather than engineering gaps.

1. **Operator payment-graph visibility.** Q necessarily relates spends to prevent double-spending; blinded cosigning compatible with fraud proofs is unsolved. Flock's privacy is below Lightning's.
2. **Uncheckpointed-receipt data availability.** Arkoor exit data is not seed-derivable; a coordinated relay purge plus archival-relay denial strands long-offline uncheckpointed receipts. Structural; bounded only by the VaR cap, read-back replication, the operators' paid archival relays, and auto-checkpointing. Full transfer-log availability — needed for nullifier-collision scanning (§6.1) — is part of the same problem.
3. **Replacement cycling / fee-spike timing.** No complete mempool-level fix exists; deadline-adjacent exits can be delayed at cost. Defense is wide margins, dedicated fee UTXOs, and tower redundancy.
4. **Dust.** Value near its ~310 vB exit cost is economically strandable; the 10,000-sat leaf floor pushes micropayments into uncheckpointed change.
5. **Grief vs. loss indistinguishability.** Ceremony withholding cannot be distinguished from relay loss; intent bonds either tax honest unlucky users or under-deter attackers. There is no clean point on that dial.
6. **Accountability without covenants.** Operator "slashing" reduces to portable fraud proofs plus reputation and legal recourse; on-chain bonds would need trusted arbitration.
7. **Structural carrying cost.** 0.5–0.9% of refreshed value **per refresh** (break-even ≈0.8% per refresh at 5% APY with the canonical sweep_delay; float ≈ hot_balance × 58 d / refresh_cycle, §9) is intrinsic to covenant-less Ark capital fronting. A once-a-cycle user pays ≈0.5–0.9% per ~23-day cycle; merchants at hop-4 velocity pay proportionally more and will rationally prefer **LN-gateway sweeps or on-chain offboards** to high-frequency refreshes — conceded, with the gateway line priced as the relief valve. The pricing levers are `sweep_delay` and pool expiry: shorter sweep_delay cheapens refreshes but shrinks the dormancy grace window (§5.5). Competitive for casual payment float, not for merchant velocity or savings (savings go dormant — no carrying fee, one automatic exit — instead).
8. **Recovery-leaf stale window.** Sub-cap uncheckpointed value of receivers offline past T_exp+288 is claimable by the prior checkpointed owner.
9. **Quorum liveness.** Fewer than k live operators halts new payments and forfeit *creation* (exit-only mode); already-held forfeits still need *someone* to broadcast within the 144–720-block forfeit stale window (depending on the stale owner's hop, §5.3).
10. **Watchtower market thinness.** Height- and conflict-triggered deposits make tower action automatic, and the contingent ~2k-sat fee output gives towers an explicit funding path — but a real tower economy at scale is unproven, and it now carries more weight than in v0.2: the dormancy guarantee itself (a dormant wallet's funds reach self-custody only if one of its ≥3 standing deposit towers acts within the 3,744-block window, §5.5) and the offline receiver's stale-unroll defense (§8.2) both rest on it. The symmetry with this paper's own critique of Lightning watchtowers (§1.2, "no sustainable fee model") is acknowledged: Flock makes the tower economy load-bearing and differs only in giving it a funding instrument.

## 12. Future work under covenants

Flock assumes none of this, but the upgrade path is concrete. **CTV** would make tree expansion non-interactive: no MuSig2 ceremonies, no ephemeral-key-deletion assumption, no cells, no aborter griefing — rounds collapse to a single quorum broadcast, and offline users refresh trustlessly (Erk-style [28]), deleting open problem 5 and most of 9. **CSFS** would let leaves verify quorum signatures over *future* state: consensus-enforced forfeit binding, true per-hop recovery-leaf rotation (closing problem 8), and compact rebindable exits. Together, CTV+CSFS would roughly halve exit vbytes and remove every "someone must broadcast in time" dependency — leaving only the economic and privacy problems, which no opcode fixes.

## References

1. Lightning Engineering — How to get inbound capacity. https://docs.lightning.engineering/the-lightning-network/liquidity/how-to-get-inbound-capacity-on-the-lightning-network
2. Bitcoin Design Guide — Liquidity. https://bitcoin.design/guide/how-it-works/liquidity/
3. Plaitr — Lightning Network for merchant payments: a 2026 reality check. https://www.plaitr.com/blog/lightning-network-for-merchant-payments-a-2026-reality-check
4. Spark Research — Lightning Network liquidity explained. https://www.spark.money/research/lightning-network-liquidity-explained
5. BitcoinAndLightningLayerSpecs/lsp (archived Jan 2025). https://github.com/BitcoinAndLightningLayerSpecs/lsp
6. Bitcoin Optech — 2025 year-in-review. https://bitcoinops.org/en/newsletters/2025/12/19/
7. Bitcoin Optech — Splicing. https://bitcoinops.org/en/topics/splicing/
8. Bitcoin Optech — Async payments. https://bitcoinops.org/en/topics/async-payments/
9. lightning/bolts PR #1149 — Async payments. https://github.com/lightning/bolts/pull/1149
10. CCN — Watchtowers in Bitcoin. https://www.ccn.com/education/crypto/watchtowers-in-bitcoin/
11. Voltage — Handling high on-chain fees. https://voltage.cloud/blog/lightning-node-management-handling-high-on-chain-fees
12. LND — safety.md. https://github.com/lightningnetwork/lnd/blob/master/docs/safety.md
13. Lightning Engineering — Disaster recovery. https://docs.lightning.engineering/lightning-network-tools/lnd/disaster-recovery
14. Bitcoin Optech — Channel jamming attacks. https://bitcoinops.org/en/topics/channel-jamming-attacks/
15. Bitcoin Magazine — Postmortem on the Lightning replacement cycling attack. https://bitcoinmagazine.com/technical/postmortem-on-the-lightning-replacement-cycling-attack
16. Sguanci & Sidiropoulos — Mass exit attacks on the Lightning Network. https://arxiv.org/abs/2208.01908
17. CryptoSlate — Why Lightning Network capacity declining 20% in 2025 is not as bad as it sounds. https://cryptoslate.com/why-lightning-network-capacity-declining-20-in-2025-is-not-as-bad-as-it-sounds/
18. Bitcoin.com News — Sustained slide in Lightning capacity and channels through 2025. https://news.bitcoin.com/data-shows-sustained-slide-in-lightning-network-capacity-channels-through-2025/
19. Second — Ark protocol introduction. https://second.tech/docs/learn/intro
20. Bitcoin Optech — Ark. https://bitcoinops.org/en/topics/ark/
21. Ark Protocol — clArk (covenant-less Ark). https://ark-protocol.org/intro/clark/index.html
22. Steven Roose — The Ark case for CTV (Delving Bitcoin). https://delvingbitcoin.org/t/the-ark-case-for-ctv/1528
23. Second — Ark payments (arkoor). https://second.tech/docs/learn/payments
24. Ark Protocol — Connectors. https://ark-protocol.org/intro/connectors/index.html
25. Second — Forfeits. https://second.tech/docs/learn/forfeits
26. Bitcoin Magazine — Second launches Bark on Bitcoin mainnet. https://bitcoinmagazine.com/news/second-launches-bark-on-bitcoin-mainnet
27. Spark Research — Ark protocol explained. https://www.spark.money/research/ark-protocol-explained
28. Steven Roose — hArk/Erk: evolving the Ark protocol using CTV and CSFS. https://roose.io/blog/hark-erk-evolving-the-ark-protocol-using-ctv-and-csfs/
29. Bitcoin Magazine — Statechains: sending keys, not coins. https://bitcoinmagazine.com/technical/statechains-sending-keys-not-coins-to-scale-bitcoin-off-chain
30. conduition.io — Mercury Layer vulnerability disclosure. https://conduition.io/code/mercury-disclosure/
31. Spark Docs — Transfers. https://docs.spark.money/learn/transfers
32. Spark Docs — Trust model. https://docs.spark.money/learn/trust-model
33. Bitcoin Layers — Spark assessment. https://www.bitcoinlayers.org/layers/spark
34. Shinobi — Trustodial: an ontological dilemma. https://bitcoinmagazine.com/technical/trustodial-an-ontological-dilemma
35. Seth For Privacy — Spark and Ark: a look at our newest Bitcoin layer twos. https://bitcoinmagazine.com/technical/spark-and-ark-a-look-at-our-newest-bitcoin-layer-twos
36. Spark Docs — Lightning interop. https://docs.spark.money/learn/lightning
37. NIP-47 — Nostr Wallet Connect. https://github.com/nostr-protocol/nips/blob/master/47.md
38. NIP-60 — Cashu wallets. https://nips.nostr.com/60
39. NIP-61 — Nutzaps. https://nips.nostr.com/61
40. FROSTR. https://github.com/FROSTR-ORG/
41. bifrost — FROST over Nostr. https://github.com/frostr-org/bifrost
42. joinstr — Coinjoin over Nostr. https://joinstr.xyz/
43. Mostro — P2P exchange over Nostr. https://mostro.network/
44. OpenArk / BOLT-ARK. https://openark.tech/
45. Nostr relay replication measurement. https://arxiv.org/html/2402.05709
46. "95% of Nostr relays can't cover costs." https://dev.to/jonathan_greenallidoizc/95-of-nostr-relays-cant-cover-costs-heres-why-that-matters-189l
47. BIP-327 — MuSig2. https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
48. BitGo / Bitcoin Optech — MuSig2 nonce-reuse field report. https://bitcoinops.org/en/bitgo-musig2/
49. Zcash Foundation FROST (RFC 9591 implementation). https://github.com/ZcashFoundation/frost
50. BIP-433 — Pay-to-Anchor (P2A). https://github.com/bitcoin/bips/blob/master/bip-0433.mediawiki
51. Bitcoin Optech — Core 28 wallet integration guide (TRUC, P2A, 1p1c). https://bitcoinops.org/en/bitcoin-core-28-wallet-integration-guide/
52. NIP-59 — Gift wrap. https://github.com/nostr-protocol/nips/blob/master/59.md
53. NIP-13 — Proof of work. https://github.com/nostr-protocol/nips/blob/master/13.md

---

*Research digests with full citation trails: [research/sources.md](research/sources.md). Normative companion specifications: [specs/event-kinds.md](specs/event-kinds.md), [specs/onchain.md](specs/onchain.md), [specs/ceremonies.md](specs/ceremonies.md). Analyses: [analysis/attacks.md](analysis/attacks.md), [analysis/economics.md](analysis/economics.md), [analysis/comparison.md](analysis/comparison.md).*



