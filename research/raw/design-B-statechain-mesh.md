# SPECTER — Statechain Payments over an Erasure-Committed Threshold federation, coordinated entirely by Nostr Relays

## 1. Architecture Overview

```
                          BITCOIN L1
      statecoin UTXOs:  Taproot, key = U_owner + F (FROST group)
     deposits ↑        batched rollovers ↑          exits ↑ (~125 vB each)
              │                  │                       │
 ┌────────────┴──────────────────┴───────────────────────┴──────────┐
 │  FEDERATION: 7 independent operators, 5-of-7 FROST               │
 │  proactive share resharing on EVERY transfer + signed            │
 │  deletion attestations; each operator runs one public relay      │
 └───────┬───────────────────────────────┬──────────────────────────┘
   k24400/24401 signing+RPC        k31400/31401/30404 announcements,
   (gift-wrapped, ephemeral)        attestations, blinded coin state
         │                               │
 ════════╪═══════════ NOSTR RELAY MESH ══╪═══════════════════════════
   7 operator relays + users' NIP-65 relays; every event ≥5 relays
         │                  │                        │
   ┌─────┴────┐   k1402     │              ┌─────────┴─────────┐
   │  SENDER  │ ──────────▶ │ RECEIVER     │ WATCHTOWERS (any- │
   │  wallet  │  gift-wrap  │ (offline OK) │ one) + LN BRIDGE  │
   └──────────┘  "cheque"   └──────────────┘ DESKS             │
```

SPECTER moves whole denominated UTXOs ("statecoins") off-chain, Mercury/Spark-style: each coin is a Taproot UTXO whose key is the aggregate of the current owner's key and a 5-of-7 FROST federation key; transfer hands the user-side key to the recipient while the federation tweaks and **reshares** its side, co-signing the new owner a pre-signed exit transaction with a decrementing absolute locktime so the newest owner can always exit first. All discovery, signing traffic, transfer delivery, encrypted backup, and watchtower feeds are ordinary Nostr events (NIP-01/44/59/65); relays are treated as lossy redundant broadcast, never as authority — authoritative serialization lives in the federation's replicated log, and fund safety never depends on any relay.

## 2. On-Chain Construction

**Statecoin output.** P2TR. Internal key `K = U_i + F` (current owner key plus FROST group key; `K` is constant across transfers — ownership moves by tweaking shares so `U_i + F_i = U_j + F_j`). One script leaf, the **recovery leaf**, rotated at each rollover to the owner-of-record:

```
<U_rollover_owner> OP_CHECKSIGVERIFY <epoch_end + 288> OP_CLTV
```

**Epochs and locktimes.** Coin lifetime per epoch `W = 4032` blocks (~4 weeks). At epoch start the current owner's pre-signed **exit tx** carries `nLockTime L = epoch_end − 288`. Each transfer decrements: `L_{i+1} = L_i − δ`, `δ = 36` blocks (~6 h) → up to 104 transfers per coin per epoch; the federation refuses transfers that would breach the floor and forces a rollover instead. Decrementing absolute locktimes give **consensus-enforced staleness ordering**: a stale owner's exit tx is consensus-invalid until ≥δ·(hops) blocks after the rightful owner's, so defense is "act before a known deadline," never a mempool fee race.

**Exit tx (pre-signed at every transfer).** v3/TRUC, zero-fee, SIGHASH_DEFAULT keyspend by `K` (MuSig2 between user share and FROST partial): one input (57.5 vB), one P2TR output to the owner (43 vB), one keyless P2A ephemeral anchor (~11 vB) ≈ **125 vB**, plus a ~110 vB CPFP child anyone (owner, tower, federation) can fund via `submitpackage`. Per the Mercury audit lesson, operators **fully validate** locktime, sequence, sighash, and amounts on everything they sign — no blind signing.

**Rollover.** Before `L_current`, the federation batch-spends all transferred-this-epoch coins by keyspend into fresh statecoin outputs (~100 vB/coin), resetting locktimes, killing every stale exit tx at consensus level, rotating the recovery leaf to the current owner, and performing full re-keying. Untransferred coins skip rollover (no stale state exists).

