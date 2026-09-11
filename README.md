# Implementation Plan.

## Succinct Accountable Stake-Weighted Multi-Signatures for PoS Finality

Companion to the research proposal. This plan covers the full path — cryptographic primitive → consensus integration → evaluation. It assumes the stack you chose (**Rust primitive + FFI to Go CometBFT**) as the default, and flags at each step where switching to **all-Go** simplifies the work.

---

### 0. Read this first: calibration and honest constraints

This project stacks four independently hard things: originating a sound cryptographic primitive, implementing crypto correctly, learning Rust and Go from zero, and bridging them with FFI. That is ambitious for a solo effort, and the plan is structured to protect you from the two ways it most commonly fails.

- **Two hard gates, unchanged from the proposal.** (M0) Do not design the primitive until the literature sweep confirms the novelty gap. (M2) Do not harden or optimize any code until an external cryptographer has reviewed the scheme and proof. Everything below is downstream of M0.
- **Validate the math before the systems code.** The single most important sequencing decision here is Phase 1: build a *throwaway* high-level prototype (Python/SageMath) that proves the scheme is correct and its soundness properties hold, *before* you write a line of Rust. Discovering an unsound scheme after three months of Rust+FFI is the failure mode to avoid.
- **Strong recommendation: reconsider all-Go.** You are learning both languages from scratch. CometBFT is Go, so you must learn Go regardless. `gnark-crypto` provides BLS12-381 pairings and KZG polynomial commitments — everything the primitive needs. Going all-Go deletes Phase 3 (FFI) entirely and halves the language-learning load, for the same research result. The plan keeps your Rust+FFI choice as default but marks the all-Go branch throughout. If you take one piece of advice from this document, take this one.
- **Realistic horizon.** From your stated starting point (no Go, no Rust, some-not-deep crypto, solo), plan for roughly 12–18 months to a submittable result — the bulk of the first quarter is skills, not research. All-Go shaves an estimated 2–4 months off by removing FFI and a second language.

### 1. Phase 0 — Foundations and skills ramp (est. 8–14 weeks)

Nothing project-specific is built here; the goal is a working toolchain and enough fluency to start.

**Go (required for CometBFT regardless of stack).**
- Work through the Go Tour and "Effective Go"; build 2–3 small CLI tools to internalize goroutines, interfaces, error handling, and `cgo` basics.
- Target competence: can read and modify an unfamiliar Go package with tests.

**Rust (skip if you switch to all-Go).**
- Work through "The Rust Programming Language" (the book) chapters 1–13; do the `rustlings` exercises. Rust's ownership model is the main hurdle and is worth over-investing in early.
- Target competence: can write a small crate with unit tests and use an external dependency.

**Cryptography libraries, hands-on (both stacks).**
- Read a short primer on bilinear pairings and BLS signatures (concept level — you do not need to build pairings, only use them).
- Build a *toy* working example: aggregate three BLS signatures and verify them (Rust: `blst`; Go: `gnark-crypto` or `blst` bindings).
- Build a *toy* KZG polynomial commitment: commit to a vector, open one position, verify the opening (Rust: `arkworks` poly-commit; Go: `gnark-crypto` KZG).

**Environment.**
- Install `rustup` (if Rust) and the Go toolchain; set up an editor with language servers; initialize a Git monorepo with CI (GitHub Actions) running `go test` and `cargo test` from day one.

**Phase 0 deliverable:** a repo with a working toolchain and two toy programs (BLS aggregate-verify, KZG commit-open) passing in CI. This proves the environment and the minimum understanding before real work starts.

### 2. Phase 1 — Scheme design and math validation (est. 6–10 weeks, runs with M1/M2)

This is where the proposal's §4.1 design direction becomes a concrete, testable specification, in a high-level language chosen for clarity, not speed.

