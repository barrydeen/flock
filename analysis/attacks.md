# Attack Matrix

Flock protocol - DRAFT v0.3 - 2026-06-12

This document disposes of all 42 adversarial findings raised by the red-team panel against the pre-synthesis winning design (design A, federated Ark). v0.2 changed the design in direct response to these findings; v0.3 amends the mechanisms again after a second adversarial review (adjudicated in [../research/AMENDMENTS-v0.3.md](../research/AMENDMENTS-v0.3.md)). Each entry states explicitly whether the current architecture:

- **(a) eliminates** the attack by construction — the attack steps no longer apply to any v0.3 mechanism;
- **(b) mitigates** it — the attack is still expressible but its impact is bounded, with the residual risk stated; or
- **(c) concedes** it as an open problem — no satisfactory defense exists; the concession is cross-referenced to the open-problems list (whitepaper §Open problems, OP1–OP10; index reproduced in [§7](#7-open-problem-index)).

Panel severity ratings are preserved unchanged except where v0.3 eliminates the attack by construction, marked "eliminated (was: *severity*)".

Companion documents: mechanism details in [../specs/onchain.md](../specs/onchain.md), [../specs/ceremonies.md](../specs/ceremonies.md), [../specs/event-kinds.md](../specs/event-kinds.md); cost figures in [economics.md](economics.md); design rationale in [../whitepaper.md](../whitepaper.md).

## 1. Threat model

- **Operators.** Up to 3 of 7 FROST operators are fully malicious: safety (no theft of checkpointed funds, no minting, no ladder bypass) must hold against any 3-of-7 collusion, and holds against full 7-of-7 collusion for checkpointed vTXOs **until T_exp+sweep_delay (default 4032) absent an unroll**: the claim ladder and recovery tapleaf are consensus-enforced rather than honesty-enforced, but the cell root carries a sweep leaf, so a never-unrolled cell is sweepable at its root — confiscation is prevented only if the owner or a tower acts within the 3,744-block recovery window (L2, → OP10). A 4-of-7 collusion gains exactly one theft capability against an online receiver: double-spending **uncheckpointed** arkoor receipts with the sender's cooperation, until the next checkpoint (the bounded window in [§6](#6-residual-trust-explicit-time-bounds)). Against a receiver who stays offline beyond the ladder differential, a stale prior owner (typically the sender) acting **alone — no operator collusion** — can unroll early and outwait the head start unless a tower acts (T5 residual 2, L3). Operators are additionally assumed to be passive surveillants of everything they cosign.
- **Relays.** Fully adversarial: drop, delay, reorder, selectively serve, spam-gate, purge, and read all metadata. No relay is trusted for storage, ordering, or delivery. Clients must fail closed on absence.
- **Griefers.** Well-funded third parties able to sustain DDoS against operators and relays, flood mempools to move L1 feerates, run Sybil seats and dust floods, and operate attractive watchtowers/relays to harvest metadata. They seek to impose cost, not necessarily to profit.
- **Network position.** Targeted ISP/state-level partitioning of an individual victim is in scope; it is mitigated with an ISP/state-level residual — no client-side defense fully defeats it, but fail-closed behavior bounds it to DoS plus forced exit (L5).

### Terminology drift

The panel attacked the pre-synthesis design. Attack descriptions below preserve the panel's steps; read old terms through this map:

| Pre-synthesis (attacked) | v0.3 (current) |
|---|---|
| 3-of-5 quorum | FROST 4-of-7 |
| Kind 38403 cleartext cosign log | Kind 30178 transfer log (quorum-signed Merkle roots over (nullifier, commitment) pairs, one root per arkoor cosign; no cleartext amounts) |
| Kind 38405 tower alert | Kind 8405 (regular) |
| One ≤1024-cosigner round, radix-4 depth-5 | 16 independent cells × 64 leaves, radix-4 depth-3, each funded by its own on-chain cell tx |
| 30 s phase timers, ≤3 re-proposals | 120 s adaptive timers, post-nonce participant finalization with same-attempt rebuild |
| Reactive 144-block CSV contest + live-Q forfeit | Decrementing claim ladder (nSequence 720 → 144) + nSequence-0 forfeits broadcastable by anyone |
| Hash-lock (preimage) refresh atomicity | Connector outputs (transaction-graph binding to the confirmed cell txid) |
| Delegated offline refresh | Deleted; sender-funded height-deposits (38404, height- **and conflict-**triggered) + recovery tapleaf + standing dormancy deposits at every checkpoint |
| `<T_exp> CLTV <Q>` sweep | Sweep at T_exp+sweep_delay (default 4032) only; recovery tapleaf to checkpointed owner at T_exp+288 |
| Boarding refund 4320 CSV (~30 d) | 288 CSV (~2 d) |
| Stealth key `P_B = B + H(ECDH(a,B))·G` | `P_B = B + H(ECDH(a,B) ‖ cell_txid ‖ counter)·G` |
| Exit ≈ 2.2 kvB first mover (4.5k–22k sats) | ≈ 1.6 kvB first exiter (~16k sats @10 sat/vB); ≈ 310 vB marginal (~3.1k sats); +235 vB per arkoor hop; mass exit ≈ 451 vB/user |

## 2. Theft and double-spend

### T1. Deterministic secnonce reuse extracts private keys / FROST shares

**Severity:** critical. **Disposition:** (b) mitigated — residual is an implementation-bug class, not a protocol property.

**Attack.** The old design derived secnonces deterministically from session metadata. A malicious cosigner or relay collects one partial signature, induces a crash or same-attempt retry, and obtains a second partial under the same secret nonce `k` but a different aggregate nonce/challenge. Two partials, one `k`, two challenges yield the signing key algebraically. Against an operator this recovers a FROST share; collect k shares and reconstruct Q — whole-pool theft.

**v0.3 disposition.** The deterministic scheme is banned at spec level. All MuSig2/FROST round-1 nonces are CSPRNG-fresh, RAM-only, never persisted, never re-derived; one partial per secnonce ever; any crash, timeout, or duplicate aborts the session and restarts with fresh nonces; idempotency lives in session-ids, never nonce reuse. All signers sign and cross-verify a canonical session transcript (full pubnonce set, signer set, message) before emitting partials. The panel's own residual — a wiped/restored device re-deriving a used nonce — is closed by construction: nonces are never derivable, so a restored device *cannot* recompute one.

**Residual risk.** Implementation bugs (an implementer reintroducing derivation or persisting a secnonce). Bounded only by spec-level prohibition, test vectors, and audit. See [../specs/ceremonies.md](../specs/ceremonies.md).

### T2. FROST-threshold-of-MuSig2 composition leaks the group key

**Severity:** eliminated (was: critical). **Disposition:** (a) eliminated by construction.

**Attack.** The old design had Q participate as one MuSig2 cosigner whose nonces and binding-factor partials were FROST-threshold-generated — an unshipped, unproven composition. A coordinator splitting the relay view feeds different pubnonce sets to different operators inside one session; operators compute different binding factors and emit partials over inconsistent challenges; combining them leaks group-secret algebra (Drijvers/ROS-class). Many concurrent lossy sessions grind candidates; one success compromises Q entirely.

**v0.3 disposition.** The composition is **removed**. Every quorum-gated output uses a tapscript requiring two independent BIP-340 signatures (`<X> OP_CHECKSIGVERIFY <Q> OP_CHECKSIG`): users/cohorts sign standard MuSig2 or single-key Schnorr; Q signs the same digest separately via plain FROST (RFC 9591 / FROSTR-class, shipped). No nonce coupling between the two protocols exists anywhere. Transcript canonicalization and bounded concurrent sessions per group key are retained as defense in depth.

**Residual risk.** Standard-cryptography risk only, plus a cost: ~+70 vB witness per quorum-gated input.

### T3. Arkoor double-spend with eclipsed checkpoint-log detection

**Severity:** critical. **Disposition:** (b) mitigated; the accountability component is conceded (→ OP6).

**Attack.** Sender U plus a colluding quorum majority cosign U's leaf to Bob (seq = n) and publish the checkpoint entry only to attacker-controlled relays serving Bob, while a conflicting spend back to U goes everywhere else. Eclipsed on his read relays, Bob sees "paid" and ships goods; his later refresh is rejected. The portable fraud proof carries no on-chain bond, so a smash-and-grab quorum is undeterred — the "bounded trust window" becomes uncapped custodial loss.

**v0.3 disposition.** Receiver finality is fail-closed: fetch 30178 transfer-log roots (published per arkoor cosign, seq strictly increasing) from **disjoint relay sets including operator-independent relays**, failing closed on any absence or timeout; verify Merkle inclusion under a quorum-signed root; require read-back-verified replication on ≥k independent mirrors before treating funds as spendable (relay-signed receipts are an optional SHOULD, not the mechanism). The 50,000-sat VaR cap applies to the **aggregate uncheckpointed balance**: any accepted receipt pushing the aggregate over it triggers auto-checkpoint at the next available round seat (VaR-triggered refreshes take seat priority), so structuring many sub-cap receipts does not evade it; below the cap, the wallet displays "uncheckpointed". Equivocation (two signed 30178 roots at one seq) is a portable fraud proof; and because the nullifier is derived from the outpoint alone (`SHA256("FlockNullifier/v0.3" ‖ outpoint)`), any double cosign of one outpoint collides as an identical nullifier under any seq — a mechanically detectable collision, though detection requires scanning the log (towers/auditors do; receivers check their own outpoints; full-log availability → OP2).

**Residual risk.** Fail-closed checking converts withholding into DoS (funds unspendable, not stolen — see L10). A global network-position eclipse can still blind one victim for sub-cap value. Bonded slashing of operators would need trusted arbitration — **conceded** as OP6 (accountability without covenants). Severity retained because the theft path survives for sub-cap uncheckpointed value under 4-of-7 collusion; that is exactly the residual-trust window in [§6](#6-residual-trust-explicit-time-bounds).

### T4. Forfeit/new-leaf hash-lock atomicity break enables refresh theft and mass forfeit

**Severity:** eliminated (was: critical). **Disposition:** (a) eliminated at the script level by construction.

**Attack.** Old refresh atomicity rested on a hash-lock: forfeit and new leaf gated on a preimage Q reveals "after the round confirms" — asserted, not script-bound. Colluding Q confirms a round tx omitting the victim's new leaf, reveals the preimage legitimately for included users, then broadcasts the victim's forfeit and sweeps the old leaf. A shared round preimage upgrades this to mass refresh theft.

**v0.3 disposition.** Hash-locks are replaced by **connector outputs**: each forfeit spends the old leaf *plus* a distinct per-user connector output of the new cell tx, so the forfeit is consensus-invalid unless the exact cell txid containing the user's verified new leaf confirms. There are no preimages, shared or otherwise, and no reveal step to withhold or reorder. A mutated cell tx invalidates the forfeit, leaving the old vTXO intact.

**Residual risk.** None at the script level. The user must verify L1 confirmation of the correct cell tx honestly — that residual lives in T9 (view-split), not here.

### T5. Dead quorum makes stale exits uncontestable, robbing arkoor recipients

**Severity:** eliminated for the named attack (was: critical). **Disposition:** (a) eliminated by construction for the dead-quorum case; two residuals: refresh-stale forfeit broadcast (→ OP9) and stale-owner early-unroll theft against offline receivers (mitigated — see residual 2 and [§6](#6-residual-trust-explicit-time-bounds)).

**Attack.** In the old design, punishing a stale exit required a *live* Q partial on the forfeit. U sells a vTXO via arkoor, then waits for (or induces) quorum death and unilaterally exits the pre-sale leaf. No forfeit can be produced; the stale exit matures after CSV; the buyer is robbed. Quorum liveness was silently a theft-prevention dependency.

**v0.3 disposition.** The reactive contest is replaced by the **decrementing claim ladder**: every owner holds a pre-signed claim tx with consensus-enforced BIP-68 nSequence — checkpointed owner 720; each arkoor hop −144 (576/432/288/144; hop cap 4, reset at every checkpoint). The current owner's claim matures *strictly before* any prior owner's once the leaf is on-chain: no mempool race, no live quorum needed. Q cosigns the claim atomically with each arkoor transfer; a transfer without its claim is invalid and the receiver rejects it. The seller's stale exit is consensus-blocked, dead quorum or not.

**Residual risk.** Two adjacent cases. (1) Refresh-stale leaves (already forfeited): forfeits have nSequence 0, pay Q, and are broadcastable by **anyone** (watchtower copies include the full ancestor arkoor chain plus per-tx CPFP children — TRUC 1p1c means serial confirmation), so a forfeit beats every laddered claim once broadcast — but *someone* must broadcast it within the **forfeit stale window** of 144–720 blocks, depending on the attacking hop (→ OP9). (2) **Stale-owner early-unroll theft — a first-class vector needing no operator collusion.** The claim ladder orders maturity; it does not preclude stale claims. A prior hop owner — typically the sender — can unilaterally unroll early and outwait an offline receiver's head start. The attacker here is a prior owner at hop ≤3, so the current owner's head-start differential is **144–576 blocks**, and the defense deadline — the attacker's claim maturity, the **ladder defense deadline** — is **288–720 blocks after the unroll** (the 144-block floor belongs only to the forfeit stale window of residual 1). Mitigations: height-deposits (38404) are **conflict-triggered as well as height-triggered** — towers MUST broadcast the deposited package immediately on detecting any unroll or conflicting spend touching its ancestor outpoints; Q SHOULD lodge a deposit for every arkoor cosign it performs (it holds the claim it cosigned). The mandatory sender deposit is filed by this very attacker and is unenforceable against a never-online receiver — conceded. Finality framing: an uncheckpointed receipt is not **accepted** until the receiver's fail-closed wake-up verification passes; clawback before acceptance is a failed payment, not theft of accepted funds. The residual that remains: an accepted-but-uncheckpointed receipt is revocable by the sender alone if the receiver stays offline beyond the ladder differential after a hostile unroll and no tower acts — bounded in [§6](#6-residual-trust-explicit-time-bounds).

### T6. Delegated offline refresh lets the quorum re-sign a sock-puppet branch

**Severity:** eliminated (was: critical). **Disposition:** (a) eliminated by construction — the feature is deleted.

**Attack.** Delegated refresh re-treed an offline user's leaf with no key contribution the user could delete. Q places the leaf in a subtree of Q-controlled sock-puppets, retains all ephemeral keys (deletion unenforceable), and re-signs the parent node to redirect the subtree to itself before the user returns. Full-balance theft under the standard collusion threshold, mis-framed as a "weaker covenant".

**v0.3 disposition.** Delegated refresh is **deleted** — the panel's own conclusion ("a user cannot delete a key they never generated") is accepted as terminal. Offline users are protected instead by sender-funded watchtower height-deposits (kind 38404, mandatory at send, height- and conflict-triggered), the standing dormancy deposit lodged at every checkpoint (trigger_height = T_exp+288, replicated to ≥3 towers), and the recovery tapleaf (T_exp+288 to the checkpointed owner's seed-derived key).

**Residual risk.** Not a theft risk but a cost — with a stated worst case: a user offline longer than ~3.5 weeks (past the T_exp−1008 refresh deadline) pays on-chain exit fees via their tower, or waits out recovery-leaf latency. Dormancy degrades to **tower-protected** self-custody; if the owner stays absent *and* no tower acts within the 3,744-block (~26-day) window after recovery maturity, the quorum sweep at T_exp+sweep_delay confiscates (L2, → OP10).

### T7. Forfeit race lost via TRUC/P2A pinning or replacement cycling

**Severity:** high. **Disposition:** (b) mitigated; replacement cycling itself conceded (→ OP3).

**Attack.** A seller who arkoor-sold a vTXO exits the pre-sale leaf; the old design gave Q 144 blocks to broadcast and CPFP the forfeit, both packages zero-fee. The attacker wins the fee race or replacement-cycles the forfeit's CPFP package until the contest window lapses, then the stale claim matures — seller double-spends the sold coin.

**v0.3 disposition.** The owner-vs-owner mempool race is gone: forfeits are nSequence 0 and valid as soon as both prevouts exist on-chain — for forfeits of arkoor-tip outputs the ancestor arkoor chain must first confirm serially (TRUC 1p1c), which is why tower forfeit copies include the full ancestor chain plus per-tx CPFP children — while every stale claim must wait out its ladder floor: the defender starts 144–720 blocks ahead (the forfeit stale window, by the attacking hop) instead of racing. Forfeits are broadcastable by anyone (towers hold copies). Pinning defense: all packages are nVersion=3 (TRUC) with P2A anchors; the exiter pre-signs its own minimum-feerate child at creation and submits 1p1c via `submitpackage`; CPFP is funded from dedicated UTXOs that cannot be conflicted; deadlines carry 1008-block margins.

**Residual risk.** Replacement cycling has no complete mempool-policy fix; a miner-colluding or well-capitalized attacker can still delay a forfeit's confirmation across the 144–720-block forfeit stale window during a fee spike. **Conceded** as OP3; defense is margins, dedicated fee UTXOs, and tower redundancy.

### T8. PTLC/vHTLC adaptor-secret leak via nonce reuse and swap cycling

**Severity:** high. **Disposition:** (b) mitigated.

**Attack.** Adaptor signatures are maximally nonce-sensitive: finalizing two adaptors under one nonce, or one adaptor against two contexts (exactly what deterministic nonces plus relay-driven retries enabled), leaks the adaptor secret. The attacker pulls an incoming payment without revealing the preimage downstream, or claims a swap without paying. Separately, replacement cycling around the swap's 144-block CLTV boundary steals the timeout claim.

**v0.3 disposition.** The T1 nonce discipline applies in full to adaptor/PTLC signing: CSPRNG-fresh nonces, one finalize per nonce, journal-recorded; the adaptor point and recomputed challenge are verified before finalizing. Swap legs use conservative CLTV deltas, watchtower-backed rebroadcast, and PTLC-preference at gateways.

**Residual risk.** Adaptor-secret leakage is a silent, catastrophic failure class — only conservative engineering defends it. HTLC-path cycling remains unsolved at the mempool level (→ OP3); bridge liquidity is exposed for the 144-block CLTV delta ([§6](#6-residual-trust-explicit-time-bounds)).

### T9. Round-assembly view-split commits a victim's input to an inescapable key

**Severity:** high. **Disposition:** (b) mitigated.

**Attack.** Q shows victim U a participant set/tree in which U's leaf appears, collects U's partial, but assembles the actual on-chain root from different keys/coefficients, capturing U's input in an output U cannot unilaterally exit. Combined with the old unbound hash-lock forfeit, U released its input before confirming the real output.

**v0.3 disposition.** Three independent locks. (1) The cell proposal (24401) carries a signed **canonical participant-set commitment** plus the cell-tx template; every signer verifies its own leaf and full root-to-leaf path against it, and the participant set is finalized post-nonce — any divergence aborts. (2) The forfeit binds to the exact cell txid via its connector input: a cell tx that differs from what U verified makes U's forfeit consensus-invalid, so U's old vTXO is never spent for a mutated cell tx. (3) Before treating a refresh as final, the wallet verifies the confirmed cell-tx output script (after that cell's SET-FINAL) against a local recomputation from the signed participant set.

**Residual risk.** Step (3) needs a reliable L1 view; an eclipsed or SPV-light wallet can be shown false confirmations. Mitigated with compact block filters or a full node as the baseline L1 view — block explorers only ever as additional cross-checks, never the sole source for fail-closed decisions; a full network-position adversary remains (L5).

### T10. Recovery tapleaf names the stale pre-arkoor owner between checkpoints

**Severity:** high. **Disposition:** (b) mitigated, with the residual explicitly **conceded** (→ OP8).

**Attack.** The recovery tapleaf pays the *checkpointed* owner and rotates only at refresh — not per arkoor hop. After Alice pays Bob, the on-chain leaf's recovery path still names Alice until Bob's next checkpoint. Alice waits T_exp+288 and sweeps via the stale recovery path, stealing Bob's uncheckpointed funds.

**v0.3 disposition.** The window is bounded on every axis rather than closed: receives are **refused** against cells with <2016 blocks to T_exp (the sender must refresh first), so a fresh receipt always has ≥2016 blocks of margin before the recovery leaf can even be relevant; an aggregate uncheckpointed balance above the 50k-sat VaR cap is auto-checkpointed at the next available round seat (VaR-triggered refreshes take seat priority), rotating the leaf to Bob (online-only — an offline receiver's exposure is bounded by the acceptance rule and Q's optional `uncheckpointed_ceiling`); sender-funded height-deposit towers exit by T_exp−720, well before T_exp+288. Per-transfer leaf rotation was evaluated and rejected: it requires an on-chain or quorum action per hop, destroying offline receive.

**Residual risk.** **Conceded** (OP8): sub-cap uncheckpointed value whose holder stays offline past T_exp+288 is claimable by the prior checkpointed owner. Listed verbatim in the residual-trust bounds ([§6](#6-residual-trust-explicit-time-bounds)).

### T11. Refresh hash-lock atomicity freeze (preimage withholding)

**Severity:** eliminated (was: high). **Disposition:** (a) eliminated by construction.

**Attack.** Old refresh spent the old vTXO into the round, then gated the new leaf on a preimage revealed as a censorable Nostr event. A colluding quorum lets the round confirm, withholds the reveal, and the user has neither old nor new coin — an indefinite freeze.

**v0.3 disposition.** There is no reveal step to withhold. Connector atomicity makes confirmation of the cell tx *simultaneously* activate the new leaf and validate the forfeit; the old leaf is only spendable via the forfeit, which is only valid if the cell tx containing the verified new leaf confirmed. If the cell tx never confirms (or is mutated), the forfeit is invalid and the old vTXO remains intact and spendable.

**Residual risk.** None of this class. Censoring the round itself is refresh censorship, handled at L2.

### T12. FROST resharing/DKG split view over untrusted relays

**Severity:** medium. **Disposition:** (b) mitigated.

**Attack.** Monthly proactive resharing and the genesis PedPoP DKG run as kind-24404 events over adversarial relays. Selective drops/reordering drive operators to inconsistent share sets (bricking the group); replayed old-epoch reshare messages induce cross-epoch combination that aids a mobile adversary's reconstruction across months.

**v0.3 disposition.** All DKG/reshare traffic uses epoch-tagged, signed, ACKed transcripts with explicit complaint rounds; every participant signs and verifies one canonical reshare transcript before committing; cross-epoch material is rejected outright; new shares are verified against the unchanged group public key Q before old shares are deleted.

**Residual risk.** Relays provide no ordering or consensus, so a sufficiently disruptive adversary can make resharing fail — bricking **liveness** (exit-only mode, → OP9) but never safety: a failed reshare leaves the old, verified share set in place. Proactive resharing bounds, not eliminates, mobile-adversary accumulation.

### T13. Decrementing-locktime ladder creates an exhaustible exit window

**Severity:** eliminated (was: low). **Disposition:** (a) eliminated by construction.

**Attack.** Against design B's *absolute*-locktime ladder: forced refresh churn or maximal arkoor chaining burns finite locktime headroom until legitimate exits crowd into shrinking validity windows or become unbroadcastable.

**v0.3 disposition.** The v0.3 ladder uses **relative** timelocks (BIP-68 nSequence floors), which open and never close — no claim ever becomes unbroadcastable. The decrement budget is consumed only by arkoor hops, is capped at 4, and resets to hop 0 at every checkpoint; refreshes reset rather than consume it, so churn cannot burn headroom.

**Residual risk.** The 4-hop cap is a functional constraint (long arkoor chains must checkpoint), not a security window.

## 3. Liveness and censorship

### L1. Data-availability stranding of arkoor receipts defeats seed-only recovery

**Severity:** critical. **Disposition:** (c) **conceded** as an open problem (→ OP2).

**Attack.** Arkoor-received value never touches L1; its exit data (tx chain + sigs in the 1059→24406 gift wrap) exists only as relay copies plus Q's archive — not derivable from a seed. Flood the victim's read relays to pressure retention/LRU eviction, then DDoS or legally pressure the operator archives. A long-offline receiver returns to find no gift wrap anywhere and cannot reconstruct the spend path. The recovery tapleaf cannot help: the current off-chain owner's claim is not seed-derivable.

**v0.3 disposition.** Bounded, not solved: replication-verified finality (funds not shown as received until confirmed on ≥k independent mirrors, verified by read-back over disjoint connections); mandatory local persistence of every received package; the quorum's ciphertext archive, concretely operator-run **paid archival relays speaking plain NIP-01** with long retention, listed in 38400 `params.archive_relays` (ciphertext useless to operators, useful to the owner); auto-checkpoint of the aggregate uncheckpointed balance above the 50k-sat VaR cap so larger sums promptly become L1-recoverable.

**Residual risk.** **Conceded** (OP2): coordinated relay purge plus archival-relay denial strands long-offline *uncheckpointed* receipts. Seed-only recovery of off-chain receipts is structurally impossible; the synthesis says so explicitly and v0.3 does not pretend otherwise.

### L2. Refresh censorship plus watchtower suppression yields a confiscating T_exp sweep

**Severity:** high. **Disposition:** (b) mitigated — confiscation is downgraded to fee taxation, conditional on the owner or a tower acting within the 3,744-block recovery window (→ OP10).

**Attack.** Q stalls a target's refresh (censorship is an admitted operator capability), withholds the target's watch requests and tower alerts via operator-run relays, AUTH-gates public relays, and at T_exp executes the "legitimate" sweep, confiscating the offline user's funds.

**v0.3 disposition.** The sweep no longer confiscates a defended user. The recovery tapleaf (`T_exp+288 CLTV` to the checkpointed owner's seed-derived key) matures **3,744 blocks (~26 days) before** Q's sweep leaf (`T_exp+sweep_delay`, default 4032). But the cell root carries a sweep leaf, so Q can sweep a never-unrolled cell **at its root** without any recovery tapleaf ever materializing on-chain — the sweep-vs-recovery race is real, and the recovery leaf alone does not win it. The standing defense: at every checkpoint the wallet pre-signs its recovery claim and lodges {unroll branch + recovery claim + CPFP children} as a self-funded 38404 height-deposit (trigger_height = T_exp+288, replicated to ≥3 towers), so tower action is automatic without alert traffic to censor; ≥3 mandated operator-independent relays plus read-back-verified multi-relay publication must confirm watch-request delivery before each refresh deadline.

**Residual risk.** Confiscation is prevented **only if the owner or a tower acts within the 3,744-block (~26-day) window** between recovery maturity (T_exp+288) and the quorum sweep (T_exp+sweep_delay). With standing tower deposits in place, a resourced adversary can **tax** the victim (on-chain unroll/claim fees, delay) but not confiscate; if the owner stays absent *and* every tower fails to act for the full window, the sweep confiscates. The tower market therefore carries the dormancy guarantee itself — tower-market thinness residual → OP10 (extended).

### L3. Never-online arkoor receiver confiscated at T_exp

**Severity:** high. **Disposition:** (b) mitigated.

**Attack.** A receiver who never comes online files no watch request. A motivated quorum mines its archive for dormant one-time keys, waits out their clocks, and sweeps at T_exp — targeted confiscation of exactly the offline users the protocol promises to protect.

**v0.3 disposition.** Both panel-proposed grafts shipped, together: a **funded 38404 height-deposit filed by the sender is a mandatory part of every arkoor send** ("broadcast package P at height H **or immediately on any conflicting spend of its ancestor outpoints**, unless superseded", fee-budgeted via a contingent ~2,000-sat tower-fee output riding each of the pre-signed escalating bump children — 10/50/200 sat/vB, spending the owner's dedicated fee UTXOs, so an acting tower is never out of pocket), so protection exists from the moment of payment with zero receiver action; Q SHOULD additionally lodge a deposit for every arkoor cosign it performs; the recovery tapleaf catches checkpointed value whose owner returns within the 3,744-block window even if every tower fails; Q's sweep is delayed to T_exp+sweep_delay (default 4032).

**Residual risk.** Tower-market thinness — a real fee-budgeted tower economy is unproven (**conceded**, → OP10). The mandatory sender deposit is filed by the would-be attacker and is unenforceable against a never-online receiver — conceded; conflict triggering and Q's per-cosign deposits are the compensating controls (T5 residual 2). Truly abandoned sub-dust residue is eventually swept at T_exp+sweep_delay. Withheld sender deposits are covered by L9.

### L4. Quorum-offline expiry coercion (timelock-boundary liveness DoS)

**Severity:** high. **Disposition:** (b) mitigated; forced pool-wide exit conceded as a liveness cost (→ OP9).

**Attack.** T_exp ladders are publicly computable. Knock enough operators offline (DDoS, BGP leaks, legal takedowns on relay hosts) across the refresh margin; with <k signers nothing signs, and every holder must exit on-chain simultaneously at peak fee pressure.

**v0.3 disposition.** 4-of-7 (liveness survives any 3 down) across diverse ASNs/jurisdictions behind hidden services; per-operator 31401 heartbeats trigger client-side pre-emptive exits on detected degradation rather than at the cliff; the recovery tapleaf plus standing tower deposits are the safety net for anyone who misses the window (L2); mass-exit load is ≈451 vB/user (each TRUC parent needs its own ≈115 vB CPFP child — no batch bumping), ≈0.46 blocks per 1,024-leaf pool round (~46 blocks per 100k users), spread across hourly-laddered T_exp rungs with 1008-block margins.

**Residual risk.** A funded adversary sustaining a multi-operator outage across the margin still forces a pool-wide exit — fees and disruption, never theft. <k operators halts new payments and forfeit *creation* (exit-only mode) — **conceded** under OP9.

### L5. NIP-65 relay-set eclipse hides conflicts and refresh windows

**Severity:** high. **Disposition:** (b) mitigated, with an ISP/state-level residual.

**Attack.** Read the victim's kind-10002, DDoS or repoint those relays, satisfy the old "3 relays say no-conflict" check entirely from attacker relays, and withhold round announcements so the victim misses the deadline ladder — stranding them past T_exp−1008 while exposed in the receive window.

**v0.3 disposition.** Clients use a hard-coded diverse bootstrap relay set independent of operator curation; finality requires attestations from operator-independent relays under distinct operators/ASNs, failing closed; cell txids are cross-checked against L1 ground truth via compact filters or a full node (explorers only as additional cross-checks); staleness alarms fire on round-announcement gaps, with timeout-to-exit.

**Residual risk.** An ISP/state-level adversary can still partition one target's entire reachable relay set, and relay "independence" is unverifiable from the client's vantage. The fail-closed posture converts this from theft exposure into DoS plus a forced (possibly pre-emptive) exit.

### L6. Sweep-at-expiry confiscation of offline or censored users

**Severity:** high. **Disposition:** (b) mitigated — at-expiry confiscation removed for checkpointed value, defended by the recovery window plus standing tower deposits (→ OP10); uncheckpointed residual → OP8.

**Attack.** The old `<T_exp> CLTV <Q>` path let a live quorum sweep everything immediately at expiry; combined with watch-request censorship, a passive holder's funds moved to the operator "by protocol".

**v0.3 disposition.** There is no Q path at T_exp. The leaf script order is: cooperative path; recovery tapleaf at **T_exp+288** paying the checkpointed owner's seed-derived key; Q sweep only at **T_exp+sweep_delay** (default 4032, late garbage collection). For checkpointed value the at-expiry confiscation is gone by consensus — the owner's recovery claim matures 3,744 blocks before Q can act. The race that remains: the recovery tapleaf must be materialized by an unroll, and Q can sweep an untouched cell at its root, so the window must actually be used — by the owner, or by the standing dormancy deposits lodged at every checkpoint (L2). Mandatory sender-funded height-deposits act first in the normal case.

**Residual risk.** Uncheckpointed value still rides the recovery leaf of the *prior* checkpointed owner (T10, **conceded** as OP8, bounded by the VaR cap and the 2016-block receive ban). Owner-and-all-towers absence for the full 3,744-block window forfeits to the sweep (L2, → OP10). Unroll fees tax late claimants.

### L7. Replacement-cycling / TRUC child-slot grief on exits

**Severity:** high. **Disposition:** (b) mitigated; cycling itself conceded (→ OP3).

**Attack.** Exit txs are zero-fee TRUC with anyone-can-spend P2A anchors and a single child slot. Occupy the slot with a low-fee child spending a conflictable funded input; when the honest party RBFs, conflict the funded input elsewhere, evicting the replacement and dropping the parent to childless/zero-fee. Repeat each block near a deadline; the parent never confirms.

**v0.3 disposition.** CPFP children are funded **only from dedicated anchor-spend UTXOs that cannot be conflicted** — removing the eviction lever the cycle depends on. The exiter's own pre-signed minimum-feerate child travels with the parent as a 1p1c `submitpackage` unit; multiple independent watchtowers race bumps; deadlines carry 1008-block margins; claims themselves are ladder-ordered, so no owner-vs-owner race rides on confirmation timing.

**Residual risk.** Replacement cycling is unsolved at the consensus level (**conceded**, OP3): sustained cycling plus fee spikes can still delay confirmation toward deadlines at real cost to the attacker. Tower funding adds its own cost and attack surface.

### L8. Targeted relay eviction forces expensive exits

**Severity:** high. **Disposition:** (b) mitigated.

**Attack.** DoS precisely the victim's public read relays during round windows so their intents and signing messages miss the (old 30 s) phase timers; the coordinator evicts the "unresponsive" victim every round; they can never checkpoint and are forced into an on-chain exit or expiry risk.

**v0.3 disposition.** Phase timers are 120 s and adaptive; eviction now costs the victim one 64-leaf cell attempt, not a 1024-user round; intents and ceremony traffic may be submitted across many independent relays not tied to the public 10002; clients detect targeted relay outages and reschedule into a later round; post-nonce rebuild means an evicted user is excluded cleanly rather than aborting others.

**Residual risk.** A determined adversary can DoS even a large relay set for one high-value target; the victim's fallback is a pre-emptive unilateral exit (~310 vB if the path is unrolled, ≈1.6 kvB first-exiter) — cost imposition, not fund loss. Longer timers widen the griefing window for G1/G2 (→ OP5 tension).

### L9. Operator/paid-relay censorship of watch requests and alerts

**Severity:** high. **Disposition:** (b) mitigated.

**Attack.** Operators run some pool relays and know each victim's relay set. Drop the offline victim's 38404 watch requests and 8405 alerts on relays they control, pressure the rest; no tower acts before the deadline; sweep at expiry.

**v0.3 disposition.** Height-deposits are *fire-and-forget*: a tower holding one broadcasts at height H — or immediately on detecting a conflicting spend of the package's ancestor outpoints — with no further alert traffic to censor. Requests are encrypted to per-request tower keys and **sharded across multiple independent towers**; ≥3 operator-independent relays are mandated per pool and the sender verifies publication by read-back over disjoint connections at send time; the recovery tapleaf backstops total tower failure for checkpointed value whose owner acts within the 3,744-block window (L2); 8405 alerts are public and any observer can escalate.

**Residual risk.** At least one honest, funded tower must have received the deposit. Operator tower monoculture plus relay collusion narrows that set — tower-market thinness **conceded** as OP10.

### L10. Coordination-event censorship strands receive-finality and refresh discovery

**Severity:** medium. **Disposition:** (b) mitigated.

**Attack.** Selectively withhold checkpoint-log entries and round announcements from one target (no funds move): the receiver is pinned in permanent "uncheckpointed" limbo; the holder never sees refresh windows and misses the deadline ladder. Operator-curated relay sets make stacking feasible.

**v0.3 disposition.** The 30178 transfer log has a strictly increasing quorum-signed sequence (one root per arkoor cosign) with inclusion proofs, so a withheld latest entry is *detectable* as a gap, not silent; clients ignore operator relay curation in favor of the pinned diverse bootstrap set and require operator-independent attestations; direct L1 verification covers round existence; stalled attestations trigger timeout-to-exit rather than indefinite limbo.

**Residual risk.** Low-rate targeted withholding is never provable as an attack (grief/loss indistinguishability, → OP5); the victim's recourse is a unilateral exit they should not have needed. DoS, not theft.

### L11. Anti-spam gate (PoW + paid AUTH) as a censorship chokepoint

**Severity:** medium. **Disposition:** (b) mitigated.

**Attack.** (a) Flood pool relays so they raise minimum PoW under load; mobile clients cannot mine an intent inside the phase timer and silently drop out of refresh near deadlines. (b) Operator-run paid relays deniably deny AUTH or stall writes for a targeted npub, blocking their intents and watch events.

**v0.3 disposition.** Intent PoW is **≥20 bits with relay/Q demand capped at 20** by protocol — relays cannot inflate it against mobile clients; wallets pre-mine PoW ahead of windows; intents are accepted across many independent relays so one AUTH denial is not fatal; wallets verify their writes by read-back over disjoint connections, making denial detectable (relay-signed receipts are an optional SHOULD, defense-in-depth, format [FUTURE] — only where deployed do they additionally make denial attributable); a non-paid public-relay fallback path is mandated; phase timers are 120 s.

**Residual risk.** A relay can always slow or deny a specific identity deniably; per-npub slowdowns remain possible and unprovable. Bounded to delay — the refresh deadline (T_exp−1008) and tower exits cap the consequence.

### L12. Boarding-deposit strand via post-broadcast quorum stall

**Severity:** medium. **Disposition:** (b) mitigated.

**Attack.** Boarding outputs are recognizable on-chain. Time a quorum DoS to land after a victim's boarding tx confirms but before conversion; the deposit sits in a 2-of-2 with an uncooperative quorum, escapable only via the refund CSV — 30 days in the old design — denying onboarding at will.

**v0.3 disposition.** The boarding refund CSV is shortened to **288 blocks (~2 days)**; boarding outputs are converted in the same round they confirm when possible; clients refuse to board while 31401 heartbeats show a degraded quorum.

**Residual risk.** Any deposit caught between confirmation and conversion is strandable for ≤2 days per attempt. Onboarding remains a soft DoS target around quorum outages; capital is delayed, never lost.

### L13. Blossom-hosted tree-commitment data-availability loss

**Severity:** eliminated (was: medium). **Disposition:** (a) eliminated by construction.

**Attack.** Round results stored the tree commitment off-relay as a Blossom blob on one or two hosts; take those down (or wait out retention) and a returning user cannot reconstruct their root-to-leaf exit path despite holding keys and vault.

**v0.3 disposition.** Inline tree commitments in the 38402 round result are **mandatory**; a Blossom mirror is optional and **never the sole copy**. Clients additionally persist their full path locally at signing time, and rounds are not treated as final without ≥k-mirror replication.

**Residual risk.** None of this class — external mirrors are convenience only.

### L14. vHTLC swap stalls and replacement cycling strand bridge funds

**Severity:** medium. **Disposition:** (b) mitigated (→ OP3 for the cycling component).

**Attack.** Replacement-cycle the HTLC-spending tx around the timeout boundary so neither claim nor refund confirms when expected, locking user and gateway funds for the full CLTV; stalling one quorum mid-cross-pool-swap locks in-flight value until timeout.

**v0.3 disposition.** Aggressive rebroadcast and fee-bumping of HTLC claims; short, randomized CLTV deltas; PTLC/adaptor constructions preferred (no preimage to replay in mempool games); gateway liquidity buffers and fail-safe-to-refund timeouts; both legs monitored across independent relays.

**Residual risk.** Cycling is mitigated, not eliminated; swap and cross-pool funds remain lockable for the 144-block CLTV window by a determined mempool adversary ([§6](#6-residual-trust-explicit-time-bounds), OP3).

## 4. Economic griefing and privacy

### Griefing

### G1. Single-aborter round-collapse griefing (free, hourly)

**Severity:** critical. **Disposition:** (b) mitigated; the deterrence gap is conceded (→ OP5).

**Attack.** The old root spend needed MuSig2 of *all* ≤1024 participants. Submit cheap valid intents, complete the pubnonce phase, then withhold partials. One withholder blocks the whole round; aborting forfeits nothing; ~4 Sybil intents exhaust the re-proposal limit and kill every hourly round, freezing all refreshes pool-wide for free.

**v0.3 disposition.** Rounds are restructured into **16 independent 64-leaf cells, each funded by its own cell tx** — a rebuild re-signs one cell only, so one aborter delays one cell by one attempt, never the round. The participant set is **finalized after nonce submission** — non-responders are excluded and the cell rebuilt within the same attempt, so withholding does not even cost the cell a full cycle. Intents post a **forfeitable bond** (`params.intent_bond_sats`, canonical default 1,000 sats), instrumented as a **ledger debit netted at next service**: on post-nonce abort Q records the slash in the transfer log and deducts the bond from the user's next successful refresh output or arkoor change, refusing service until netted; as backstop, the refresh intent (24400) carries a pre-signed **lien tx** spending the refreshing leaf's cooperative path — (value − bond) back to the owner, bond to Q — valid on any future unroll of that leaf (collection there costs ≈ an unroll; netting is the economic path). Q waives the bond on honest completion or honest pre-nonce exit. No standalone bond vTXO exists, so no 10k-floor exemption is needed. Boarding users (no vTXOs yet) are exempt — PoW-only. Plus PoW ≥ 20 bits (demand capped at 20) and per-key rate limits; phase timers are 120 s adaptive.

**Residual risk.** Malice is indistinguishable from packet loss, so honest unlucky users occasionally lose bonds, and a bond sized to spare them under-deters attackers (**conceded**, OP5). Q's power to slash is unilateral; the exposure is bounded at ≤ the bond per intent and is counted in [§6](#6-residual-trust-explicit-time-bounds). Griefing now has a positive price and a 1/16-round blast radius, but the defense is imperfect.

### G2. Refresh-ceremony griefing (Sybil dust seats + branch stall)

**Severity:** high. **Disposition:** (b) mitigated (→ OP5).

**Attack.** Accrue hundreds of free dust vTXOs as penalty-immune "existing holder" seats across many branches; join every round; withhold round-2 partials just inside the timer; one staller per radix-4 branch stalls everyone above it; repeat hourly until legitimate refreshers cascade into forced exits.

**v0.3 disposition.** The seat is no longer free or reusable: minimum leaf denomination 10,000 sats; bonded intents (instrument: G1's ledger-debit netting with lien-tx backstop) slashed on non-completion (not only proven equivocation); per-key intent quotas; cells segregate blast radius; post-nonce rebuild excludes stallers in-attempt; 120 s adaptive timers absorb honest jitter.

**Residual risk.** Same OP5 concession as G1: cost-to-grief is now nonzero but remains below cost-to-defend for a well-funded adversary, and honest packet-loss victims share the penalty surface.

### G3. Exit-storm fee spike strands sub-economic vTXOs

**Severity:** high. **Disposition:** (b) mitigated; dust strandability conceded (→ OP4).

**Attack.** Trigger a correlated exit (quorum DoS) while fee-sniping L1 to 50–100 sat/vB. Exits cost real vbytes; vTXOs worth less than their exit cost are rationally abandoned and swept at expiry.

**v0.3 disposition.** Exit weight is reduced (depth-3 trees: ≈1.6 kvB first exiter, ≈310 vB marginal incl. own CPFP child, vs ~2.2 kvB attacked; mass exit ≈451 vB/user); 10k-sat minimum denomination keeps leaves exit-economical at moderate feerates; wallets consolidate dust before deadlines (VaR policy) and exit pre-emptively on fee-oracle signals; exits batch across shared branches (each TRUC parent still needs its own CPFP child); operators run subsidized emergency batching at mass-exit; dormant value need not join the storm: it pays no carrying fee and is never confiscated while towers function — its standing tower deposits exit it automatically to on-chain self-custody ~32 days after its last checkpoint, at the owner's expense, off the attacker's chosen fee peak (L2).

**Residual risk.** **Conceded** (OP4): any value near its ~310 vB exit cost is economically strandable on demand; sub-floor change rides uncheckpointed.

### G4. Operator liquidity amplification via refresh churn

**Severity:** high. **Disposition:** (b) mitigated (→ OP7 for the structural cost).

**Attack.** Each refresh strands a fresh tranche of Q's fronted capital until the root sweep — (4320 + sweep_delay)/144 ≈ 58 days at the default. Refresh a small balance hourly from many keys: locked float ≈ balance × refreshes-in-the-lock-window — **~1,390× theoretical leverage** for an unthrottled hourly refresher (58 days of hourly locks) — draining the pool's float and halting honest refreshes.

**v0.3 disposition.** Refresh fees are priced on **capital-lock duration**, not remaining lifetime — the amplification lever's economics are inverted: each extra refresh pays for exactly the capital it strands. A minimum dwell time D between refreshes plus per-leaf rate limits cap fronted-capital leverage directly at **≤ 58 d / D** (canonical D = 7 days ⇒ ≈8×, matching economics.md §1.1); rapid re-refresh is surcharged.

**Residual risk.** An adversary spreads across keys and pools to evade caps; honest high-velocity users feel the same limits. The underlying **0.5–0.9%-per-refresh** carrying cost on actively refreshed balances (break-even ≈0.8% per refresh at 5% APY — each refresh locks operator capital ~58 days; float ≈ hot_balance × 58 d / refresh_cycle) is structural to covenant-less Ark (**conceded**, OP7).

### G5. Targeted fee-spike timing on T_exp deadlines

**Severity:** high. **Disposition:** (b) mitigated (→ OP3, OP4).

**Attack.** Every pool's tower-broadcast window (T_exp−720) is publicly computable. Flood the mempool during it, pushing feerates above small vTXOs' exit economics and fixed tower bounties; exits stall; the operator sweeps at expiry.

**v0.3 disposition.** The confiscation endpoint is largely removed: checkpointed value missed at the deadline rides the recovery tapleaf (T_exp+288, backed by standing tower deposits) rather than being swept (T_exp+sweep_delay, default 4032) — provided the owner or a tower acts within the 3,744-block window (L2). Height-deposit packages carry pre-signed escalating bump children (10/50/200 sat/vB, spending the owner's dedicated fee UTXOs) with the contingent tower fee riding each variant — the tower picks the lowest variant that clears, though feerates above the pre-signed ceiling stall the package (OP3); wallets consolidate dust and exit pre-emptively off-peak; expiries are laddered hourly with 1008-block margins so the adversary must sustain pressure across rungs for a month.

**Residual risk.** Sustained fee pressure still strands dust-sized vTXOs (OP4) and taxes everyone's exit; deadline-adjacent delay is the OP3 concession. Users pay elevated fees regardless — cost imposition stands even where theft fails.

### G6. Dust vTXO flooding of leaf slots and the checkpoint log

**Severity:** high. **Disposition:** (b) mitigated (→ OP4).

**Attack.** Cheap arkoor self-splits generate thousands of dust vTXOs; a refresh intent per dust vTXO starves honest refreshers of the round's leaf slots while per-vTXO log entries bloat relays and client scans unboundedly.

**v0.3 disposition.** 10,000-sat minimum leaf denomination; bonded intents with a flat fee floor make dust refresh uneconomical; per-key slot quotas per round; the 30178 log holds fixed-size (nullifier, commitment) pairs and is **pruned to live nullifiers**; 16×64 cell structure caps any single cell's exposure.

**Residual risk.** **Conceded** in part (OP4): the anti-dust floor pushes legitimate micropayments into uncheckpointed change; an adversary spreads dust across keys to evade quotas; log growth still burdens relays during an active attack.

### G7. TRUC anchor-slot contention / CPFP fee-escalation grief

**Severity:** medium. **Disposition:** (b) mitigated (slot-occupancy vector removed; escalation residual → OP3).

**Attack.** TRUC permits one unconfirmed child and the P2A anchor is anyone-can-spend. Occupy the slot with a minimal-fee child first; force the honest CPFP into an RBF escalation war, re-pinning just below each replacement; one contested shared-branch parent stalls every descendant exit.

**v0.3 disposition.** The anchor is **never a free slot**: the exiter pre-signs its own minimum-feerate child at package creation and submits parent+child atomically as 1p1c via `submitpackage`, so an attacker cannot get there first. Replacing the honest child requires strictly higher fees — which confirms the package faster. Children spend dedicated unconflictable UTXOs (see L7).

**Residual risk.** An attacker can still pay to push defenders into higher fees and marginal delay during congestion; cheap relative to the delay imposed on shared branches. Folded into the OP3 cycling concession.

### G8. Gift-wrap spam flood on receiver relays

**Severity:** medium. **Disposition:** (b) mitigated.

**Attack.** Flood a victim's public read relays with millions of well-formed bogus kind-1059 wraps p-tagged to random one-time keys. The waking receiver must NIP-44 trial-decrypt everything — battery, bandwidth, and a buried real payment, possibly missing a refresh deadline.

**v0.3 disposition.** Arkoor wraps carry **PoW tags ≥20 bits** (the spam floor; relay/Q demand capped at 20). The pre-auth filter tag (event-kinds §3.11) narrows trial decryption only for **re-fetches and tower handoffs** — the receiver cannot pre-derive sender-chosen counters, so first-discovery wraps must still be trial-decrypted in full. Relays rate-limit and may require PoW/paid writes; clients dedup and prioritize; the mandatory sender-funded height-deposit (L3, height- and conflict-triggered) protects the underlying funds even if discovery is delayed.

**Residual risk.** Trial-decrypt cost under a determined paid flood remains nonzero — and is the full cost for first discovery, where no filter tag helps; PoW taxes legitimate senders; curated receive relays shrink the anonymity set (tension with P3).

### G9. vHTLC gateway liquidity exhaustion (free option)

**Severity:** medium. **Disposition:** (b) mitigated.

**Attack.** Open many swaps committing the gateway's HTLC capacity, then never reveal the preimage. Capital freezes for the CLTV window per swap; concurrency exhausts the gateway; aborts cost the attacker only fees.

**v0.3 disposition.** Non-refundable upfront reservation fees price the option; per-identity in-flight value caps bound concurrency; CLTV deltas are short and randomized; PTLCs are preferred; gateways diversify and rebroadcast.

**Residual risk.** Upfront fees deter casual but not well-funded attackers; bridge liquidity remains lockable for 144-block windows; a thin gateway market concentrates targets.

### Privacy

### P1. Operator as global payment-graph surveillance point

**Severity:** critical. **Disposition:** (c) **conceded** as an open problem (→ OP1).

**Attack.** At each arkoor cosign Q sees the spent vTXO, both outputs, the one-time key, and the amount; the receiver's later refresh binds the one-time key to their identity; boarding ties keys to on-chain identity. Any single operator reconstructs the full payment graph — strictly worse than Lightning, where no single party sees the whole graph.

**v0.3 disposition.** Bounded at the edges, conceded at the core: the public surface is reduced to commitments/nullifiers (P2); stealth keys are per-payment (P5); vaults are unlinkable (P6). But Q necessarily relates spends to prevent double-spending, and **blinded cosigning compatible with portable fraud proofs is unsolved**. v0.3 states plainly that Flock's privacy is below Lightning's.

**Residual risk.** **Conceded** (OP1). Operators see amounts, linkage, and timing at cosign time; 4-of-7 jurisdictional diversity is the only practical bound on what is done with it.

### P2. Public amount-revealing payment DAG in the cosign log

**Severity:** eliminated (was: high). **Disposition:** (a) eliminated by construction for the public surface; operator-side visibility remains P1.

**Attack.** The old kind-38403 log publicly named spent vTXO, payment output, change output, sequence, and amount per arkoor — anyone scraping relays rebuilt the full vTXO-level DAG with amounts; one deanonymized node collapsed the component.

**v0.3 disposition.** The public transfer log (kind 30178, one root per arkoor cosign) contains only quorum-signed Merkle roots over `(nullifier, commitment)` pairs with a strictly increasing sequence, where `nullifier = SHA256("FlockNullifier/v0.3" ‖ outpoint)` — derived from the **outpoint alone**, so any double cosign of one outpoint produces an identical nullifier under any two seqs, a mechanically detectable collision — and **no cleartext outputs, amounts, or DAG edges**. Receivers verify by private Merkle inclusion proof; proof-of-payment is an inclusion proof, not a public log row.

**Residual risk.** Per-cosign root cadence and timing metadata leak coarse activity (one root per cosign batch is finer-grained timing than the old hourly roots); collision detection requires scanning the log — towers/auditors do this, receivers check their own outpoints — so full-log availability is part of OP2; commitments must remain Q-relatable for double-spend detection, which is exactly the conceded P1/OP1 surface. Gateway swaps and L1 exits remain deanonymization anchors.

### P3. Recipient leak via NIP-65 delivery relays

**Severity:** high. **Disposition:** (b) mitigated.

**Attack.** Gift wraps hide the recipient's npub inside the event, but delivery to "Bob's read relays" fingerprints him: a relay receiving the wrap maps it to the owner of that relay set and correlates inbound-wrap timing with Bob's later activity from the same set.

**v0.3 disposition.** Wraps are addressed to one-time keys and delivered to Bob's read relays **plus the pool relays**, which serve every pool user and form a shared anonymity set; clients access relays over Tor (operators themselves run hidden services); vault and intent traffic use derived keys unlinkable to the npub, weakening cross-correlation.

**Residual risk.** The anonymity set is bounded by shared-relay popularity; a global relay observer correlating wrap arrival with subsequent 24400 intents still narrows the recipient (→ OP1). Decoy traffic and mix-style delivery are costed but not mandated. Severity retained.

### P4. Watchtower watch-request deanonymization harvesting

**Severity:** high. **Disposition:** (b) mitigated.

**Attack.** Run attractive (high-bounty, reliable) towers at scale to harvest watch requests, building a database mapping vTXO ↔ L1 address ↔ amount ↔ npub.

**v0.3 disposition.** Height-deposits (38404) are encrypted to **freshly derived per-request tower keys** and **sharded across multiple towers** so no single tower holds the full package; towers are keyless executors ("broadcast P at height H or on conflict, unless superseded") and the claim's final L1 address need not be embedded until claim time; the public 8405 alert stream carries no user linkage.

**Residual risk.** Colluding towers can recombine shards; the tower that actually broadcasts necessarily learns the on-chain footprint at exit; a thin tower market concentrates whatever is harvested (→ OP10).

### P5. Stealth one-time key reuse (missing per-payment tweak)

**Severity:** eliminated (was: medium). **Disposition:** (a) eliminated by construction.

**Attack.** The old derivation `P_B = B + H(ECDH(a,B))·G` had no per-payment component: every payment A→B produced the identical one-time key and scriptpubkey, trivially linking repeat counterparties in the public log and on-chain at exit.

**v0.3 disposition.** The derivation is `P_B = B + H(ECDH(a,B) ‖ cell_txid ‖ counter)·G` — a fresh unlinkable key per payment, Silent-Payments-style; no scriptpubkey reuse exists.

**Residual risk.** Implementation correctness: the receiver scans a bounded counter space per cell tx; sender bookkeeping must never reuse a counter. Engineering, not protocol.

### P6. Wallet vault (37400) metadata leakage

**Severity:** medium. **Disposition:** (b) mitigated.

**Attack.** The old vault was authored by the user's main npub: ciphertext size growth and update cadence yielded a public per-identity wealth-and-activity estimator without any decryption.

**v0.3 disposition.** Vaults are authored by a **seed-derived unlinkable key** (deterministic for recovery, unlinkable to the npub) and **padded to fixed size buckets**; replication uses paid pinning plus the operator-run archival relays (38400 `params.archive_relays`), none of which sees plaintext.

**Residual risk.** Coarse buckets still leak order-of-magnitude wealth to anyone who breaks the author-key unlinkability or correlates upload timing; update cadence under load may correlate with activity. Accepted as a bounded leak.

## 5. Summary table

| # | Attack | Panel severity | v0.3 disposition | Primary mechanism |
|---|---|---|---|---|
| T1 | Deterministic nonce reuse extracts keys/shares | critical | Mitigated | CSPRNG-only, RAM-only nonces; one partial per nonce; transcript cross-verification |
| T2 | FROST-of-MuSig2 composition leaks Q | eliminated (was: critical) | Eliminated | Composition removed; two-signature tapscripts; plain FROST for Q |
| T3 | Arkoor double-spend via eclipsed checkpoint log | critical | Mitigated; accountability conceded (OP6) | Fail-closed disjoint-relay checks; ≥k-mirror read-back; VaR cap; fraud proofs (equivocation + nullifier collision) |
| T4 | Hash-lock atomicity break / mass forfeit | eliminated (was: critical) | Eliminated | Connector outputs bind forfeit to confirmed cell txid |
| T5 | Dead-quorum stale exits rob arkoor recipients | eliminated (was: critical) | Eliminated (dead-quorum case); residuals: stale-owner early unroll (mitigated), refresh-stale forfeits (OP9) | Decrementing claim ladder; nSequence-0 permissionless forfeits; conflict-triggered deposits |
| T6 | Delegated-refresh sock-puppet theft | eliminated (was: critical) | Eliminated | Feature deleted; height-deposits + recovery tapleaf |
| T7 | Forfeit race via pinning/cycling | high | Mitigated; cycling conceded (OP3) | Ladder removes race; instant forfeits; dedicated fee UTXOs; 1p1c |
| T8 | PTLC adaptor-secret leak | high | Mitigated | Journaled fresh nonces; one finalize per nonce; adaptor verification |
| T9 | Round-assembly view-split captures inputs | high | Mitigated | Canonical participant-set commitment; connector cell-txid binding; confirmed-script verification |
| T10 | Recovery tapleaf names stale pre-arkoor owner | high | Mitigated; residual conceded (OP8) | 2016-block receive ban; VaR auto-checkpoint; tower exit by T_exp−720 |
| T11 | Refresh preimage-withholding freeze | eliminated (was: high) | Eliminated | Connector atomicity; no reveal step exists |
| T12 | FROST resharing/DKG split view | medium | Mitigated | Epoch-tagged signed transcripts; complaint rounds; group-key invariant |
| T13 | Exhaustible decrementing-locktime window | eliminated (was: low) | Eliminated | Relative (BIP-68) ladder; 4-hop cap; reset at checkpoint |
| L1 | DA stranding of arkoor receipts | critical | **Conceded (OP2)** | Bounded by ≥k-replication finality, local persistence, archival relays, VaR auto-checkpoint |
| L2 | Refresh censorship → T_exp confiscation | high | Mitigated (confiscation → fee tax, if owner/tower acts in window; OP10) | Recovery tapleaf (T_exp+288) before Q sweep (T_exp+sweep_delay); standing tower deposits |
| L3 | Never-online receiver confiscated at T_exp | high | Mitigated (OP10 residual) | Sender-funded 38404 (height+conflict triggered); Q per-cosign deposits; recovery tapleaf; delayed sweep |
| L4 | Quorum-offline expiry coercion | high | Mitigated; forced exit conceded (OP9) | 4-of-7 AS/geo diversity; 31401 heartbeats; pre-emptive exits |
| L5 | NIP-65 relay-set eclipse | high | Mitigated (ISP/state-level residual) | Pinned bootstrap set; operator-independent attestations; L1 cross-check |
| L6 | Sweep-at-expiry confiscation | high | Mitigated (recovery window + tower deposits; OP8/OP10 residuals) | Recovery tapleaf precedes sweep by 3,744 blocks; standing dormancy deposits |
| L7 | Replacement-cycling / child-slot grief on exits | high | Mitigated; cycling conceded (OP3) | Dedicated unconflictable fee UTXOs; 1p1c; tower redundancy; margins |
| L8 | Targeted relay eviction forces exits | high | Mitigated | 120 s adaptive timers; cell blast radius; relay diversity; reschedule |
| L9 | Operator-relay censorship of watch requests | high | Mitigated (OP10 residual) | Fire-and-forget height-deposits; sharded towers; ≥3 independent relays |
| L10 | Coordination-event censorship | medium | Mitigated (OP5-adjacent residual) | Monotonic 30178 seq + inclusion proofs; fail-closed; timeout-to-exit |
| L11 | PoW/AUTH censorship chokepoint | medium | Mitigated | 20-bit PoW cap; pre-mining; many-relay submission; read-back verification |
| L12 | Boarding strand | medium | Mitigated | 288-block refund CSV; same-round conversion; heartbeat gating |
| L13 | Blossom tree-commitment DA loss | eliminated (was: medium) | Eliminated | Inline commitments mandatory; mirrors never sole copy |
| L14 | vHTLC swap stalls / cycling | medium | Mitigated (OP3 residual) | Rebroadcast; short randomized CLTVs; PTLC preference |
| G1 | Single-aborter round collapse | critical | Mitigated; deterrence gap conceded (OP5) | 16×64 per-cell-funded cells; post-nonce finalization; bonded intents (ledger debit netted at next service + lien backstop, ≤1k sats) |
| G2 | Sybil-seat branch-stall griefing | high | Mitigated (OP5) | 10k-sat floor; bonded intents; quotas; same-attempt rebuild |
| G3 | Exit-storm fee spike strands dust | high | Mitigated; dust conceded (OP4) | Min denomination; batched/pre-emptive exits; recovery-leaf dormancy |
| G4 | Operator liquidity amplification | high | Mitigated (OP7 residual) | Capital-lock-duration pricing; minimum dwell (≤58 d/D leverage bound); per-leaf rate limits |
| G5 | Targeted fee-spike timing on T_exp | high | Mitigated (OP3/OP4 residual) | Recovery tapleaf; escalating bump children (10/50/200 sat/vB); laddered expiries; margins |
| G6 | Dust flooding of slots and log | high | Mitigated (OP4 residual) | 10k floor; fee floor; quotas; log pruned to live nullifiers |
| G7 | TRUC anchor-slot contention | medium | Mitigated (OP3 residual) | Pre-signed own min-fee child; atomic 1p1c submission |
| G8 | Gift-wrap spam flood | medium | Mitigated | PoW-tagged wraps; filter tag (re-fetches/tower handoffs only); rate limits |
| G9 | Gateway free-option exhaustion | medium | Mitigated | Upfront reservation fees; in-flight caps; short randomized CLTVs |
| P1 | Operator payment-graph surveillance | critical | **Conceded (OP1)** | Edges bounded (P2/P5/P6); blinded cosigning unsolved |
| P2 | Public amount-revealing payment DAG | eliminated (was: high) | Eliminated | 30178 holds (outpoint-only nullifier, commitment) Merkle roots only |
| P3 | Recipient leak via delivery relays | high | Mitigated (→ OP1 residual) | One-time keys; pool-relay anonymity set; Tor |
| P4 | Watchtower harvesting | high | Mitigated (OP10-adjacent) | Per-request tower keys; sharded packages; keyless towers |
| P5 | Stealth one-time key reuse | eliminated (was: medium) | Eliminated | Per-payment tweak: `H(ECDH(a,B) ‖ cell_txid ‖ counter)` |
| P6 | Vault metadata leakage | medium | Mitigated | Derived unlinkable author key; padded size buckets |

Tally (recounted for v0.3): 9 eliminated by construction, 31 mitigated with stated residual risk, 2 conceded outright (L1 → OP2, P1 → OP1). T5's "eliminated" rating covers the dead-quorum case only — its stale-owner early-unroll residual is mitigated, not eliminated. Several mitigated entries carry partially conceded residuals, each cross-referenced above.

## 6. Residual trust: explicit time bounds

The complete list of conditions under which a Flock user trusts anyone, with the duration of each exposure (normative, from the synthesis trust model as amended by [../research/AMENDMENTS-v0.3.md](../research/AMENDMENTS-v0.3.md)):

- **Uncheckpointed arkoor.** Revocable by the **sender alone** if the receiver remains offline beyond the ladder differential (144–576 blocks, depending on the hops involved; the tower's absolute deadline is the attacker's claim maturity — the ladder defense deadline, 288–720 blocks after the unroll) after a hostile unroll and no tower acts; revocable by **sender + 4-of-7 collusion** otherwise; **extinguished at checkpoint**. An uncheckpointed receipt is not *accepted* until the receiver's fail-closed wake-up verification passes — clawback before acceptance is a failed payment, not theft of accepted funds. Bound: until the next checkpoint — automatic at the next available round seat once the **aggregate uncheckpointed balance** crosses the 50,000-sat VaR cap (VaR-triggered refreshes take seat priority; structuring sub-cap receipts does not evade the cap; enforceable only while online — offline exposure is bounded by the acceptance rule and Q's optional `uncheckpointed_ceiling`); hard bound T_exp (receives are banned in the final 2016 blocks). Operator equivocation (two signed 30178 roots at one sequence number) yields a portable fraud proof; a double cosign of one outpoint collides as an identical nullifier (outpoint-only derivation), detectable by anyone scanning the log (OP2 for log availability).
- **Offline safety.** No third party is needed before **T_exp − 720**; at T_exp − 720 the height-deposit tower must act. Under attack the dependency starts earlier: deposits also fire on conflict detection; against a stale-owner unroll of an uncheckpointed chain the tower's deadline is the ladder defense deadline — **288–720 blocks** after the unroll (head-start differential 144–576) — while a held forfeit against a refresh-stale claim must land within the **144–720-block** forfeit stale window. Checkpointed value missed by every tower rides to the recovery tapleaf (T_exp + 288 → **tower-protected** self-custody); if the owner *and* all towers stay absent for the full 3,744-block window, the sweep at T_exp + sweep_delay confiscates (L2, OP10). Worst case for a never-online receiver: tower-dependent from the moment of a hostile unroll, and unconditionally from T_exp − 720.
- **Recovery-leaf stale window.** Uncheckpointed value whose holder stays offline past **T_exp + 288** is claimable by the prior checkpointed owner — bounded by the VaR cap and the receive-near-expiry ban (OP8).
- **Intent bonds.** Q's power to slash an intent bond on post-nonce abort is unilateral; exposure is bounded at ≤ `intent_bond_sats` (default 1,000 sats) per intent (a ledger debit netted at next service, with a lien-tx backstop — no standalone bond vTXO). Honest completion or pre-nonce exit voids it; boarding users are exempt (PoW-only).
- **Swaps.** Standard HTLC exposure for the **144-block CLTV delta** per swap leg.

Beyond these: a 4-of-7-colluding quorum **cannot** reverse checkpointed vTXOs, mint state, sweep before T_exp + sweep_delay (default 4032), or beat the claim ladder. These guarantees are deadline-conditioned, not absolute: a forfeit beats every laddered claim *once broadcast*, but someone must broadcast it within the 144–720-block forfeit stale window (OP9); the recovery claim beats the sweep only if the owner or a tower acts within the 3,744-block window (L2); deposit execution needs at least one live, funded tower (OP10). Relays can deny service and read metadata, never steal.

## 7. Open-problem index

Conceded items above cross-reference this list (normative statement in [../whitepaper.md](../whitepaper.md), §Open problems):

| OP | Open problem | Conceding attacks |
|---|---|---|
| 1 | Operator payment-graph visibility; blinded cosigning unsolved | P1; residuals of P2, P3 |
| 2 | Uncheckpointed-receipt data availability (incl. archival-relay retention); receipts not seed-recoverable; nullifier-collision detection needs full-log availability | L1; log-availability residuals of T3, P2 |
| 3 | Replacement cycling / fee-spike timing; no mempool-level fix | T7, T8, L7, L14, G5, G7 |
| 4 | Dust: value near ~310 vB exit cost economically strandable; 10k floor pushes micropayments into uncheckpointed change | G3, G5, G6 |
| 5 | Grief vs. loss indistinguishability: bonds tax the honest or under-deter | G1, G2, L10 |
| 6 | Accountability without covenants: slashing reduces to fraud proofs + reputation; bonds need trusted arbitration; share/key deletion attestations (31401) carry reputational and legal weight only — retention is cryptographically unverifiable | T3 |
| 7 | Structural 0.5–0.9%-per-refresh carrying cost on actively refreshed balances (break-even ≈0.8% per refresh at 5% APY; each refresh locks capital (4320 + sweep_delay)/144 ≈ 58 days) | G4 |
| 8 | Recovery-leaf stale window for sub-cap value offline past T_exp+288 | T10, L6 |
| 9 | Quorum liveness: <k halts payments and forfeit creation (exit-only mode); held forfeits need a broadcaster within the 144–720-block forfeit stale window | T5 (residual), T12, L4 |
| 10 | Watchtower market thinness: height-deposits automate action, but a fee-budgeted tower economy is unproven — and it now also carries the dormancy guarantee (standing deposits are the defense against the root sweep) | L2, L3, L6, L9, P4 |
