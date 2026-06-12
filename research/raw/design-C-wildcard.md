# EMBER — Epoch-checkpointed Bearer Rails over Nostr

A Bitcoin L2 combining **Ark-style epoch timeout-trees** (unilateral exit, amortized footprint) with **Spark-style statechain key-tweak transfers inside the tree** (round-free, receiver-offline payments), operated by a **FROST federation whose entire coordination plane — including operator-to-operator signing — is Nostr events**.

## 1. Architecture overview

```
                          BITCOIN L1
   Epoch-pool root UTXO (one per pool, laddered daily, expiry T_exp=4032 blk)
      ├─ pre-signed radix-4 tree (clArk n-of-n, TRUC/v3 + P2A anchors)
      │     └─ leaf outputs: P_leaf = agg(U_k, F_k)  ← on-chain key constant;
      │        (U,F) rotated OFF-CHAIN per payment via FROST share-tweak
      └─ script path: Fed-sweep after CLTV T_exp
──────────────────────────────────────────────────────────────────
   OPERATOR FEDERATION  n=7, FROST 5-of-7; each operator runs one
   public relay + one Blossom server. NO HTTP/gRPC API exists.
──────────────────────────────────────────────────────────────────
   NOSTR RELAYS (untrusted; k-of-m replication, NIP-42/13 gated)
   discovery 39100/39101 │ state commitments 30178 │ signing 21178(eph)
   transfer delivery 1059 giftwrap │ self-backup 30179 │ watchtower 30180
──────────────────────────────────────────────────────────────────
   WALLETS (npub = payment address)      WATCHTOWERS (anyone)
```

Funds live as leaves of pre-signed timeout-trees. A **payment is a statechain transfer**: the federation tweaks its FROST share so control of a leaf moves from sender-key to receiver-key, and sender+federation pre-sign the receiver a new exit package — no round, no receiver interaction, no new on-chain bytes. **Rounds exist only as hourly checkpoints** (refresh/cleanse/expiry), and only forfeiting holders participate. Trees expire every 4 weeks; pools are laddered daily so expiries never synchronize.

**Why this beats straightforward federated Ark-over-Nostr (e.g., OpenArk):**
1. **Payment path has zero round interactivity.** clArk's known killer — receivers with nothing to forfeit can DoS rounds costlessly (Roose, Delving) — vanishes: receivers never touch rounds; every round participant forfeits a leaf (skin in the game).
2. **Native offline receive.** Arkoor needs the receiver online to take delivery and adds one exit-chain tx per hop; Ember's key-tweak keeps exit depth constant per transfer and the receiver can be fully offline.
3. **Capital efficiency.** Arkoor/round sends consume operator liquidity (full vTXO value fronted per round); Ember payments move value *inside* an already-funded tree — operator float covers only checkpoint churn (~10% of pool value vs ~100%).
4. **Bounded trust, not chained trust.** Arkoor chains accumulate trust in *every* prior sender; Ember's exposure is to one specific prior owner + a t-of-n quorum, and is cleansed at the next checkpoint.

The price: we inherit the statechain "unprovable deletion" critique. We pay it knowingly, bound it tightly (§6), and cleanse it on-chain every checkpoint.

## 2. On-chain construction

