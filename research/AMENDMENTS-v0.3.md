# Amendments v0.3 — adjudicated resolutions to the verification panel's findings

Status: NORMATIVE. These amendments override `raw/synthesis-v0.2.md` and the v0.2 drafts
wherever they conflict. ERRATA.md has been updated to match. All docs carry the status
line "Flock protocol — DRAFT v0.3 — 2026-06-12".

Each amendment names the defect it resolves (panel lens in brackets).

## A1. Per-cell funding transactions (replaces the single round tx)

**Defect** [bitcoin-validity]: all 16 cells shared one round tx, so any cell rebuild
changed the round txid and invalidated every other cell's pre-signed node txs, forfeits,
and boarding signatures — cell isolation was fictional.

**Resolution**: each cell is funded by its **own on-chain cell transaction** (inputs:
Q capital + that cell's boarding outputs; outputs: 1 cell root + up to 64 per-user
connectors + Q change). The hourly **round** is now the *batch* of up to 16 cell txs.
Forfeits commit to their **cell txid** via their connector input. A rebuild re-signs one
cell tx only. The stealth-tweak formula uses `cell_txid` in place of `round_txid`.
Kind 38402 (round result) lists all cell txids with per-cell inline tree commitments;
kind 24401 (cell proposal) carries the **cell tx template** (nVersion=2, fee-paying),
while all pre-signed txs (node/claim/forfeit) remain nVersion=3 zero-fee P2A.
Cell-tx fee-bumping is **CPFP-only** (spend Q change). RBF is forbidden — any txid
change voids the pre-signed graph. Wallet verification steps that referenced "round
txid" now verify their **cell txid** after that cell's SET-FINAL.

## A2. Dormancy is tower-protected, not unconditional; "Q must unroll to sweep" is false

**Defect** [requirements, attack-coverage, cold-read — critical]: cell roots carry a
sweep leaf, so Q sweeps a never-unrolled cell **at its root** at `T_exp+4032` without
ever materializing leaf-level recovery tapleafs. "Zero-cost dormancy / never swept /
Q must unroll to sweep" were false. Also "432 blocks (~3 days)" is an arithmetic error:
the recovery-to-sweep gap is **3,744 blocks (~26 days)**.

**Resolution**:
1. Delete every "Q must unroll to sweep" claim. The sweep-vs-recovery race is real.
2. New normative refresh step: at **every checkpoint**, the wallet pre-signs its
   **recovery claim** (recovery tapleaf → a seed-derived cold key) and lodges
   {unroll branch + recovery claim + CPFP children} as a self-funded kind-38404
   height-deposit with `trigger_height = T_exp+288`, replicated to ≥M towers
   (M = 3 default). Dormancy therefore degrades to **tower-protected self-custody**.
3. Restated guarantee (use this wording): *"A dormant wallet pays nothing and need
   never come online; its funds are exited by its standing tower deposits at
   T_exp+288. Absolute worst case — the owner stays absent AND no tower acts within
   the 3,744-block (~26-day) window — the quorum sweep at T_exp+4032 confiscates."*
4. The comparison table's "Idle-funds cost: Zero" becomes "≈0 (standing tower
   deposit); confiscation only if owner and all towers fail to act for ~26 days
   after recovery maturity". Extend open problem 10 (tower-market thinness) to note
   it now also carries the dormancy guarantee.

## A3. Stale-owner theft against offline receivers; conflict-triggered deposits; finality framing

**Defect** [requirements, bitcoin-validity, cold-read — critical]: the claim ladder
orders maturity, it does not preclude stale claims. A prior owner (e.g. the sender)
acting **alone** can unroll early and outwait an offline receiver's 144-block-per-hop
head start. Height-deposits were height-triggered only, and the "mandatory" sender
deposit is filed by the attacker.

**Resolution**:
1. **Conflict trigger** added to kind 38404 and the tower protocol: towers MUST watch
   L1 for any spend touching the deposited package's ancestor outpoints (cell root,
   node outputs, leaf) and broadcast the deposited package **immediately** upon
   detecting an unroll or conflicting spend — not only at `trigger_height`.
2. Tower copies of defensive packages MUST include the **full ancestor arkoor chain**
   and per-tx CPFP children (TRUC 1p1c ⇒ serial confirmation; state it). The
   response window is **144–720 blocks depending on which hop attacks** (hop-k stale
   owner vs hop-0), not a flat 144.
