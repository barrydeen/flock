Flock protocol - DRAFT v0.3 - 2026-06-12

# On-Chain Construction

This document specifies every Bitcoin-level object in Flock: keys, output script
templates, transaction templates, the pre-signed tree, unilateral exit, and the
complete timelock schedule. Signing ceremonies live in [ceremonies.md](ceremonies.md),
the Nostr transport in [event-kinds.md](event-kinds.md), and the adversarial analysis
in [../analysis/attacks.md](../analysis/attacks.md).

Two design rules govern everything below:

1. **No novel cryptographic composition.** Every quorum-gated spend is a tapscript
   requiring **two independent BIP-340 signatures**: the user side signs single-key
   Schnorr or standard MuSig2 ([BIP-327](https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki));
   the quorum side signs plain FROST-Schnorr
   ([RFC 9591](https://www.rfc-editor.org/rfc/rfc9591), implemented by
   [ZF frost](https://github.com/ZcashFoundation/frost)). No nonce coupling exists
   between the two protocols anywhere. The cost — ~+70 vB of witness per
   quorum-gated input versus a single aggregate key — is paid deliberately.
2. **No consensus changes assumed.** Everything below runs on deployed Bitcoin
   (Taproot, BIP-68/65/112 timelocks, TRUC/v3 and P2A relay policy per Bitcoin
   Core 28+). What CTV/CSFS would simplify is noted in the
   [whitepaper](../whitepaper.md), not relied on here.

---

## 1. Keys

### 1.1 Master seed and derivation

A wallet holds exactly one secret: a BIP-39 seed. From it derive:

| Key | Derivation (draft paths) | Used for |
|---|---|---|
| Nostr identity key `nsec` | NIP-06: `m/44'/1237'/0'/0/0` | All Nostr authorship, gift-wrap ECDH, payment addressing (npub) |
| Bitcoin `xprv` | BIP-32 root | Everything below |
| Checkpointed-owner key `U_ckpt` | `m/84021'/0'/0'/<idx>` | Recovery tapleaf destination; re-derivable from seed alone, with no stored state |
| Leaf cooperative key `U` | `m/84021'/0'/1'/<idx>` | Owner key of leaf L1 on tree leaves at checkpoint time; needed in the seed-only scan to match leaf scriptPubKeys and rebuild control blocks |
| Claim destination keys | BIP-86: `m/86'/0'/0'` | Outputs of claim and refund transactions |
| Vault author key | `m/84021'/0'/2'/0` | Publishing encrypted wallet vaults, unlinkable to `nsec` (see [event-kinds.md](event-kinds.md), kind 37400) |

`<idx>` is the non-hardened index
`int(SHA256("FlockIdx/v0.3" ‖ pool_id ‖ round_seq ‖ leaf_slot)) mod 2^31` —
computable from public round data alone, so the derivation is stateless: *seed →
all of the above* with no auxiliary data and no stored indices. The `m/84021'`
purpose field is a draft placeholder pending BIP-style registration; the paths are
draft, the structure (one seed, deterministic fan-out, no stored indices) is
normative.

**Seed-only scan (recovery with no wallet state).** "Recoverable from the seed
alone" means, precisely: **seed plus any surviving copy of the public round
data**. The procedure:

1. Crawl kind-38400 pool advertisements from the compiled-in bootstrap relay set
   to enumerate candidate pools: `Q`, params, relay and archival-relay sets.
2. Fetch each pool's kind-38402 round results: every **cell txid** plus the
   per-cell inline tree commitments.
3. Confirm the cell txids on L1 via **compact block filters or a full node**.
   Block explorers MAY serve as additional cross-checks; they MUST NOT be the
   sole L1 view for any fail-closed decision.
4. Rebuild each cell's taptree from the inline tree commitments; derive `U` and
   `U_ckpt` for each (pool, round) and match against leaf scriptPubKeys.
5. Reconstruct control blocks from the rebuilt taptree (sibling tapleaf hashes +
   NUMS internal keys, §1.5) and spend: the recovery leaf after `T_exp + 288`,
   or unroll + claim before that.

The round data — 38402 events on pool and archival relays, vault copies, tower
copies, any cell peer — is public, replicated, and non-secret, but its
availability is a **nonzero assumption**: if every copy vanishes, the control
block cannot be rebuilt and the recovery leaf cannot be spent. Uncheckpointed
arkoor outputs additionally require the gift-wrapped package (§1.4).

### 1.2 Quorum key Q

`Q` is the FROST group public key of the pool's operators: **k = 4 of n = 7**,
PedPoP DKG at pool genesis, monthly proactive resharing that preserves `Q`
(share rotation with group-key invariance, as implemented in ZF frost refresh;
see [ceremonies.md](ceremonies.md)). `Q` produces ordinary BIP-340 signatures
via the secp256k1 "tr" ciphersuite
([frost-secp256k1-tr](https://github.com/ZcashFoundation/frost/pull/730)).

Placement rules:

- `Q` **never** appears as the Taproot internal key of any output holding user
  funds, and never participates in any MuSig2 aggregate. It appears only as the
  second key in two-signature tapleaves (rule 1 above).
- Outputs that are purely operator funds — connectors, Q change, sweep
  destinations — MAY use `Q` (BIP-341-tweaked) as a key-path internal key, since
  no user property depends on their script structure.

### 1.3 Ephemeral tree-signing keys and the deletion assumption

Each round-ceremony participant generates fresh **ephemeral signing keys**, one per
node on their root-to-leaf path. These keys are:

- generated from a CSPRNG, **never** derived from the seed, never written to disk,
  held in RAM only;
- used exactly once, as the user's contribution to the MuSig2 aggregate
  `AggU_branch` on each path node (§2.2);
- **deleted immediately after the user's partial signatures are emitted and the
  aggregate signatures have been verified**, before the cell transaction is
  broadcast. Rebuilds use fresh trees and fresh keys, so early deletion loses
  nothing.

The deletion assumption is the pseudo-covenant of covenant-less Ark
([clArk](https://ark-protocol.org/intro/clark/index.html)): a tree node can only be
spent by the pre-signed node transaction if no complete signing-key set survives.
Because every user is a MuSig2 cosigner on *every node above their own leaf*, **a
user's own deletion alone binds their entire path** — no reliance on anyone else
deleting anything. Non-derivability is the point: a key re-derivable from seed could
be re-signed with, so recoverability here would be a vulnerability. Operators publish
signed key-deletion attestations (kind 31401) for their own ephemeral material;
these carry reputational and legal weight only — retention is not cryptographically
provable, and the attestations are evidence, not enforcement.

### 1.4 Stealth per-payment keys

Arkoor payments pay a one-time key so that neither relays nor chain observers can
link payments to a receiver's npub. For sender Alice (secret `a`, pubkey `A`) paying
receiver Bob (pubkey `B`, taken from his Flock payment descriptor — kind 38407, see
[event-kinds.md](event-kinds.md); fallback when Bob has published no 38407 and no
kind-10002: `B` = Bob's Nostr pubkey):

```
t   = H_tag( ECDH(a, B) ‖ cell_txid ‖ counter )
P_B = B + t·G
```

- `H_tag` is a BIP-340-style tagged hash; tag string [TBD].
- `cell_txid` is the txid of the confirmed cell transaction (§3.1) funding the
  leaf being spent.
- `counter` is a per-(sender, receiver, round) monotonic counter, included in the
  arkoor package. Its presence makes every payment's key unique even for repeated
  payments along the same edge — omitting it was a flagged linkage bug
  (see [../analysis/attacks.md](../analysis/attacks.md), P5).

Bob computes `t` from `ECDH(b, A)` using the sender pubkey and counter carried
inside the gift-wrapped package, and therefore controls `p_B = b + t`. `P_B` is what
appears in the arkoor output's tapleaves and in the claim-transaction destination;
`B` itself never appears on-chain or on a relay in cleartext. Consequence (conceded):
uncheckpointed arkoor outputs are recoverable only with the package, not from seed
alone, since `t` requires `A`, `cell_txid`, and `counter`.

### 1.5 Unspendable internal keys (NUMS)

Every output template holding user funds uses a provably unspendable Taproot
internal key, so that *only* the enumerated tapleaves can spend it
([BIP-341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)).
To avoid giving all Flock outputs an identical internal key (a chain-analysis
fingerprint and a cross-output malleability hazard), the internal key is the
BIP-341 recommended NUMS point `H` tweaked by a publicly re-derivable scalar:

```
K_int = H + H_tag'( cell_txid ‖ output_path )·G
```

with tag [TBD]. Every ceremony participant recomputes `K_int` for every output on
their path and refuses to sign on mismatch; the derivation is public, so
unspendability is verifiable by anyone.

---

## 2. Output templates

Notation: scripts are Bitcoin Script with named placeholders in angle brackets.
`<X> OP_CHECKSIGVERIFY <Y> OP_CHECKSIG` is the canonical two-signature quorum-gated
leaf. All CLTV/CSV values are block heights. `T_exp = cell-tx height + 4320`
(~30 days) is the cell's expiry anchor. `T_exp` is **strictly per cell**: sibling
cells of a round usually confirm within a few blocks of each other, but a rebuilt
cell may confirm at a different height than its siblings, so these values are
never round-wide (see [event-kinds.md](event-kinds.md) §3.9).

### 2.1 Boarding output

On-chain deposit awaiting conversion into a vTXO leaf. Internal key: NUMS (§1.5).
Tapleaves:

```
leaf B1 (conversion):  <U> OP_CHECKSIGVERIFY <Q> OP_CHECKSIG
leaf B2 (refund):      <288> OP_CHECKSEQUENCEVERIFY OP_DROP <U> OP_CHECKSIG
```

- **B1** is spent as an input of a cell transaction (§3.1), with `U` signing
  in-ceremony and `Q` signing via FROST. Conversion targets a cell of the same
  round in which the deposit confirms when phase timing allows.
- **B2** is the user's unconditional escape: after 288 blocks (~2 days,
  [BIP-112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) relative
  lock) the depositor can refund with their key alone. 288 rather than a
  pool-lifetime value deliberately bounds the boarding-strand window — a quorum
  that stalls after the deposit confirms can strand the deposit for at most ~2 days
  (see [../analysis/attacks.md](../analysis/attacks.md), L12).

### 2.2 Cell root / tree node output

The cell-root output of each cell transaction (one per cell, §3.1) and the outputs
of every tree node transaction. Internal key: NUMS. Tapleaves:

```
leaf N1 (tree path):   <AggU_branch> OP_CHECKSIGVERIFY <Q> OP_CHECKSIG
leaf N2 (node sweep):  <T_exp + sweep_delay> OP_CHECKLOCKTIMEVERIFY OP_DROP <Q> OP_CHECKSIG
```

`sweep_delay` is a pool policy parameter published in the 38400 `params`
(canonical default **4032**; it MUST exceed 288 so recovery matures first).

- **N1**: `AggU_branch` is the BIP-327 MuSig2 aggregate of the *ephemeral* keys
  (§1.3) of exactly the users whose leaves sit below this node. The only spend ever
  signed for N1 is the pre-signed child node transaction; after key deletion no
  other spend can ever be authorized, which is what makes the pre-signed tree a
  covenant in practice.
- **N2** is late garbage collection only: if a path is unrolled but its subtree is
  abandoned, `Q` can reclaim it `sweep_delay` blocks *after* expiry — long after
  every user remedy (recovery leaf at `T_exp + 288`, all claims) has matured. There is no
  per-level CSV on the tree path: relative locks along a path add serially
  (CSV clocks restart at each parent confirmation —
  [BIP-68](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki)), so
  any per-node delay would multiply worst-case exit latency by tree depth. All
  delay lives in the terminal claim instead.

### 2.3 vTXO leaf output

The user-facing output: tree leaves, and (with substituted keys) every arkoor chain
output. Internal key: NUMS. Tapleaves:

```
leaf L1 (cooperative): <U> OP_CHECKSIGVERIFY <Q> OP_CHECKSIG
leaf L2 (recovery):    <T_exp + 288> OP_CHECKLOCKTIMEVERIFY OP_DROP <U_ckpt> OP_CHECKSIG
leaf L3 (leaf sweep):  <T_exp + sweep_delay> OP_CHECKLOCKTIMEVERIFY OP_DROP <Q> OP_CHECKSIG
```

- **L1 (cooperative)** is the working path: arkoor transfers, forfeits, offboards,
  the pre-signed laddered claim (§3.3), and the intent-bond lien backstop
  ([ceremonies.md](ceremonies.md)) all spend it. `U` is the current owner
  key — the refresher's key for a tree leaf, the stealth key `P_B` for an arkoor
  output.
- **L2 (recovery)** pays the **checkpointed owner's seed-derivable key** after
  absolute height `T_exp + 288`
  ([BIP-65](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki)).
  This is the dormancy path, and it is **not unconditional**: an untouched cell is
  swept at its *root* at `T_exp + sweep_delay` (§3.5), so this leaf only ever
  materializes if someone unrolls first — the sweep-vs-recovery race is real. It
  is closed by a **standing tower deposit**: at every checkpoint the wallet
  pre-signs its recovery claim (recovery tapleaf → a seed-derived cold key) and
  lodges {unroll branch + recovery claim + CPFP children} as a self-funded
  kind-38404 height-deposit with `trigger_height = T_exp + 288`, replicated to
  ≥ 3 towers ([ceremonies.md](ceremonies.md) §5). Dormancy therefore degrades to
  **tower-protected self-custody**, not unconditional self-custody (anyone may
  still unroll — the node transactions are fully signed and third-party
  broadcastable — but no one is obliged to except the deposit towers). For arkoor
  outputs the recovery key is the receiver's `P_B`; note that this protects the
  receiver only at *this* output — the parent leaf's recovery key still names the
  pre-arkoor checkpointed owner, which is the conceded stale-owner window
  ([../analysis/attacks.md](../analysis/attacks.md), T10).
- **L3 (leaf sweep)** mirrors N2: quorum garbage collection `sweep_delay − 288`
  blocks (3,744 blocks ≈ 26 days at the canonical default) after the recovery
  leaf matures, never before.

**Why there is deliberately no bare CSV exit leaf.** Single-operator Ark gives the
leaf a `<delay> OP_CSV <U> OP_CHECKSIG` exit path and defends against exit of
already-spent vTXOs with a reactive forfeit race inside the CSV window
([second.tech forfeits](https://second.tech/docs/learn/forfeits)). Flock removes
that leaf entirely, for two reasons:

1. **Stale owners could bypass the ladder.** After Alice pays Bob by arkoor, the
   on-chain leaf script still names Alice as `U` (the leaf is pre-signed and
   immutable). A bare CSV leaf would let Alice — and every intermediate stale owner
   in a chain — exit unilaterally after a *uniform* delay, racing the rightful
   owner's claim in the mempool: exit ordering would devolve to a fee race plus a
   live-watchtower assumption, exactly what the ladder exists to eliminate.
2. **Exit must stay safe under a dead quorum.** Every self-spend is therefore
   forced through the cooperative path via a **Q-cosigned claim transaction whose
   nSequence encodes the owner's recency** (§3.3). Recency ordering is enforced by
   consensus (BIP-68), not by anyone being online — though it is *maturity*
   ordering, not preclusion: someone must still broadcast the newer chain before a
   matured stale claim spends (§3.3). The price: the claim must be
   obtained *at transfer time* — hence a transfer without its cosigned claim is
   invalid and receivers reject it.

### 2.4 Connector output

Per-refreshing-user output of the cell transaction; the atomicity gadget for
forfeits (§3.4), following the classical Ark connector design
([ark-protocol.org/intro/connectors](https://ark-protocol.org/intro/connectors/index.html)).

- Script: P2TR, key path only, internal key = tweaked `Q` (§1.2). No user funds.
- Value: policy dust for P2TR (330 sats at current relay policy; exact value [TBD]).
- One connector per refreshing user, at a deterministic output index committed in
  the cell proposal, so the user can verify *their* connector before signing their
  forfeit.

A connector's only purpose is to be an input of one forfeit transaction. Because
the connector is identified by `(cell_txid, vout)`, the forfeit that spends it is
consensus-invalid unless *that exact cell transaction* confirms.

### 2.5 Arkoor chain outputs

An arkoor transfer spends a leaf's L1 path off-chain and creates new outputs using
the **same template as §2.3** with substitutions: payment output `U := P_B`,
recovery key `:= P_B`; change output `U :=` sender's fresh key. Chain depth is
capped at 4 hops (§3.3); minimum leaf denomination is **10,000 sats** (sub-floor
change rides as uncheckpointed value).

---

## 3. Transaction templates

All pre-signed transactions are **nVersion = 3 (TRUC, BIP-431), zero fee, with one
zero-value P2A anchor output**
([BIP-433](https://github.com/bitcoin/bips/blob/master/bip-0433.mediawiki));
fees are attached at broadcast time by a CPFP child (§3.7). The cell and sweep
transactions are operator-funded v2 transactions signed at broadcast time. A segwit
txid commits to inputs and outputs but not witnesses
([BIP-141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki)), so a
cell txid is fixed the moment its input/output set is fixed — *before* any
signature exists. The whole pre-signing choreography leans on this.

### 3.1 Cell transaction (one per cell; the hourly round is the batch of up to 16)

Each cell is funded by its **own on-chain cell transaction**. The hourly **round**
is the batch of up to 16 cell txs — there is no shared funding transaction. This is
what makes cell isolation real: a rebuild re-signs **one cell tx only**, and no
other cell's txid, pre-signed node txs, forfeits, or boarding signatures move.

| Field | Value |
|---|---|
| nVersion | 2 |
| Inputs | Q capital UTXOs (key path, FROST); recycled sweep/forfeit outputs (key path, FROST); **this cell's** boarding outputs via leaf B1 (`U` in-ceremony + `Q`) |
| Outputs | 1 cell-root output (§2.2); one connector per refreshing user in this cell, ≤ 64 (§2.4); Q change |
| nSequence | 0xFFFFFFFE (locktime-enabling; no RBF signal) |
| nLockTime | current height (anti-fee-sniping) |
| Pre-signed? | No — Q signs at broadcast; boarding users sign their own input during the ceremony |
| Fees | Paid by Q at market rate; fee-bumping is **CPFP-only**, by spending the Q change output. **RBF is forbidden**: any input/output change — even shrinking Q's change for fees — changes the cell txid, voiding every pre-signed node tx, every collected forfeit, and every boarding signature of this cell |
| Broadcast | Q, at ceremony close |

The cell-tx template travels in the kind-24401 cell proposal; it is the only
nVersion=2, fee-paying transaction there — all pre-signed txs (node, claim,
forfeit) are nVersion=3 zero-fee P2A. Wallets verify their **cell txid** as soon
as their cell reaches SET-FINAL, and treat a refresh as final only after verifying
that the **confirmed** cell transaction's outputs byte-match their locally
recomputed templates (defense against view-split substitution; see
[../analysis/attacks.md](../analysis/attacks.md), T9).

### 3.2 Tree node transactions (pre-signed, broadcast only on exit)

| Field | Value |
|---|---|
| nVersion | 3 (TRUC) |
| Input | Parent output (cell root or node) via leaf N1: MuSig2 sig by `AggU_branch` + FROST sig by `Q` |
| Outputs | 4 child outputs (node template §2.2, or leaf template §2.3 at depth 3) + 1 zero-value P2A anchor |
| nSequence | 0xFFFFFFFD (no relative lock — see §2.2 on serial CSV) |
| nLockTime | 0 |
| Pre-signed | In-ceremony, before cell-tx broadcast, by all users below the branch + Q; ephemeral keys deleted immediately after partials are emitted and the aggregate signatures verify (§1.3) |
| Fees | Zero; CPFP child at broadcast |
| Broadcast | Anyone holding the package (owner, watchtower, Q), only during exit/unroll/sweep-preparation |

Per cell: 1 + 4 + 16 = **21 node transactions** create 64 leaves; 16 cells per
round = 336 node transactions and 1024 leaves per pool round.

### 3.3 Claim transactions — the decrementing ladder

Every owner of a (v)TXO holds a pre-signed **claim transaction**: the only
authorized self-spend of the cooperative path.

| Field | Value |
|---|---|
| nVersion | 3 (TRUC) |
| Input | The owned output (leaf or arkoor output) via leaf L1: owner key + `Q` |
| Input nSequence | **Ladder value** (BIP-68, height type): hop 0 = 720, hop 1 = 576, hop 2 = 432, hop 3 = 288, hop 4 = 144 |
| Output | Owner's destination key (seed-derived BIP-86 for hop 0; `P_B` for arkoor hops) + P2A anchor |
| nLockTime | 0 |
| Pre-signed | `Q` signs (FROST, SIGHASH_DEFAULT) at checkpoint time (hop 0) or atomically with each arkoor cosign (hops 1–4). The owner adds their own signature at broadcast time — they always can, so only Q's half must be obtained in advance |
| Broadcast | Owner (or tower under a kind-38404 deposit — height- **and conflict-**triggered), after unroll, once nSequence matures |

Rules:

- **Decrement 144 per hop; cap 4 hops; reset to hop 0 at every checkpoint.**
  `Q` refuses to cosign a fifth hop; the holder must checkpoint (refresh) to spend
  onward.
- **A transfer without its Q-cosigned claim is invalid**; receivers verify Q's
  claim signature before accepting payment. This is what guarantees the ladder is
  complete even if the quorum later dies.
- The ladder decrements (rather than incrementing from the oldest owner) because
  nSequence is fixed at signing time: later owners sign later, and nothing can
  retroactively lengthen an already-signed claim — so hop 0 starts high and recency
  buys shorter delays. The 144 floor preserves a one-day forfeit-broadcast window
  (§3.4). The construction follows Spark's decrementing-timelock exit precedent
  ([docs.spark.money/learn/technical-definitions](https://docs.spark.money/learn/technical-definitions)),
  moved from absolute to relative locks so it composes with unroll-at-any-time.

**Worked example.** Alice refreshes into a leaf at round R (hop 0). She pays Bob
(hop 1), Bob pays Carol (hop 2), Carol pays Dave (hop 3), Dave pays Erin (hop 4).
The pool later unrolls; the leaf output confirms at height `X`. Each owner's claim
spends their own chain tip, so reaching hop k's output costs k additional arkoor-tx
confirmations (nSequence 0, broadcastable immediately; assume ~1 block each):

| Owner | Hop | Claim nSequence | Chain txs first | Earliest claim height |
|---|---|---|---|---|
| Erin (current) | 4 | 144 | 4 (confirm ≈ X+4) | **X + 148** |
| Dave (stale) | 3 | 288 | 3 | X + 291 |
| Carol (stale) | 2 | 432 | 2 | X + 434 |
| Bob (stale) | 1 | 576 | 1 | X + 577 |
| Alice (stale) | 0 | 720 | 0 | X + 720 |

Maturity at hop k ≈ `X + k·c + (720 − 144k)` with per-tx confirmation latency `c`:
strictly decreasing in k whenever `c < 144` blocks (~1 day per hop — generous).
The current owner wins by consensus ordering, with no mempool race and no live
quorum. Moreover Erin's arkoor-chain broadcast (nSequence 0) *conflicts* with every
stale claim: once arkoor₁ confirms, Alice's claim is permanently dead, not merely
delayed. But the ladder orders **maturities**, it does not preclude matured stale
claims: a prior owner who unrolls early and outwaits an absent Erin collects
unless Erin — or a tower holding her conflict-triggered kind-38404 deposit —
broadcasts the chain before the stale claim matures. That **ladder defense
deadline** is the attacker's claim maturity, **288–720 blocks after the unroll**
depending on the attacker's hop (a hop-3 staler matures at 288, a hop-0 at 720);
Erin's head-start **differential** over the attacker is **144–576 blocks**
(144 per hop of staleness)
([../analysis/attacks.md](../analysis/attacks.md), T5). If the current owner
never appears and no tower acts, the most recent owner who does appear
collects — laddered claims double as last-resort recovery. When Erin checkpoints,
the new leaf restarts at hop 0 / 720 with the recovery leaf rotated to her, and her
old chain is forfeited (§3.4).

### 3.4 Forfeit transactions — connector atomicity

A forfeit hands the old vTXO to Q when its owner refreshes into a new leaf,
replacing hash-lock atomicity (Bark-style preimage gating) with transaction-graph
binding.

| Field | Value |
|---|---|
| nVersion | 3 (TRUC) |
| Input 1 | Old leaf via L1: old owner `U` (signed in-ceremony) + `Q` (FROST) — nSequence **0** |
| Input 2 | The user's connector `(cell_txid, vout)` of the **new** cell tx: key path `Q` — nSequence 0 |
| Outputs | Old-leaf value + connector dust to `Q` + P2A anchor |
| nLockTime | 0 |
| Pre-signed | Fully signed in-ceremony, *after* the cell txid is fixed (SET-FINAL) and the user has verified their new leaf in it; copies distributed to watchtowers **together with the full ancestor arkoor chain and each tx's pre-signed CPFP bump children (escalating ladder, §3.7)** — never the forfeit alone |
| Broadcast | **Anyone**, once both prevouts exist on-chain: the connector exists when the new cell tx confirms; the old-leaf prevout exists only after someone unrolls **and — for forfeits of arkoor chain tips — after every ancestor arkoor tx confirms serially (TRUC 1p1c: one package per hop)** |

**Why this is atomic, step by step:**

1. The cell ceremony fixes the cell transaction's complete input/output set
   (SET-FINAL). Its txid is now determined (BIP-141: txid excludes witnesses) even
   though nothing is signed yet.
2. The user verifies, against the canonical cell proposal: their new leaf's value
   and scripts in the proposed tree, and their own connector at its committed index.
3. Only then do they sign the forfeit. Input 2's outpoint embeds the exact cell
   txid: **the forfeit is consensus-invalid unless precisely that cell transaction
   confirms.** A cell tx mutated in any way — one output changed, one input
   swapped — has a different txid, the connector outpoint never exists, and the
   forfeit is garbage; the old vTXO remains intact and exitable. The blast radius
   of any mutation or rebuild is one cell (§3.1).
4. Q broadcasts the cell tx. Confirmation is a single atomic L1 event that
   *simultaneously* (a) brings the user's verified new leaf into existence and
   (b) makes the forfeit valid.
5. **Nothing can be withheld.** Contrast hash-locks: there, a post-confirmation
   preimage reveal sits between the user and atomicity, and withholding it freezes
   the refresh (see [../analysis/attacks.md](../analysis/attacks.md), T11).
   Here there is no reveal step, no shared secret, and no message
   after broadcast: both outcomes of the only remaining event (confirm / not
   confirm) leave the user whole. The user's residual obligation is honest L1
   verification of which cell tx actually confirmed.

A confirmed forfeit beats even the shortest claim (nSequence 0 vs 144) — including
after quorum death, since watchtowers hold complete copies. But "confirms
immediately upon a stale unroll" holds only when the forfeited output is the leaf
itself. When it is an **arkoor chain tip** — the common case for checkpointed
arkoor-received value — its prevout exists only after up to 4 intermediate arkoor
transactions confirm **serially**: TRUC permits 1-parent-1-child packages only, so
the chain cannot be broadcast in one shot. This is why tower copies MUST include
the full ancestor arkoor chain with per-tx CPFP bump children (§3.7). The
defender's window is the **forfeit stale window**: the attacking stale owner's
claim nSequence — **144–720 blocks depending on which hop attacks** (a hop-4
staler allows 144 blocks; a hop-0 staler allows 720) — and the serial
confirmations consume part of it. (Distinct from the 288–720-block ladder
defense deadline of §3.3, which governs stale-owner attacks on uncheckpointed
chains.) Residual (conceded): *someone* must
broadcast the held forfeit package within that window
([../analysis/attacks.md](../analysis/attacks.md), T5; open problem 9).

### 3.5 Sweep transactions

| Field | Value |
|---|---|
| nVersion | 2 |
| Inputs | Expired outputs via sweep leaves N2/L3 (FROST sig by Q); at whatever level the tree currently sits on-chain — an untouched cell is swept at its root in one input |
| Outputs | Q consolidation |
| Input nSequence | 0xFFFFFFFE (locktime-enabling) |
| nLockTime | ≥ `T_exp + sweep_delay` (CLTV requirement; canonical default 4032) |
| Pre-signed | No — Q is alive by definition; signed at broadcast, market fees |

Sweeping a never-unrolled cell at its root reclaims abandoned value in one input —
which means a dormant user's recovery claim requires the tree to be unrolled
*before* `T_exp + sweep_delay`. The sweep-vs-recovery race is real, and **it makes
dormancy tower-dependent**: the recovery leaf never materializes on-chain unless
someone unrolls, and the only party obliged to is the standing tower deposit of
§2.3 (lodged at every checkpoint, `trigger_height = T_exp + 288`, ≥ 3 towers). The
guarantee, stated plainly (per amendments A2/A21): *a dormant wallet pays no
carrying fees and need never come online, and its funds are not confiscated
while towers function — but ~32 days after its last checkpoint the standing
deposit automatically exits them to on-chain self-custody at the owner's
expense* (≈3.1k–16k sats at 10 sat/vB, 5× at 50, plus the ~2,000-sat contingent
tower fee — §3.7). Absolute worst case — the owner stays absent AND no tower
acts within the 3,744-block (~26-day) window — the quorum sweep at
`T_exp + 4032` confiscates. `sweep_delay` is a pool policy parameter (canonical default 4032);
shortening it cheapens refreshes by shrinking Q's float lockup, at the price of a
narrower dormancy grace window
([../analysis/economics.md](../analysis/economics.md)). See
[../analysis/attacks.md](../analysis/attacks.md), L2/L6; the tower market (open
problem 10) now carries the dormancy guarantee.

### 3.6 Boarding refund

| Field | Value |
|---|---|
| nVersion | 2 |
| Input | Boarding output via leaf B2, `U` only; nSequence = **288** (BIP-68) |
| Output | User's BIP-86 key |
| nLockTime | 0 |
| Pre-signed | Not needed — single-key; user signs whenever, valid 288 blocks after the deposit confirms |

### 3.7 CPFP children and fee UTXOs

Every pre-signed package carries its fees in a child:

- At creation time, the owner **pre-signs their own minimum-feerate child** spending
  the parent's P2A anchor plus a **dedicated fee UTXO** — a confirmed, single-purpose
  P2TR UTXO never spent elsewhere, so no third party can conflict it out from under
  the child. This minimum-feerate variant serves the owner's own broadcasts, where
  the child can be replaced live (§5.1); packages lodged as height-deposits carry
  the escalating ladder below. Fee-UTXO provisioning policy (count, denominations)
  is [TBD].
- The anchor is zero-value ephemeral dust, legal only if spent in the same package
  (Bitcoin Core 29, [PR 30239](https://github.com/bitcoin/bitcoin/pull/30239)) —
  so **the anchor is never a free slot**: the package always ships parent+child, and
  TRUC's 1-parent-1-child topology plus sibling eviction prevent an attacker from
  squatting the only child slot with a low-fee tx
  ([Core 28 wallet integration guide](https://bitcoinops.org/en/bitcoin-core-28-wallet-integration-guide/)).
- P2A is keyless: anyone — owner, watchtower, even Q — can attach or replace a
  child at a higher feerate without holding any key.
- **Tower bump funding and fee output.** A tower bumping via the keyless anchor
  alone would pay out of its own pocket (a dormant first exit is ~1.6 kvB —
  ~16k sats at 10 sat/vB — against a ~2k fee), so every package lodged as a
  watchtower height-deposit (kind 38404) carries **pre-signed escalating bump
  children** — canonical ladder **10 / 50 / 200 sat/vB** — each spending the
  parent's P2A anchor plus the owner's dedicated fee UTXOs, one variant set per
  named tower. Each variant carries a **fee output (canonical default
  ~2,000 sats) paying that tower's per-request key**: the acting tower
  broadcasts the lowest variant that clears current feerates, and its
  out-of-pocket cost is ≈ 0. The variants conflict — whichever tower completes
  the broadcast collects — so at most one tower fee is ever paid, and **nothing
  is paid if no tower ever needs to act**: the fee exists only inside the
  contingent children, a contingent allocation rather than a per-send payment.
  Residual (conceded): feerates above the 200 sat/vB pre-signed ceiling stall
  the package until the owner returns or a third party funds the keyless
  anchor — this folds into the replacement-cycling/fee-spike problem
  ([../analysis/attacks.md](../analysis/attacks.md), L7).

---

## 4. Tree construction

Each hourly round is a batch of up to **16 cell transactions** (§3.1), each
funding one **independent cell**: a **radix-4, depth-3** pre-signed tree over
**64 leaves** (1024 leaves per round). Cells are independent failure domains in
the strong sense: one ceremony aborter delays one 64-user cell by one attempt,
not the round, and a rebuild re-signs that one cell tx only — no other cell's
txid or signatures move (see [ceremonies.md](ceremonies.md) for phase timers,
post-nonce participant finalization, and intent bonds — a ledger debit netted at
next service, backstopped by a pre-signed lien tx spending the refreshing leaf's
cooperative path L1; no standalone bond vTXO exists).

Structure per cell:

| Level | Txs | Each creates | Cosigner set per node (`AggU_branch`) |
|---|---|---|---|
| Cell root | (cell tx output) | — | all 64 users |
| 1 | 1 | 4 node outputs | 64 |
| 2 | 4 | 4 node outputs each | 16 |
| 3 | 16 | 4 leaf outputs each | 4 |

**What each user co-signs: their root-to-leaf path only** — exactly 3 node
transactions, in MuSig2 cohorts of size 64, 16, and 4, using ephemeral keys (§1.3).
They additionally sign: the forfeit for their old leaf (§3.4), and — if boarding —
their boarding input on their cell tx. Q cosigns every node (FROST) and the hop-0
claim for every new leaf.

**What each user verifies before emitting any partial signature:**

1. Their leaf exists in the proposed tree with the correct value (≥ 10,000 sats)
   and the exact §2.3 script set, `T_exp` matching the round announcement, recovery
   key = their own seed-derived `U_ckpt`.
2. Every output script on their path recomputes byte-for-byte from published data:
   `AggU_branch` from the canonical participant ephemeral-key list, NUMS internal
   keys (§1.5), sweep leaves at `T_exp + sweep_delay` per the announced pool
   parameters.
3. The canonical participant-set commitment in the cell proposal matches the set
   they are signing with (post-nonce finalization re-fixes this; any change to the
   set or message after nonces mandates fresh nonces — nonce discipline in
   [ceremonies.md](ceremonies.md)).
4. Their connector is present at its committed index and the forfeit they are about
   to sign spends exactly `(cell_txid, that index)`.
5. Their hop-0 claim: Q's signature valid, nSequence = 720, destination = their key.
6. The full session transcript (signer set, pubnonce set, message digests) — all
   cosigners cross-verify the same transcript before partials.

Immediately after emitting partials and verifying the aggregate signatures —
before the cell tx is broadcast — delete the ephemeral keys (§1.3). Persist the
path package (node txs + claim + forfeit copies) locally and to the vault
(kind 37400). Verify the cell txid at SET-FINAL; after the cell tx confirms,
verify the confirmed outputs against the local recomputation before treating the
refresh as final.

---

## 5. Unilateral exit

### 5.1 Unroll procedure

1. Assemble the package: from local storage or vault, else rebuild from the
   kind-38402 inline tree commitment. Check the transfer log (kind 30178) and
   held forfeits — exiting an already-forfeited leaf is harmless (the forfeit
   collects it) but wastes fees.
2. Broadcast the depth-1 node transaction as a **1p1c package** — `[node tx,
   its pre-signed min-fee child]` — via `submitpackage`. Bump by replacing the
   child at a higher feerate if needed (package RBF). 1p1c relay is opportunistic
   until cluster-mempool nodes dominate
   ([Core 28 guide](https://bitcoinops.org/en/bitcoin-core-28-wallet-integration-guide/),
   [cluster mempool](https://bitcoinops.org/en/topics/cluster-mempool/) — merged for
   Core 31, **not** assumed network-wide); rebroadcast across diverse peers.
3. Await confirmation; repeat for depth-2, then depth-3. No CSV accrues along the
   path (§2.2): total latency is 3 confirmations plus the terminal claim delay.
4. For arkoor-chained funds, broadcast the chain transactions (nSequence 0) the
   same way, one package per hop.
5. Broadcast the claim after its nSequence matures (144–720 blocks per §3.3).
   Watchtower height-deposits (kind 38404) automate steps 2–5 for offline owners;
   they fire at `trigger_height` **and immediately on conflict detection** (any
   spend touching the deposited package's ancestor outpoints — see
   [ceremonies.md](ceremonies.md) §5).

TRUC limits are respected by construction: node txs ≈ 321 vB ≪ 10,000 vB parent
cap; children ≈ 115 vB ≪ 1,000 vB child cap.

### 5.2 Vbyte accounting

Component sizes (canonical figures from the synthesis; arithmetic shown):

| Component | vB | Arithmetic |
|---|---|---|
| Two-sig tapscript input | **126** | 41 non-witness (36 outpoint + 1 + 4 nSequence) + witness ≈ 300 WU / 4 ≈ 75 (two 64-B sigs ≈ 130, 70-B script + len ≈ 71, ≤ 97-B control block + len ≈ 98, count ≈ 1) ⇒ ≈ 116 computed; **126 is the canonical budget** (margin for sighash bytes, deeper control blocks, serialization variance). All totals below use 126. |
| P2TR output | 43 | 8 amount + 1 len + 34 script |
| P2A anchor output | 13 | 8 + 1 + 4 (`OP_1 <0x4e73>`) |
| Tx overhead | 10.5 | 4 version + 4 locktime + 2 counts + 0.5 marker/flag |
| **Node tx** | **321** | 126 + 4×43 + 13 + 10.5 = 321.5 |
| **Claim tx** | **193** | 126 + 43 + 13 + 10.5 = 192.5 |
| **Arkoor hop tx** | **235** | 126 + 2×43 (payment+change) + 13 + 10.5 = 235.5 |
| **CPFP child** | **≈115** | P2A input ≈ 41 + P2TR key-spend fee input ≈ 58 (41 + 66 WU/4) + overhead 10.5 + minimal residual output ≈ 5–13 |

Exit totals at 10 sat/vB:

| Scenario | Arithmetic | Total | Cost |
|---|---|---|---|
| First exiter (full path) | 3×321 + 193 + 4×115 = 963 + 193 + 460 | **≈ 1,616 vB ≈ 1.6 kvB** | ~16k sats |
| Path already unrolled | 193 + 115 | **≈ 310 vB** | ~3.1k sats |
| Per arkoor hop in chain | +235 (parent; its bump child, where needed, is additional) | **+235 vB/hop** (≤ 4) | +2.35k sats |

### 5.3 Mass exit

Full unroll of one 1024-leaf pool round. Batch-bumping is impossible under the
design's own relay rules — ephemeral-dust anchors must be spent in the same
package and TRUC forbids a child with more than one unconfirmed parent (1p1c
only) — so **every TRUC parent needs its own ≈115 vB CPFP child**:

```
336 node txs   × 321 vB = 107,856 vB
1,024 claims   × 193 vB = 197,632 vB
1,360 children × 115 vB = 156,400 vB   (one per parent — no batch-bumping)
total                   = 461,888 vB   ≈ 451 vB/user
                                       ≈ 0.46 of a 1 MvB block per pool round
```

100,000 users ≈ 98 pool rounds ≈ **46 blocks** of total demand. Mitigations against
the known mass-exit fee dynamics
([Sguanci & Sidiropoulos 2023](https://arxiv.org/abs/2208.01908) — pre-TRUC, but
the congestion mechanism stands): expiries are laddered hourly (≈ 6-block rungs,
deadlines never synchronize), wallets carry a 1008-block refresh margin, exits are
pre-signed and pinning-hardened, and keyless P2A lets third parties fund the
priced-out. Expensive and slow, but safe and counterparty-free. Conceded residuals:
replacement cycling has no complete mempool-level fix (L7), and value near its
~310 vB marginal exit cost is economically strandable
([../analysis/economics.md](../analysis/economics.md),
[../analysis/attacks.md](../analysis/attacks.md), G3).

---

## 6. Timelock ledger

Consensus-enforced locks:

| Lock | Mechanism | Value | Lives in | Spender | Purpose |
|---|---|---|---|---|---|
| Forfeit | nSequence | 0 | forfeit tx inputs | anyone (fully signed) | Beats every claim on refresh-stale leaves, quorum dead or alive |
| Claim, hop 4…0 | nSequence (BIP-68) | 144 / 288 / 432 / 576 / 720 | claim tx input | owner + pre-signed Q | Consensus-ordered owner recency; newest matures first |
| Boarding refund | CSV (BIP-112) | 288 | leaf B2 | depositor alone | Bounds boarding strand to ~2 days |
| Recovery leaf | CLTV (BIP-65) | `T_exp + 288` | leaf L2 | `U_ckpt` alone | Dormancy degrades to tower-protected self-custody (§2.3, §3.5); seed + public-round-data recovery (§1.1) |
| Node sweep | CLTV | `T_exp + sweep_delay` (pool parameter; canonical default 4032) | leaf N2 | Q | Garbage collection of abandoned branches |
| Leaf sweep | CLTV | `T_exp + sweep_delay` (pool parameter; canonical default 4032) | leaf L3 | Q | Garbage collection of abandoned leaves |
| Gateway swap delta | CLTV (vHTLC) | 144 | swap contracts | gateway/user | Standard HTLC exposure window ([event-kinds.md](event-kinds.md), kind 38406) |

Protocol heights (wallet/tower policy, not script):

| Deadline | Value | Meaning |
|---|---|---|
| Receive ban | `T_exp − 2016` | No arkoor receives against older rounds; bounds the recovery-leaf stale-owner window |
| Wallet refresh deadline | `T_exp − 1008` | Wallet refreshes or begins exit; ~1 week of fee-market margin |
| Tower height-deposit trigger | `T_exp − 720` | No third party is needed before `T_exp − 720`; at `T_exp − 720` the height-deposit tower must act. Deposits also fire **immediately on conflict detection** (any spend touching the deposited package's ancestor outpoints), not only at `trigger_height` |
| `T_exp` | `cell-tx height + 4320` | Pool expiry anchor (~30 days) |
| Standing dormancy deposit trigger | `T_exp + 288` | Lodged at every checkpoint, replicated to ≥ 3 towers; exits dormant funds the moment the recovery leaf matures (§2.3, §3.5) |

**Ordering invariants** (all enforced by the constants above; any parameter change
must re-verify them):

1. **Forfeits beat all claims.** `0 < 144` — a held forfeit confirms before even the
   current owner's fastest claim matures. Stale-state exit is never profitable
   while one forfeit-package holder is awake within the attacker's 144–720-block
   forfeit stale window, net of the serial ancestor confirmations a chain-tip
   forfeit needs (§3.4).
2. **Current owner beats all stale owners.** Hop maturities `X + k·c + (720 − 144k)`
   are strictly decreasing in k for per-hop confirmation latency `c < 144`; the
   chain broadcast itself kills stale claims outright on confirmation (§3.3).
3. **Recovery matures before the quorum sweep.** `T_exp + 288 < T_exp +
   sweep_delay` (`sweep_delay > 288` is a hard pool-parameter constraint): 3,744
   blocks (~26 days) at the canonical default in which the checkpointed owner —
   or the standing tower deposits acting for them — moves before Q can collect.
   Dormancy is **tower-protected** custody, not unconditional: if the owner stays
   absent and no tower acts within the window, the sweep confiscates (§3.5).
4. **Exits fit inside expiry margins.** Worst-case exit latency (3 node
   confirmations + 720-block hop-0 claim) is well under the 1008-block wallet
   margin plus the post-expiry sweep gap (`sweep_delay`, canonical 4032); no path
   CSV accumulates (§2.2), so depth does not multiply delay.
5. **The receive ban outruns recovery staleness.** `2016 > 1008 > 720`: any value
   received before the ban can still be checkpointed (hourly rounds) before wallet
   and tower deadlines pass, bounding the value exposed to a stale recovery key to
   sub-VaR-cap (50,000 sats, applied to the aggregate uncheckpointed balance;
   enforced while the wallet is online) uncheckpointed remainders — the conceded residual of
   [../analysis/attacks.md](../analysis/attacks.md), T10.
6. **Boarding refund precedes everything.** `288 ≪ 4320`: an unconverted deposit
   returns to its owner ~28 days before any pool-lifetime event could touch it.