**Deposit.** Vault pattern: funding txid fixed first, exit tx ceremony completes, then funding broadcasts; the recovery leaf names the depositor, so an aborted ceremony can never strand funds.

**Three exit paths:** (1) cooperative keyspend — instant, anytime; (2) pre-signed exit tx — unilateral, matures at `L_current` ≤ 4032 blocks away; (3) recovery leaf — seed-only, matures `epoch_end + 288`, needs neither federation nor any saved file.

## 3. Nostr Protocol Surface

All "k2xxxx" kinds are ephemeral (relayed, not stored); gift wrap = NIP-59 with NIP-44; everything is published to ≥5 relays (operator relays + counterparty NIP-65 read relays).

| Kind | Storage | Wrapped? | Content |
|---|---|---|---|
| 31400 | addressable, public | no | Federation announcement: roster, `F`, params (n,t,W,δ), denominations, fees, relay list |
| 31401 | addressable, public | no | Per-operator epoch attestation: resharing log head, hash-committed **deletion attestation**, heartbeat |
| 31402 | addressable, public | no | Watchtower service announcement (NIP-89 style), pricing |
| 10133-compatible payment-target entry | replaceable, public | no | "this npub accepts SPECTER via federation X" — npub addressing, zap-flow discovery |
| 21401 | regular, NIP-40 expiry | optional | Invoice / payment request (amount, memo, federation id) |
| 24400 | ephemeral | gift-wrapped | Signing-session messages: MuSig2 pubnonces/partial sigs, FROST nonce commitments, resharing rounds; BIP-373-style fields; session-ID'd |
| 24401 | ephemeral | gift-wrapped | NWC-style RPC: `transfer_init`, `claim`, `list_coins`, `swap_denoms`, `bridge_pay`, responses |
| 1402 | regular | gift-wrapped to receiver | **Transfer package ("cheque")**: signed exit tx, encrypted `x_u` key handover, lineage + federation attestation; the offline-receive object |
| 1403 | regular | gift-wrapped to tower | Maturity/justice delegation: exit tx + anchor + fee budget |
| 1404 | regular, public optional | no | Federation-co-signed payment receipt; zap-receipt-compatible tags |
| 30403 | addressable | NIP-44 to seed-derived key | Encrypted wallet backup: coin list, exit txs, indices |
| 30404 | addressable, public | no | Blinded per-coin state commitment `H(coin_id‖exit_txid‖L)` — watchtower feed |

Nonce discipline over lossy relays (per BitGo/BIP-327 guidance): fresh nonces per attempt, secnonce delete-on-read, abort-and-restart on any timeout/duplicate/ambiguity; the federation coordinator alone may act as the stateless last signer.

## 4. Payment Flows

**Send (receiver online or not).** (1) Sender resolves recipient npub → payment-target → federation. (2) `transfer_init` (24401) to ≥3 operator relays. (3) Operators FROST-co-sign, with the sender's MuSig2 partial, a new exit tx paying the receiver's static key, `L −= δ` (24400 traffic). (4) Federation executes **VSS share resharing with the ownership tweak folded in** (ZF-frost refresh plus a public additive tweak — a small, stated extension to shipped refresh code); each operator publishes a 31401 deletion attestation. (5) Federation updates 30404, returns a 1404 receipt to the sender (proof-of-payment), and gift-wraps the 1402 cheque to the receiver's read relays. ~2–4 s end-to-end.

**Receive while offline.** Steps 1–5 need nothing from the receiver. On waking, the receiver fetches 1402 cheques, verifies lineage, locktime ordering, and attestations, then runs **claim**: decrypt the handed-over `x_u`, pick fresh `x_u'`, send `Δ = x_u − x_u'` to the federation, which tweaks+reshares and co-signs a fresh exit tx. Pre-claim, the cheque already pays only the receiver and matures before all stale txs; the residual exposure (§6) is federation serialization, not relay behavior.