3. Q SHOULD additionally lodge a deposit for every arkoor cosign it performs
   (it holds the claim it cosigned); sender deposits remain mandatory protocol rules
   but are unenforceable against a never-online receiver — concede this.
4. **Finality framing** (use everywhere): an uncheckpointed arkoor receipt is **not
   accepted** until the receiver's fail-closed wake-up verification passes. A sender
   clawing back value before acceptance is a *failed payment*, not theft of accepted
   funds. Merchants requiring finality MUST require checkpoint inclusion or verify
   online (including deposit existence). The §8 residual-trust row becomes:
   *"uncheckpointed receipts: revocable by the sender alone if the receiver remains
   offline beyond the ladder differential after a hostile unroll and no tower acts;
   revocable by sender + 4-of-7 collusion otherwise; extinguished at checkpoint."*
5. Whitepaper §1.3's "sub-second irreversible finality" bar: concede explicitly that
   arkoor finality (1–2 s, collusion-bounded until checkpoint) is **weaker than
   Lightning's payee finality** and why the design accepts this.

## A4. Mass-exit cost corrected (TRUC packaging)

**Defect** [bitcoin-validity]: batch-bumping violates TRUC/ephemeral-dust rules
(1p1c only) — each of the ~1,360 parents needs its own ≈115 vB child.

**Resolution**: canonical mass-exit figures are now **≈451 vB/user, ≈0.46 blocks per
1,024-leaf pool round, ≈46 blocks per 100k users**. First-exiter (≈1.6 kvB) and
marginal single-exiter (≈310 vB incl. child) figures stand. ERRATA updated.

## A5. Economics re-derived around true capital turnover

**Defect** [requirements, cold-read]: capital fronted for a round is recovered at the
**root sweep** (`T_exp+4032`), so float turns over in ~58 days, not ~30; dormant
leaves occupy float too; headline refresh revenue (0.3–0.4%/mo) sat at or below the
5%-APY cost of capital.

**Resolution**: model capital lock as `(4320 + sweep_delay)/144` days with
`sweep_delay` a pool policy parameter (canonical default 4032). Float-to-hot-balance
ratio ≈ 1.9. Break-even refresh fee at 5% APY ≈ **0.8%/month**; canonical fee band
becomes **0.5–0.9%/month on actively refreshed balances** (parameterized by APY and
sweep_delay, cross-subsidized by arkoor/gateway/pinning lines — show the full stack
margin reconciliation in whitepaper §9 with the float-ratio derivation inline).
Open problem 7's magnitude is updated accordingly. Frame honestly: competitive with
Lightning's channel open/close/rebalance amortization for *active payment balances*;
savings go dormant (≈0 cost) instead. Note the structural trade-off: a shorter
`sweep_delay` cheapens refreshes but shrinks the dormancy grace window (A2).

## A6. Checkpoint-log cadence: per-cosign incremental roots

**Defect** [cold-read]: docs said both "hourly roots" and "root published inside the
1–2 s send".

**Resolution**: kind-30178 roots are published **per arkoor cosign** (seq strictly
increasing, one root per cosign batch). Terminology note everywhere: a **checkpoint**
is a round refresh (extinguishes residual trust, hourly opportunity); the **transfer
log** (30178) is the per-cosign anti-equivocation spine. Receivers always verify
against a fresh root; replication requirement applies per root.

## A7. Nullifier derived from outpoint alone

**Defect** [cold-read]: `H(outpoint‖seq)` makes cross-seq double-spends unlinkable.

**Resolution**: `nullifier = SHA256("FlockNullifier/v0.3" ‖ outpoint)` — any double
cosign of the same outpoint produces an identical nullifier under two seqs, a
mechanically detectable collision. Concede that detection requires scanning the log
(towers/auditors do this; receivers check their own outpoints), and that full-log
availability is part of open problem 2.

## A8. Quorum ciphertext archive = operator-run archival relays

**Defect** [requirements]: the archive was load-bearing but had no transport — an
implicit operator HTTP endpoint, violating R2.

