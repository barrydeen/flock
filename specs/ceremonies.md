Flock protocol - DRAFT v0.3 - 2026-06-12

# Signing Ceremonies

This document specifies every multi-party signing ceremony in Flock: the nonce
rules that all ceremonies inherit, quorum genesis and maintenance (DKG and
proactive resharing), the cell refresh ceremony, the arkoor transfer ceremony,
the watchtower protocol, recovery from seed, and LN gateway swaps.

Event kinds, tags, and serialization are specified in
[event-kinds.md](event-kinds.md). Scripts, transaction templates, and exit
paths are specified in [onchain.md](onchain.md). The attack analysis that
motivates the rules below is in [../analysis/attacks.md](../analysis/attacks.md).
The key words MUST, MUST NOT, SHOULD, and MAY are to be interpreted as in
RFC 2119.

Two signing protocols appear in Flock, and they never share nonces, keys, or
algebra:

- **MuSig2** ([BIP-327](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki))
  among *user cohorts* — cell-tree path signing and cooperative leaf spends.
- **FROST** ([RFC 9591](https://www.rfc-editor.org/rfc/rfc9591)) among the
  *operators* — every signature by the group key Q is a plain BIP-340
  FROST-Schnorr signature over the same digest the user side signs, placed in
  a separate tapscript signature slot (`<X> OP_CHECKSIGVERIFY <Q> OP_CHECKSIG`).

There is no FROST-inside-MuSig2 composition anywhere in Flock. v0.1's
composition was removed because no proven construction exists and split relay
views enable Drijvers/ROS-class attacks on the combined challenge (attack T2
in the matrix). The cost is one extra signature per quorum-gated input
(~+70 vB); the benefit is that both protocols run exactly as standardized.

---

## 1. Nonce discipline

These rules are load-bearing. Every signer — wallet, operator, gateway — MUST
enforce all of them in every MuSig2, FROST, and adaptor-signature session.
Implementations SHOULD ship the spec test vectors that exercise each rule.

**N1.** All round-1 nonces (MuSig2 `secnonce`, FROST hiding and binding
nonces) MUST be generated from a CSPRNG at the moment the session begins.

**N2.** Deterministic nonce derivation is BANNED. A nonce MUST NOT be derived
from the session-id, message, key set, attempt counter, seed, or any other
re-computable input, in whole or in part. Session identifiers (rule N7) MUST
NOT be used to seed nonce generation.

**N3.** Secret nonces MUST be held in RAM only. They MUST NOT be written to
disk, keychains, cloud backup, swap, or replicated HSM state. A device
restored from any backup therefore *cannot* reuse a nonce, because no nonce is
ever recoverable; restoration aborts every in-flight session by construction.

**N4.** A signer MUST emit at most **one partial signature per secret nonce,
ever**. Implementations MUST delete the secret nonce at the moment it is read
for signing (delete-on-first-use), before the partial leaves the device.

**N5.** On any timeout, crash, duplicate or conflicting proposal, missing
counterparty message, or other ambiguity, the signer MUST abort the session
and restart with **fresh nonces** in a new session. Resuming an interrupted
session is forbidden.

**N6.** A signer MUST sign at most one message per session and MUST ignore any
second proposal carrying the same session-id. Coordinator equivocation
therefore yields nothing.

**N7.** Retry idempotence is achieved via **session-ids, never via nonce
reuse**. The session-id is
`SHA256("FlockSession/v0.3" ‖ pool_id ‖ round_seq ‖ cell_index ‖ attempt ‖ sub_attempt ‖ pset_commitment ‖ msg_digest)`
(identical in [event-kinds.md](event-kinds.md) §1.2). REBUILD increments
`sub_attempt` only; a new attempt resets `sub_attempt` to zero.
Relays may replay events freely: events are deduplicated by id, and partials
are bound to dead sessions, so replay is harmless.

**N8. Canonical session transcript.** Before emitting any partial signature,
every signer MUST construct the canonical transcript — the **full ordered
pubnonce set, the full signer set, and the exact message digest** — sign it,
publish the transcript signature, and verify every other signer's transcript
signature over a byte-identical transcript. Any divergence MUST abort the
session (rule N5). This makes a split relay view (different participants
shown different nonce sets or messages within one session) detectable before
any partial exists, rather than after a signature has leaked information.

**N9.** Pubnonces and partials are carried in kinds 24402/24403 using
[BIP-373](https://github.com/bitcoin/bips/blob/master/bip-0373.mediawiki)
PSBT MuSig2 fields, NIP-44 encrypted and session-bound. Kinds 24402/24403
are reserved for true MuSig2 cohort sessions; the arkoor cosign uses the
kind-24405 request/response envelope (§4.1), which involves no nonce
coupling.

**N10.** Signers MUST bound the number of concurrently open signing sessions
per key (operators per FROST share; wallets per leaf key). Concurrency limits
cap the surface for nonce-grinding and ROS-style concurrent-session attacks.

**N11. Adaptor signatures (PTLC/vHTLC).** All rules above apply unchanged.
Additionally, before finalizing any adaptor signature the signer MUST verify
the adaptor point and the challenge it is finalizing against, and MUST record
the finalize in an append-only local **finalize journal** keyed by nonce. At
most **one finalize per nonce, ever**; a journal hit MUST abort. The journal
exists because an adaptor secret leaks if one adaptor is finalized against two
contexts — the failure is silent and catastrophic (the payment secret is the
discrete log that escapes).

**N12.** Operators apply rules N1–N10 identically to FROST signing and to all
DKG/resharing traffic (§2). The FROST rule mirror of N4: never resume an
interrupted FROST session; fresh nonces per session, one partial per nonce.

### The attack this prevents

v0.1 derived `secnonce = H(round-id ‖ attempt ‖ signer-set ‖ tx-digest)` for
retry-idempotence over lossy relays. That rule is a key-extraction oracle.
MuSig2 commits a signer to its nonce *before* it sees counterparties' nonces,
so the aggregate nonce — and hence the challenge `e` — depends on inputs the
attacker controls. A malicious co-signer or relay delivers a proposal with
nonce-set N₁; the victim emits `s₁ = k + e₁·a·x` and "deletes" `k`. The
attacker drops `s₁`, induces a crash or same-attempt retry, and the victim
re-derives the identical `k` against nonce-set N₂, yielding `s₂` with
`e₂ ≠ e₁`. Two partials under one nonce give `x = (s₁ − s₂)/(e₁ − e₂)` — the
victim's private key, with no signature ever reaching the chain. Applied to
an operator, the same algebra recovers a FROST share; repeat against k = 4
operators and the attacker reconstructs Q and drains the entire pool. This is
not theoretical: deterministic MuSig2 nonces caused a real key-extraction
vulnerability in production
([BitGo field report](https://bitcoinops.org/en/bitgo-musig2/)), and it is
why rules N1–N5 are stated as bans rather than guidance. Lossy, replaying,
reordering relays are Flock's *normal* transport; the nonce rules are written
so that the worst any relay can do to a session is abort it.

---

## 2. Quorum genesis and maintenance

### 2.1 Genesis DKG

The pool's group key Q (FROST 4-of-7) is created by a **PedPoP** distributed
key generation among the seven operators, carried as kind-24404 events
(NIP-44 encrypted operator↔operator, ephemeral). Reference construction: ZF
FROST's PedPoP DKG ([github.com/ZcashFoundation/frost](https://github.com/ZcashFoundation/frost));
FROSTR/bifrost demonstrates FROST signing over Nostr relays in practice
([github.com/frostr-org/bifrost](https://github.com/frostr-org/bifrost)).

Relays provide no ordering, no delivery guarantee, and no transport
authentication, so the DKG supplies its own:

1. **Epoch tag.** Every DKG/reshare message carries the epoch number. Genesis
   is epoch 0. Operators MUST reject any message whose epoch differs from the
   session they are in — cross-epoch material is never combined (a replayed
   old-epoch share alongside new-epoch shares is the classic aid to a mobile
   adversary).
2. **Signed transcript.** Each operator accumulates the full ordered message
   transcript of the round, signs it under its operator npub, and publishes
   the transcript signature. All seven transcript signatures over a
   byte-identical transcript are required to proceed — the DKG analogue of
   rule N8.
3. **ACK round.** Every operator explicitly ACKs receipt and verification of
   every other operator's contribution. Missing ACK after the phase timer ⇒
   abort the epoch and restart.
4. **Complaint round.** An operator that received an invalid share or detects
   transcript divergence publishes a complaint (kind 24404) identifying the
   message. Any complaint aborts the epoch; the transcript makes the fault
   attributable. Complaint-round timer: [TBD].

On success, Q is published in the kind-38400 pool advertisement, signed by
the quorum, and each operator's first kind-31401 heartbeat for the epoch
attests the DKG transcript hash.

### 2.2 Monthly proactive resharing

Shares are proactively rotated **monthly** so that an adversary must
compromise k = 4 operators *within one epoch*, not cumulatively across the
pool's life. Resharing runs over kind-24404 with the same epoch-tag,
transcript, ACK, and complaint machinery as genesis.

Additional invariants:

- **Group-key invariance.** Resharing rotates shares; Q MUST NOT change. Each
  operator independently verifies that the post-reshare group public key
  equals the advertised Q before ACKing. Any mismatch ⇒ abort, retain old
  shares.
- **Old-share deletion.** Old-epoch shares are deleted only after the full ACK
  set for the new epoch is observed. Each operator then publishes a signed
  **deletion attestation** in its kind-31401 heartbeat (covering reshare
  epochs and the ephemeral tree keys of §3), retained on relays for a rolling
  90 days. Attestations create **reputational and legal exposure only**:
  deletion is attested, not provable — FROST signatures do not attribute
  which share subset signed, and resharing leaves Q unchanged, so retention
  is cryptographically unverifiable. This is an honesty assumption on
  individual operators and is acknowledged as such.
- **Failure mode.** If resharing fails — missing ACKs, complaints, invariance
  failure, partitioned operators — the epoch is abandoned and the previous
  share set remains live. Repeated failure degrades the pool toward
  **exit-only mode** (no new rounds or cosigns once fewer than k operators
  are mutually consistent). Resharing failure is therefore a **liveness
  halt, never a safety loss**: no combination of dropped, reordered, or
  replayed reshare messages can yield a spendable wrong key, because nothing
  is signed under a share set that lacks full ACKs and a verified-invariant
  group key. Users exit unilaterally; the claim ladder and recovery tapleaf
  (see [onchain.md](onchain.md)) do not need a live quorum.

---

## 3. Cell refresh ceremony

Refreshes (and boarding conversions) happen in hourly rounds. A **round** is
the batch of up to **16 cell txs**: each 64-leaf cell (radix-4, depth-3 tree)
is funded by its **own on-chain cell transaction** — inputs: Q capital plus
that cell's boarding outputs; outputs: one cell root, up to 64 per-user
connectors, and Q change. Each cell runs its own ceremony against its own
cell tx; a misbehaving participant delays *one cell by one attempt*, never
the round — a rebuild re-signs that one cell tx only and touches no other
cell's pre-signed node txs, forfeits, or boarding signatures. Wallets MUST
refresh by `T_exp − 1008`.

### 3.1 States and messages

| State | Entered by | Wallet obligation in this state |
|---|---|---|
| `ANNOUNCED` | Q publishes kind-38401 (round-id, phase deadlines) | Decide whether to refresh |
| `INTENT` | Wallet sends kind-24400, gift-wrapped to Q | Prove vTXO ownership; attach the pre-signed lien tx backstopping the intent bond (§3.4; `params.intent_bond_sats`, canonical default **1,000 sats**, a ledger debit netted at next service — boarding users have no vTXO yet and are exempt, PoW only) and NIP-13 PoW **≥ 20 bits** (relay/Q demand capped at 20 — the cap prevents PoW escalation from becoming a censorship chokepoint) |
| `PROPOSED` | Q sends kind-24401 to cell members, carrying the **cell-tx template** (nVersion=2, fee-paying) and all pre-signed tx templates (nVersion=3, zero-fee, P2A) | Verify the proposal (§3.2, step P) |
| `NONCE` | Members publish kind-24402 pubnonces | CSPRNG nonces per §1 |
| `SET-FINAL` | Q finalizes the participant set **after** the nonce phase; this **freezes the cell tx and its txid** | Verify the finalized set ⊆ proposed set (non-responders excluded); recompute the final cell txid locally and re-verify own leaf + connector under it (§3.2, step P′) |
| `PARTIAL` | Members publish kind-24403 partials | Transcript sign + cross-verify (N8) **before** emitting partials; delete the ephemeral path key immediately after emitting partials and verifying the aggregates (§3.2, step N) |
| `FORFEIT` | Members sign forfeits | Only after the checks in §3.3 |
| `BROADCAST` | Q broadcasts the cell tx (fee-bumping is **CPFP-only**, spending Q change; RBF is forbidden — any txid change voids the cell's pre-signed graph) | Watch L1 |
| `CONFIRMED` | Cell tx confirms | Verify confirmed outputs (§3.4) before treating the refresh as final |
| `REBUILD` | Any exclusion or divergence | Same attempt, sub-attempt++, fresh session-id, fresh nonces — **this cell only**; every other cell's tx and pre-signed graph are unaffected |
| `ABORTED` | Phase-timer expiry without quorum progress | Wallet keeps old vTXO; retry next round |

Every phase has a nominal **120 s adaptive timer**; concrete deadlines are
published in the kind-38401 announcement and are part of the canonical
session context, so all participants time out identically.

**Post-nonce participant finalization.** The participant set is *not* final
at proposal time, and therefore neither is the cell txid: SET-FINAL is the
moment both freeze. After the nonce phase, Q fixes the set to those who
delivered valid pubnonces; non-responders are excluded and the cell is
**rebuilt in the same attempt** — a new session-id (sub-attempt incremented),
new proposal restricted to responders, fresh nonces all around (rule N5). A
silent dropper therefore costs one 120 s phase, not the cell and not the
round. Because each cell has its own funding tx, a rebuild re-signs that one
cell tx only — no other cell's node txs, forfeits, or boarding signatures are
invalidated. This bounds the v0.1 single-aborter collapse, where one withheld
partial killed a 1024-cosigner ceremony every hour for free.

### 3.2 What the wallet verifies, per step

**Step P — on receiving the kind-24401 proposal, before sending any nonce:**

1. **Own leaf**: the proposed tree contains the wallet's new vTXO leaf with
   the exact expected script — cooperative path keyed to the wallet's key,
   recovery tapleaf `<T_exp+288> CLTV` keyed to the wallet's **own**
   seed-derived `U_ckpt`, leaf sweep at `T_exp + sweep_delay` per the announced
   pool parameters (canonical default 4032), amount ≥ 10,000 sats.
2. **Own path**: all node txs on the wallet's root-to-leaf path (depth 3)
   match the templates in [onchain.md](onchain.md) — NUMS internal keys,
   correct branch MuSig2 aggregate including the wallet's ephemeral path key,
   node sweep `<T_exp + sweep_delay> CLTV <Q>` (per pool params, default 4032).
3. **Canonical participant-set commitment**: the proposal's signer-set
   commitment matches the set the wallet will later see in nonces, transcript,
   and finalization. This commitment is what defeats round view-splits
   (attack T9): two members shown different sets produce divergent
   transcripts and the session aborts before partials.
4. **Own connector outpoint**: the proposed cell tx contains a distinct
   dust-sized connector output assigned to this wallet.
5. **Proposed cell txid**: the wallet computes the cell txid locally from
   the proposal and uses *its own computation* — never a Q-asserted txid — in
   everything that follows. This txid is provisional: it freezes only at
   `SET-FINAL`.
6. **Transaction versions**: all pre-signed txs (node/claim/forfeit) are
   nVersion=3 zero-fee P2A; the cell-tx template is nVersion=2, fee-paying
   (CPFP-only). Any other combination ⇒ reject the proposal.

**Step P′ — after `SET-FINAL`, before emitting any partial or forfeit:** the
participant set, and therefore the cell tx, are frozen only at `SET-FINAL`.
The wallet MUST recompute the **final cell txid** from the finalized set and
re-verify its own leaf and its connector outpoint under that txid. Every
later commitment — partials, forfeits — binds to the post-`SET-FINAL` cell
txid, never to the provisional one.

**Step N — nonce and partial phases:** rules N1–N10 apply. The wallet signs
path txs only for its own root-to-leaf path. The wallet MUST delete its
ephemeral path key **immediately after emitting its partial signatures and
verifying the aggregate signatures — before the cell tx is broadcast**
(rebuilds use fresh trees and fresh keys, so nothing is lost); its own
deletion is what binds its branch's pseudo-covenant (and is the reason no
third party can ever be delegated to refresh on a user's behalf — the
trust-downgraded delegated refresh of v0.1 is deleted from the protocol).

### 3.3 Forfeit signing order

The forfeit tx is the atomicity device: inputs = the old leaf's cooperative
path **plus the wallet's connector output of the new cell tx**, nSequence 0,
paying Q. Because the connector outpoint commits to the exact cell txid, the
forfeit is consensus-invalid unless the precise cell tx containing the
wallet's verified new leaf confirms. There is no preimage, no reveal step,
nothing Q can withhold (this replaces v0.1's asserted hash-locks, attack T4).

Ordering is normative:

1. The wallet MUST sign the forfeit **only after** `SET-FINAL`, verifying
   that the frozen cell tx contains (a) its new leaf exactly as verified in
   step P and (b) its own connector outpoint, both under the locally
   recomputed final cell txid (step P′).
2. Q MUST collect every cell member's forfeit before broadcasting the cell
   tx (otherwise a member would receive a new leaf without forfeiting the old
   one).
3. Forfeits are broadcastable by **anyone**; watchtowers hold copies (with
   the full ancestor arkoor chain and CPFP children where the forfeited
   output is an arkoor chain tip — §5). With nSequence 0 a forfeit beats
   every laddered claim (minimum maturity 144 blocks) **once broadcast**,
   even if Q later dies — but someone must broadcast it within the stale
   window (open problem 9 in the [whitepaper](../whitepaper.md)).

### 3.4 Aborts, slashing, and confirmation

**The intent bond.** The bond (`params.intent_bond_sats` in the 38400 pool
ad, canonical default **1,000 sats**) is a **ledger debit, netted at next
service**: on a slash, Q records the debit in the transfer log and deducts
it from the wallet's next successful refresh output or arkoor change,
refusing service until netted. As backstop, the kind-24400 intent carries a
pre-signed **lien tx** spending the refreshing leaf's cooperative path —
(value − bond) back to the owner, bond to Q — valid on any future unroll of
that leaf (collecting it costs about as much as an unroll; the netting path
is the economic one). No standalone bond vTXO exists, so no exemption from
the 10,000-sat leaf floor is needed. Boarding users have no vTXOs yet and
are exempt (PoW only). Refinements [FUTURE].

- **Abort before the nonce phase** (intent withdrawn, proposal rejected,
  honest completion): Q MUST discard the lien tx and record no debit — the
  bond never leaves the wallet's control.
- **Abort after emitting pubnonces** — withholding partials, garbage
  partials, refusing the forfeit: the member is excluded, the cell is rebuilt
  in the same attempt, and Q MAY slash the **intent bond** — recorded as a
  ledger debit and netted at the wallet's next service, with the lien tx as
  backstop. Q's power to slash is unilateral, but its exposure is bounded:
  at most `intent_bond_sats` per intent, an entry in the residual-trust
  accounting. Conceded honestly: ceremony withholding is indistinguishable
  from relay loss, so bonds occasionally tax honest unlucky users and only
  imperfectly deter a resourced griefer (open problem 5 in the
  [whitepaper](../whitepaper.md)).
- **Cell tx mutation**: if the broadcast cell tx differs from the
  post-`SET-FINAL` template, every forfeit in that cell is invalid (wrong
  connector txid) and every old vTXO remains intact. A mutated cell tx is a
  liveness event, not a loss. This is also why RBF on the cell tx is
  forbidden: any txid change voids the cell's pre-signed graph; fee-bumping
  is CPFP-only via Q change.
- **Confirmation handling**: on seeing its cell txid confirm, the wallet
  MUST verify the confirmed transaction's actual output scripts — its leaf,
  its cell root, its connector — against its **local recomputation**, using
  compact block filters or its own full node as the baseline L1 view (block
  explorers MAY serve as additional cross-checks, never as the sole source
  for this fail-closed decision), before treating the refresh as final and
  retiring the old exit package (the ephemeral path key was already deleted
  at step N). An SPV-eclipsed wallet that skips this check reintroduces
  attack T9; the residual risk of an eclipsed L1 view is conceded and
  mitigated only by view diversity.
- **Standing dormancy deposit (normative, new in v0.3)**: at **every
  checkpoint**, the wallet MUST pre-sign its **recovery claim** (recovery
  tapleaf → a seed-derived cold key) and lodge {unroll branch + recovery
  claim + escalating bump children (10/50/200 sat/vB, spending the wallet's
  dedicated fee UTXOs — §5)} as a **self-funded kind-38404 height-deposit** with
  `trigger_height = T_exp + 288`, replicated to **≥ 3 towers** (§5). This is
  what makes dormancy tower-protected self-custody rather than a promise:
  the quorum sweep at `T_exp + 4032` is real, and an absent owner's exit is
  executed by its towers, not by Q's forbearance (§6).

A completed refresh yields: fresh 30-day clock (`T_exp = cell-tx height +
4320`), claim ladder reset to hop 0 (nSequence 720), recovery tapleaf rotated
to the current owner, a standing dormancy deposit on ≥ 3 towers, zero
residual trust on the refreshed value.

---

## 4. Arkoor transfer ceremony

Arkoor (out-of-round) transfers are the payment path: instant, no receiver
liveness, no per-payment operator capital.

### 4.1 Sender side (Alice → Bob, ~1–2 s target, pending measurement over relay+Tor paths)

1. **Stealth key.** Alice derives the per-payment claim key
   `P_B = B + H(ECDH(a,B) ‖ cell_txid ‖ counter)·G`. The per-payment tweak
   (cell txid + counter) guarantees a fresh unlinkable key and
   scriptPubKey per payment; relays and the public log never see `B`.
2. **Build.** Alice constructs the arkoor tx spending her leaf's cooperative
   path into a **payment vTXO** (leaf template keyed to `P_B`) and a
   self-made **change vTXO** keyed to herself. No denomination inventory
   exists; change below the 10,000-sat leaf floor rides as uncheckpointed
   value.
3. **Cooperative signing.** The arkoor leaf's cooperative path is a
   two-signature script, not an aggregate key — there is no MuSig2 between
   Alice and Q and no nonce coupling. The exchange is a gift-wrapped
   **kind-24405 arkoor cosign request/response**: Alice sends her complete
   Schnorr signatures plus the tx chain; Q replies with its FROST
   signatures, the updated transfer-log root, and the cosigned claim.
   (Kinds 24402/24403 are reserved for true MuSig2 cohort sessions; Q's own
   FROST signing falls under all of §1.)
4. **Quorum checks (before cosigning).** Q verifies: no **nullifier
   conflict** — `nullifier = SHA256("FlockNullifier/v0.3" ‖ outpoint)`,
   derived from the outpoint alone, unseen in the transfer log. Any double
   cosign of one outpoint therefore yields an identical nullifier under two
   seqs — a mechanically detectable collision; detection requires scanning
   the log (towers and auditors do; receivers check their own outpoints),
   and full-log availability is part of open problem 2. Q also verifies: the
   **transfer-log seq strictly increments** (one root per cosign batch — the
   nullifier-absence check above is what covers double-cosigns); arkoor hop
   count ≤ 4 since the last checkpoint; and the **receive-ban window** — the
   transfer is refused if the backing cell has fewer than 2016 blocks to
   `T_exp` (Alice must refresh first).
5. **Claim cosigned atomically.** Q cosigns the transfer **and** Bob's
   laddered claim tx — nSequence `720 − 144·hop` (hop 1 = 576 … hop 4 = 144)
   — in the same ceremony. A transfer package without its claim is invalid
   and MUST be rejected by the receiver. (Decrementing-timelock exits are the
   Spark-precedented mechanism:
   [docs.spark.money/learn/technical-definitions](https://docs.spark.money/learn/technical-definitions).)
6. **Transfer-log root.** Q publishes the updated kind-30178 **transfer-log**
   root — a quorum-signed Merkle root over `(nullifier, commitment)` pairs,
   **one root per cosign**, seq strictly increasing, no cleartext amounts.
   (Terminology: a **checkpoint** is a round refresh — the hourly opportunity
   that extinguishes residual trust; the **transfer log** is the per-cosign
   anti-equivocation spine.) Two signed roots at one seq, or a nullifier
   collision, are portable fraud proofs.
7. **Delivery.** Alice gift-wraps the package (kind 1059 wrapping a kind-24406
   rumor: full tx chain + signatures back to a confirmed cell tx, the
   Q-cosigned laddered claim, and the Merkle inclusion proof) to a one-time
   key, publishing to **Bob's NIP-65 read relays plus the pool relays** —
   including the operator-run **archival relays** (paid, plain NIP-01, long
   retention) listed in 38400 `params.archive_relays`. That is all "Q's
   archive" means; no bespoke operator endpoint exists.
8. **Mandatory height-deposit.** Alice files a funded kind-38404 watchtower
   height-deposit for Bob (§5) as a **mandatory part of send** — a
   never-online receiver is protected toward expiry without ever acting.
   Conceded: against a never-online receiver this rule is unenforceable —
   the depositor is the only party positioned to attack. Q SHOULD therefore
   also lodge a deposit for every arkoor cosign it performs (it holds the
   claim it cosigned), and the receiver checks deposit existence on waking
   (§4.2).
9. **Replication verification.** Send completion is defined by **read-back
   verification over disjoint connections** per
   [event-kinds.md](event-kinds.md) §1.5: Alice MUST read the wrap and the
   30178 root back from **≥ k relays** (k = 4), including
   operator-independent relays, over connections disjoint from those she
   published on. NIP-01 OK responses prove nothing. Relay-signed write
   receipts are an optional SHOULD — defense-in-depth, format [FUTURE] —
   and never substitute for read-back.

Proof-of-payment = the Merkle inclusion proof under a quorum-signed 30178
root naming the payment's nullifier/commitment.

### 4.2 Receiver side, on waking (fail-closed)

An uncheckpointed arkoor receipt is **not accepted** until every step below
passes; absence of data is treated as failure, never as success. This
framing is normative: a sender clawing back value before acceptance is a
*failed payment*, not theft of accepted funds. Merchants requiring finality
MUST require checkpoint inclusion or verify online — including the deposit
check in step 9.

1. Fetch gift wraps from own read relays; fall back to the operator-run
   archival relays listed in 38400 `params.archive_relays`.
2. **Verify the signature chain** from the received vTXO back to a
   **confirmed cell tx**, checking L1 via compact block filters or a full
   node (block explorers only as additional cross-checks, never the sole
   L1 view).
3. **Fetch fresh kind-30178 transfer-log roots from disjoint relay sets**,
   including at least one operator-independent relay, and **fail closed on
   absence**: no fresh root, no acceptance.
4. Verify the **Merkle inclusion proof** for this payment and check that no
   other log entry carries this outpoint's nullifier (outpoint-only
   derivation makes any double cosign an identical, visible collision).
5. Verify **replication by read-back on ≥ k mirrors** over the receiver's
   own disjoint connections (event-kinds §1.5 — the sender's read-back
   evidence is local and non-portable).
6. **Persist the package locally** (mandatory — relays owe nothing).
7. **VaR cap**: the wallet's value-at-risk cap (default **50,000 sats**)
   applies to the **aggregate uncheckpointed balance**, never per receipt —
   structuring many sub-cap receipts does not evade it. Any accepted
   receipt pushing the aggregate over the cap triggers auto-checkpoint at
   the **next available round seat** (VaR-triggered refreshes take priority
   in Q's seat assignment; checkpoint-queue congestion proportionally
   extends above-cap exposure). Below the cap, the balance is displayed as
   *uncheckpointed* until refreshed. The cap is enforceable only while
   online; offline exposure is bounded by the acceptance rule above, not by
   the cap.
8. **Receive-ban window**: packages built against cells with < 2016 blocks
   to `T_exp` are rejected outright.
9. **Deposit check**: verify the sender's mandatory kind-38404 deposit
   exists on at least one tower; if absent, the wallet SHOULD file its own
   before accepting.

Residual trust, stated plainly: an uncheckpointed receipt is **revocable by
the sender alone** if the receiver remains offline beyond the ladder
differential (head start 144–576 blocks by hop; the defender's absolute
deadline is the attacker's claim maturity, 288–720 blocks after the unroll)
after a hostile unroll *and* no tower acts; revocable by **sender + 4-of-7
operators** colluding
otherwise; **extinguished at checkpoint** (automatic above the VaR cap,
hard-bounded by `T_exp`). Checkpointed funds are theft-proof with unilateral
exit, subject to the dormancy bounds in §6.

---

## 5. Watchtower protocol

Watchtowers are permissionless and keyless: they hold no user keys, only
pre-signed packages and a fee budget. Towers advertise on the kind-38406
**service offer book** (`type: "tower"`: fee expectations, watch capacity,
per-request key endpoint).

- **Height-deposits (kind 38404).** "Broadcast package P at `trigger_height`
  **or on conflict**, unless superseded," plus a fee budget. Both triggers
  are normative:
  - *Height trigger*: broadcast P at `trigger_height`.
  - *Conflict trigger*: the tower MUST watch L1 for **any spend touching the
    deposited package's ancestor outpoints** (cell root, node outputs, leaf)
    and broadcast the deposited package **immediately** upon detecting an
    unroll or conflicting spend — not only at `trigger_height`. This is what
    defends a sleeping receiver against a prior owner who unrolls early and
    outwaits the claim ladder. Two distinct windows govern the response, and
    they must not be conflated: against a **refresh-stale leaf unroll**, the
    held forfeit (nSequence 0) must be broadcast within the **forfeit stale
    window — 144–720 blocks**, set by the stale owner's hop; against a
    **stale prior owner attacking an uncheckpointed chain** (attacker hop
    ≤ 3), the current owner's head start is **144–576 blocks** and the
    tower's absolute deadline is the attacker's claim maturity, **288–720
    blocks after the unroll**.

  Deposited defensive packages MUST include the **full ancestor arkoor
  chain** plus pre-signed CPFP bump children per tx (the escalating ladder
  under *Keyless P2A bumping* below) — TRUC's 1p1c rule means the
  chain confirms serially, and a chain-tip claim or forfeit prevout exists
  only after its ancestors confirm. Deposits are encrypted to **freshly
  derived per-request tower keys** and **sharded across multiple towers** so
  no single tower learns the full vTXO ↔ L1-address ↔ amount mapping
  (colluding towers can recombine shards — conceded). Retention: until
  `H + 288`. Sender-funded deposits at arkoor send time are mandatory (§4.1)
  but unenforceable against a never-online receiver — the depositor is the
  attacker in that scenario; conceded. Q SHOULD lodge a deposit for every
  arkoor cosign it performs. Wallets SHOULD file deposits for their own
  leaves and MUST lodge the standing dormancy deposit at every checkpoint
  (§3.4: `trigger_height = T_exp + 288`, ≥ 3 towers). The dependency bound,
  stated unambiguously: **no third party is needed before `T_exp − 720`; at
  `T_exp − 720` the height-deposit tower must act** — and conflict-triggered
  deposits may fire earlier.
- **Tower fees.** Each deposit names a specific tower (its per-request key).
  Each of the package's pre-signed bump children for that tower pays a
  **fee output (canonical default ~2,000 sats) to the tower's per-request
  key** — the fee rides every feerate variant. Multiple towers hold
  conflicting child variants; whichever tower completes the broadcast
  collects. **At most one fee is ever paid, and nothing is paid if no tower
  ever needs to act** — the deposit budget is a contingent allocation inside
  the pre-signed children, not a per-send payment.
- **Supersession.** Kind 38404 is addressable: republishing under the same
  `d`-tag replaces the deposit (e.g., after a refresh makes the old package
  obsolete). Before broadcasting, a tower MUST check L1 for the package's
  inputs already being spent and treat a spend as supersession.
- **Keyless P2A bumping.** All packages are nVersion=3 (TRUC), zero-fee, with
  a P2A anchor (never a free slot). Every deposited package carries
  **pre-signed escalating bump children** — canonical ladder **10 / 50 /
  200 sat/vB** — spending the owner's **dedicated fee UTXOs that cannot be
  conflicted** (the replacement-cycling defense), with the contingent tower
  fee output riding each variant. The acting tower selects the lowest
  variant that clears, submits 1p1c packages via `submitpackage`, and
  rebroadcasts aggressively; its out-of-pocket cost is ≈ 0. Residual, stated
  plainly: feerates above the pre-signed 200 sat/vB ceiling stall the
  package — this folds into the replacement-cycling/fee-spike open problem.
  Towers also hold forfeit copies — with the full ancestor
  arkoor chain and CPFP children where the forfeited output is an arkoor
  chain tip — and broadcast them (nSequence 0) against any stale-leaf
  unroll.
- **Alerts (kind 8405, regular, public, 30-day retention).** A tower
  publishes an alert on: a detected conflict (double-spend or stale exit
  attempt), a sweep anomaly (Q-path spend earlier than `T_exp + 4032`), or
  completion of a deposited broadcast.

Conceded: a real fee-budgeted tower market is unproven (open problem 10),
and it now also carries the dormancy guarantee (§3.4, §6); operator-run
backstop towers reintroduce a censorship surface that is mitigated only by
tower diversity and the recovery tapleaf. The symmetry with the Lightning
watchtower critique is acknowledged: Flock makes a tower economy
load-bearing, but gives it an explicit funding path (the contingent fee
output).

---

## 6. Recovery from seed

A wallet is stateless by design: the seed derives the Nostr key, the Bitcoin
xprv, and the vault-author key. Recovery is a decision tree; each fallback is
strictly worse but never unsafe.

1. **Seed → keys.** Derive Nostr key + xprv + the unlinkable vault-author
   key.
2. **Vault fetch.** Fetch kind-10002 relay lists and the kind-37400 vault
   (NIP-44 self-encrypted, padded to size buckets, authored by the derived
   key) from the hard-coded bootstrap relay set. Decrypt; rebuild the vTXO
   set, exit packages, and history.
3. **Archive fallback.** If ordinary relays purged the vault, fetch it from
   the operator-run **archival relays** (paid, plain NIP-01, long retention)
   listed in 38400 `params.archive_relays` — operators serve ciphertext they
   cannot read or use. There is no bespoke operator endpoint; the "archive"
   is these relays.
4. **Cross-check.** Validate the rebuilt state against kind-30178
   transfer-log roots and L1 (compact block filters or a full node;
   explorers only as cross-checks) before trusting any of it.
5. **Last resort — recovery tapleaf.** If *every* vault copy is gone,
   checkpointed funds are still recoverable from the seed plus any surviving
   copy of the public round data: each leaf carries
   `<T_exp+288> CLTV DROP <U_ckpt> CHECKSIG` paying the **checkpointed**
   owner's seed-derivable key. Once the tree is unrolled, the owner claims
   with `U_ckpt` and nothing else. **Anyone can unroll** — but no one has
   to: cell roots carry a sweep leaf, so Q sweeps a never-unrolled cell **at
   its root** at `T_exp + 4032` without ever materializing the leaf-level
   recovery tapleafs. The recovery window is the **3,744-block (~26-day)
   gap** between recovery maturity and the sweep, and someone must unroll
   and claim inside it. The standing dormancy deposits (§3.4) are the
   normative answer: *dormant funds pay no carrying fees and the owner need
   never come online; the funds are never confiscated while towers
   function, but ~32 days after the last checkpoint (`T_exp + 288`) the
   standing deposits automatically exit them to on-chain self-custody at
   the owner's expense — ≈3.1k–16k sats at 10 sat/vB (5× at 50) plus the
   ~2,000-sat contingent tower fee. Absolute worst case — the owner stays
   absent AND no tower acts within the 3,744-block (~26-day) window — the
   quorum sweep at `T_exp + 4032` confiscates.* Dormancy is
   **tower-protected self-custody**, not an unconditional guarantee.

**Why stale vaults are safe.** Broadcasting an exit for an already-forfeited
leaf only exposes an already-spent coin: the held forfeit (Q, towers, anyone)
spends at nSequence 0 and, **once broadcast**, beats every laddered claim —
someone must broadcast it within the stale window (open problem 9). A
recovered wallet can act on arbitrarily old state and never lose funds it
still owns. There is no Lightning-style toxic state.

**What is NOT recoverable — conceded.** Uncheckpointed arkoor receipts are
not seed-derivable: the package exists only on relays (including the
archival relays) and in local storage. A coordinated relay purge that
reaches the archival relays strands them (open problem 2). The bounds are
the VaR cap (default 50k sats of aggregate uncheckpointed balance,
enforceable only while online), read-back-verified ≥ k-relay replication, mandatory local
persistence, and auto-checkpointing — bounds, not a fix. Additionally,
sub-cap value whose holder stays offline past `T_exp + 288` without
checkpointing is claimable by the prior checkpointed owner via the recovery
leaf (open problem 8), and a prior hop owner can attempt the same theft
earlier by unrolling and outwaiting the claim ladder — the
conflict-triggered deposits (§5) exist to cover exactly these cases; the
receive-ban window and auto-checkpoint shrink them.

---

## 7. LN gateway swaps

Gateways bridge Flock ↔ Lightning (BOLT11/12) with vHTLCs, advertising on the
kind-38406 offer book (NIP-69-style). Swap fees: 0.1–0.5%. The same
construction between two quorums federates pools horizontally without
inter-pool trust.

**Ceremony sketch (swap-out, Flock → LN).** (1) User selects an offer and
pays the gateway's **upfront, non-refundable reservation fee**. (2) User and
Q cosign a vHTLC vTXO: cooperative path to the gateway gated on the payment
secret, refund path to the user after the CLTV delta. (3) Gateway pays the
user's BOLT11/12 invoice; the revealed preimage (HTLC) or adaptor secret
(PTLC) lets the gateway claim the vHTLC cooperatively, with unilateral
enforcement available on-chain. Swap-in mirrors this: the gateway funds the
vHTLC vTXO to the user's stealth key, gated on the secret of the LN payment
the user routes to the gateway.

Normative points:

- **PTLC preference.** Where Flock controls both endpoints (cross-pool
  swaps, PTLC-capable gateways), adaptor-signature PTLCs MUST be preferred
  over hash HTLCs; LN-route interop still means HTLCs today. All adaptor
  signing falls under rule N11: fresh journaled nonces, one finalize per
  nonce, adaptor point and challenge verified before finalize. The
  catastrophic failure class — adaptor-secret leak via an induced
  double-finalize — has no recovery; conservative engineering is the only
  defense.
- **CLTV deltas.** Nominal delta **144 blocks**; gateways SHOULD shorten and
  randomize deltas within policy bounds [TBD] to blunt timed
  replacement-cycling around the timeout boundary, and MUST rebroadcast
  claim/refund txs aggressively with dedicated fee UTXOs.
- **Free-option mitigation.** A swap initiator holds a free option for the
  CLTV window (commit the gateway's liquidity, reveal only if the price
  moves favorably, or simply grief). Mitigations: the upfront reservation
  fee (priced to the option value), **per-identity in-flight value caps**,
  short randomized deltas, and gateway diversity. Conceded: bridge liquidity
  remains lockable for CLTV windows at the cost of reservation fees;
  replacement cycling is unsolved at the consensus level and applies to the
  swap boundary (the core protocol has no preimage-bearing second-stage
  transactions).
- **Exposure.** Users and gateways carry standard HTLC exposure for the
  144-block delta — identical in kind to Lightning's, narrower in scope:
  swaps are the only place it appears in Flock.

---

*Cross-references: protocol surface in [event-kinds.md](event-kinds.md);
scripts and exit construction in [onchain.md](onchain.md); the full
adversarial matrix in [../analysis/attacks.md](../analysis/attacks.md);
ceremony-relevant economics (bonds, fees, tower market) in
[../analysis/economics.md](../analysis/economics.md); external sources in
[../research/sources.md](../research/sources.md).*
