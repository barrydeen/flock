Flock protocol - DRAFT v0.3 - 2026-06-12

# Flock Event Kinds: the Nostr Protocol Surface

This document specifies every Nostr event kind used by Flock, NIP-style. It is normative for wire formats, tag semantics, encryption, validation, and retention. On-chain scripts and transactions are specified in [onchain.md](onchain.md); ceremony state machines and nonce rules in [ceremonies.md](ceremonies.md); the threat model in [../analysis/attacks.md](../analysis/attacks.md); design rationale in [../whitepaper.md](../whitepaper.md).

Kind numbers are placeholders pending NIP registration. All events follow NIP-01 semantics. The words MUST, SHOULD, MAY are used per RFC 2119.

---

## 1. Conventions

### 1.1 Pool identifier

A pool is identified by its FROST group public key **Q** (32-byte x-only, lowercase hex). `pool_id = hex(Q)`. This is self-certifying: monthly proactive resharing changes operator shares but never the group key, so `pool_id` is stable for the life of the pool. Clients MUST pin `pool_id` on first use and verify it out-of-band where possible; a different Q is a different pool, full stop.

Because `pool_id` is a public key, the **pool tag** is an ordinary indexed `p` tag: `["p", "<pool_id>"]`. Every Flock event that is not authored by Q itself MUST carry the pool tag. Events authored by Q are already filterable by `authors`.

### 1.2 Round, cell, and session identifiers

- `round_seq`: strictly increasing u64 per pool, one per hourly round. A **round** is the hourly batch of up to 16 per-cell funding transactions (**cell txs**); there is no single round-wide funding tx (see [onchain.md](onchain.md)).
- `cell_index`: 0–15 (16 cells of 64 leaves per round).
- `attempt`: u32, a fresh ceremony attempt for the cell; starting a new attempt resets `sub_attempt` to 0.
- `sub_attempt`: u32, incremented on every REBUILD within the same attempt (member exclusion + re-propose; [ceremonies.md](ceremonies.md) rule N7).
- **Session id** (32-byte hex):

```
session_id = SHA256("FlockSession/v0.3" || pool_id || round_seq || cell_index ||
                    attempt || sub_attempt || pset_commitment || msg_digest)
```

where `pset_commitment` is the canonical participant-set commitment (SHA-256 over the lexicographically sorted list of participant session pubkeys, hex, no separators) and `msg_digest` is the BIP-341 sighash of the transaction being signed. Sessions are idempotent **via session ids, never via nonce reuse**: any retry, rebuild, or timeout produces a new `session_id` and fresh CSPRNG nonces (see [ceremonies.md](ceremonies.md)). Ceremony events carry `["s", "<session_id>"]` — a single-letter tag so relays index it; the possibility of collision with other protocols' use of `s` is acceptable on pool relays.

### 1.3 Quorum-authored events

Events "authored by Q" are ordinary NIP-01 events whose `pubkey` is Q and whose `sig` is a plain BIP-340 Schnorr signature produced by the 4-of-7 FROST quorum over the event id (the same signing path used for Bitcoin transactions; see [ceremonies.md](ceremonies.md)). On the wire they are indistinguishable from single-key events. Any 4 operators can author them; fewer cannot.

### 1.4 Canonical transcript signing

Every MuSig2 ceremony binds its participants to one view of the session before any partial signature is released:

```
transcript_hash = SHA256("FlockTranscript/v0.3" || session_id ||
                         sorted(pubnonce_list) || pset_commitment || msg_digest)
```

Serialization is lowercase hex, lexicographically sorted where a list is named, concatenated without separators. Each signer signs `transcript_hash` (BIP-340, session key) and includes that signature in its kind-24403 event. A signer MUST verify that every other participant's transcript signature covers an identical `transcript_hash` before treating the session as complete; any divergence aborts the session (fresh nonces, new `session_id`). Equivalent epoch-tagged transcripts govern intra-quorum DKG/resharing over kind 24404.

### 1.5 Replication verification (what ≥k-relay replication actually means)

Several kinds (transfer log, round results, arkoor finality) require proof of replication on **≥ k = 4** relays, of which at least one MUST be operator-independent (the pool relay set is ≥ 8 relays with ≥ 3 mandated operator-independent, listed and flagged in kind 38400).

Honesty about NIP-01: an `OK` response is **not signed** and proves nothing beyond "this socket said yes." Flock therefore defines replication verification as **read-back**: the verifier fetches the event by id over **disjoint, independently established connections** — separate TCP/TLS (or Tor circuit) sessions, ideally established at different times, optionally NIP-42-authenticated — to k distinct relays from the pool set, and requires byte-identical events from each. Disjoint connections defend against a single intercepted session mirroring writes back; they do not defend against a global network-position adversary (conceded — T3/L5 in [../analysis/attacks.md](../analysis/attacks.md)).

Read-back verification over disjoint connections is the **normative completion condition** for every send and publish step in this protocol; wherever ceremonies.md or the whitepaper say a send "completes", they mean this section. Replication evidence gathered this way is **local**, not portable: you can act on it, but you cannot prove it to a third party. Pool relays SHOULD additionally implement a non-standard signed proof-of-write receipt (relay-signed `(event_id, timestamp)`) as defense-in-depth; where available these receipts make AUTH denial and retention failure attributable, but they are never the completion condition. Their format is [FUTURE] pending a receipts NIP.

Clients verifying finality-critical events (kind 30178 above all) MUST fetch from disjoint relay sets **including operator-independent relays and fail closed**: absence or timeout means "not final," never "probably fine."

### 1.6 Outbox routing (NIP-65)

Wallets, operators, gateways, and towers publish kind-10002 relay lists. Routing rules:

- **Arkoor gift wraps (1059)**: published to the **pool relay set (mandatory in all cases)** plus the receiver's read relays when known (best-effort; a zero-footprint receiver has none — §1.10).
- **Quorum and operator events** (38400/38401/38402/30178/31401): published to the full pool relay set.
- **Vaults (37400)**: author's write relays plus pool relays plus the client's pinned bootstrap set.
- **Payment descriptors (38407)**: the receiver's write relays plus the relay sets of its preferred pools.
- **Ceremony and cosign traffic (24400–24405)**: pool relay set only.

NIP-65 alone is an eclipse vector: clients MUST keep a hard-coded, diverse bootstrap relay list and MUST NOT treat a kind-10002 list as the sole source of read locations for finality-critical fetches.

### 1.7 Proof of work (NIP-13)

Refresh intents and arkoor gift wraps carry NIP-13 proof of work mined on the **outer** event id (the 1059 wrap), with a `nonce` tag declaring the target.