**Resolution**: operators MUST run **paid archival relays speaking plain NIP-01**
with long retention, listed in 38400 `params.archive_relays`. The "Q archive" in all
flows means: fetch the ciphertext events (37400 vaults, 1059 wraps) from those relays.
Scope R2's claim: no operator endpoint is load-bearing **for checkpointed funds**;
uncheckpointed receipts additionally depend on relay retention incl. archival relays
(open problem 2).

## A9. Seed-only recovery scoped to "seed + public pool data"

**Defect** [requirements]: recovery-leaf spends need the control block (sibling leaf
hashes), which is not derivable from the seed; derivation paths were [TBD] and the
leaf cooperative key U was missing from the derivation table.

**Resolution**: add U to the seed-derivation table; assign concrete BIP-32 paths
(purpose field `m/84021'` placeholder, document as draft). Specify the seed-only scan:
crawl 38400 pool ads from bootstrap relays → fetch 38402 round results → confirm cell
txids on L1 via compact filters or a full node → rebuild taptrees from the inline tree
commitments → derive U/U_ckpt and match leaf scriptPubKeys → reconstruct control
blocks. Restate the headline honestly: recovery needs the **seed plus any surviving
copy of the public round data** (38402 events, vault, towers, archival relays, any
cell peer — public, replicated, non-secret data, but a nonzero availability
assumption). Block explorers may only ever be cross-checks, never the sole L1 view.

## A10. Ephemeral-key deletion timing unified

**Defect** [bitcoin-validity, consistency]: ceremonies said delete after CONFIRMED;
onchain/synthesis said after emitting partials.

**Resolution**: delete ephemeral path keys **immediately after emitting partials and
verifying the aggregate signatures, before broadcast**. Rebuilds use fresh trees and
fresh keys, so nothing is lost. Ceremonies aligns to onchain.

## A11. Send completion = read-back verification

**Defect** [requirements, consistency]: ceremonies made non-existent signed write
receipts the mandatory send-completion condition.

**Resolution**: send completion is defined by **read-back verification over disjoint
connections** per event-kinds §1.5. Relay-signed receipts are an optional SHOULD
(defense-in-depth, format [FUTURE]). Align ceremonies §4.1 step 9, whitepaper §7.1,
and attacks.md L11 wording.

## A12. Intent bond specified

**Defect** [requirements]: the G1/G2 mitigation had no instrument.

**Resolution**: `params.intent_bond_sats` added to 38400 (canonical default
**1,000 sats**). Instrument: the refresh intent (24400) carries a pre-signed tx
spending a small seed-derived **bond vTXO** to Q, immediately broadcastable. Q
discards it on honest completion or honest pre-nonce exit; Q MAY broadcast it on
post-nonce abort. Q's power to slash is unilateral and bounded by the bond; add this
exposure (≤ bond per intent) to the residual-trust tables. Boarding users (no vTXOs
yet) are exempt — PoW-only. Refinements [FUTURE].

## A13. Height-deposit economics specified

**Defect** [bitcoin-validity, cold-read]: tower payment was unspecified; deposit cost
missing from user fees.

**Resolution**: each deposit names a specific tower (per-request key). The package's
pre-signed CPFP child for that tower pays a **fee output (~2,000 sats canonical
default) to the tower's per-request key**. Multiple towers hold conflicting child
variants — whichever tower completes the broadcast collects; at most one fee is ever
paid, and **nothing is paid if no tower ever needs to act** (the fee exists only
inside the contingent child). State this in the user-cost summaries: the deposit
budget is a contingent allocation, not a per-send payment. Acknowledge the symmetry
with the Lightning watchtower critique: Flock makes a tower economy load-bearing
(open problem 10) but gives it an explicit funding path.

## A14. Arkoor cosign envelope (new kind 24405)

**Defect** [bitcoin-validity minor]: Alice↔Q arkoor signing was misdescribed as a
MuSig2 session; the two-sig leaf needs no nonce coupling.

**Resolution**: new ephemeral kind **24405** — arkoor cosign request/response,
gift-wrapped: Alice sends her complete Schnorr signatures + tx chain; Q replies with
its FROST signatures + transfer-log root + cosigned claim. Kinds 24402/24403 are
reserved for true MuSig2 cohort sessions only.

## A15. Discovery and payment descriptors

**Defect** [requirements]: first-payment discovery for a zero-footprint npub was
unspecified; onchain referenced an undefined "payment-target entry".

