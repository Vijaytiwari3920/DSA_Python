# Research Proposal

## Succinct Accountable Stake-Weighted Multi-Signatures for Proof-of-Stake Finality

**Principal investigator:** VS Tiwari
**Area:** Proof-of-Stake consensus / applied cryptography for distributed systems
**Status:** Draft proposal. The novelty claim in §3 is stated against the closest prior art known at time of writing and is *contingent on completing the systematic literature review in §7 (Milestone M0)*.

---

### 1. Motivation and Problem Statement

In Proof-of-Stake (PoS) Byzantine-fault-tolerant consensus, finality is established when validators controlling at least a threshold fraction of total stake (typically two-thirds) vote for a block. As validator sets grow into the tens of thousands (and, for Ethereum-style designs, far beyond), the cost of collecting, transmitting, storing, and verifying these votes becomes a first-order bottleneck. Signature aggregation is the standard remedy: many validator signatures are compressed into a single short object that a verifier can check against an aggregated key.

Aggregation, however, is in tension with two properties that PoS specifically requires. First, **accountability**: the protocol must know *which* validators signed and which did not, because rewards, inactivity penalties, and slashing all depend on per-validator participation. Standard aggregate signatures discard this information, and the usual fix — attaching an *n*-bit signer bitmap — reintroduces an O(*n*) term that dominates an otherwise near-constant-size aggregate at scale. Second, **stake-weighting**: the finality condition is a threshold over *stake*, not over a *count* of validators, yet most compact multi-signature constructions are count-based and encode weights only by clumsy replication.

The result is that no widely-used voting primitive simultaneously delivers a compact signature, verification of a *stake-weighted* threshold, and *succinct* accountability. Chains today pick two of the three and pay for the missing one operationally. This proposal targets that gap directly, as a cryptographic construction integrated into and measured within a real PoS consensus system.

### 2. Research Questions and Objectives

The central research question is whether the three properties above can be achieved together without introducing a new cryptographic hardness assumption, and at a verification and bandwidth cost that improves on current practice in a deployed-style PoS finality path.

Concretely, the project pursues four objectives:

1. **RQ1 (Primitive).** Design a multi-signature scheme, from standard pairing-based assumptions, whose signature size is (near-)independent of the validator count and of the stake distribution.
2. **RQ2 (Weighting).** Enable a verifier to confirm that signers control at least a threshold τ of total stake, without per-signer data in the common path.
3. **RQ3 (Accountability).** Enable succinct recovery — sub-linear in *n* — of the non-signer set, sufficient for reward/penalty accounting, ideally via short membership proofs rather than a full bitmap.
4. **RQ4 (Systems).** Integrate the primitive into an existing PoS BFT protocol and quantify the effect on finality latency, message size, verification time, and reconfiguration cost against the protocol's native vote path.

### 3. Novelty

The novel target of this work is the **simultaneous** combination of (a) compactness, (b) stake-weighted threshold verification, and (c) succinct accountability in a *single* multi-signature primitive, together with (d) its integration and empirical evaluation inside a live PoS finality gadget. To the best of current knowledge, existing constructions each provide a strict subset:

- **BLS multi-signatures** (aggregate signatures over pairing-friendly curves) give compactness but are, by default, neither accountable (a verifier cannot determine the signer set from the aggregate) nor natively stake-weighted.
- **Accountable-subgroup multi-signatures** (the Boneh–Drijvers–Neven line) restore accountability, but convey the signing subgroup with an O(*n*) description and are count-based rather than stake-weighted. This is the closest prior art and the primary point the novelty must be measured against.
- **Weighted-threshold and weighted multi-signature constructions** address stake-weighting but, to current knowledge, do not also deliver succinct (sub-linear) accountability in the same object.
- **SNARK/zk proofs of stake-weighted signing** (as used in PoS light-client designs) achieve succinctness and weighting together, but at high proving cost and in a light-client setting rather than the full-consensus incentive-accounting setting this proposal targets.

The specific technical delta this project hypothesizes is to **replace the O(*n*) subgroup/bitmap encoding of accountable multi-signatures with a succinct, commitment-based representation of the weighted signer set** — for example, a vector/polynomial commitment to the stake-ordered validator set that supports short proofs of which indices signed — so that a verifier checks the *weighted* threshold in the common path and can later extract non-signers with sub-linear proof size for accounting. If this holds, the primitive achieves (a)–(c) together from known assumptions.

**Honest scoping of the claim.** This is a novelty *hypothesis*, not a settled result. Two outcomes of the §7/M0 review would narrow it, and the proposal is deliberately structured to remain viable in either case: (i) if a construction already combines weighting with succinct accountability, the contribution refocuses on the consensus-level integration and the empirical trade-off study (Objective RQ4), which no prior work is known to provide for this exact primitive; (ii) if only weighting *or* only succinct accountability exists in the literature, the primitive contribution stands as the unification of the two. The systematic review is therefore a gating deliverable, not a formality.

### 4. Proposed Approach

**4.1 The primitive (design direction).** The construction will build on BLS multi-signatures over a standard pairing-friendly curve (e.g., BLS12-381), using proofs-of-possession or the message-augmentation technique to defend against rogue-key attacks. Stake weights and validator identities will be bound through a vector or polynomial commitment to the stake-ordered validator set, fixed per epoch. A signature over a block will carry (i) the aggregated BLS signature, (ii) a short proof, relative to the committed set, that the aggregated public key corresponds to a signer set whose committed weights sum to at least τ, and (iii) the material needed to later produce sub-linear non-signer proofs for accounting. The core research difficulty — and where the security argument must be earned — is making weighting and succinct accountability *compatible with aggregation* while proving existential unforgeability under chosen-message attack by reduction to the underlying pairing assumption, with no new assumption introduced.