- Required difficulty: **PoW ≥ 20 bits (relay/Q demand capped at 20)**.
- **Cap: 20 bits.** Relays in the pool set and the quorum MUST NOT require more, even under load. The cap is load-bearing: an inflatable PoW floor is a deniable censorship lever against mobile clients exactly during congestion (L11). Anti-spam beyond 20 bits comes from bonded intents, per-key quotas, and rate limits — not from PoW escalation.
- Pre-mining is permitted: NIP-59 wrap timestamps are already randomized into the past, so clients SHOULD mine intent/wrap PoW ahead of deadline windows.

### 1.8 Kind-number rationale (NIP-01 ranges)

| Range | NIP-01 class | Flock usage |
|---|---|---|
| 1000–9999 | regular (stored) | 8405 tower alerts, 9735 zap receipts, 1059 gift wraps — history that must accumulate |
| 20000–29999 | ephemeral (relayed, not stored) | 24400–24405 ceremony and cosign chatter, 24406 (inner only) — worthless or hazardous after the session; storing partial-signature material invites replay analysis and bloats relays for nothing. Loss is handled by abort-and-retry, never by re-signing |
| 30000–39999 | addressable (latest per `(kind, pubkey, d)`) | 38400/38401/38402/38404/38406/38407, 31401, 30178, 37400 — records where "current version" is the meaningful object |

Kind 24406 never appears bare on a relay; it is the rumor inside a kind-1059 wrap. Its 2xxxx number signals "do not store unwrapped." Kind **8405** is the corrected tower-alert kind: the synthesis's 38405 fell in the addressable range, and alerts are evidence that must accumulate, so they are regular events ([ERRATA](../research/ERRATA.md)).

### 1.9 Archival relays

Operators MUST run **paid archival relays speaking plain NIP-01** with long retention, listed in kind-38400 `params.archive_relays`. Wherever this document or its companions say "the Q archive" or "the quorum's ciphertext archive", the meaning is: fetch the ciphertext events — 37400 vaults and 1059 arkoor wraps — from those archival relays with ordinary NIP-01 filters. There is no bespoke operator endpoint, and none may be load-bearing (requirement R2) **for checkpointed funds**. Honest scope: uncheckpointed receipts additionally depend on relay retention, archival relays included — a real availability dependency, conceded as open problem 2.

### 1.10 First-contact discovery

How a sender finds the key and relays for any npub — including one with zero prior footprint — and how a brand-new wallet finds its first payment:

- **Sender:** fetch the receiver's kind-38407 payment descriptor (§3.17) for the stealth base key B, preferred pools, and read relays. If there is no 38407, fall back to kind-10002 for read relays. If there is **neither 38407 nor 10002**, set **B = the receiver's Nostr pubkey** and rely on pool-relay delivery alone. In **all** cases the wrap MUST be published to the pool relay set (§1.6); the receiver's read relays are additive.
- **Receiver (new wallet, nothing published):** crawl kind-38400 pool advertisements from the compiled-in bootstrap relay set, then scan each pool's relay set for pool-tagged 1059 wraps and trial-decrypt (§3.11). Scaling concession: first-contact scan cost grows linearly with the number of pools; publishing a 38407 collapses the scan to the receiver's declared pools.

---

## 2. Kind index

| Kind | Class | Author | Purpose |
|---|---|---|---|
| 38400 | addressable | Q | Pool advertisement (quorum-signed; params incl. fees, sweep_delay, intent_bond_sats, archive_relays, optional uncheckpointed_ceiling) |
| 38401 | addressable | Q | Round/cell announcement + phase deadlines |
| 31401 | addressable | operator | Operator heartbeat + key-deletion attestation (reputational/legal weight only) |
| 24400 | ephemeral | wallet (wrapped) | Refresh intent (bonded, PoW-capped, gift-wrapped to Q) |
| 24401 | ephemeral | Q | Cell proposal incl. cell-tx template (NIP-44 to cell members) |
| 24402 | ephemeral | participant / Q | MuSig2 pubnonces (cohort sessions only) |
| 24403 | ephemeral | participant | MuSig2 partial signatures (cohort sessions only) |
| 24404 | ephemeral | operator | Intra-quorum FROST / DKG / reshare traffic |
| 24405 | ephemeral | wallet / Q (wrapped) | Arkoor cosign request/response (gift-wrapped; no MuSig2) |
| 24406 | ephemeral (inner) | one-time key | Arkoor payment package, carried in kind-1059 gift wrap |
| 38402 | addressable | Q | Round result: all cell txids + per-cell inline tree commitments |
| 30178 | addressable | Q | Transfer log (quorum-signed Merkle roots, per-cosign, increasing seq) |
| 37400 | addressable | derived key | Wallet vault (self-encrypted, unlinkable author key) |
| 38404 | addressable | one-time key | Watchtower height-deposit (height **and conflict** triggered) |
| 8405 | regular | tower | Tower alert *(corrected from 38405)* |
| 38406 | addressable | gateway / tower | Service offer book: gateways **and towers** |
| 38407 | addressable | receiver | Flock payment descriptor (stealth base key B, preferred pools, relays) |
| 9735 | regular | gateway/LNURL | Zap receipt (NIP-57 interop) |

---

## 3. Kind specifications

### 3.1 Kind 38400 — Pool advertisement

**Purpose.** The root discovery object: everything a wallet needs to evaluate and join a pool. **Class:** addressable — there is exactly one current advertisement per pool, and fee/relay changes must supersede old copies; `d` = `pool_id`. **Author:** Q.

```json
{
  "id": "...",
  "pubkey": "9f4e2a…c21a",
  "created_at": 1781136000,
  "kind": 38400,
  "tags": [["d", "9f4e2a…c21a"]],
  "content": "{\"v\":1,\"quorum\":{\"k\":4,\"n\":7,\"operators\":[\"a1…\",\"b2…\",\"c3…\",\"d4…\",\"e5…\",\"f6…\",\"07…\"]},\"relays\":[{\"url\":\"wss://r1.example\",\"operator_independent\":false},{\"url\":\"wss://r8.example\",\"operator_independent\":true}],\"fees\":{\"arkoor_base_sats\":2,\"arkoor_rate\":0.0005,\"refresh_per_refresh\":0.007,\"gateway\":[0.001,0.005],\"pinning_sats_month\":1500},\"params\":{\"round_interval_s\":3600,\"cells\":16,\"leaves_per_cell\":64,\"t_exp_delta\":4320,\"recovery_delta\":288,\"sweep_delay\":4032,\"ladder\":{\"hop0\":720,\"decrement\":144,\"max_hops\":4},\"boarding_csv\":288,\"receive_ban_blocks\":2016,\"refresh_deadline_delta\":1008,\"tower_exit_delta\":720,\"var_cap_sats\":50000,\"min_leaf_sats\":10000,\"intent_bond_sats\":1000,\"uncheckpointed_ceiling\":200000,\"phase_timer_s\":120,\"pow_bits\":20,\"archive_relays\":[\"wss://arc1.example\",\"wss://arc2.example\"]},\"services\":{\"offer_book_kind\":38406,\"gateways\":[\"<gateway npub>\"],\"towers\":[\"<tower npub>\"]},\"active_rounds\":[{\"round_seq\":712,\"funding_height\":957013,\"t_exp\":961333}]}",
  "sig": "..."
}
```