**Resolution**: new addressable kind **38407 — Flock payment descriptor**: stealth
base key B, preferred pools, read relays (optional; complements NIP-65).
**Fallback rule**: if a receiver has no 38407 and no kind-10002, senders set
B = the receiver's Nostr pubkey and publish the wrap to the **pool relay set**
(mandatory in all cases) plus the receiver's read relays when known. New wallets
crawl 38400 pool ads from compiled-in bootstrap relays and scan pool relay sets with
the pool tag + trial decryption. Document the scaling concession (scan cost grows
with pool count).

## A16. Service offer book covers towers

38406 becomes the **service offer book**: entries carry `type: "gateway" | "tower"`
with per-type fields (gateway: pairs/fees/CLTV deltas; tower: fee expectations,
watch capacity, per-request key endpoint). Tower discovery [TBD] is resolved.

## A17. Attestations downgraded; fraud proofs scoped

**Defect** [cold-read]: "share retention becomes provable fraud" is unfalsifiable —
FROST signatures don't attribute share subsets and resharing keeps Q unchanged.

**Resolution**: 31401 deletion attestations create **reputational and legal exposure
only**; share retention remains cryptographically unverifiable (consistent with open
problem 6 — same standard the paper applies to Spark). "Provable fraud" is reserved
for transfer-log equivocation (two quorum-signed roots at one seq) and nullifier
collisions (A7), which are genuinely cryptographic.

## A18. Scoped absolutes

- "Liveness can be attacked; safety cannot" → safety holds **for checkpointed funds
  whose owner or towers act within stated deadlines**; enumerate the deadline-driven
  exceptions inline (held-forfeit broadcast, recovery-vs-sweep window, deposit
  execution), not only in the open-problems list.
- "A forfeit always beats every laddered claim" → true once broadcast; **someone must
  broadcast it** within the stale window (open problem 9).
- VaR cap: enforceable **only while online**; offline exposure is bounded by the
  acceptance rule (A3.4), and `params.uncheckpointed_ceiling` (38400, optional) lets
  Q refuse cosigns beyond a per-recipient uncheckpointed total (privacy trade-off
  noted: Q already sees the graph).
- Velocity bound stated explicitly: ≤4 chained transfers per leaf between refreshes;
  a hop-4 holder must checkpoint at the next round before re-spending; analyze round
  capacity (16×64 = 1,024 leaves/hour) vs auto-checkpoint demand for a 10k-user pool.

## A19. Presentation fixes (apply throughout)

- Fix "432 blocks (~3 days)" → **3,744 blocks (~26 days)** everywhere.
- Compare exit costs at the **same feerates** for Lightning and Flock (table at 10
  and 50 sat/vB, typical and worst case).
- Remove self-praise and appeals to the internal contest's authority ("judges called
  this the single best…", "not a sales page"); cite `research/raw/` artifacts
  factually instead.
- One-line rationale (or pointer) for every headline constant (4-of-7, ≥8/≥3 relays,
  50k VaR, 10k floor, 20-bit PoW, 120 s timers); mark 1–2 s arkoor latency as a
  **target pending measurement** over relay+Tor paths.
- Residual-risk "four families" sentence: scope to mitigated findings and point to
  the open-problems list for conceded/structural residuals.
- Tower dependency bound: unambiguous phrasing — *"no third party is needed before
  T_exp−720; at T_exp−720 the height-deposit tower must act"*.
- PoW phrasing: "PoW ≥ 20 bits (relay/Q demand capped at 20)".
- Re-key every "attack #N" cross-reference to attacks.md's T/L/G/P identifiers.
- 24401 validation rule: "all pre-signed txs are nVersion=3 zero-fee P2A; the cell
  tx template is nVersion=2, fee-paying (CPFP-only)".
- Session-id: single canonical preimage in both specs —
  `SHA256("FlockSession/v0.3" ‖ pool_id ‖ round_seq ‖ cell_index ‖ attempt ‖
  sub_attempt ‖ pset_commitment ‖ msg_digest)`; REBUILD increments `sub_attempt`,
  a new attempt resets it.
