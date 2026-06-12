# Covenant-less L2 Toolbox on Bitcoin — mid-2026 (no new soft forks)

## 1. MuSig2 (BIP-327)

- **What it gives you:** n-of-n key aggregation producing a plain BIP-340 signature in 2 rounds; the on-chain footprint is a single Taproot keyspend. Spec: https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
- **Nonce safety:** the secret nonce (`secnonce`) must be used at most once, ever. Reusing it across two sessions lets a co-signer extract your private key algebraically. Crucially, **deterministic nonces are unsafe**: a malicious co-signer opens two sessions on the same message with different nonces of their own, obtaining two of your partial sigs under one nonce → key leak (BitGo field report, https://bitcoinops.org/en/bitgo-musig2/).
- **Why stateless/async signing is dangerous:** a signer must hold `secnonce` between round 1 (nonce exchange) and round 2 (partial sig). Any state cloning — VM snapshots, replicated HSMs, crash-recovery, or naive retry logic over a lossy transport — can replay round 2 with the same `secnonce` against different aggregate nonces.
- **Safe patterns over unreliable transport:**
  - Delete-on-read `secnonce` (BitGo's SDK erases it the moment it's accessed for signing).
  - Only the **last** signer may be stateless: after receiving all other pubnonces plus the final message, it can run NonceGen→NonceAgg→Sign in one shot, even deterministically (BIP-327 §"stateless signer"; BitGo report).
  - Feed message/key-set/session-ID into NonceGen's auxiliary input; on any timeout, duplicate, or ambiguity, **abort and restart with fresh nonces** — idempotency via session IDs, never via nonce reuse.
  - Use BIP-373 PSBT MuSig2 fields to carry pubnonces/partial sigs across retries safely: https://github.com/bitcoin/bips/blob/master/bip-0373.mediawiki. Core has BIP-390 `musig()` descriptor support (PR https://github.com/bitcoin/bitcoin/pull/31244).

## 2. FROST (threshold Schnorr, t-of-n)

- **Maturity:** standardized as IETF RFC 9591 (2024); produces standard BIP-340 signatures when instantiated with the secp256k1 "tr" ciphersuite. A consensus-facing signing BIP is still draft: https://github.com/siv2r/bip-frost-signing.
- **Libraries:** ZF FROST (Rust, audited, actively released — https://github.com/ZcashFoundation/frost); `frost-secp256k1-tr` crate adds BIP-340/341 compatibility incl. Taproot tweaks (https://github.com/ZcashFoundation/frost/pull/730, on crates.io at 2.x). FROSTR/bifrost runs FROST over Nostr relays — a working example of threshold signing over async, unreliable transport (https://github.com/frostr-org/bifrost). Frostsnap ships FROST hardware signers commercially (https://frostsnap.com/).
- **DKG options:** trusted dealer (simplest), PedPoP-style DKG in ZF frost, and **ChillDKG** — a purpose-built, recoverable DKG BIP draft with unspendable-script-path commitments (https://github.com/BlockstreamResearch/bip-frost-dkg); still a draft as of mid-2026.
- **Proactive resharing:** ZF frost supports share *refresh* (rotate shares, group key unchanged) and *repair* of lost shares; FROSTR exposes rotation as a first-class operation. Same nonce rules as MuSig2: fresh random nonces only, never resume an interrupted session.

## 3. Adaptor signatures / PTLCs

- **Status:** Schnorr adaptor signatures work on Taproot today and compose with MuSig2 (musig adaptor support in libsecp256k1-zkp; Rust `musig2` and `schnorr_fun` crates implement them — https://docs.rs/musig2/latest/musig2/) and with FROST.
- **In production:** DLCs (ECDSA adaptors) and cross-chain atomic swaps; recent academic work continues (e.g., https://arxiv.org/pdf/2503.12719).
- **Lightning PTLCs:** still not deployed. An Eclair proof-of-concept exists (https://github.com/bitcoin-s/eclair/blob/develop/docs/PTLC.md), but Optech still characterizes PTLCs as "mostly a discussion topic" (https://bitcoinops.org/en/topics/ptlc/); 2025 discussion noted CTV+CSFS would simplify them. **For a greenfield L2 where you control both protocol endpoints, PTLCs are usable now**; LN-route interop still means HTLCs.

## 4. Pre-signed transaction trees

- **Lineage:** Burchert–Decker–Wattenhofer channel factories (2018, older material) — n-of-n pre-signed; one offline participant freezes the factory.
- **Timeout Trees** (John Law / jlspc, "Scaling Lightning With Simple Covenants," bitcoin-dev 2023 — https://gnusha.org/pi/bitcoindev/vUfA21-18moEP9UiwbWvzpwxxn83yJQ0J4YsnzK4iQGieArfWPpIZblsVs1yxEs9NBpqoMBISuufMsckbuWXZE1qkzXkf36oJKfwDVqQ2as=@protonmail.com/): a covenant tree with an expiry after which the LSP sweeps the root in one tx. As specified it wants CTV; covenant-less emulation = all-of-all pre-signing.
- **Exact interactivity without covenants:** every user with funds in the tree must be online **at tree creation** to verify the tree and complete a MuSig2 nonce round + signing round for each tx on their root-to-leaf path, before the funding tx confirms. Afterward, unilateral exit needs no cooperation.
- **Ark-style (clArk):** per-round trees co-signed n-of-n (round participants + server) via MuSig2; only that round's users must be online; costs one extra signing round vs covenant Ark (https://ark-protocol.org/intro/clark/index.html, https://arkdev.info/docs/learn/clark/). Covenant-less Ark implementations (Second's Bark; Ark Labs) reached mainnet-beta in 2025 — status evolving, verify before relying (https://bitcoinops.org/en/topics/ark/).
- **Interactivity-reduction tricks:** ephemeral cosigner keys deleted after the ceremony ("pseudo-covenant" — trust = deletion); stateless-deterministic *last* signer for the coordinator (BitGo pattern); batching all path-txs into one multi-session ceremony; periodic refresh rounds instead of on-chain exits; **SuperScalar** (ZmnSCPxj, Delving 2024: https://delvingbitcoin.org/t/superscalar-laddered-timeout-tree-structured-decker-wattenhofer-factories-with-pseudo-spilman-leaves/1242) ladders timeout-trees with Decker–Wattenhofer layers so subtrees update without re-onboarding everyone — design critiques exist and project status is uncertain. Tradeoff map: Peter Todd's covenant-dependent L2 review (2024, https://petertodd.org/2024/covenant-dependent-layer-2-review).

## 5. Timelocks (CLTV/CSV)

- CLTV (BIP-65) = absolute; CSV (BIP-112) = relative; both only *compare* against spending-tx fields, so the spender must set nLockTime (with non-final nSequence) / nSequence (with nVersion ≥ 2). BIPs: https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki, bip-0065, bip-0112, bip-0113.
- **Pitfalls:** height-vs-time unit mismatch between opcode argument and tx field fails the script; time locks use median-time-past (BIP-113), lagging wall clock ~1–2 h; off-by-one (spendable strictly after the lock); **CSV clocks start at parent confirmation, so relative delays along a tree path add serially** — worst-case exit latency = tree depth × CSV; any absolute expiry (timeout-tree sweep) must exceed the sum of all path-relative delays plus contest periods plus congestion margin; users must exit *before* expiry, so wallets need deadline tracking.

## 6. Exit fee-bumping — what Core actually ships

- **Core 28.0 (Oct 2024):** opt-in TRUC/v3 (BIP-431) — topology-limited txs killing rule-3 pinning; sibling eviction; package RBF; keyless **P2A** anchor output (BIP-433: https://github.com/bitcoin/bips/blob/master/bip-0433.mediawiki); opportunistic 1-parent-1-child package relay. Guide: https://bitcoinops.org/en/bitcoin-core-28-wallet-integration-guide/
- **Core 29.0 (Apr 2025):** **ephemeral dust** — one below-dust output allowed iff the tx is zero-fee TRUC and the child spends it in the same package (PR https://github.com/bitcoin/bitcoin/pull/30239; https://www.nobsbitcoin.com/bitcoin-core-v29-0/).
- **Core 30.0 (Oct 2025):** wallet creates/spends TRUC; relay fee floor lowered to 0.1 sat/vB (https://bitcoincore.org/en/2025/10/10/release-30.0/).
- **Core 31.0 (rc-phase, due 2H 2026):** cluster mempool merged Nov 2025 (https://bitcoincore.org/en/releases/31.0/; https://bitcoinops.org/en/topics/cluster-mempool/) — do **not** assume network-wide deployment yet.
- **Design guidance:** pre-sign exits as v3/TRUC, zero fee, with a single zero-value P2A ephemeral anchor; anyone (watchtower, LSP, user) can CPFP without keys; broadcast via `submitpackage`; treat 1p1c relay as best-effort until cluster-mempool nodes dominate; respect TRUC limits (parent ≤10,000 vB, child ≤1,000 vB, no chains beyond 1p1c).

## 7. Mass-exit fee-market dynamics

- "Mass Exit Attacks on the Lightning Network" (Sguanci & Sidiropoulos, IEEE ICBC 2023; https://arxiv.org/abs/2208.01908 — older material, pre-TRUC): zombie and forced-double-spend attacks by small coalitions under historically observed congestion can push honest justice/exit txs past their timelock windows; fee spikes are self-reinforcing as exits compete.
- Factories amplify this: one UTXO backs many users, so forced unfurling costs O(log n) txs per user, O(n) total — exit demand can exceed multiple blocks' capacity.
- Literature mitigations: timeout-tree expiries make cooperative sweep the default and on-chain exit exceptional (Law); ladder/stagger expiries so deadlines never synchronize (SuperScalar); size timelocks against simulated worst-case congestion; keyless P2A anchors let third parties fund exits when users can't.