**Content schema.** `quorum` (k, n, the 7 operator npubs); `relays` (≥ 8 entries, ≥ 3 flagged `operator_independent`); `fees` (arkoor 1–3 sats base + 0.05%, refresh **0.5–0.9% of refreshed value per refresh** — each refresh locks operator capital ~58 days, break-even ≈0.8% per refresh at 5% APY — gateway 0.1–0.5%, pinning ~1–2k sats/month); `params` (the canonical constants — clients MUST reject a pool whose params diverge from their compiled-in expectations rather than silently adopt them), including `sweep_delay` (pool policy, canonical default 4032; a shorter value cheapens refreshes but shrinks the dormancy grace window), `intent_bond_sats` (canonical default 1,000; a ledger debit netted at next service, lien backstop — §3.4), `archive_relays` (operator-run archival relays, §1.9), and the optional `uncheckpointed_ceiling` (per-recipient uncheckpointed total above which Q refuses further cosigns — §3.16; privacy trade-off: Q already sees the graph); `services` (pointers into the kind-38406 offer book — discovery convenience, not endorsements); `active_rounds` (the T_exp ladder; canonical per-cell values live in 38402).

**Encryption:** none, public. **Validation:** valid sig by Q; relay set meets the 8/3 floor; every operator listed has a live kind-31401 heartbeat; params match client policy; `pool_id` matches a pinned value if previously seen (TOFU). **Retention:** life of pool + 60 days; all pool relays. **Typical filter:** `{"kinds":[38400],"authors":["<pool_id>"]}`; discovery crawlers use `{"kinds":[38400]}` on bootstrap relays.

### 3.2 Kind 38401 — Round/cell announcement

**Purpose.** Announces a round — the hourly batch of up to 16 per-cell funding transactions — its 16 cells, and the ceremony phase deadlines; updated in place as adaptive 120 s phase timers move. **Class:** addressable, `d` = `<pool_id>:<round_seq>` — one mutable schedule object per round. **Author:** Q.

```json
{
  "id": "...",
  "pubkey": "9f4e2a…c21a",
  "created_at": 1781139600,
  "kind": 38401,
  "tags": [["d", "9f4e2a…c21a:713"]],
  "content": "{\"round_seq\":713,\"target_funding_height\":957019,\"cells\":16,\"phases\":{\"intent_close\":1781139720,\"proposal_by\":1781139840,\"nonce_by\":1781139960,\"partial_by\":1781140080,\"broadcast_by\":1781140200},\"boarding_open\":true}",
  "sig": "..."
}
```

**Content schema.** `round_seq`; `target_funding_height`; per-phase unix deadlines (each phase nominally 120 s, adaptive); `boarding_open` (MUST be false while operator heartbeats are degraded). **Encryption:** none. **Validation:** signed by Q; `round_seq` strictly increasing; deadlines monotone and within sane bounds of `created_at`. **Retention:** until the round closes; pool relays only. **Filter:** `{"kinds":[38401],"authors":["<pool_id>"],"limit":1}` plus a live subscription during refresh windows.

### 3.3 Kind 31401 — Operator heartbeat + key-deletion attestation

**Purpose.** Per-operator liveness signal and the signed attestation that ephemeral tree-signing keys were deleted after each ceremony — the liveness basis for pre-emptive exit on quorum degradation. Honest scope: an attestation cannot make key or share retention provable — FROST signatures do not attribute which share subset signed, and proactive resharing leaves Q unchanged — so attestations carry **reputational and legal weight only** (open problem 6). "Provable fraud" in this protocol is reserved for transfer-log equivocation and nullifier collisions (§3.10), which are genuinely cryptographic. **Class:** addressable, `d` = `pool_id`; identity is `(31401, operator_pubkey, pool_id)`, so each operator has exactly one current heartbeat per pool. **Author:** individual operator npub (never Q — the point is per-operator accountability).

```json
{
  "id": "...",
  "pubkey": "a1f09b…77de",
  "created_at": 1781139600,
  "kind": 31401,
  "tags": [["d", "9f4e2a…c21a"], ["p", "9f4e2a…c21a"]],
  "content": "{\"epoch\":14,\"height\":957018,\"last_round_seq\":712,\"deleted_keys\":[{\"round_seq\":712,\"cells\":[0,15],\"pubkeys\":[\"02ab…\",\"03cd…\"]}],\"prev\":\"<sha256 of previous heartbeat event id>\",\"status\":\"ok\"}",
  "sig": "..."
}
```

**Content schema.** `epoch` (current reshare epoch); observed `height`; `last_round_seq`; `deleted_keys` — the ephemeral cosigning pubkeys this operator attests it destroyed, per round/cell; `prev` — hash chain over this operator's heartbeat history; `status` (`ok`/`degraded`). **Encryption:** none. **Validation:** author is in the 38400 operator set; `epoch` matches the current reshare epoch; `prev` chain is continuous — a broken chain is itself an alarm. Clients MUST treat a heartbeat older than one round interval (1 hour) as degraded, refuse boarding against a degraded quorum, and begin pre-emptive exit evaluation when fewer than k = 4 operators are live.