**Arbitrary amounts.** Coins come in powers-of-two denominations 2^10–2^30 sats. Payments are atomic multi-coin bundles; exact change is a simultaneous swap against the operator change desk inside one federation state update (the federation executes the bundle all-or-nothing). Native granularity is 1,000 sats (dust/exit-economics floor); sub-1k amounts ride the Lightning bridge. *Future-work note: CTV would permit off-chain coin splitting to msat granularity and non-interactive receive trees.*

**Refresh/rollover.** Automatic, federation-initiated, batched on-chain every epoch for transferred coins; users need not participate (keyspend uses current shares; the user's wallet pre-authorizes rollover co-signing at claim time, or co-signs lazily when next online — towers cover the deadline otherwise).

**Unilateral exit.** Broadcast your exit tx after `L_current` (≤28 days wait, typically days), CPFP via P2A; or instant cooperative exit while the federation lives. One 125 vB tx per coin, cost independent of payment activity.

**Recovery from seed alone.** Seed → Nostr key + wallet keys (BIP-32). (1) Fetch 30403 backup from relays; (2) failing that, `list_coins` RPC — the federation re-serves co-signed exit txs; (3) failing *both* (all relays purged AND federation dead), the recovery leaf returns every rolled-over coin to your key at `epoch_end + 288` with nothing but the seed. **Stale backups are safe**: old exit txs are consensus-invalid (locktime) or spend already-spent UTXOs; there is no penalty mechanism, hence no toxic state.

## 5. How R3 and R4 Are Met

**R3 — no single point of failure.**
- *Operator failure:* 5-of-7 threshold; two operators can vanish with zero impact. Three or more dead ⇒ transfers freeze but **funds don't**: exit txs mature within ≤4032 blocks, recovery leaves behind them; locktimes are naturally staggered.
- *Relay failure:* relays carry copies, never the only copy — federation log + user device + ≥5 relays. Any relay subset failing degrades latency, not safety.
- *User offline:* receiving needs no receiver liveness; safety deadlines (rollover at `L−288`) are met by the federation by default and by permissionless watchtowers (1403 packages, paid per 31402 terms) as backstop. Stated bound: a coin transferred this epoch is attackable by its previous owner only if owner, federation, *and* every delegated tower all stay inactive past `L_current + 36` blocks (~6 h grace, growing 6 h per hop of staleness).
- *Seed-only recovery:* three independent paths (§4); worst-case (everything dead, backups purged) the recovery leaf needs only the seed and a Bitcoin node, for every coin owned at the last rollover; coins received since rely on the 1402 cheque surviving on ≥5 relays for ≤4 weeks.

**R4 — zero liquidity management.**
- A brand-new user receives their first payment with literally nothing on-chain: a cheque to their npub. No channels, no inbound capacity, no splicing — coins are bearer UTXOs, fungible across counterparties.
- Operators front **zero** capital per transfer (contrast Ark's full-value-per-round fronting). Operator float exists only for the change desk, on-ramp coin inventory (minted ~45 vB/coin in 100-output batches), and LN bridge channels — recycled in hours and priced into swap fees (§7).

## 6. Honest Trust & Attack Self-Assessment

**The key-deletion problem, precisely.** Theft of a coin requires a signature under `K`, i.e. **(a) ≥5 of 7 operators retaining their epoch-k shares AND (b) the epoch-k owner of that coin colluding**. Operators alone can never steal (they never hold any user-side key); an old owner alone holds only a consensus-invalid-until-deadline exit tx. Per-transfer proactive resharing **does** kill: accidental retention (backups, snapshots), hacks that exfiltrate fewer than 5 operators' stores within one transfer interval, and any later compromise of old material. It **does not** kill premeditated logging by 5 colluding operators — no cryptographic proof of deletion exists. Residual mitigations are non-cryptographic and we say so: 7 independent, jurisdictionally-separated businesses; signed public deletion attestations (31401) making retention provable fraud after the fact; theft is on-chain visible, triggering mass exit and destroying the federation's fee stream. This is the irreducible statechain trust, narrowed from Mercury's "one server, trust me" to "5-of-7 premeditated conspiracy plus a coin's past owner."

**Bounded windows.**
- *Fresh-receive window:* from cheque issuance until claim, the receiver relies on federation serialization (not for theft-resistance — theft still needs the 5+owner coalition — but to refuse the sender a double-spend attempt). Bound: ends at claim; hard-stops at next rollover (≤4 weeks).
- *Unilateral-exit latency:* ≤4032 blocks without cooperation. Acknowledged regression vs. Lightning's CSV-only delay; bought deliberately to get consensus-ordered staleness with no penalty/toxic state and no mempool races.
- *Bridge in-flight:* one payment amount, ≤144-block HTLC window, trust = same 5-of-7 (preimage shards).

**What a malicious quorum CAN do:** freeze all transfers (remedy: everyone exits within one epoch, unstoppable); censor specific users (same remedy); see the full transfer graph — privacy is operator-visible by design because the Mercury audit showed blind signing is how you get robbed; relays and outsiders see only encrypted blobs. **Cannot:** steal without an old-owner co-conspirator, inflate, or block matured exits.

**What relays CAN do:** drop/delay/reorder/selectively serve — worst effects are stalled payments (sender retries other relays; receipts are federation-signed) and delayed cheque/backup delivery. Eclipse-of-user delays deadline awareness; towers and federation auto-rollover cover it. Relays can never steal, forge, or freeze funds.

**Resource exhaustion:** transfers cost a fee; RPC requires NIP-13 PoW (≥20 bits) and per-pubkey limits on operator relays; no slots or liquidity can be locked for free (no HTLCs in the native path). No replacement-cycling surface: exits are single TRUC txs with P2A anchors, no second-stage contests.

## 7. Economics

**Operator revenue:** transfer fee 1 sat + 0.05%; denomination/change swaps 0.1%; LN bridge 0.1–0.3% (Boltz-comparable); rollover surcharge ~200 sats/coin-epoch folded into transfer fees; tower subscriptions optional. Illustrative: 10M transfers/month at avg 5,000 sats ⇒ ~275M sats/yr (~3 BTC) fee revenue across 7 operators against costs of signers, relays, nodes, and a float of ~5% of daily volume plus ~2 BTC of LN channel capacity — capital turns over in hours, not Ark's 30 days.

**User costs:** receive: 0 on-chain. Send: sub-cent fee. Rollover: ~100 vB/coin/epoch only when the coin moved (≈200 sats at 2 sat/vB, ~2.6%/yr on a 100k-sat coin — falls linearly with coin size; acknowledged as material for small coins at high feerates).

**Exit cost:** ~125 vB exit + ~110 vB CPFP ≈ **235 vB/coin**: ~470 sats at 2 sat/vB, ~4,700 at a 20 sat/vB spike. An 8-coin wallet worst-case ≈ 1,880 vB — comparable to one Lightning force-close, **independent of in-flight payments and of tree depth (there is none)**.

**Mass exit:** strictly O(1) tx per coin — no tree unrolling. 1M coins ⇒ ~235 Mvb ≈ 235 full blocks (~40 h saturated); locktime maturities are staggered across 4032-block epochs by construction, so deadlines cannot synchronize; recovery leaves extend the runway a further 288+ blocks. Coins below ~3k sats may be uneconomical to exit at spike feerates — disclosed floor, mitigated by the change desk consolidating dust cooperatively.

**Footprint honesty:** statechains amortize *transfers* (≈100+ per coin-epoch, thousands over a coin's life as it circulates user-to-user like cash), not UTXO creation — each coin is one L1 UTXO forever. That is the structural trade against Ark's trees, bought back as O(1) exits and no interactive rounds.

**R8/interop:** payments address npubs via payment-target entries; 1404 receipts are zap-compatible; the bridge speaks BOLT11/BOLT12 both directions from day one, so any Lightning payer can pay a SPECTER npub and any SPECTER wallet can pay any Lightning invoice, with migration being "receive over the bridge, never open a channel again."
