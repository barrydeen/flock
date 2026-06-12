# Flock v0.2 — Federated Ark over Nostr (Consolidated Architecture)

## 1. Executive summary

Flock is a Bitcoin Layer-2 successor to Lightning: covenant-less Ark pools operated by a k-of-n FROST quorum, with **all** off-chain coordination, state backup, and watchtower traffic carried as Nostr events over untrusted public relays. Users receive instantly to an npub with zero on-chain setup, no channels, and no liquidity management; the capital burden moves to operators and is explicitly priced. v0.2 eliminates the contest's worst flaw: the unproven FROST-inside-MuSig2 composition is **removed** — every quorum-gated output now uses a tapscript requiring two independent BIP-340 signatures, so Q signs only plain FROST-Schnorr (RFC 9591 / FROSTR-class, shipped) and MuSig2 runs only among user cohorts with CSPRNG nonces. Refresh atomicity is rebuilt on Ark-style **connector outputs** (transaction-graph binding) instead of asserted hash-locks. Stale-exit defense becomes a Spark-precedented **decrementing relative-timelock claim ladder** — race-free and valid even under a dead quorum — replacing the reactive 144-block contest. A rotating **CLTV recovery tapleaf** lets dormant funds degrade to plain self-custody instead of being swept at expiry, and enables seed-only recovery once a tree is unrolled. Round liveness is restructured: one 1024-cosigner ceremony becomes 16 independent 64-leaf **cells** with post-nonce participant finalization and forfeitable intent bonds, so one aborter delays one cell by one attempt. The trust-downgraded delegated refresh is **deleted**; offline users are protected by sender-funded watchtower height-deposits plus the recovery leaf. Residual trust is explicit and bounded: uncheckpointed arkoor receipts are trusted (sender plus k-of-n collusion) until the next checkpoint, capped by a wallet value-at-risk policy; checkpointed funds are theft-proof with unilateral exit. Open problems are conceded, not glossed: operator payment-graph visibility, data availability of uncheckpointed receipts, mempool replacement cycling, and a structural ~0.3–0.5%/month carrying cost on actively refreshed balances.

## 2. Architecture

**Roles.**
- **Operators O1..O7** (independent orgs, diverse ASNs/jurisdictions, hidden services): hold FROST shares of group key **Q** (k=4 of n=7 — theft needs 4 colluding, liveness survives 3 down). PedPoP DKG at genesis; monthly proactive resharing over signed, epoch-tagged transcripts.
- **Relays** (≥8 per pool, ≥3 mandated operator-independent): untrusted transport and best-effort storage. Clients additionally use a hard-coded diverse bootstrap set.
- **Wallets**: hold a seed → Nostr key + Bitcoin xprv; otherwise stateless (vault-recoverable). Enforce value-at-risk and finality policies client-side.
- **Watchtowers**: permissionless, keyless; execute height-deposits ("broadcast package P at height H unless superseded") and CPFP via P2A.
- **LN gateways**: vHTLC/PTLC swap providers; existing LSPs can pivot here with the same capital and no per-edge lockup (LSP migration path for R1).

```
                          BITCOIN L1
   round tx (hourly/pool: 16 cell roots + connectors)   forfeits/claims/sweeps
            ▲                                   ▲
            │   FROST 4-of-7 quorum Q — plain BIP-340 sigs only
            └───────────────┬───────────────────┘
                            │ ALL traffic = Nostr events
 ┌──────────────────────────┴───────────────────────────────────┐
 │  PUBLIC NOSTR RELAYS (untrusted; ≥8/pool, ≥3 non-operator)   │
 │  pool ads · cell ceremonies · arkoor gift wraps · Merkle     │
 │  checkpoint log · vaults · height-deposits · gateway offers  │
 └──┬───────────────┬──────────────────┬───────────────┬────────┘
  Alice           Bob (OFFLINE        Watchtowers     LN Gateways
  (sender)        receiver: npub only) (keyless CPFP + (vHTLC ↔ BOLT11/12)
                                       height-deposits)
```