**Retention.** Rolling 90 days as accountability evidence (reputational/legal, per above). Honest caveat: addressable semantics mean generic relays only guarantee the **latest** version; the `prev` hash chain makes suppressed history detectable, and pool relays SHOULD archive superseded versions for the 90-day window. **Filter:** `{"kinds":[31401],"#d":["<pool_id>"]}` (yields all 7 operators' latest).

### 3.4 Kind 24400 — Refresh intent

**Purpose.** A wallet's request to enter the next round: proves vTXO ownership, posts the forfeitable intent bond, pays the PoW toll. **Class:** ephemeral — valueless after the intent window; relays must not store it. **Author/wrapping:** the 24400 is an unsigned rumor sealed and **gift-wrapped (NIP-59) to Q**; the outer 1059 is signed by a one-time key, p-tags Q, and carries the NIP-13 PoW (20 bits, capped — §1.7).

Inner rumor:

```json
{
  "id": "...",
  "pubkey": "<one-time session pubkey>",
  "created_at": 1781139650,
  "kind": 24400,
  "tags": [["p", "9f4e2a…c21a"]],
  "content": "{\"round_seq\":713,\"vtxo\":{\"outpoint\":\"c4a9…ee01:3\",\"value\":120000},\"ownership_proof\":\"<BIP-340 sig by leaf key U over SHA256('FlockIntent/v0.3'||pool_id||round_seq||outpoint)>\",\"session_pubkey\":\"02f3…\",\"lien\":\"<pre-signed lien tx spending the refreshing leaf's cooperative path: value − bond to owner, bond to Q; valid on any future unroll>\"}"
}
```

**Content schema.** Target `round_seq`; the vTXO being refreshed; an ownership proof signed by the current leaf key over a domain-separated challenge that binds pool, round, and outpoint (replay across rounds is impossible); the fresh `session_pubkey` for this ceremony; the bond instrument: a **ledger debit, netted at next service**, amount = `params.intent_bond_sats` (canonical default **1,000 sats**). On abort-after-nonce Q MAY slash: it records the debit in the transfer log and deducts the bond from the user's next successful refresh output or arkoor change, refusing service until netted; on honest completion or honest pre-nonce exit nothing is recorded. Backstop: the intent carries a pre-signed **lien tx** spending the refreshing leaf's cooperative path — (value − bond) back to the owner, bond to Q — valid on any future unroll of that leaf (collection cost ≈ unroll cost; the netting path is the economic one). No standalone bond vTXO exists, so no exemption from the 10,000-sat leaf floor is needed. Honest accounting: slashing is unilateral — Q needs no proof and no arbiter — so the wallet's exposure is the bond amount per intent, and that exposure belongs in the residual-trust tables. Boarding users (no vTXOs yet) post no bond — PoW only. Refinements [FUTURE]. **Encryption:** NIP-44 inside NIP-59, readable only by Q. **Validation (by Q):** PoW ≥ 20 bits on the wrap; ownership proof valid against the live nullifier set; per-key and per-vTXO quotas; leaf ≥ 10,000 sats; lien tx well-formed. **Retention:** none. **Filter (operators only):** `{"kinds":[1059],"#p":["<pool_id>"],"limit":...}` on pool relays.

### 3.5 Kind 24401 — Cell proposal

**Purpose.** Q's proposal to the ≤ 64 members of one cell: the unsigned transaction set **including the cell-tx template** — the per-cell funding transaction (inputs: Q capital + this cell's boarding outputs; outputs: one cell root + up to 64 per-user connectors + Q change) — each member's leaf and path, connector assignments, forfeit templates, and the canonical participant-set commitment. **Class:** ephemeral — superseded by the next attempt; never stored. **Author:** Q, one event per cell member, content NIP-44-encrypted Q→member (session pubkey from the intent).

```json
{
  "id": "...",
  "pubkey": "9f4e2a…c21a",
  "created_at": 1781139850,
  "kind": 24401,
  "tags": [["p", "<member session pubkey>"], ["s", "<session_id>"]],
  "content": "<NIP-44(Q→member): {\"round_seq\":713,\"cell_index\":4,\"attempt\":0,\"pset_commitment\":\"7b1d…\",\"txs\":{\"node_txs\":[\"<raw tx hex>\"],\"leaf_assignment\":{\"index\":17,\"value\":119500,\"scripts\":{...}},\"connector\":\"<vout in proposed cell tx>\",\"forfeit_template\":\"<raw tx hex>\"},\"cell_tx_template\":\"<raw tx hex>\"}>",
  "sig": "..."
}
```

**Validation (by each member, all MUST pass before sending nonces):** own leaf value and scripts correct; recovery tapleaf pays the member's own checkpointed seed-derived key with CLTV `T_exp + 288`; node and leaf sweep tapleaves use `T_exp + sweep_delay` per the pool's announced params (canonical default 4032); claim nSequence is 720 (ladder hop 0); the forfeit spends the old leaf plus **this member's own connector output of the proposed cell txid** (nSequence 0); recomputed `pset_commitment` matches; full root-to-leaf path (depth 3) verifies; all **pre-signed** txs (node, claim, forfeit) are nVersion=3, zero-fee, with P2A anchors, while the **cell-tx template is nVersion=2, fee-paying** — its fee-bumping is CPFP-only (spend Q change); RBF is forbidden, because any txid change voids the pre-signed graph. Any failure → silent abort (no nonces released). Cell isolation is real: a rebuild re-signs **this cell's tx only**, leaving every other cell's pre-signed graph untouched; members verify their **cell txid** (there is no round-wide txid) against 38402 after this cell's SET-FINAL. **Retention:** none. **Filter:** `{"kinds":[24401],"#p":["<session pubkey>"]}` live during the ceremony.

### 3.6 Kind 24402 — MuSig2 pubnonces

**Purpose.** Round-1 MuSig2 messages, both directions: members → Q (individual pubnonces) and Q → members (the aggregate nonce set, which fixes the transcript and finalizes the participant set — non-responders are excluded and the cell rebuilt in the same attempt, `sub_attempt`+1). Reserved for **true MuSig2 cohort sessions only** (tree pre-signing); arkoor cosigns involve no nonce coupling and use kind 24405 (§3.16), never this kind. **Class:** ephemeral; a stored nonce is at best garbage and at worst an attack surface. **Author:** participant session key, or Q for the aggregate. **Encryption:** NIP-44, session-bound (member→Q, or Q→member per recipient). Payloads use BIP-373 PSBT fields.

```json
{
  "id": "...",
  "pubkey": "02f3…",
  "created_at": 1781139960,
  "kind": 24402,
  "tags": [["p", "9f4e2a…c21a"], ["s", "<session_id>"]],
  "content": "<NIP-44(member→Q): {\"role\":\"participant\",\"pubnonces\":[\"<66B hex per BIP-373>\"]}>",
  "sig": "..."
}
```

**Validation:** `s` tag matches a live session; nonces are CSPRNG-fresh, RAM-only, never persisted, never re-derived — deterministic nonce derivation is **banned** ([ceremonies.md](ceremonies.md)); a second 24402 for the same session from the same key MUST abort the session. **Retention:** none.

### 3.7 Kind 24403 — MuSig2 partial signatures

**Purpose.** Round-2 partials plus the transcript signature (§1.4). MuSig2 cohort sessions only — never arkoor traffic (§3.16). **Class:** ephemeral. **Author:** participant session key; NIP-44 to Q.

Content (decrypted): `{"partials":["<32B hex>"], "transcript_sig":"<BIP-340 over transcript_hash>", "transcript_hash":"7e90…"}` — plus, in refresh ceremonies, the member's signature on their forfeit tx.

**Validation:** Q MUST verify each partial against the announced pubnonce set and the transcript signature against the canonical `transcript_hash`; participants MUST have verified the echoed transcript before sending. One partial per secnonce, ever; duplicate session ids, crashed sessions, or divergent transcripts abort with fresh nonces. **Retention:** none. Replay by relays is harmless: events dedupe by id and partials are bound to dead sessions.

### 3.8 Kind 24404 — Intra-quorum FROST / DKG / reshare traffic

**Purpose.** Operator↔operator transport for FROST signing rounds (the Q side of every cosign), PedPoP DKG at genesis, and monthly proactive resharing. **Class:** ephemeral. **Author:** individual operators; NIP-44 pairwise operator→operator.