**Pool root.** Taproot output. Internal key = MuSig2 agg(all round participants' ephemeral keys, Fed FROST key) — the clArk pseudo-covenant (keys deleted post-ceremony). Script path: `<T_exp> CLTV DROP <Fed> CHECKSIG` (one-tx expiry sweep).

**Tree.** Radix-4, 4,096 leaves ⇒ depth 6. Each branch tx: 1 Taproot keyspend input, 4 P2TR outputs + 1 P2A anchor (BIP-433), nVersion=3, zero fee (~253 vB). Pre-signed branch-only (arkd-v0.5 style) at the checkpoint ceremony.

**Leaf output.** `P_leaf = agg(U_k, F_k)` keyspend only. `U_k` = current owner key (BIP-32-derived from the Nostr seed); `F_k` = federation FROST share-aggregate. Transfers tweak: `U_k + F_k = U_{k+1} + F'_{k+1}` — on-chain key never changes; old shares are deleted and **proactively reshared** (ZF-FROST refresh) across all 7 operators after every transfer.

**Exit package** (held by owner, re-issued per transfer):
- *Exit tx*: spends leaf via keyspend (pre-signed by U_k+F_k), zero-fee TRUC, output `O_k` + P2A anchor. ~124 vB.
- `O_k` Taproot: keyspend `agg(U_k, F_k)`; script path `<CSV_k> CSV DROP <U_k> CHECKSIG`.
- *Decrementing relative lock*: CSV₀ = 432 blocks; each transfer decrements Δ=3; floor 72 (12 h) ⇒ **120 transfers per leaf** between checkpoints. Current owner's exit always matures soonest.
- *Forfeit tx* (signed by owner at transfer/refresh): spends `O_k` keyspend to Fed — broadcastable by **anyone** (published in watchtower feed) if a stale exit appears; the CSV window is the dispute period. Forfeit-vs-new-state atomicity uses Bark-style hash-locks (one federation preimage validates both forfeit and new leaf).

**Splits** (change): owner+Fed co-sign a tx spending the leaf into 2–4 child leaves (each with fresh exit packages); adds one exit-chain level; wallets cap depth at 3; checkpoints compress chains back to depth 0.

**Boarding**: on-chain deposit into 2-of-2 (user+Fed) with user-only CLTV refund at T_exp+288; converted to a leaf at the next checkpoint. Optional — most users onboard via the LN bridge or by simply receiving.

**Unilateral exit cost**: worst case alone: 6 branch txs (1,518 vB) + exit (124) + CSV claim (~138) + CPFP child (~110) ≈ **1,890 vB ≈ 19k sats @10 sat/vB**. Fully amortized (everyone exits, branches shared): ≈ **350 vB/user**. Timelock arithmetic: path CSVs are zero on branch txs (only the exit output carries CSV), so worst-case exit latency = confirmation of 8 txs + CSV_k ≤ 432 blocks, with T_exp margin of 432 blocks reserved.

## 3. Nostr protocol surface

All operator endpoints are npubs; every interaction is an event. Operators' relays require NIP-42 AUTH + NIP-13 PoW (20 bits) on writes; clients also fan out via NIP-65 to ≥8 public relays.

| Kind | Class | Content | Wrapping |
|---|---|---|---|
| 39100 | addressable | Federation descriptor: operator npubs, FROST group key, params, fee schedule, relay/Blossom URLs | public |
| 39101 | addressable | Pool descriptor: root txid, T_exp, tree-blob sha256 (full signed tree on Blossom, mirrored on all 7 operators) | public |
| 30178 | replaceable | **State commitment**: epoch, strictly-increasing seq, Merkle root of leaf-ownership map, sig by FROST key. Two conflicting seqs = portable fraud proof | public |
| 10178 | replaceable | User payment capability: federation id, static claim pubkey (zap-config-style, NIP-A3-adjacent) | public |
| 21178 | ephemeral | Signing-session msgs: FROST/MuSig2 nonces & partial sigs — operator↔operator and user↔federation; session-id tagged | NIP-44 |
| 21179 | ephemeral | Transfer/split/refresh RPC (NWC-pattern request/response) | gift wrap (1059) |
| 1059 | regular | Transfer delivery: exit package, Merkle inclusion proof, receipt | gift wrap |
| 30179 | replaceable | Encrypted wallet backup: leaf set + exit packages, self-NIP-44 (key from seed) | self-encrypted |
| 30180 | regular | Watchtower feed: forfeited-exit txids + ready-to-broadcast forfeit txs; "broadcast-exit-at-height-H" deposits | feed public; deposits gift-wrapped |
| 9912 | regular | Payment receipt (PoP scalar commitment); emitted as kind-9735-compatible zap receipt when requested | public (zaps) / wrapped |

Ephemeral kinds carry all nonce material — never stored, never replayable from relays. MuSig2/FROST sessions follow BitGo discipline: delete-on-read secnonces, session-IDs in NonceGen aux input, **abort-and-restart with fresh nonces on any timeout/duplicate** (idempotency via session-id, never nonce reuse). Lossy relays therefore cost only retries, never key leakage.

## 4. Payment flows

**Send (1–3 s).** ① Sender fetches receiver's 10178 + NIP-65 lists. ② Gift-wrapped 21179 transfer request → federation relays (leaf ids, receiver key derived from npub, optional PTLC adaptor point). ③ Operators verify against leaf map, run share-tweak + resharing over 21178 events (5-of-7). ④ Sender co-signs: receiver's new exit tx (CSV decremented by 3) and his own forfeit. ⑤ Federation publishes 30178 (seq+1), gift-wraps the exit package to the receiver's read relays, returns the adaptor-completed scalar to the sender as proof-of-payment, emits 9912/zap receipt.

**Receive while offline.** Steps ①–⑤ need nothing from the receiver — the exit package is signed by sender+federation and pays to the receiver's key unconditionally. On wake: AUTH-fetch 1059s, verify tree sigs + Merkle proof against 30178, verify replication (acks from ≥5 relays + 2 Blossom mirrors) before displaying "final", then **cleanse** at the next checkpoint if value > wallet threshold (default 50k sats).

**Refresh/checkpoint (hourly).** ① Round announce (ephemeral). ② Forfeit intents from participants (each must own a leaf — free-rider DoS impossible). ③ Branch-only MuSig2 ceremony over 21178; staller's branch evicted and restarted, staller merely misses this round. ④ Round tx: inputs = expired pool roots (sweep-and-fund) + operator float; outputs = new tree roots + connectors. ⑤ Hash-lock preimage release atomically validates forfeits and new leaves; wallets update 30179 backups.

**Unilateral exit.** Pull tree blob (Blossom/backup/watchtower), `submitpackage` branch path + exit tx with CPFP on P2A anchors (anyone can fee-bump, keylessly), wait CSV_k, claim via script path. No operator involvement at any step.

**Recovery from seed alone.** Seed → Nostr identity + all leaf keys (BIP-32). ① Fetch 39100/39101 from bootstrap relay list. ② AUTH as self; pull own 1059s + 30179 backups. ③ Verify against latest 30178. ④ Federation alive → cooperative re-issue of exit packages (free); federation dead → broadcast exit packages from backups, or rely on watchtower-deposited exits already queued at height H. **Stale backups are safe**: the worst a stale state can do is broadcast an already-forfeited exit, which the public forfeit in 30180 sweeps — you lose nothing you still own. No LN-style penalty; no toxic state.

## 5. R3 and R4, mechanism by mechanism

**R3 — no SPOF.**
- *Operator failure*: 5-of-7 FROST — two operators can vanish with zero impact; signing, preimage shards, and deletion duty are all thresholdized.
- *Relay failure*: every event class written k-of-m (7 operator relays + ≥8 public via NIP-65); payments aren't final until replication verified; relays can't forge, only withhold — and withholding is detected at finality-check time, not after.
- *User offline*: receiving needs no user liveness (§4). Safety needs none either: signed exit txs are third-party-broadcastable; wallets deposit them with independent watchtowers ("broadcast at height T_exp−576 unless superseded"), so even a user asleep past epoch expiry lands on L1 automatically instead of being swept. Stated liveness bound without watchtowers: **come online once per 28-day epoch**.
- *Seed-only recovery*: §4; no channel state, no counterparty honesty required.

**R4 — zero liquidity management.**
- A brand-new npub receives its first payment with zero on-chain setup: the federation tweaks toward a key the receiver derives from seed. No inbound capacity exists as a concept — leaves are bearer objects, not edges.
- No channels, no rebalancing, no splicing; change handled by federation-co-signed splits; denominations invisible to users.
- Liquidity burden sits with operators only at checkpoints, mostly recycled via sweep-and-fund of expiring laddered pools; the residual float is priced into refresh fees scaled by remaining epoch lifetime (Bark-style cost-of-capital pricing).

## 6. Honest trust & attack self-assessment

**Residual trust windows (explicit):**
- **Received funds, until your next checkpoint participation** (wallet default ≤1 h for leaves >50k sats; hard bound = epoch expiry, 28 d): theft requires **5-of-7 operators deliberately retaining a reshared-away share AND colluding with that leaf's specific prior owner** (on-chain leaf key is constant, so any historical share-pair reconstructs it; only on-chain re-rooting cleanses). This is the statechain deletion problem — unprovable, per the Mercury audit and "Trustodial" critique. Mitigations: proactive resharing makes retention deliberate-by-quorum, not accidental; fraud-proofable 30178 equivocation destroys the federation's business; wallets cap value-at-risk pending cleanse. We state it; we do not hide it.
- **Received leaves also inherit the pseudo-covenant**: tree branches were co-signed by the original round's participants — full key-retention by *all* cosigners of your path + federation could re-sign a conflicting branch. Same cleanse, same bound. Self-refreshed leaves are covenant-safe for you (you co-signed your path and deleted your key — your own deletion suffices).
- **Federation halt**: cannot steal, but can freeze new payments and refuse refreshes; everyone exits within the 432-block margins (mass-exit math in §7). Funds are never lost to a halt, only forced on-chain.
- **Stale-exit games**: old owner broadcasts a stale exit ⇒ its CSV is ≥3 blocks longer than the current owner's; the public forfeit in 30180 sweeps it within the window. Requires ≥1 of (federation, any watchtower, the current owner) to act within 72 blocks — stated watchtower assumption.

**What relays CAN do**: drop/delay/reorder/selectively serve. Consequences: delayed payments (finality rule blocks premature acceptance), lost backups (k-of-m + operator mirrors + watchtower deposits make total loss require simultaneous failure of ~10 independent parties), metadata leakage (gift wrap hides content/sender; recipient p-tags and IPs leak — Tor recommended; operators see the full pseudonymous payment graph, **privacy is below Lightning's onion routing** — acknowledged).

**Jamming/griefing**: no HTLC slots; transfers settle or fail atomically with fees charged on execution; LN-bridge locks carry an upfront unconditional fee and a 40-block timeout; round participation requires forfeiting a leaf; relay writes cost PoW+AUTH. Replacement-cycling-class attacks don't apply: no preimage-gated multi-hop relay; exits are TRUC/P2A packages immune to rule-3 pinning.

**Future work (consensus changes, not assumed):** CTV makes trees self-enforcing (rounds become non-interactive, receiver covenant-trust disappears); CTV+CSFS enables Erk-style perpetual offline refresh, deleting the once-per-epoch liveness bound.

## 7. Economics

**User costs**: transfer fee 0.1% (min 1 sat); cleanse/refresh fee = float cost × remaining epoch fraction (≈10–30 sats typical); LN-bridge 0.3%; exit gas only when exiting.

**Operator model**: example federation, 100k MAU, 20 payments/user/mo, avg 20k sats ⇒ fee revenue ≈ 40M sats/mo (~0.4 BTC). Costs: 7×(server+relay+Blossom) ≈ trivial; checkpoint chain fees ~1 round-tx/hour ≈ 0.7M sats/mo; float ≈ 5 BTC (≈10% of a 50-BTC pool: early cleanses awaiting ladder expiry) at 5%/yr opportunity cost ≈ 2M sats/mo. Healthy margin without custody or yield games; operators are the natural pivot for today's LSPs (R1 migration: same capital, no per-edge lockup).

**Exit costs**: lone worst-case 1,890 vB (~19k sats @10 sat/vB; 3.8k @2); amortized 350 vB/user. **Mass exit**: one 4,096-leaf pool fully unrolled ≈ 1.43 MvB ≈ 3.6 blocks; 100k users ≈ 35 MvB ≈ ~88 full blocks ≈ 15 h of saturation — survivable inside the 432-block dispute margin and the 4,032-block epoch, *because expiries are laddered daily* (≤1/28 of pools deadline-bound on any day) and P2A anchors let third parties fee-bump priced-out users. Acknowledged limit: leaves below ~25k sats are uneconomical to exit above ~10 sat/vB; wallets surface this and steer small balances through the LN bridge during stress instead of on-chain exit.

**R8 surface**: payments address npubs natively; receipts are zap-compatible (9912→9735); LN bridge gives day-one universality — receive BOLT11 invoices offline via federation preimage shards (Spark-precedented), pay any invoice via PTLC-locked leaf transfer to a gateway. Ember is useful to its first user, not just its millionth.