A pool's hourly **round tx** funds 16 independent 64-leaf pre-signed trees ("cells", radix-4, depth 3) expiring at **T_exp = funding height + 4320 (~30 d)**. Payments are out-of-round (**arkoor**) cosigns: instant, no receiver liveness, no per-payment operator capital. Rounds serve only refreshes/boarding (senders = receivers).

## 3. On-chain construction

All quorum-gated spends use script paths with **two separate signatures** (`<X> OP_CHECKSIGVERIFY <Q> OP_CHECKSIG`): users/cohorts sign standard MuSig2 or single-key Schnorr; Q signs the same digest independently via plain FROST. No nonce coupling between the two protocols exists anywhere.

**Round tx** (per pool per hour; inputs: Q capital + boarding outputs + forfeited value): outputs = 16 **cell roots**, per-user **connector outputs** (dust-sized, one per refreshing user), Q change.

**Cell root / node outputs**: internal key = NUMS (unspendable). Tapscripts: (a) *tree path*: `<AggU_branch> CHECKSIGVERIFY <Q> CHECKSIG` where AggU = MuSig2 aggregate of the users below that branch (each user cosigns only their root-to-leaf path, depth 3, then deletes the ephemeral key — their own deletion binds the pseudo-covenant); (b) *node sweep*: `<T_exp+4032> CLTV DROP <Q> CHECKSIG` (late garbage collection only).

**vTXO leaf output**: NUMS internal key. Tapscripts: (a) *cooperative*: `<U> CHECKSIGVERIFY <Q> CHECKSIG` — used for arkoor, forfeits, offboards, and pre-signing claims; (b) *recovery leaf*: `<T_exp+288> CLTV DROP <U_ckpt> CHECKSIG` paying the **checkpointed** owner's seed-derivable key — dormancy degrades to self-custody; (c) *leaf sweep*: `<T_exp+4032> CLTV DROP <Q> CHECKSIG`. There is **no** bare CSV exit script (it would let stale owners bypass the ladder).

**Claim ladder (replaces the 144-CSV contest).** Every owner holds a pre-signed *claim tx* spending the leaf's cooperative path to their own key, with consensus-enforced BIP-68 nSequence: checkpointed owner (hop 0) nSequence = 720; each arkoor hop decrements by 144 (hop 4 = 144; hop cap 4, reset at every checkpoint). The **current owner's claim matures strictly before any prior owner's** once the leaf is on-chain — no mempool race, no live quorum needed. Q cosigns the claim atomically with each arkoor transfer; a transfer without its claim is invalid and the receiver rejects it.

**Forfeit tx** (refresh atomicity, replaces hash-locks): inputs = old leaf (cooperative path, pre-signed in-ceremony) + a **connector output of the new round tx**. The forfeit is consensus-invalid unless the round tx that actually contains the user's verified new leaf confirms (the connector outpoint commits to that exact txid). Distinct connector per user; no shared preimages; nothing to withhold. Forfeits have nSequence 0, pay Q, and are broadcastable by **anyone** (watchtowers hold copies) — so they beat every laddered claim (min 144) even if Q later dies.

**Boarding output**: `<U> CHECKSIGVERIFY <Q> CHECKSIG` + refund `288 CSV <U>` (~2 days, shortened from 30 — bounds the boarding-strand window); converted in the same round it confirms when possible.

**Fees/pinning**: all tree/claim/forfeit txs are nVersion=3 (TRUC), zero-fee, P2A anchor; the exiter pre-signs its own minimum-feerate child at creation (the anchor is never a free slot) and submits 1p1c via `submitpackage`; CPFP funded from dedicated UTXOs that cannot be conflicted.

**Exit cost** (script-path inputs ≈126 vB): depth-3 unroll = 3 node txs ≈ 321 vB each + claim ≈ 193 vB + CPFP children ≈ 460 vB ⇒ **first exiter ≈ 1.6 kvB (~16k sats @10 sat/vB)**; path already unrolled ⇒ **≈ 310 vB (~3.1k sats)**. Each arkoor hop in an exit chain adds ≈ 235 vB.

