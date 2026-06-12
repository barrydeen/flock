# Flock: Federated-Ark over Nostr

A Bitcoin L2 where covenant-less Ark pools are operated by a k-of-n FROST quorum, and every byte of off-chain coordination, state backup, and watchtower signaling travels over public, untrusted Nostr relays.

## 1. Architecture overview

```
                         BITCOIN L1
     round tx (hourly/pool)      sweep tx (1 per expired round)
          ▲                              ▲
          │  FROST 3-of-5 quorum Q (operators O1..O5, independent orgs)
          └──────────────┬───────────────┘
                         │ ALL traffic = signed Nostr events
 ┌───────────────────────┴────────────────────────────────────────┐
 │        PUBLIC NOSTR RELAYS (untrusted; ≥8 per pool,            │
 │        incl. ≥2 operator-run paid relays)                      │
 │  pool ads · round ceremonies · arkoor gift wraps · cosign log  │
 │  encrypted vaults · watch requests · gateway offer book        │
 └───┬────────────────┬─────────────────┬───────────────┬─────────┘
  Alice (sender)   Bob (OFFLINE       Watchtowers     LN Gateways
  vTXO + presigned  receiver; has     (permissionless, (vHTLC swaps
  exit package)     only an npub)     keyless CPFP)    ↔ Lightning)
```

Flock pools are clArk-style shared-UTXO rounds: an hourly on-chain **round tx** funds a radix-4 pre-signed vTXO tree (≤1024 leaves) expiring in ~30 days. The "server" is replaced by a 3-of-5 FROST group key **Q**: any 3 operators produce Q's BIP-340 signatures over Nostr (FROSTR/bifrost precedent), so one or two operator failures change nothing. Payments are **out-of-round (arkoor)** cosigns — instant, requiring no receiver liveness and no operator liquidity. Rounds are used only for refreshes and boarding (senders=receivers), eliminating clArk's receiver-DoS problem. Every wallet's unilateral-exit package is pre-signed at creation, NIP-44-encrypted to its own key, and replicated to ≥8 relays, making wallets stateless and seed-recoverable. Permissionless watchtowers fee-bump exits via keyless P2A anchors.

## 2. On-chain construction