Tags: `["p", "<recipient operator>"]`, `["s", "<session or epoch transcript id>"]`. Content (decrypted): `{"epoch":14, "phase":"reshare-share|sign-commit|sign-share|ack|complaint", "payload":"<RFC 9591 message hex>", "transcript":"<hash>"}`.

**Validation:** epoch tag MUST match the operator's current epoch; **cross-epoch material MUST be rejected** (split-view/replay defense — T12); every reshare requires a signed canonical transcript, explicit ACKs, and a complaint round; after resharing, operators MUST verify the group key Q is unchanged before deleting old shares. Relays provide no ordering — a stalled reshare can brick liveness, never safety. **Retention:** none. Operators SHOULD route 24404 over their own relays plus ≥ 2 independent relays, over Tor/hidden services.

### 3.9 Kind 38402 — Round result

**Purpose.** The authoritative record of a confirmed round: **all cell txids** plus the **inline per-cell tree commitments** from which any member can reconstruct their exit path. A round is a batch of up to 16 independently confirmed cell txs — there is no round-wide txid. **Class:** addressable, `d` = `<pool_id>:<round_seq>` — one permanent record per round (per-round d-tags mean later rounds never overwrite earlier ones). **Author:** Q.

```json
{
  "id": "...",
  "pubkey": "9f4e2a…c21a",
  "created_at": 1781143200,
  "kind": 38402,
  "tags": [["d", "9f4e2a…c21a:713"]],
  "content": "{\"round_seq\":713,\"cells\":[{\"index\":0,\"cell_txid\":\"88c0…41aa\",\"funding_height\":957019,\"t_exp\":961339,\"root_vout\":0,\"tree\":\"<full node-tx set, inline>\",\"pset_commitment\":\"7b1d…\",\"connectors\":\"<outpoint per refreshed user in this cell, inline>\"}],\"blossom\":[\"https://mirror.example/<sha256>\"]}",
  "sig": "..."
}
```

**Content schema.** Per cell: `cell_txid`, funding height, `t_exp = cell-tx height + 4320`, inline tree data, participant-set commitment, and this cell's connector map (every connector is an output of **its own cell tx**; a rebuilt cell may confirm at a different height than its siblings, so these fields are per-cell, never round-wide). Blossom mirrors are optional convenience and MUST NOT be the sole copy. **Validation:** signed by Q; the member's **cell tx** is confirmed on L1 **and its outputs byte-match the client's local recomputation** of what it cosigned (compact filters or a full node as the baseline L1 view; block explorers only as additional cross-checks, never the sole source for fail-closed decisions — an SPV-light view is an accepted residual risk, T9); `pset_commitment` equals the value signed during the ceremony. A refresh is final only after this check against the member's own cell txid — the connector then makes the forfeit valid and the new leaf live, atomically. **Retention:** life of round + 90 days, all pool relays, replication ≥ k = 4 verified by read-back. **Filter:** `{"kinds":[38402],"authors":["<pool_id>"],"#d":["<pool_id>:713"]}`.

### 3.10 Kind 30178 — Transfer log

**Purpose.** The double-spend prevention spine: quorum-signed Merkle roots over the live `(nullifier, commitment)` set, where `nullifier = SHA256("FlockNullifier/v0.3" ‖ outpoint)` — the outpoint **alone**, no seq, so any double cosign of one outpoint produces an **identical nullifier under two seqs**: a mechanically detectable collision. Roots are published **per arkoor cosign** (one root per cosign batch, `seq` strictly increasing) — not hourly. Terminology, used everywhere: a **checkpoint** is a round refresh (extinguishes residual trust, hourly opportunity); the **transfer log** is this per-cosign anti-equivocation spine. No cleartext amounts, no pubkeys — the public learns set size and update rate only. Two signed roots at one `seq` constitute a portable equivocation fraud proof; a nullifier collision is the other genuinely cryptographic one. Honest cost: collision detection requires scanning the log — towers and auditors do this continuously, receivers check their own outpoints — and full-log availability is itself part of open problem 2. **Class:** addressable with `d` = `<pool_id>:<seq>` — an append-only log realized as a family of addressable events; each entry persists for the life of the pool, while "the latest entry" is simply the highest `seq`. **Author:** Q.

```json
{
  "id": "...",
  "pubkey": "9f4e2a…c21a",
  "created_at": 1781143260,
  "kind": 30178,
  "tags": [["d", "9f4e2a…c21a:48211"]],
  "content": "{\"seq\":48211,\"root\":\"e3c9…0b77\",\"prev_root\":\"91aa…d2c3\",\"height\":957021,\"count\":14302}",
  "sig": "..."
}
```

**Content schema.** `seq` (strictly increasing, +1 per cosign batch); `root`; `prev_root` (hash chain); observed `height`; `count` (live nullifiers; the log is pruned to live entries). **Encryption:** none. **Validation (receiver-critical):** signed by Q; `seq` continuity via `prev_root`; fetched from **disjoint relay sets including operator-independent relays, failing closed on absence** (§1.5); replicated on ≥ k = 4 relays by read-back; Merkle inclusion proofs in arkoor packages verify against this root. Any two distinct roots for one `seq` — and any verified nullifier collision — MUST be persisted, broadcast as an 8405 alert, and treated as pool failure (exit). **Retention:** life of the pool; ≥ k-relay replication is a standing requirement on the pool relay set. **Filters:** latest — `{"kinds":[30178],"authors":["<pool_id>"],"limit":3}` then take max `seq` and verify continuity; specific entry — `{"kinds":[30178],"authors":["<pool_id>"],"#d":["<pool_id>:48211"]}`.

### 3.11 Kind 24406 — Arkoor payment package (inside kind-1059 gift wrap)

**Purpose.** The payment itself: everything the receiver needs to own, checkpoint, or unilaterally exit the new vTXO. **Class:** ephemeral inner rumor — it MUST only ever appear NIP-59-wrapped; the outer 1059 (regular range) is what relays store. **Author:** the rumor is unsigned; the seal is signed by a **one-time sender key** (sender unlinkability); authenticity comes from the Q-cosigned transaction chain inside, not from the Nostr signature.

**Addressing and discovery (honest trade-off).** The wrap does **not** p-tag the receiver's long-term npub (that would publish a payment graph). It carries the pool tag `["p", "<pool_id>"]`, the 20-bit PoW, and a pre-auth filter tag `["f", "<first 8 hex chars of SHA256(\"FlockFilter/v0.3\" ‖ P_B)>"]`. The receiver's claim key is the stealth key `P_B = B + H(ECDH(a,B) ‖ cell_txid ‖ counter)·G` (B from the receiver's 38407 descriptor, falling back to the receiver's Nostr pubkey — §1.10), which the receiver cannot precompute — so receivers fetch all pool-tagged 1059s from their read relays since last sync and **trial-decrypt** (one ECDH+HKDF+HMAC failure per non-matching wrap). **Scope of the `f` tag — decided here, v0.3:** the receiver cannot pre-derive sender-chosen counters, so the tag cannot narrow *first discovery*; it is specified and retained because it turns **re-fetches and tower handoffs** (where P_B is already known) into a single indexed filter instead of a re-scan. The trial-decrypt cost under first-discovery flood therefore remains a conceded residual (G8); the PoW requirement is the spam floor.