## 4. Nostr protocol surface

Placeholder kinds; NIP-01 semantics; NIP-44 encryption; NIP-59 gift wrap; NIP-65 outbox.

| Kind | Type | Content | Visibility | Retention requirement |
|---|---|---|---|---|
| 38400 | addressable | Pool ad: Q key, operator npubs, k/n, fees, relay set, T_exp ladder | Public, quorum-signed | Life of pool + 60 d |
| 38401 | addressable | Round/cell announcement, phase deadlines | Public | Until round closes |
| 31401 | addressable | Per-operator heartbeat + signed key-deletion attestation (ephemeral tree keys, reshare epochs) | Public, per-operator-signed | Rolling 90 d (fraud evidence) |
| 24400 | ephemeral | Refresh intent: vTXO ownership proof + bonded intent fee, PoW ≥ 20 bits (capped) | Gift-wrapped to Q | None |
| 24401 | ephemeral | Cell proposal: txs, canonical participant-set commitment, session-id | NIP-44 to cell members | None |
| 24402/3 | ephemeral | MuSig2 pubnonces / partials (BIP-373 fields), transcript-signed | NIP-44, session-bound | None |
| 24404 | ephemeral | Intra-quorum FROST + DKG/reshare msgs, epoch-tagged, ACKed | NIP-44 operator↔operator | None |
| 38402 | addressable | Round result: txid, **inline** tree commitment (Blossom mirror optional, never sole copy) | Public | Life of round + 90 d |
| 30178 | addressable | **Checkpoint log**: quorum-signed Merkle root over (nullifier=H(outpoint‖seq), commitment) pairs, strictly increasing seq; no cleartext amounts | Public | Life of pool; ≥k-relay replication proof |
| 1059→24406 | gift wrap | Arkoor package: tx chain + sigs to confirmed round + Q-cosigned laddered claim + inclusion proof | Wrapped to one-time key | Receiver-paid pinning; Q archive (ciphertext) |
| 37400 | addressable | Wallet vault: vTXO set, exit packages, history | NIP-44 self-encrypted, authored by seed-derived unlinkable key, padded to size buckets | Paid pinning SLA + Q archive + local |
| 38404 | addressable | Watch height-deposit: "broadcast P at height H unless superseded" + fee budget | Encrypted to per-request tower keys, sharded | Until H + 288 |
| 38405 | regular | Tower alert: conflict / sweep anomaly / broadcast done | Public | 30 d |
| 38406 | addressable | Gateway offer book (NIP-69-style) | Public | Until superseded |
| 9735 | regular | Zap receipt (NIP-57) | Public | Client norm |

**Nonce discipline (load-bearing, corrected):** all MuSig2/FROST round-1 nonces are **CSPRNG-fresh**, RAM-only, never persisted, never re-derived — deterministic session-keyed derivation is banned (it enabled key extraction via induced re-signing). One partial per secnonce ever; on crash/timeout/duplicate, abort and restart with fresh nonces; sessions are idempotent via session-ids, never via nonce reuse. A restored device cannot reuse a nonce because nonces are never derivable. All signers sign and cross-verify a canonical session transcript (full pubnonce set, signer set, message) before emitting partials; any divergence aborts. Same rules for adaptor/PTLC signing (one finalize per nonce, journal-recorded).

## 5. Payment flows

**Send (Alice→Bob, ~1–2 s).** Alice derives a per-payment key **P_B = B + H(ECDH(a,B) ‖ round_txid ‖ counter)·G** (per-payment tweak fixes reuse linkage). She builds the arkoor tx (payment vTXO + self-made change), signs cooperatively; Q verifies no nullifier conflict, increments seq, cosigns the transfer **and** the receiver's laddered claim, publishes the 30178 root. Alice gift-wraps the package to Bob's read relays + pool relays, files a funded 38404 height-deposit for Bob (mandatory part of send), and obtains ≥k-relay replication receipts. Proof-of-payment = Merkle inclusion proof under a quorum-signed root.