**4.2 Consensus integration and base system.** Per the decision to modify an existing system rather than build from scratch, the primary integration target is **CometBFT (Tendermint Core)**: it already performs stake-weighted BFT voting, is modular and widely deployed, and exposes a clean insertion point at the vote-and-commit layer where the native signature scheme can be swapped for the proposed primitive. A HotStuff-based PoS implementation is the backup target if CometBFT's aggregation seam proves too invasive. The native (non-succinct) vote path of the same system provides the apples-to-apples baseline.

**4.3 Security and system model.** Standard PoS assumptions: an adversary controlling less than one-third of total stake; static per-epoch validator set (churn/reconfiguration is explicitly *out of scope* for this first paper and reserved as follow-up, consistent with the churn-free decision); partial synchrony for liveness, safety independent of timing. Security goals: existential unforgeability of the multi-signature; soundness of the weighted-threshold proof (a valid signature implies signer stake ≥ τ); and soundness of accountability (non-signer proofs cannot falsely implicate or exonerate). Cryptographic proofs will be game-based in the random-oracle model, consistent with the BLS lineage.

### 5. Evaluation Plan

Evaluation proceeds on two fronts. Cryptographic evaluation reports concrete signature size, signing cost, aggregation cost, verification cost, and accountability-proof size as functions of *n* and of the stake distribution, benchmarked against BLS multi-signature and accountable-subgroup baselines. Systems evaluation, on the modified CometBFT deployment, measures end-to-end finality latency, per-round message size, verification CPU, and block-header/certificate size, swept across validator-set sizes (e.g., 100 / 1k / 10k) and realistic skewed stake distributions, each compared to the unmodified native vote path. The headline result the paper must produce is a quantified trade-off curve: how much verification/bandwidth is saved at scale, and what the accountability layer costs to obtain it.

### 6. Related Work (to be completed in M0)

The review will classify prior work by which subset of {compact, stake-weighted, accountable} it achieves, covering at minimum: BLS multi-signatures and proof-of-possession defenses; accountable-subgroup multi-signatures (Boneh–Drijvers–Neven); weighted-threshold signatures and recent weighted-to-unweighted reductions; forward-secure multi-signatures for PoS (e.g., Pixel); vector/polynomial commitment and cryptographic accumulator techniques as accountability substrates; and SNARK-based stake-weighted signing in PoS light clients. The output is a capability matrix that either confirms the §3 gap or reduces the claim as described there.

### 7. Timeline and Milestones

- **M0 — Novelty verification (weeks 1–3).** Systematic literature review; produce the capability matrix; finalize or narrow the novelty claim. *Gate: do not proceed to primitive design until the gap is confirmed or the claim is refocused.*
- **M1 — Primitive design and security argument (weeks 4–12).** Formal definitions, construction, unforgeability and soundness proofs.
- **M2 — Cryptographer review checkpoint (week 12–13).** Independent expert review of the proofs before any implementation or write-up hardens around them. *This checkpoint is mandatory given a solo, non-specialist crypto effort at this venue tier.*
- **M3 — Reference implementation of the primitive (weeks 13–18).** Standalone library plus microbenchmarks.
- **M4 — Consensus integration (weeks 18–26).** Modify CometBFT vote path; correctness testing.
- **M5 — Evaluation (weeks 26–32).** Full trade-off study vs baselines.
- **M6 — Write-up and submission (weeks 32–40).** Target the venue selected in §9.

### 8. Risks and Mitigations

The dominant risk is cryptographic: a solo researcher with limited provable-security background may produce a scheme that is unsound or not novel. This is mitigated by scoping the construction to known assumptions (no new assumption to defend), by the M0 gate (novelty confirmed before design), by the M2 external proof review (soundness confirmed before implementation), and — structurally — by ensuring the consensus integration and empirical trade-off study (RQ4) constitute a publishable contribution on their own, so that a narrowed primitive does not sink the project. A secondary risk is that the accountability-proof cost erases the bandwidth savings; the evaluation is designed to surface this early (M5 depends only on M3), allowing a pivot to reporting the negative/threshold result honestly. A tertiary risk is integration friction in CometBFT; the HotStuff backup target mitigates it.

### 9. Target Venues

The integrated systems-and-crypto result fits IEEE S&P, ACM CCS, and USENIX Security; financial-cryptography venues (FC, AFT) are a strong secondary fit and a more accessible bar. If the primitive contribution proves substantial and self-contained, it may alternatively target a cryptography venue (PKC, Asiacrypt) as a separate paper, with the systems integration published independently.

### 10. References

*Note: the following are named from the author's and assistant's recollection and have not been verified against a database. Several titles, authors, years, and venues may be inaccurate or conflated. Every reference must be checked during M0 before the proposal is circulated or submitted.*

- D. Boneh, M. Drijvers, G. Neven. "Compact Multi-Signatures for Smaller Blockchains." (Asiacrypt, ~2018.)
- Boneh, Lynn, Shacham. "Short Signatures from the Weil Pairing." (BLS signatures.)
- M. Drijvers et al. "Pixel: Multi-signatures for Consensus." (Forward-secure multi-signatures for PoS.)
- Works on weighted-threshold signatures and weighted-to-unweighted reductions (authors/venue to be identified in M0).
- Vector/polynomial commitment literature (e.g., KZG) and cryptographic accumulators, as accountability substrates.
- CometBFT / Tendermint consensus documentation and specification.