**Round tx** (one per pool per hour; quorum-funded plus boarding inputs): one P2TR root output. Internal key = MuSig2-agg(all round participants' ephemeral session keys, Q). Script path: sweep — `<T_exp> CLTV DROP <Q> CHECKSIG`, where **T_exp = funding height + 4320 blocks (~30 d)**.

**Tree node txs** (pre-signed, never broadcast unless exiting): radix-4; each node tx spends its parent via keyspend and creates 4 outputs with the same template (internal key = MuSig2-agg of the users *below that branch* + Q; same CLTV sweep script). Branch-only signing: each user cosigns exactly the txs on their root-to-leaf path (depth 5 for 1024 leaves), then **deletes the ephemeral key** — since the user is a cosigner on every node above their own leaf, their own deletion alone makes their path's pseudo-covenant binding. No per-level CSV, so path delays don't sum.

**vTXO leaf output**: internal key = MuSig2(U, Q) (used for forfeits, arkoor, cooperative spends). Script A (exit claim): `144 CSV DROP <U> CHECKSIG` — the 24 h contest window lets the quorum broadcast a forfeit against exit of an already-spent vTXO. Script B: the T_exp sweep.

**Pre-signed at creation, stored in the vault**: the full path txs *and* a claim tx spending script A to the user's BIP-86 address — so a keyless watchtower can complete the entire exit. All tree/exit/claim txs are **nVersion=3 (TRUC), zero-fee, with a P2A ephemeral anchor**; anyone CPFPs via `submitpackage` (Core 28+ policy, no consensus change).

**Forfeit tx**: spends the old leaf keyspend; atomicity with the replacement uses Bark-style hash-locks — the forfeit and the user's new leaf both gate on a preimage Q reveals only after the new round tx confirms. Connector outputs in the round tx are retained for offboards.

**Boarding output**: P2TR 2-of-2 (U,Q), refund path `4320 CSV <U> CHECKSIG` if never converted.

**Unilateral exit cost** (radix-4, 1024 leaves, depth 5): 5 node txs (~253 vB each: 1 keyspend input 57.5 + 4×43 outputs + 13 vB P2A + overhead) + leaf claim (~165 vB script-path) + per-tx CPFP children (~115 vB). Worst-case first exiter ≈ **2.2 kvB ≈ 22,000 sats at 10 sat/vB**; if the path is already unrolled by an earlier exiter, marginal cost ≈ **450 vB**. Each arkoor hop adds ~152 vB (chain depth capped at 4 by wallet policy).

## 3. Nostr protocol surface

Kinds follow NIP-01 semantics: 3xxxx addressable, 2xxxx ephemeral, 1059 gift wrap (NIP-59). Placeholder numbers.

| Kind | Type | Content | Crypto / visibility |
|---|---|---|---|
| 38400 | addressable | Pool announcement: Q pubkey, operator npubs, k/n, fee schedule, relay set, T_exp ladder | Public, quorum-signed |
| 38401 | addressable | Round announcement: round-id, phase deadlines | Public |
| 24400 | ephemeral | Round intent: vTXO proof-of-ownership sig, refresh request | Gift-wrapped to Q; NIP-13 PoW ≥ 20 bits |
| 24401 | ephemeral | Tree proposal (txs, cosigner sets, session-id) | NIP-44 to participants |
| 24402/24403 | ephemeral | MuSig2 pubnonces / partial sigs (BIP-373 PSBT fields) | NIP-44, session-bound |
| 24404 | ephemeral | Intra-quorum FROST nonce/share messages | NIP-44 between operators |
| 38402 | addressable | Round result: txid, vTXO-tree commitment (inline or Blossom sha256) | Public |
| 1059→24406 | gift wrap | **Arkoor transfer package**: full tx chain + sigs back to a confirmed round | Gift-wrapped to receiver's one-time key |
| 38403 | addressable | **Cosign log checkpoint**: quorum-signed, per-vTXO monotonic sequence numbers | Public — double-spend detection / fraud proof |
| 37400 | addressable | **Wallet vault**: vTXO set, exit packages, arkoor history | NIP-44 self-encrypted ciphertext, public event |
| 38404 | addressable | Watch request: exit package + deadline height + bounty | Encrypted to chosen watchtowers, or public |
| 38405 | regular | Watchtower alert: double-spend / sweep anomaly / exit broadcast | Public |
| 38406 | addressable | LN gateway offer book (NIP-69-style) | Public |
| 9735 | regular | Zap receipt (NIP-57 compatible) | Public |

**Nonce safety over lossy relays** (the load-bearing rule set): one `secnonce` per session-id = H(round-id ‖ attempt ‖ signer-set ‖ tx digest); session-id fed into NonceGen aux input; `secnonce` held in RAM only, delete-on-first-use; a client signs **at most one message per attempt** and ignores any second proposal for the same attempt (coordinator equivocation yields nothing); on timeout/duplicate/ambiguity, abort and restart with fresh nonces — idempotency via session-ids, never nonce reuse. Relay replay is harmless: events are deduped by id and partial sigs are bound to dead sessions. Operators follow identical rules for FROST (RFC 9591, ZF frost), with proactive share refresh monthly and PedPoP DKG at pool genesis. Q participating as one MuSig2 cosigner whose partials are produced by internal FROST is the largest novel-crypto engineering risk and gets an external audit gate.

## 4. Payment flows

**Send (Alice→Bob, ~1–2 s).** (1) Alice derives a one-time claim key P_B = B + H(ECDH(a,B))·G so relays can't link payments to Bob's npub. (2) Builds an arkoor tx spending her leaf keyspend into a payment vTXO (P_B,Q template) and a change vTXO (A′,Q) — change is self-made, no denomination inventory. (3) MuSig2 with Q over kinds 24402/24403. (4) Q verifies the vTXO is unspent in its log, increments its sequence, cosigns, publishes the 38403 entry. (5) Alice gift-wraps the package to Bob's NIP-65 read relays + pool relays; Q archives a copy. Proof-of-payment = quorum-signed log entry naming P_B.

**Receive while offline.** Bob needs only a Nostr keypair — no on-chain setup, no liquidity (R4). Asleep, his phone receives nothing; the gift wrap waits on relays and in Q's archive. On waking: fetch wraps, verify the signature chain to a confirmed round tx (L1 view via any explorer/compact filters), check 38403 on ≥3 independent relays for conflicts, request a quorum no-conflict attestation, then queue a refresh. Until that refresh, the funds are trusted (§6).

**Refresh / checkpoint (hourly opportunity; wallet deadline T_exp − 1008).** Post 24400 intent → verify own leaf and path in the 24401 proposal → two MuSig2 rounds over relays for path txs → sign hash-locked forfeit → Q broadcasts the round tx → on confirmation Q reveals the preimage; the old vTXO is forfeited, the new leaf carries zero arkoor trust and a fresh 30-day clock. Stallers: 30 s phase timers, evicted intents, ≤3 re-proposals; intents require PoW plus a forfeitable-vTXO ownership proof, so only existing holders (penalizable, rate-limitable per vTXO) can grief, and only refreshers — never receivers — are in the ceremony.

**Unilateral exit.** Check 38403 for the latest forfeit state; broadcast path txs as sequential 1p1c TRUC packages, CPFP each via P2A (any party can fund); after leaf confirmation wait 144 blocks; the pre-signed claim pays the user's L1 address. No operator cooperation anywhere.

**Recover from seed alone.** Seed → Nostr key + Bitcoin xprv (deterministic per-pool/per-round derivation). Fetch own kind-10002 and 37400 vaults from a bootstrap relay list; discover pools via 38400; if relays purged the vault, request Q's archive (operators serve ciphertext they cannot use); rebuild vTXO set; cross-check against 38403 and L1. **Stale vaults are safe**: broadcasting an exit for an already-forfeited vTXO only lets Q claim that already-spent coin during the CSV window — you can never lose funds you still own. No toxic state.

## 5. R3 and R4, mechanism by mechanism

**R3 — no single point of failure.** *Operator:* 3-of-5 FROST — any single (or double) operator failure leaves rounds, arkoor, and sweeps running; if ≥3 die permanently, service halts but theft is impossible and every user already holds a complete pre-signed exit; watchtowers see missed 38401 heartbeats and auto-broadcast exits well before T_exp (the sweep path needs Q, so a dead quorum can't sweep either — coins go nowhere except to exiting users). *Relay:* every event published to ≥8 relays across operators' paid relays and public ones; loss of any subset degrades latency, not safety. *User device offline:* receiving needs no liveness; safety needs none until **T_exp − 720 blocks (~25 days)**, when watch-requested towers broadcast the exit (cost: on-chain fees, never loss). Optionally the user pre-authorizes ≤2 delegated refreshes (§6 for the trust downgrade). *Recovery:* seed + any one surviving ciphertext copy (≥8 relays + 5 operator archives + local).

**R4 — zero liquidity management.** Receiving is arkoor: no inbound capacity, no channels, no operator capital consumed, first payment with zero prior on-chain footprint. There is nothing to rebalance, splice, or close — vTXOs split via arkoor change. The capital burden moves to the quorum, which fronts new leaves at refresh time and recycles capital at each 30-day sweep; this is priced into the refresh fee (§7), not into user workflow.

## 6. Honest trust and attack self-assessment

**Residual trust windows (explicit, bounded):**
- **Arkoor receipt**: until the receiver's next refresh (typically ≤1 h once online; hard bound T_exp), a double-spend requires the **sender AND ≥3 of 5 operators** colluding. One honest party in that set suffices. Equivocation produces two quorum-signed 38403 entries — portable cryptographic fraud proof. Chained arkoor extends this to every prior sender in the ≤4-hop chain; wallets display "uncheckpointed" balance distinctly.
- **Delegated offline refresh** (opt-in): the user wasn't in the ceremony, so their new path's pseudo-covenant rests on **≥1 of {3 quorum signers, co-branch users} having deleted ephemeral keys** instead of on their own deletion. Flagged in-wallet; capped at 2 consecutive delegations (~8 weeks); default is watchtower exit instead.
- **vHTLC/LN swaps**: standard HTLC exposure during the swap's 144-block CLTV delta; replacement-cycling-class mempool games apply only here (the core protocol has no preimage-bearing second-stage txs), mitigated by aggressive rebroadcast, not eliminated.

**What the quorum CAN do:** censor (refuse cosigns/rounds → users unilaterally exit; funds never frozen); sweep expired rounds (legitimate); with k colluding: double-spend *unrefreshed* arkoor with the sender (above), and retain FROST shares — useless for theft because every spend still needs the user's key, and useless against tree paths the user cosigned and key-deleted. It can never confiscate, freeze, or reverse a checkpointed vTXO. It CANNOT mint state: every claim is bounded by pre-signed, user-verified txs.

**What relays CAN do:** drop/withhold (ceremonies retry; transfers re-fetched from Q's archive; total withholding by *all* relays *and* all operators blocks seed-only recovery — a data-availability failure, never theft; probability addressed by 8-relay + 5-operator replication and local storage); delay/reorder (sessions abort safely — nonce rules); read metadata (gift-wrap p-tags hidden by one-time claim keys; IP via Tor; amounts in 38403 visible to all — **privacy is roughly Spark-level, below Lightning's onion routing; acknowledged regression**); spam (PoW + NIP-42 AUTH + paid writes).

**Other acknowledged weaknesses:** quorum liveness <k halts new payments (exit-only mode); hourly rounds mean checkpoint latency, not payment latency; mass-exit fee spikes can make sub-10k-sat vTXOs uneconomical to exit (mitigated, not solved, by laddered expiries); watchtower compensation is a thin market (operator-run towers as backstop); FROST-inside-MuSig2 needs bespoke security review. **Future work (explicitly not in scope per R5):** CTV+CSFS (Erk-style) would give non-interactive rounds, offline refresh without trust downgrade, and self-enforcing trees.

## 7. Economics

**Operator revenue.** (1) Refresh fees priced on cost-of-capital for the leaf's remaining lifetime: a 100k-sat vTXO at 5 % APY for 28 d ≈ 384 sats ≈ **0.38 %/month**, decreasing for late refreshes. (2) Arkoor cosign fee: flat 1–3 sats + ~0.05 % — near-zero marginal cost, pure margin. (3) Gateway/vHTLC swap fees 0.1–0.5 %. (4) Paid relay/vault retention subscriptions (~1–2k sats/month, nostr.wine-comparable). A 10,000-user pool ladder at 100k sats average needs ~10 BTC working capital recycling every 30 days plus an overlap buffer — revenue ≈ 0.4–0.5 %/month on deployed capital against 5 % APY cost: sustainable at moderate scale, honest about being a capital-intensive business that concentrates among well-funded operators (federation, not elimination, of that pressure).

**User costs.** Receive: free. Send: ~1–3 sats + 0.05 %. Refresh: ~0.4 %/month on refreshed balance (comparable to LN rebalancing+channel amortization, but predictable and with zero ops burden). Exit: 450 vB–2.2 kvB (≈4.5k–22k sats at 10 sat/vB).

**Mass exit.** Full unroll of one 1024-leaf pool: 341 node txs (~86 kvB) + 1024 claims (~169 kvB) ≈ **255 kvB ≈ 0.26 blocks**, i.e. ~250 vB/user amortized — early exiters subsidize later ones on shared branches. 100k users ≈ 100 pools ≈ 26 full blocks; hourly-laddered T_exp values mean deadlines never synchronize, and the 1008-block refresh margin plus keyless P2A bumping gives watchtowers a week of fee-market room. Worst case is expensive and slow but *safe*: exits are pre-signed, pinning-resistant (TRUC), and need no counterparty.

**Interop (R8).** Payments address npubs natively; gateways emit NIP-57 kind-9735 receipts so existing zap clients work unchanged; vHTLC swaps bridge to BOLT11/BOLT12 Lightning in both directions from day one; cross-pool payments use the same vHTLC/PTLC adaptor construction between two quorums, so Flock pools federate horizontally without trusting each other.
