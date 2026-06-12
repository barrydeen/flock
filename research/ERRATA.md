# Errata and canonical constants (v0.3)

`raw/synthesis-v0.2.md` is the historical baseline; `AMENDMENTS-v0.3.md` records the
adjudicated changes after the verification panel. **Synthesis + amendments together are
normative; where they conflict, the amendments win.** This file is the quick sheet —
all documents MUST match it exactly.

## Corrections to the synthesis (see AMENDMENTS-v0.3.md for rationale)

1. **Tower alert kind** is `8405` (regular range), not 38405.
2. **Per-cell funding transactions** replace the single round tx (A1). A "round" is the
   hourly batch of up to 16 cell txs; forfeits and stealth tweaks commit to `cell_txid`.
3. **Mass-exit cost** is ≈451 vB/user (each TRUC parent needs its own CPFP child), not
   ≈310 vB/user (A4).
4. **Refresh fee is priced per refresh**: 0.5–0.9% of refreshed value **per refresh**
   (break-even ≈0.8% per refresh at 5% APY with sweep_delay 4032 — each refresh locks
   operator capital ~58 days). The earlier "%/month" framings (0.3–0.4 and 0.5–0.9)
   are superseded (A5, A20).
5. **Transfer-log roots (30178) are per-cosign**, not hourly; "checkpoint" = round
   refresh (A6).
6. **Nullifier** = `SHA256("FlockNullifier/v0.3" ‖ outpoint)` — outpoint only (A7).
7. **Recovery-to-sweep gap** is 3,744 blocks (~26 days); dormancy is *tower-protected*
   self-custody, not unconditional (A2).
8. **"Q must unroll to sweep" is false** — the cell-root sweep leaf exists (A2).

## Canonical constants

- Quorum: FROST **4-of-7**, PedPoP DKG, monthly proactive resharing.
- Round: hourly batch of **16 cell txs × 64 leaves** each (radix-4, depth-3 trees).
- Pool expiry: `T_exp = cell-tx height + 4320` (~30 days).
- Recovery tapleaf: `T_exp + 288` CLTV to the checkpointed owner's seed-derived key.
- Quorum garbage sweep: `T_exp + sweep_delay` CLTV; `sweep_delay` is a pool policy
  parameter, canonical default **4032**. Recovery-to-sweep gap: **3,744 blocks (~26 d)**.
- Claim ladder: checkpointed owner nSequence **720**; each arkoor hop −144
  (hop1 576, hop2 432, hop3 288, hop4 144); **max 4 hops**, reset at checkpoint.
  Two distinct defense windows (A22): **forfeit stale window 144–720 blocks** (held
  forfeit vs a refresh-stale claim, by stale hop); **ladder defense deadline 288–720
  blocks** after an unroll (stale prior owner vs an uncheckpointed chain; head-start
  differential 144–576).
- Forfeit txs: nSequence 0, spend old leaf + per-user connector output of the new
  **cell tx**; tower copies include the full ancestor arkoor chain + CPFP children.
- Boarding refund: **288 CSV** (~2 days).
- Receive ban window: no arkoor receives against cells with **< 2016 blocks** to T_exp.
- Wallet refresh deadline: `T_exp − 1008`. Tower height-deposits: no third party needed
  before `T_exp − 720`; at `T_exp − 720` the deposit tower must act. Deposits also fire
  on **conflict detection** (any spend touching the package's ancestor outpoints).
- Standing dormancy deposit: lodged at every checkpoint, `trigger_height = T_exp + 288`,
  replicated to ≥ **3** towers.
- Value-at-risk cap (default): **50,000 sats**, applied to the **aggregate
  uncheckpointed balance** (any accepted receipt crossing it triggers auto-checkpoint
  at the next available round seat; VaR-triggered refreshes take seat priority;
  enforceable online only).
- Minimum leaf denomination: **10,000 sats**.
- Intent bond: `params.intent_bond_sats`, canonical default **1,000 sats**; instrument
  = **ledger debit netted at next service**, with a pre-signed lien tx on the
  refreshing leaf's cooperative path as backstop (A23 — no standalone bond vTXO);
  boarding users exempt (PoW only).
- Tower fee output (contingent): canonical default **~2,000 sats**, riding each of the
  pre-signed **escalating bump children (10 / 50 / 200 sat/vB)** that spend the
  owner's dedicated fee UTXOs (A24); paid at most once, only if a tower acts.
- Ceremony phase timers: **120 s** adaptive; intent PoW **≥ 20 bits (demand capped at 20)**.
- Exit costs (@10 sat/vB): first exiter ≈ **1.6 kvB** (~16k sats); marginal single
  exiter ≈ **310 vB** (~3.1k sats, incl. own CPFP child); each arkoor hop in an exit
  chain ≈ **+235 vB**; script-path input ≈ 126 vB; node tx ≈ 321 vB; claim tx ≈ 193 vB;
  CPFP child ≈ 115 vB. **Mass exit ≈ 451 vB/user ≈ 0.46 blocks per 1,024-leaf pool
  round ≈ 46 blocks per 100k users.**
- Fees: arkoor cosign 1–3 sats + 0.05%; refresh **0.5–0.9% per refresh** (break-even
  ≈0.8% per refresh at 5% APY); gateway swaps 0.1–0.5%; vault pinning ~1–2k sats/month.
- Float model (A20): boarding-funded leaves cost Q ≈ nothing; float ≈ hot_balance ×
  58 d / refresh_cycle. Canonical worked example (single source: economics.md §1.3,
  quoted verbatim by whitepaper §9): 10k users, 100k-sat avg, 50% hot, 23-day cycle ⇒
  float ≈ 12.6 BTC; refresh line sold ≈ at cost; margin lives in flow fees + services
  (pinning modeled at ~30% adoption).
- Dormant balances (A21): no carrying fee; **automatic on-chain exit ~32 days after
  the last checkpoint** at the owner's expense (≈3.1k–16k sats @10 sat/vB + ~2k tower
  fee); "not confiscated", never "not force-exited".
