# Research sources

Digests produced by the design workflow (2026-06-12), each with inline citation URLs:

- [research-lightning-pain.md](research-lightning-pain.md) — Lightning's structural
  problems (inbound liquidity, always-online, toxic state, jamming, LSP centralization)
  and the mitigations that already exist; checklists of what a successor must solve and
  must not regress.
- [research-ark.md](research-ark.md) — the Ark protocol family: vTXOs, rounds,
  connectors/forfeits, covenant-less clArk interactivity, arkoor trust model,
  implementations (Ark Labs bark, Second, Arkade), exit costs, operator economics,
  open problems.
- [research-statechains-spark.md](research-statechains-spark.md) — Mercury Layer and
  Spark: statechain transfer mechanics, threshold operator groups, the key-deletion
  trust problem and mitigations, offline receive, criticisms.
- [research-nostr-prior-art.md](research-nostr-prior-art.md) — Nostr as payments
  infrastructure: NWC, zaps, Cashu/nutzaps, FROSTR, Mostro/Civkit/joinstr, prior
  proposals for L2-over-Nostr, and relay-network properties (retention, outbox,
  ephemeral events, AUTH, PoW, delivery reliability).
- [research-crypto-toolbox.md](research-crypto-toolbox.md) — the no-soft-fork toolbox:
  MuSig2 nonce safety over async transport, FROST maturity, PTLCs/adaptor signatures,
  pre-signed tree constructions (channel factories, Timeout Trees), CLTV/CSV patterns,
  TRUC/P2A/package relay status, mass-exit fee dynamics.

## Raw contest artifacts (`raw/`)

- `synthesis-v0.2.md` — **normative** consolidated architecture (see `../ERRATA.md`).
- `design-A-federated-ark.md` / `design-B-statechain-mesh.md` / `design-C-wildcard.md`
  — the three competing designs (A won, 2/3 judge votes; B contributed the claim
  ladder, recovery tapleaf, and deletion attestations; C contributed the Merkle
  checkpoint log, height-deposits, and the value-at-risk cap).
- `contest-verdicts.json` — judge scores, fatal flaws, graft list.
- `attacks.json` — 42 adversarial findings (theft / liveness / griefing lenses) with
  mitigations and residual risk.
- `workflow-result.json` — complete workflow output.