**Receive while offline.** Bob needs only an npub — no on-chain setup, no liquidity (R4). The wrap waits on relays + Q's archive; the sender-funded height-deposit already protects the funds toward expiry. On waking, Bob: fetches wraps; verifies the sig chain to a confirmed round tx (L1 via compact filters/explorers); fetches 30178 roots from **disjoint relay sets including operator-independent relays, failing closed** on absence; verifies inclusion + replication on ≥k mirrors; persists the package locally. Funds above the **value-at-risk cap (default 50k sats)** are auto-checkpointed at the next round; below it, display as "uncheckpointed" until refreshed. Receives are **refused** against rounds with <2016 blocks to T_exp (sender must refresh first) — bounding the recovery-leaf stale-owner window.

**Refresh / checkpoint (hourly; wallet deadline T_exp − 1008).** Intent (bonded, PoW) → assignment to a 64-leaf cell → verify own leaf/path and the canonical participant-set commitment in the proposal → MuSig2 nonces, then **participant set finalized post-nonce; non-responders excluded and the cell rebuilt in the same attempt** → partials (120 s adaptive phase timers) → sign forfeit spending old leaf + own connector of the proposed round txid → Q broadcasts; confirmation simultaneously activates the new leaf and makes the forfeit valid (connector atomicity — no reveal step to withhold, and a mutated round tx invalidates the forfeit, leaving the old vTXO intact). New leaf: fresh 30-day clock, ladder reset to hop 0, recovery tapleaf rotated to current owner, zero residual trust. Abort-after-nonce slashes the intent bond.

**Unilateral exit.** Broadcast path txs as 1p1c TRUC packages with pre-signed min-fee children; CPFP via P2A (anyone can fund); claim matures after its laddered nSequence (144–720 blocks). No operator cooperation; stale prior owners are consensus-blocked by later locktimes, and live Q (or any tower holding a forfeit) beats refresh-stale leaves immediately.

**Recover from seed alone.** Seed → Nostr key + xprv. Fetch kind-10002 + 37400 vault (derived author key) from bootstrap relays; fall back to Q archives (ciphertext, useless to operators); rebuild vTXO set; cross-check 30178 + L1. If *all* vault copies are gone: checkpointed funds are still recoverable via the **recovery tapleaf** after T_exp+288 once the tree is unrolled (anyone can unroll; Q must unroll to sweep at T_exp+4032, and towers unroll for bounties), using only the seed-derived U_ckpt key. Stale vaults are safe — exiting an already-forfeited leaf only loses an already-spent coin. No toxic state. Conceded: uncheckpointed arkoor receipts are **not** seed-recoverable (§9).

## 6. Trust model and attack analysis

**Explicit residual-trust time bounds.**
- **Uncheckpointed arkoor**: theft requires sender + 4-of-7 operators. Bound: until next checkpoint — auto at next hourly round above 50k sats; hard bound T_exp (receive banned in final 2016 blocks). Equivocation = two signed 30178 roots at one seq: portable fraud proof.
- **Offline safety**: none required until T_exp − 720, when height-deposit towers exit or checkpointed value rides to the recovery leaf (T_exp+288 → self-custody). Worst case for a never-online receiver: tower-dependent until T_exp − 720.
- **Recovery-leaf stale window**: uncheckpointed value whose holder stays offline past T_exp+288 is claimable by the prior checkpointed owner — bounded by the VaR cap and the receive-near-expiry ban.
- **Swaps**: standard HTLC exposure for the 144-block CLTV delta.