- Churn leverage (A26): ~1,390× theoretical for unthrottled hourly refresh; ≤ 58 d / D
  with minimum dwell D (D = 7 d ⇒ ≈8×).
- Transactions: pre-signed txs (node/claim/forfeit) are nVersion=3 (TRUC), zero-fee,
  P2A anchor, own pre-signed min-fee CPFP child, 1p1c `submitpackage`. Cell txs are
  nVersion=2, fee-paying, **CPFP-only** bumping (no RBF, ever).
- Relays: ≥ 8 per pool, ≥ 3 operator-independent; transfer-log replication ≥ k relays;
  operator-run **archival relays** (plain NIP-01, long retention) listed in 38400.
- Stealth send key: `P_B = B + H(ECDH(a,B) ‖ cell_txid ‖ counter)·G`; fallback
  B = receiver's Nostr pubkey when no 38407/10002 exists.
- Nullifier: `SHA256("FlockNullifier/v0.3" ‖ outpoint)`.
- Session id: `SHA256("FlockSession/v0.3" ‖ pool_id ‖ round_seq ‖ cell_index ‖ attempt
  ‖ sub_attempt ‖ pset_commitment ‖ msg_digest)`; REBUILD increments sub_attempt only.
- Transfer-log root (30178): published per arkoor cosign, seq strictly increasing.
- Velocity bound: ≤ 4 chained transfers per leaf between refreshes; round checkpoint
  capacity 1,024 leaves/hour/pool.

## Event kinds (canonical)

| Kind | Class | Purpose |
|---|---|---|
| 38400 | addressable | Pool advertisement (quorum-signed; params incl. fees, sweep_delay, intent_bond_sats, archive_relays, optional uncheckpointed_ceiling) |
| 38401 | addressable | Round/cell announcement + phase deadlines |
| 31401 | addressable | Operator heartbeat + key-deletion attestation (reputational/legal weight only) |
| 24400 | ephemeral | Refresh intent (bonded, PoW-capped, gift-wrapped to Q) |
| 24401 | ephemeral | Cell proposal incl. cell-tx template (NIP-44 to cell members) |
| 24402 | ephemeral | MuSig2 pubnonces (cohort sessions only) |
| 24403 | ephemeral | MuSig2 partial signatures (cohort sessions only) |
| 24404 | ephemeral | Intra-quorum FROST / DKG / reshare traffic |
| 24405 | ephemeral | Arkoor cosign request/response (gift-wrapped; no MuSig2) |
| 24406 | ephemeral (inner) | Arkoor payment package, carried in kind-1059 gift wrap |
| 38402 | addressable | Round result: all cell txids + per-cell inline tree commitments |
| 30178 | addressable | Transfer log (quorum-signed Merkle roots, per-cosign, increasing seq) |
| 37400 | addressable | Wallet vault (self-encrypted, unlinkable author key) |
| 38404 | addressable | Watchtower height-deposit (height **and conflict** triggered) |
| 8405 | regular | Tower alert *(corrected from 38405)* |
| 38406 | addressable | Service offer book: gateways **and towers** |
| 38407 | addressable | Flock payment descriptor (stealth base key B, preferred pools, relays) |
| 9735 | regular | Zap receipt (NIP-57 interop) |