- Arkoor wrap pre-auth filter tag: specify in event-kinds §3.11 — tag
  `["f", first 8 hex chars of SHA256("FlockFilter/v0.3" ‖ P_B)]`; receiver-derivable
  (receiver can compute candidate P_B values from its ECDH preimages only after
  fetching… it cannot pre-derive sender-chosen counters — therefore scope the claim:
  the filter tag narrows trial decryption for *re-fetches and tower handoffs*, not
  first discovery; attacks.md G8 wording adjusted to match, or the tag is dropped
  from G8's mitigation list. Fixer for event-kinds.md decides which and records it.)
- L5 disposition: "mitigated with an ISP/state-level residual", drop the "conceded"
  label or add it to the open-problems index — pick one consistently (preferred:
  mitigated-with-residual, matching whitepaper §8.4 family 2).
- P3 cross-reference symmetry with OP1 (add the arrow where missing).
- 38400 schema gains: `intent_bond_sats`, `archive_relays`, `sweep_delay`,
  `uncheckpointed_ceiling` (optional), tower/gateway offer-book pointer.

---

# Round 2 — adjudicated resolutions to the re-verification findings

The v0.3 re-verification (19 findings, 6 major; saved at
`raw/review-v0.3-findings.json`) surfaced four substantive defects plus polish.
Resolutions below are NORMATIVE and extend Round 1. ERRATA.md updated to match.

## A20. Refresh fees are priced per refresh; float scales with cadence

**Defect**: the "0.5–0.9%/month" band and the 1.9 float ratio silently assumed one
refresh per 30-day expiry — but the refresh deadline (T_exp−1008 ≈ 23 d), the
receive ban (~16 d), and hop-4 velocity force shorter cycles, under which the float
ratio is 2.5×+ and "%/month" understates active users' true cost.

**Resolution**:
1. Canonical pricing: the refresh fee is **0.5–0.9% of refreshed value per refresh**
   (break-even ≈ APY × (4320+sweep_delay)/52560 ≈ **0.8% per refresh** at 5% APY,
   sweep_delay 4032 — a ~58-day capital lock per refresh event). A once-a-cycle
   casual user therefore pays ≈0.5–0.9% per ~23-day cycle; high-velocity users pay
   proportionally more. Concede this plainly (extend open problem 7): merchants at
   hop-4 velocity will prefer LN-gateway sweeps or on-chain offboards to
   high-frequency refreshes; `sweep_delay` and pool expiry are the pricing levers
   (shorter sweep_delay ⇒ cheaper refreshes ⇒ smaller dormancy grace, A2 trade-off).
2. Float model: **boarding-funded leaves cost Q ≈ nothing** (the user's own UTXO
   funds the cell input); Q's float comes from **refresh churn**:
   float ≈ hot_balance × 58 d / refresh_cycle. State this insight explicitly.
3. **One canonical worked example, computed in economics.md §1.3 and quoted
   verbatim by whitepaper §9** (no second variant unless labeled "optimistic,
   full-adoption"): 10k users, 100k-sat average (10 BTC custodied), 50% hot,
   hot cycle 23 d (deadline-driven) ⇒ float ≈ 5 × 58/23 ≈ **12.6 BTC**; capital
   cost ≈ 12.6 × 5%/12 ≈ **0.052 BTC/mo**; refresh revenue at 0.8% per cycle ≈
   **0.052 BTC/mo** — the refresh line is sold ≈ at cost; the margin lives in
   arkoor flow fees, gateway swaps, and pinning subscriptions (model pinning at
   ~30% adoption, not 100%). Show the stack and the resulting thin-but-positive
   margin honestly.

## A21. Dormancy wording: automatic on-chain exit, not "no forced exit"

**Defect**: "not force-exited / pays ≈0 / idle-funds cost ≈0" contradicts the A2
mechanism, which IS an automated forced exit at the owner's on-chain expense.

**Resolution** (use this framing everywhere — whitepaper §5.5/§9/§10 and §3.3 Cashu
comparison, economics saver persona, comparison.md): *dormant funds pay no carrying
fees and are never confiscated while towers function, but ~32 days after the last
checkpoint the standing deposit automatically exits them to on-chain self-custody at
the owner's expense* — ≈3.1k–16k sats at 10 sat/vB (5× at 50) plus the ~2,000-sat
contingent tower fee. Note the floor consequence: dormant leaves near the 10k-sat
minimum are substantially consumed by their own exit; savings-profile pools can
stretch expiry/sweep_delay to lengthen the dormancy horizon. "Not confiscated", never
"not force-exited". The §10 cell becomes: "Idle funds: no carrying fee; one automatic
on-chain exit (~5–18k sats) if absent >~32 d".

## A22. Two distinct defense windows, named and used correctly

**Defect**: "144–720 blocks" conflated (a) forfeit defense of refresh-stale leaves
with (b) ladder defense of uncheckpointed chains.

**Resolution**: define both, use the right one at each site:
- **Forfeit stale window** (refresh-stale leaf unrolled): defender must broadcast the
  held forfeit (nSeq 0) before the stale claim matures — **144–720 blocks** by the
  stale owner's hop.
- **Ladder defense deadline** (stale prior owner attacks an uncheckpointed chain,
  attacker hop ≤3): current owner's head start (differential) is **144–576 blocks**;
  the tower's absolute deadline is the attacker's claim maturity, **288–720 blocks**
  after the unroll.
Fix whitepaper §5.4/§6.2/§8.2, attacks.md T5/§6, ceremonies §5, onchain §3.4 (label
its 144–720 as the forfeit case).

## A23. Bond instrument made feasible

**Defect**: a standalone 1,000-sat "bond vTXO" violates the 10,000-sat leaf floor,
and "immediately broadcastable" is false (the prevout exists only after an unroll).

**Resolution**: the bond is a **ledger debit, netted at next service**: Q records the
slash in the transfer log and deducts `intent_bond_sats` from the user's next
successful refresh output or arkoor change; Q refuses service until netted. Backstop:
the intent carries a pre-signed **lien tx** spending the refreshing leaf's
cooperative path — (value − bond) back to the owner, bond to Q — valid on any future
unroll of that leaf (collection cost ≈ unroll cost; state that the netting path is
the economic one). No standalone bond vTXO exists; no 10k-floor exemption is needed.
Residual unchanged: Q can slash unilaterally, exposure bounded by the bond.

## A24. Tower bump funding specified

**Defect**: pre-signed children are minimum-feerate, so an acting tower would be out
of pocket (~16k sats unroll vs ~2k fee) — the dormancy and stale-unroll defenses
rested on a loss-making action.

**Resolution**: every deposited package carries **pre-signed escalating bump
children** (canonical ladder: 10 / 50 / 200 sat/vB) spending the owner's dedicated
fee UTXOs, with the ~2,000-sat tower fee output riding each variant. The tower
selects the lowest variant that clears; its out-of-pocket cost is ≈0. Residual
(state it): feerates above the pre-signed ceiling stall the package — this folds
into the existing replacement-cycling/fee-spike open problem.

## A25. VaR cap semantics and checkpoint queueing

- The 50k-sat VaR cap applies to the **aggregate uncheckpointed balance**; any
  accepted receipt pushing the aggregate over the cap triggers auto-checkpoint at
  the **next available round seat**. Structuring many sub-cap receipts does not
  evade it.
- Checkpoint-queue congestion proportionally extends above-cap exposure;
  VaR-triggered refreshes take priority in Q's seat assignment. Adjust whitepaper
  §4.3's "not a safety failure" sentence accordingly.

## A26. Churn-leverage figure unified

Theoretical leverage of an unthrottled hourly refresher is **~1,390×** (58 d of
hourly locks); with the minimum dwell time D the bound is **58 d / D** (D = 7 d ⇒
≈8×). economics.md §1.1 and attacks.md G4 must both use these figures and name D.

## A27. Polish (apply where they live)

- ceremonies §4.1 step 4: "per-vTXO seq" → "the transfer-log seq strictly increments
  (one root per cosign batch)" — the nullifier-absence check covers double-cosigns.
- whitepaper §5.5: "the v0.3 review" → "the v0.2 review".
- whitepaper §10 table: align Infra-dependency and Operator-trust cells with
  comparison.md (archival relays + tower market; sender-alone caveat; dormant-sweep
  caveat per A21).
- economics §1.3: re-derive labels per A20 (no "top of band" mislabel).
- ceremonies §4.1 heading: "~1–2 s target, pending measurement over relay+Tor paths".
- onchain §2 notation: T_exp is strictly per cell (a rebuilt cell may confirm at a
  different height); drop "round's cells share an effective T_exp".