Inner rumor content (NIP-44 sealed to B):

```json
{
  "kind": 24406,
  "content": "{\"pool\":\"9f4e2a…c21a\",\"cell_txid\":\"88c0…41aa\",\"sender_ecdh_pub\":\"02ab…\",\"counter\":0,\"vtxo\":{\"value\":25000,\"hop\":2},\"tx_chain\":[\"<raw arkoor txs + sigs back to the confirmed cell tx>\"],\"claim_tx\":\"<Q-cosigned via 24405, nSequence 432>\",\"inclusion\":{\"seq\":48211,\"proof\":[\"…\"]},\"deposit_event\":\"<38404 event id>\"}"
}
```

**Acceptance rule (normative).** An uncheckpointed receipt is **not accepted** until the fail-closed verification below passes on the receiver's wake-up. A sender clawing value back before acceptance is a *failed payment*, not theft of accepted funds. Merchants requiring finality MUST require checkpoint inclusion or verify online — including step 7's deposit existence.

**Validation (receiver MUST, before displaying funds, fail closed throughout):**

1. Verify the full signature chain to a cell tx confirmed in its own L1 view (compact filters or a full node; explorers only as cross-checks).
2. Verify the Q-cosigned claim: `nSequence = 720 − 144·hop` (hop 2 → 432), `hop ≤ 4`. A transfer without its claim is invalid — reject.
3. Derive `P_B` from `sender_ecdh_pub` and `counter`; confirm the vTXO pays it.
4. Fetch the `seq`-referenced 30178 entry per §3.10 (disjoint relays, fail closed) and verify the Merkle inclusion proof.
5. Verify ≥ k = 4-relay replication of that 30178 entry by read-back.
6. **Reject** if the funding cell has < 2016 blocks to `T_exp` (receive ban — the sender must refresh first).
7. Verify the referenced 38404 height-deposit exists and is funded (mandatory part of send). Honest scope: this rule binds honest senders only — it is unenforceable against a stale-owner attacker, who is the very party filing it, and a never-online receiver cannot perform this check; that is why deposits also fire on **conflict detection** (§3.13) and why Q SHOULD lodge its own deposit for every cosign it performs.
8. Persist the package locally — uncheckpointed receipts are **not** seed-recoverable; relays, archival relays included (§1.9), are best-effort only (conceded, open problem 2).
9. The 50,000-sat VaR cap applies to the **aggregate uncheckpointed balance**, never per receipt: any accepted receipt pushing the aggregate over the cap queues auto-checkpoint at the **next available round seat** (VaR-triggered refreshes take priority in Q's seat assignment; structuring many sub-cap receipts does not evade the cap). The cap binds only an online wallet; offline exposure is bounded by the acceptance rule above and, where the pool sets `params.uncheckpointed_ceiling`, by Q refusing further cosigns past that per-recipient total (§3.16).

**Retention:** the 1059 carrier rides receiver-paid pinning subscriptions (~1–2k sats/month) plus the operator archival relays (§1.9; ciphertext, useless to operators). **Filter:** `{"kinds":[1059],"#p":["<pool_id>"],"since":<last_sync>}` on the receiver's read relays; re-fetches and tower handoffs narrow with `{"kinds":[1059],"#f":["<filter tag>"]}`.

### 3.12 Kind 37400 — Wallet vault

**Purpose.** Encrypted wallet state backup: vTXO set, pre-signed exit packages, arkoor history, ladder/hop state, derivation counters — what makes wallets stateless and seed-recoverable. **Class:** addressable — only the latest vault matters; stale vaults are safe (exiting an already-forfeited leaf only loses an already-spent coin; no toxic state). **Author:** a **seed-derived key unlinkable to the main npub** (deterministic from seed, so recovery can re-derive it; defeats the per-npub wealth estimator, P6).

```json
{
  "id": "...",
  "pubkey": "<derived vault key>",
  "created_at": 1781140000,
  "kind": 37400,
  "tags": [["d", "v0"]],
  "content": "<NIP-44 self-encrypted ciphertext, padded to a size bucket>",
  "sig": "..."
}
```

**Tag semantics.** `d` = vault slot (`v0`; clients MAY shard across slots/decoy events). No pool tag — the vault must not advertise which pools its owner uses. **Encryption:** NIP-44 self-encryption under a seed-derived symmetric path; ciphertext padded to fixed-size buckets; updates batched and timing-randomized (coarse wealth-bucket leakage is conceded). **Validation:** owner-only — decrypt, then cross-check contents against 30178 and L1; never trust the vault over the chain. **Retention:** paid pinning SLA + operator archival relays (§1.9) + mandatory local copy. **Filter (recovery):** `{"kinds":[37400],"authors":["<derived key>"]}` against bootstrap relays, then the archival relays (§1.9) as fallback.

### 3.13 Kind 38404 — Watchtower height-deposit

**Purpose.** "Broadcast package P at `trigger_height` **or immediately upon conflict**, unless superseded" — automatic, fee-budgeted protection for offline receivers; filing a funded deposit is a **mandatory step of every send**. Honest scope: that mandate binds honest senders only — the stale-owner attacker is the party filing it, and a never-online receiver cannot audit it. Therefore Q SHOULD additionally lodge a deposit for every arkoor cosign it performs (it holds the claim it cosigned), and every wallet lodges a **standing dormancy deposit at each checkpoint**: its unroll branch + pre-signed recovery claim + CPFP children, `trigger_height = T_exp + 288`, replicated to ≥ 3 towers. Towers are keyless: they hold ready-to-broadcast packages and CPFP budgets, never funds keys. **Class:** addressable, `d` = request id — a deposit is cancelled/superseded by replacing its `d`. **Author:** a one-time key (deposits must not link to sender or receiver npubs).

```json
{
  "id": "...",
  "pubkey": "<one-time key>",
  "created_at": 1781140100,
  "kind": 38404,
  "tags": [["d", "<request_id>"], ["p", "<tower pubkey>"], ["p", "9f4e2a…c21a"]],
  "content": "<NIP-44(one-time→tower): {\"trigger_height\":960613,\"watch_outpoints\":[\"<cell root>\",\"<node outpoints>\",\"<old leaf>\"],\"package\":[\"<raw tx hex: full ancestor arkoor chain + claim, each with its pre-signed escalating bump children (TRUC 1p1c)>\"],\"supersede\":{\"nullifier\":\"<SHA256('FlockNullifier/v0.3'||outpoint)>\"},\"fee_budget\":{\"bump_children\":[\"<per parent, pre-signed CPFP variants at 10 / 50 / 200 sat/vB spending the owner's dedicated fee UTXOs>\"],\"utxos\":[\"<owner's dedicated fee UTXOs>\"]},\"tower_fee_sats\":2000}>",
  "sig": "..."
}
```

**Content schema.** `trigger_height` — `≤ T_exp − 720` for defensive deposits (towers must exit those by then); `T_exp + 288` for standing dormancy deposits. `watch_outpoints` — the deposited package's **ancestor outpoints** (cell root, intermediate node outputs, the old leaf). **Trigger semantics (normative):** the tower MUST broadcast the package at `trigger_height`, **and** MUST watch L1 and broadcast it **immediately** upon detecting any spend touching a watch outpoint — an unroll or conflicting claim — not only at the trigger height. The package MUST contain the **full ancestor arkoor chain** plus the claim, each tx paired with its **pre-signed escalating bump children** — canonical ladder **10 / 50 / 200 sat/vB**, spending the owner's dedicated fee UTXOs. The tower selects the lowest variant that clears, so its out-of-pocket cost is ≈ 0; residual: feerates above the pre-signed ceiling stall the package (this folds into the replacement-cycling/fee-spike open problem). TRUC 1p1c means the chain confirms **serially**, and the two defense windows are distinct: against a stale prior owner attacking an uncheckpointed chain, the tower's deadline is the attacker's claim maturity — **288–720 blocks after the unroll** by attacker hop (the current owner's head-start differential is 144–576 blocks); the **forfeit stale window** for held forfeits against refresh-stale claims is **144–720 blocks** ([onchain.md](onchain.md) §3.4). Supersession condition: the nullifier (per §3.10) appearing spent in 30178, or a replacement deposit id. Fee budget: the owner's dedicated fee UTXOs, unconflictable. **Tower fee (canonical convention):** each deposit names one tower by per-request key; the pre-signed bump children *for that tower* each carry a fee output (canonical default **~2,000 sats**) to that key — the fee rides every escalation variant, so the tower is paid whichever variant clears. Different towers hold conflicting child variants — whichever tower completes the broadcast collects; at most one fee is ever paid, and **nothing is paid if no tower ever needs to act**. The deposit budget is a contingent allocation, not a per-send payment. **Encryption:** NIP-44 to a **per-request tower key**; packages are **sharded across multiple independent towers** so reconstruction requires collusion (colluding towers recombining shards is conceded). Tower discovery: the kind-38406 service offer book (§3.15). **Validation (tower):** package decodes and is policy-valid; budget spendable; trigger plausible vs the pool's T_exp ladder; watch set matches the package's ancestors. **Validation (sender/receiver):** read-back the event from ≥ 2 relays the tower watches before treating the send as complete (§1.5). **Retention:** until `trigger_height + 288`. **Filter (towers):** `{"kinds":[38404],"#p":["<tower pubkey>"]}`.

### 3.14 Kind 8405 — Tower alert

**Purpose.** Public watchtower signal: a detected conflict (double-spend attempt, stale exit — also what fires the §3.13 conflict trigger), a sweep anomaly (Q moving before `T_exp + 4032`), or confirmation that a deposited package was broadcast. **Class:** regular — alerts are accumulating evidence; replaceable semantics would let later events erase the record (this is precisely why the kind moved from 38405 to 8405; see [ERRATA](../research/ERRATA.md)). **Author:** the tower's npub.

```json
{
  "id": "...",
  "pubkey": "<tower pubkey>",
  "created_at": 1781150000,
  "kind": 8405,
  "tags": [["p", "9f4e2a…c21a"], ["e", "<related 38404 or 30178 event id>"]],
  "content": "{\"type\":\"conflict\",\"height\":957102,\"txids\":[\"aa31…\",\"bb42…\"],\"detail\":\"stale hop-1 claim broadcast; current owner claim matures first (nSeq 432 < 576)\"}",
  "sig": "..."
}
```

**Content schema.** `type` ∈ `conflict | sweep_anomaly | broadcast_done | equivocation | nullifier_collision | heartbeat_stale`; height; relevant txids/event ids; free-text detail. For `equivocation`, content embeds both conflicting 30178 events in full; for `nullifier_collision`, both inclusion proofs of the duplicate nullifier — the portable fraud proofs (§3.10). **Validation:** alerts are **advisory**: clients MUST verify every claim against their own L1 view and signed events before acting; an unverified alert moves nothing. **Retention:** 30 days, pool relays + tower's write relays. **Filter:** `{"kinds":[8405],"#p":["<pool_id>"],"since":<last_check>}`.

### 3.15 Kind 38406 — Service offer book (gateways and towers)

**Purpose.** The service discovery surface. Entries carry `type: "gateway" | "tower"` with per-type fields: NIP-69-style offers for vHTLC/PTLC swaps between Flock vTXOs and BOLT11/BOLT12 Lightning (both directions), and watchtower service offers for the §3.13 deposit protocol — this resolves the formerly-[TBD] tower discovery kind. **Class:** addressable, `d` = `<pool_id>` (one current book per provider per pool); stale quotes must not linger. **Author:** the gateway's or tower's npub.

```json
{
  "id": "...",
  "pubkey": "<gateway pubkey>",
  "created_at": 1781140200,
  "kind": 38406,
  "tags": [["d", "9f4e2a…c21a"], ["p", "9f4e2a…c21a"], ["expiration", "1781143800"]],
  "content": "{\"type\":\"gateway\",\"directions\":[{\"from\":\"flock\",\"to\":\"bolt11\",\"rate\":0.003,\"min_sats\":10000,\"max_sats\":2000000},{\"from\":\"bolt11\",\"to\":\"flock\",\"rate\":0.004,\"min_sats\":10000,\"max_sats\":1000000}],\"cltv_delta\":144,\"reservation_fee_sats\":50,\"inflight_cap\":5,\"ptlc\":true}",
  "sig": "..."
}
```

A tower entry differs only in content: `{"type":"tower","fee_expectation_sats":2000,"watch_capacity":50000,"per_request_key_endpoint":"<npub to which key requests are gift-wrapped>"}`.

**Content schema (gateway).** Per-direction rate (within the 0.1–0.5% band), min/max, the 144-block CLTV delta, the upfront reservation fee and per-id in-flight cap (free-option/cycling defense, G9), PTLC support flag. **Content schema (tower).** `fee_expectation_sats` — the contingent fee the tower expects riding each of the deposit's pre-signed escalating bump children (canonical default ~2,000 sats, §3.13); `watch_capacity` — open deposit slots; `per_request_key_endpoint` — the npub to which a depositor gift-wraps a key request and from which it receives a fresh per-request key (deposits never reuse tower keys — P4). NIP-40 `expiration` bounds quote staleness for both types. **Validation:** signed by the provider; quote unexpired; for gateways, reservation paid before relying on a quote (swap exposure is standard HTLC risk for the CLTV window); for towers, the offer is advisory — the deposit itself (§3.13) carries the obligations. **Retention:** until superseded. **Filter:** `{"kinds":[38406],"#d":["<pool_id>"]}`.

### 3.16 Kind 24405 — Arkoor cosign request/response

**Purpose.** The sender↔Q envelope that produces an arkoor transfer. This is **not** a MuSig2 session and involves no nonce coupling: the two-sig leaf needs the sender's complete single-key Schnorr signature and Q's independent FROST signature, nothing more. Kinds 24402/24403 are reserved for true MuSig2 cohort sessions and MUST NOT carry arkoor traffic. **Class:** ephemeral — valueless after the cosign; relays must not store it. **Author/wrapping:** both directions are unsigned rumors, NIP-59 gift-wrapped: the request to Q (outer 1059 signed by a one-time key, p-tags Q, carries the §1.7 PoW), the response back to the sender's `session_pubkey`.

Request rumor:

```json
{
  "kind": 24405,
  "tags": [["p", "9f4e2a…c21a"]],
  "content": "{\"req\":{\"outpoint\":\"c4a9…ee01:3\",\"tx_chain\":[\"<raw arkoor txs + sigs back to the confirmed cell tx>\"],\"sender_sigs\":[\"<complete BIP-340 sigs, one per sender-controlled input>\"],\"new_vtxo\":{\"key\":\"<P_B>\",\"value\":25000,\"hop\":2},\"fee\":{\"sats\":2,\"rate\":0.0005},\"session_pubkey\":\"02f3…\"}}"
}
```

Response rumor (Q→sender): `{"resp":{"frost_sigs":["<BIP-340 by Q>"],"claim_tx":"<Q-cosigned, nSequence 720 − 144·hop>","log":{"seq":48211,"root":"e3c9…0b77","proof":["…"]}}}` — Q's signatures, the cosigned claim, and the **fresh transfer-log root with the inclusion proof** for the new `(nullifier, commitment)` entry. One 30178 root is published per cosign batch (§3.10), before or with this response.

**Encryption:** NIP-44 inside NIP-59, both directions. **Validation (Q, all MUST pass before signing):** sender signatures verify over the full chain back to a confirmed cell tx; `nullifier = SHA256("FlockNullifier/v0.3" ‖ outpoint)` is absent from the live set — an identical nullifier already present means double cosign: refuse and alert (§3.10); `hop ≤ 4`; the funding cell has ≥ 2016 blocks to `T_exp` (receive ban); new leaf ≥ 10,000 sats; where `params.uncheckpointed_ceiling` is set, the recipient's uncheckpointed total stays under it. Q SHOULD lodge its own 38404 deposit for the cosigned claim (§3.13). **Validation (sender):** Q's signatures verify against Q; claim `nSequence = 720 − 144·hop`; the inclusion proof verifies against the returned root, which the sender read-back-verifies per §1.5 before assembling the §3.11 payment package. **Retention:** none. **Filter (operators only):** `{"kinds":[1059],"#p":["<pool_id>"]}` on pool relays.

### 3.17 Kind 38407 — Flock payment descriptor

**Purpose.** What a sender needs to pay an npub: the stealth base key **B**, preferred pools, and optional read relays — the Flock analogue of a Lightning address, complementing (never replacing) NIP-65. **Class:** addressable, `d` = `"v0"` — one current descriptor per npub. **Author:** the receiver's long-term npub.

```json
{
  "id": "...",
  "pubkey": "<receiver npub>",
  "created_at": 1781140300,
  "kind": 38407,
  "tags": [["d", "v0"]],
  "content": "{\"v\":1,\"stealth_base\":\"<32B x-only B, hex>\",\"pools\":[\"9f4e2a…c21a\"],\"relays\":[\"wss://inbox.example\"]}",
  "sig": "..."
}
```

**Content schema.** `stealth_base` — B, a seed-derived key that is **not** the npub (keeps the payment surface unlinkable to the identity key); `pools` — the pool_ids the receiver actually scans; `relays` — optional read relays. **Fallback (normative, §1.10):** a receiver with **no 38407 and no kind-10002** is still payable — the sender sets **B = the receiver's Nostr pubkey** and publishes the wrap to the pool relay set, which is mandatory in all cases. Privacy note: publishing a 38407 marks the npub as a Flock user (though not its balances or payments); zero-footprint receivers rely on the fallback and pay the full first-contact scan cost. **Encryption:** none — public discovery data. **Validation (sender):** signed by the receiver's npub; latest `created_at` wins; fetched from the receiver's relays plus the bootstrap set. **Retention:** until superseded; receiver's write relays + preferred pools' relay sets. **Filter:** `{"kinds":[38407],"authors":["<receiver npub>"]}`.

---

## 4. Interop: zaps (9735) and NWC

Flock's native proof-of-payment is a Merkle inclusion proof under a quorum-signed 30178 root — no receipt event is required for Flock-to-Flock payments. Interop with the existing Nostr payments stack works at the gateway:

**Zaps (NIP-57, kind 9735).** A Flock wallet (or its gateway acting as LNURL server) can receive zaps: the zapper pays a BOLT11 invoice issued by the gateway, the gateway swaps the value into an arkoor vTXO for the receiver (vHTLC, 144-block CLTV delta) and emits the kind-9735 zap receipt referencing the kind-9734 request. Trust in the 9735 rests on the gateway exactly as it rests on LNURL servers today — relays attest nothing. Existing zap clients work unchanged; the receipt is a regular public event with client-norm retention, outside Flock's validation perimeter. Flock wallets MUST NOT treat a 9735 as evidence of vTXO ownership — only the arkoor package (§3.11) is.

**NWC (NIP-47).** A Flock wallet can expose the NWC request/response RPC (NIP-44-encrypted over relays) so existing Nostr apps can spend from it: `pay_invoice` routes through a gateway swap (gateway fee applies); a `pay_flock`-class extension method for native arkoor sends is [TBD]. Conversely, a Flock gateway can hold an NWC connection to an LN node as its Lightning-side liquidity. NWC events use their own kinds (13194/23194/23195) and are out of scope here; they coexist on the same relays without interaction with Flock's validation rules.

The two systems compose because both already treat relays as untrusted transport: Lightning interop adds gateway trust for the duration of a swap, never new trust in relays.

---

## 5. Cross-references

- Scripts, trees, ladder, forfeits, exits: [onchain.md](onchain.md)
- Ceremony state machines, nonce discipline, DKG/resharing, bond slashing: [ceremonies.md](ceremonies.md)
- Attack matrix backing every "fail closed" above: [../analysis/attacks.md](../analysis/attacks.md)
- Fee and retention economics: [../analysis/economics.md](../analysis/economics.md)
- Canonical constants: [../research/ERRATA.md](../research/ERRATA.md)
- Adjudicated v0.3 amendments: [../research/AMENDMENTS-v0.3.md](../research/AMENDMENTS-v0.3.md)