| # | Attack (panel) | v0.2 mitigation | Residual risk |
|---|---|---|---|
| 1 | Deterministic nonce reuse extracts keys/shares (critical) | CSPRNG-only nonces, RAM-only, never re-derived; one partial per nonce; abort-and-fresh on any retry | Implementation bug class; mitigated by spec-level ban + test vectors |
| 2 | FROST-of-MuSig2 leaks Q (critical) | **Composition removed**: two-sig tapscripts; Q signs plain FROST; transcript canonicalization; bounded concurrent sessions | Standard-crypto risk only; larger witnesses (~+70 vB/input) |
| 3 | Eclipsed 30178 + arkoor double-spend (critical) | Fail-closed disjoint-relay-set checks, ≥k-mirror replication receipts, Merkle inclusion proofs, VaR cap | Withholding-DoS; global network-position eclipse; bonded slashing needs trusted arbitration (open) |
| 4 | Hash-lock atomicity break / mass forfeit (critical) | **Connectors**: forfeit spends a per-user output of the exact confirmed round txid | None at script level; user must verify L1 confirmation honestly (see #11) |
| 5 | Dead quorum → stale exits rob receivers (critical) | Decrementing-nSequence claim ladder (consensus-ordered); forfeits permissionless, nSeq 0 | Refresh-stale leaves need *someone* to broadcast the held forfeit within 144 blocks of unroll |
| 6 | Delegated-refresh sock-puppet theft (critical) | **Feature deleted**; tower height-deposits + recovery leaf replace it | Offline >~3.5 weeks ⇒ on-chain exit fees or wait-for-recovery latency |
| 7 | Single-aborter round collapse (critical) | 64-leaf cells; post-nonce set finalization (rebuild same attempt); slashed intent bonds; 120 s timers | Malice ≈ packet loss; honest losers pay bonds occasionally; griefing cost > 0 but defense imperfect |
| 8 | DA stranding of arkoor receipts (critical) | Replication-verified finality; mandatory local persistence; Q ciphertext archive; auto-checkpoint >50k | **Conceded**: long-offline uncheckpointed receipts strandable by coordinated purge+archive denial (§9) |
| 9 | Operator payment-graph surveillance (critical) | 30178 holds commitments/nullifiers only; per-payment stealth tweak; vaults unlinkable | **Conceded**: Q still sees amounts/links at cosign time; blinded cosigning is open (§9) |
| 10 | Refresh censorship → T_exp sweep confiscation | Recovery leaf matures (T_exp+288) **before** Q sweep (T_exp+4032); height-deposits; multi-relay publication receipts | Recovery needs tree unroll; resourced adversary can tax (not confiscate) via fees |
| 11 | Round view-split captures inputs | Signed canonical participant-set commitment; forfeit binds to round txid via connector; verify confirmed output script vs local recomputation before treating refresh final | Eclipsed/SPV-light L1 view; mitigate with compact filters + multiple explorers |
| 12 | Recovery-leaf names stale pre-arkoor owner | Receive ban in last 2016 blocks; auto-checkpoint; VaR cap; tower exit before T_exp | Sub-cap value of receivers offline past T_exp+288 stealable by prior owner |
| 13 | Forfeit race / TRUC pinning / replacement cycling | Ladder removes owner-vs-owner races; forfeits valid instantly; pre-signed own CPFP child; dedicated fee UTXOs; 1008-block margins | Replacement cycling unsolved at consensus level; fee-spike delay risk remains (§9) |
| 14 | PTLC adaptor-secret leak | Fresh journaled nonces; verify adaptor point + challenge; one finalize per nonce | Silent catastrophic failure class; conservative engineering only |
| 15 | Never-online receiver confiscated at T_exp | Sender-funded 38404 mandatory at send; recovery leaf; Q sweep delayed 4032 blocks | Tower market thinness; sweep of truly abandoned sub-dust residue |
| 16 | Quorum-offline expiry coercion (DoS) | 4-of-7, AS/geo-diverse, hidden services; 31401 heartbeats trigger pre-emptive exits; recovery-leaf safety net | Funded adversary can still force pool-wide exit (fees, not theft) |
| 17 | Sybil dust seats / dust flooding / slot starvation | Min leaf 10k sats; bonded intents; per-key quotas; flat fee floor; log pruned to live nullifiers | Anti-dust floor hurts micropayments; sub-floor change rides uncheckpointed |
| 18 | Exit-storm fee spike strands dust | Min denomination; branch-batched exits; fee-oracle pre-emptive exits; operator emergency batching | Value ≈ exit cost is strandable on demand (§9) |
| 19 | NIP-65 / relay-set eclipse; coordination censorship | Pinned diverse bootstrap relays; operator-independent attestation required; L1 cross-check; staleness alarms; timeout-to-exit | ISP/state-level partitioning of a target persists |
| 20 | PoW/AUTH censorship chokepoint | Capped PoW; pre-mined PoW; many-relay submission; signed proof-of-write receipts; non-paid fallback | Deniable per-npub slowdowns remain possible |
| 21 | FROST resharing/DKG split view | Epoch-tagged signed transcripts, ACKed, complaint rounds; reject cross-epoch material; verify group key invariant | Relays give no ordering; resharing can brick liveness (never safety) |
| 22 | Boarding strand | 288-block refund CSV; same-round conversion; no boarding during degraded heartbeats | ≤2-day strand window per deposit |
| 23 | Blossom/tree-commitment DA loss | Inline commitments mandatory; local path persistence at signing; k-mirror replication | External mirrors are convenience only |
| 24 | Gift-wrap spam; vault metadata; tower harvesting | PoW-tagged wraps + pre-auth filter tag; padded vaults under derived keys; per-request tower keys, sharded packages, keyless CPFP towers | Trial-decrypt cost under flood; coarse wealth buckets leak; colluding towers recombine shards |
| 25 | Gateway free-option / vHTLC cycling | Upfront reservation fees; per-id in-flight caps; short randomized CLTVs; PTLC preference; rebroadcast | Bridge liquidity lockable for CLTV windows |

**What Q can do**: censor (→ unilateral exit, never frozen); see the payment graph (conceded); with 4-of-7: double-spend *uncheckpointed* arkoor with the sender. **Cannot**: confiscate or reverse checkpointed vTXOs, mint state, sweep before T_exp+4032, or beat the claim ladder. **What relays can do**: drop/delay/reorder (sessions abort safely; finality fails closed), read metadata (bounded by stealth keys, wrapped events, Tor), spam-gate (capped, receipt-audited).

## 7. Economics

**Operator model.** Revenue: (1) refresh fees priced on **capital-lock duration** (not remaining lifetime — kills the churn-amplification lever): ~0.3–0.4%/month on refreshed balances against ~5% APY cost of capital; minimum dwell time and per-leaf rate limits cap fronted-capital leverage; (2) arkoor cosign fees 1–3 sats + 0.05% (near-pure margin); (3) gateway swap fees 0.1–0.5%; (4) paid relay/vault pinning subscriptions (~1–2k sats/mo). A 10k-user pool at 100k-sat average needs ~10 BTC recycling float — honest framing: a capital-intensive LSP-like business; existing Lightning LSPs can migrate with the same balance sheet and better capital efficiency (no per-edge lockup).

**User costs.** Receive: free. Send: 1–3 sats + 0.05%. Hot (refreshed) balances: ~0.3–0.4%/month. **Dormant balances: zero** — skip refreshes entirely, degrade to self-custody via the recovery leaf; no forced exit, no monthly-app-open requirement (fixes the flagged carrying-cost/idle-confiscation flaw for savings; the carrying cost now applies only to actively transacted funds, where it substitutes for LN channel/rebalance amortization).

**Exit costs.** First exiter ≈1.6 kvB (~16k sats @10 sat/vB); marginal exiter ≈310 vB (~3.1k sats); +235 vB per arkoor hop (≤4).

**Mass exit.** Per 1024-leaf pool: ~336 node txs + 1024 claims + CPFP ≈ **0.33 blocks (~310 vB/user)**; 100k users ≈ ~33 blocks, spread across hourly-laddered T_exp rungs with a 1008-block margin and keyless P2A bumping. Expensive and slow but safe: pre-signed, pinning-hardened, counterparty-free, and ladder-ordered.

## 8. Comparison

| Dimension | Flock v0.2 | Lightning | Ark (single op) | Spark | Cashu/Fedimint |
|---|---|---|---|---|---|
| Operator trust | 4-of-7 theft; uncheckpointed window only | None (counterparty risk) | 1 op same window | k-of-n statechain-style | Full custody (blind) |
| Unilateral exit | Always, pre-signed, ladder-ordered | Yes (needs channel state) | Yes | Yes (timelock ladder) | No |
| New-user receive | Zero setup, offline OK | Needs inbound liquidity | Zero setup | Zero setup | Zero setup |
| Receiver liveness | None (towers + recovery leaf) | Online or LSP | Round participation | None | None |
| Seed-only recovery | Checkpointed: yes (recovery leaf); receipts: no | No (toxic state) | No | Partial | Backup-dependent |
| Idle-funds cost | Zero (dormancy → self-custody) | Channel reserve/mgmt | ~0.5%/mo or sweep | Low | Custodial risk |
| On-chain/user | ~310 vB amortized | 2 txs/channel | Similar to Flock | Low | ~0 |
| Privacy | Q sees graph; public sees commitments | Onion-routed (better) | Op sees graph | Op sees graph | Best (blind sigs) |
| Infra dependency | Public Nostr relays only | P2P gossip + watchtowers | Op API | Op API | Mint/federation API |

## 9. Open problems

1. **Operator payment-graph visibility** — Q necessarily relates spends for double-spend prevention; blinded cosigning compatible with fraud proofs is unsolved. Privacy is below Lightning's.
2. **Uncheckpointed-receipt data availability** — arkoor exit data is not seed-derivable; coordinated relay purge plus archive denial strands long-offline receipts. Structural; bounded only by VaR cap, replication receipts, and auto-checkpointing.
3. **Replacement cycling / fee-spike timing** — no complete mempool-level fix; deadline-adjacent exits can be delayed at cost; defense is margins, dedicated fee UTXOs, and tower redundancy.
4. **Dust** — value near its ~310 vB exit cost is economically strandable; the 10k-sat leaf floor pushes micropayments into uncheckpointed change.
5. **Grief vs. loss indistinguishability** — ceremony withholding cannot be told apart from relay loss; bonds tax honest unlucky users or under-deter attackers.
6. **Accountability without covenants** — operator "slashing" reduces to portable fraud proofs + reputation/legal recourse; on-chain bonds need trusted arbitration.
7. **Structural carrying cost** — ~0.3–0.4%/month on hot balances is intrinsic to covenant-less Ark capital fronting; competitive for payments balances, not for savings (savings go dormant instead).
8. **Recovery-leaf stale window** — sub-cap uncheckpointed value of receivers offline past T_exp+288 is claimable by the prior owner.
9. **Quorum liveness** — <k operators halts new payments and forfeit *creation* (exit-only mode); held forfeits still need a broadcaster within 144 blocks of a stale unroll.
10. **Watchtower market thinness** — height-deposits make action automatic, but a real fee-budgeted tower economy is unproven.

## 10. Future work under covenants

CTV would make tree expansion non-interactive: no MuSig2 ceremonies, no ephemeral-key-deletion assumption, no cells, no aborter griefing — rounds become a single Q broadcast, and offline users refresh trustlessly (Erk-style), deleting open problems 5 and most of 9. CSFS would let leaves verify quorum signatures over *future* state, enabling consensus-enforced forfeit binding, true per-hop recovery-leaf rotation (closing problem 8), and compact rebindable exits. CTV+CSFS together would roughly halve exit vbytes and remove every "someone must broadcast in time" dependency, leaving only the economic and privacy problems. None of this is assumed anywhere above.