- **Write the specification.** Pin down key generation, the stake-weighted aggregation of public keys, the weighted-threshold verification equation, and the succinct accountability mechanism (the commitment to the stake-ordered validator set and the sub-linear non-signer proof). Define the security games (unforgeability, weighted-threshold soundness, accountability soundness).
- **Prototype in Python or SageMath** using a pairing library (`py_ecc`, `petrelic`, or Sage's pairing support). Implement the full flow end to end.
- **Test the properties empirically:** valid aggregates verify; forged aggregates and out-of-threshold signer sets are rejected; non-signer proofs cannot falsely implicate a signer or exonerate a non-signer. Produce **test vectors** here — they become the ground truth for the Rust/Go implementations.
- **M2 checkpoint lands here.** Have a cryptographer review the spec, the security games, and the reduction *at this stage*, on readable prototype code, before you optimize anything.

**Phase 1 deliverable:** a written spec, a high-level reference prototype, a set of test vectors, and an external review sign-off (or a list of required fixes).

### 3. Phase 2 — Production primitive in Rust (est. 8–12 weeks)

Re-implement the validated scheme as a performant, well-tested Rust crate. *(All-Go branch: implement this as a Go package using `gnark-crypto`; skip Phase 3 entirely.)*

- **Crate layout:** modules for `keys`, `aggregate`, `weighted_threshold`, `accountability` (the KZG/commitment layer), and `serialization`.
- **Libraries:** `blst` for BLS operations (fast, audited); `arkworks` (`ark-bls12-381`, `ark-poly-commit`) for the polynomial-commitment layer. Keep serialization deterministic and versioned.
- **Security hygiene** (as far as a research prototype allows, and documented honestly as such): rogue-key defense via proofs-of-possession or message augmentation; explicit domain separation tags; deterministic canonical encoding; reject non-canonical points. Note in the write-up that full constant-time/side-channel hardening is out of scope for a research artifact.
- **Testing:** unit tests per module; property-based tests (`proptest`); cross-validation against the Phase 1 test vectors; negative tests (forgery and false-accountability attempts must fail).
- **Benchmarks:** `criterion` suite reporting sign, aggregate, verify, and accountability-proof size and time as functions of validator count `n` and stake-distribution skew.
- **FFI surface:** expose a C ABI (`#[no_mangle] extern "C"` functions over byte-slice inputs/outputs), generate headers with `cbindgen`.

**Phase 2 deliverable:** a tested Rust crate, a microbenchmark suite, and a C-compatible library artifact.

### 4. Phase 3 — Rust↔Go FFI bridge (est. 4–8 weeks) — *skip entirely if all-Go*

The most error-prone integration step, and the reason all-Go is recommended.

- **Build:** compile the Rust crate to a static or dynamic library; link it from Go via `cgo`.
- **Wrapper package:** a Go package that marshals keys/signatures/proofs across the boundary as `[]byte`, defines clear **memory-ownership rules** (which side allocates and frees), and translates Rust error codes into Go errors.
- **Boundary testing:** round-trip tests (Go → Rust → Go) for every operation; fuzz the boundary with malformed inputs; verify no leaks under repeated calls.
- **Build-system notes:** document the `cgo` link flags and the cross-compilation story; ensure CI builds the combined artifact.

**Phase 3 deliverable:** a Go module that exposes the primitive as ordinary Go functions, fully tested across the boundary.

> **All-Go simplification:** Phases 2 and 3 collapse into a single Go package built on `gnark-crypto`, imported directly by CometBFT. No C ABI, no `cgo`, no memory-ownership hazards, no dual-language CI. Given your starting point, this is the lower-risk path to the same result.

### 5. Phase 4 — CometBFT integration (est. 10–14 weeks)

Fork an existing PoS BFT system and replace its vote path with the primitive.

- **Study the codebase first.** Map CometBFT's consensus state machine, its prevote/precommit vote types, the `Commit`/`CommitSig` structures, the validator set with per-validator voting power (stake), and the signature-verification path. Write notes; this understanding is a prerequisite, not a side effect.
- **Fork strategy:** fork at a pinned CometBFT release; put the new logic behind a feature flag so the native per-signature path remains available as the evaluation baseline.
- **Insertion point:** at the precommit→commit step, replace collection of individual signatures with the aggregated multi-signature; extend the commit/certificate structure to carry the aggregate plus the accountability data; route verification through the primitive's weighted-threshold check while preserving the stake-weighted two-thirds rule.
- **Accountability wiring:** connect the primitive's non-signer proofs to participation accounting. For a research prototype it is acceptable to expose the recovered signer set to the application layer or a metrics sink to *demonstrate* accountability is preserved, rather than fully reimplementing reward/slashing.
- **Testing:** unit tests on modified components; a local Docker Compose testnet with N validators; fault injection (take validators offline and confirm the accountability layer identifies them); safety and liveness smoke tests across restarts.

**Phase 4 deliverable:** a forked CometBFT whose finality path uses the aggregated, accountable, stake-weighted vote, plus a small runnable demo chain.

### 6. Phase 5 — Evaluation harness and experiments (est. 6–10 weeks)

Produce the headline trade-off result.

- **Testbed:** a cluster of cloud VMs or a local machine with a network emulator. Be realistic about scale — 10k *real* validator processes is expensive; plan to co-locate many validators per VM and/or emulate the network, and document the method and its limits.
- **Independent variables:** validator-set size (e.g., 100 / 1k / 10k) and stake-distribution skew (uniform vs Zipf/exponential).
- **Baselines to compare against:** native CometBFT (individual signatures), plain BLS multisig without accountability, and accountable-subgroup-with-bitmap.
- **Metrics:** end-to-end finality latency, per-round message bytes, verification CPU (profiled), and certificate/header size.
- **Automation and rigor:** scripts to instantiate topologies, drive a transaction workload, and collect and plot results; multiple runs per configuration with reported variance. The deliverable plot is the compact-vs-accountability-vs-weighting trade-off curve.

**Phase 5 deliverable:** reproducible evaluation scripts, raw datasets, and the figures for the paper.

### 7. Phase 6 — Writing and artifact release (est. 6–8 weeks)

- Map results to the proposal's paper structure; write against the target venue's format.
- Prepare for artifact evaluation (common at S&P/CCS/USENIX): clean READMEs, reproducible build and run instructions, pinned dependencies.
- Open-source the crate/package and the CometBFT fork.

### 8. Consolidated milestone timeline (solo, from zero)

| Phase | Work | Rough duration | All-Go effect |
|---|---|---|---|
| 0 | Skills ramp + toolchain | 8–14 wks | ~half (one language) |
| M0 | Novelty sweep (gate) | 2–3 wks | unchanged |
| 1 | Scheme design + math validation (+M2 review) | 6–10 wks | unchanged |
| 2 | Rust primitive + benchmarks | 8–12 wks | merges with 3 |
| 3 | FFI bridge | 4–8 wks | **removed** |
| 4 | CometBFT integration | 10–14 wks | unchanged |
| 5 | Evaluation | 6–10 wks | unchanged |
| 6 | Writing + artifact | 6–8 wks | unchanged |

Phases overlap in places (e.g., M1 design proceeds while Go skills mature). Total realistic horizon: **~12–18 months**, toward the lower end if you switch to all-Go.

### 9. Tooling checklist

- Version control: Git monorepo, GitHub Actions CI running `cargo test` and `go test` from Phase 0.
- Rust: `rustup`, `blst`, `arkworks` (`ark-bls12-381`, `ark-poly-commit`), `criterion`, `proptest`, `cbindgen`. *(Omit if all-Go.)*
- Go: Go toolchain, `gnark-crypto`, `cgo` (Rust path only), CometBFT fork, Docker Compose for testnets.
- Prototype: Python (`py_ecc`/`petrelic`) or SageMath for Phase 1.
- Eval: a transaction-load generator, a metrics/profiling stack, and a plotting toolchain.

### 10. Risk register (implementation-specific)

- **Skills ramp overruns.** Most likely early risk. Mitigation: the toy deliverables in Phase 0 are hard checkpoints; if they slip badly, switch to all-Go to cut scope, and consider whether the solo constraint should be revisited.
- **Unsound scheme discovered late.** Mitigation: Phase 1 high-level validation and the M2 review both precede any optimized code.
- **FFI complexity stalls integration.** Mitigation: the all-Go branch removes this risk entirely; it is the recommended path given your starting point.
- **Accountability cost erases the bandwidth win.** Mitigation: Phase 5 depends only on Phase 2/3, so the trade-off is measurable before full integration; a negative/threshold result is still publishable if reported honestly.
- **10k-validator evaluation infeasible on budget.** Mitigation: co-location and network emulation, with the methodology and its limitations stated plainly.

---

*This is a plan to be executed only after M0 confirms the novelty gap. It does not authorize starting implementation; it defines what implementation will involve when you decide to begin.*
