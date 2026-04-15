# BlindFold

This page documents the current state of the BlindFold zero-knowledge integration into Jolt Atlas. BlindFold is the protocol used by Jolt's main repo to make sumcheck proofs zero-knowledge: instead of sending cleartext round polynomial coefficients, the prover sends Pedersen commitments, and the sumcheck verification equations are encoded into a small R1CS circuit proved via Nova folding + Spartan.

The implementation is ported from Jolt (`jolt-core/src/subprotocols/blindfold/`) and lives entirely in `joltworks/`, behind the `zk` Cargo feature flag. Jolt Atlas compiles and passes all tests in both standard and ZK modes.

---

## What is Implemented

### Feature flag

The `zk` feature is declared in `joltworks/Cargo.toml`. All BlindFold code is gated behind `#[cfg(feature = "zk")]`, so the standard-mode prover and verifier are unaffected.

### Curve abstraction (`joltworks/src/curve.rs`)

BlindFold operates on elliptic curve group elements (Pedersen commitments, Hyrax row commitments, Nova folded instances). The module introduces:

- `JoltGroupElement` trait: scalar multiplication, MSM, serialization, random sampling.
- `JoltCurve` trait: associates G1, G2, GT types and provides pairing, MSM, and generator functions.
- `Bn254Curve`: concrete implementation for BN254, using `ark-bn254` and the arkworks MSM backend.

These traits are generic, but every integration point in Jolt Atlas uses `Bn254Curve` concretely, avoiding generic-parameter propagation through the existing codebase.

### Pedersen commitments (`joltworks/src/poly/commitment/pedersen.rs`)

`PedersenGenerators<C: JoltCurve>` commits a coefficient vector with a blinding factor in a single MSM:

$$C = \sum_i m_i \cdot G_i + r \cdot H$$

Pre-converted affine bases avoid per-commit projective-to-affine conversions. The module also includes Hyrax helpers (`pedersen::hyrax`) for row-committed polynomial evaluation: `combined_row`, `evaluate`, `combined_blinding`, and `HyraxOpeningProof`.

### BlindFold subprotocol (`joltworks/src/subprotocols/blindfold/`)

The full BlindFold protocol is implemented across nine files (~4,100 lines):

| File | Purpose |
|------|---------|
| `mod.rs` | Core types: `BlindFoldAccumulator`, `ZkStageData`, `StageConfig`, `BakedPublicInputs`, `HyraxParams`, R1CS primitives (`Variable`, `Term`, `LinearCombination`, `Constraint`) |
| `output_constraint.rs` | `ValueSource`, `ProductTerm`, `OutputClaimConstraint`/`InputClaimConstraint`, `SumOfProductsVisitor` trait. These describe how sumcheck claims are derived from polynomial openings and challenges as sum-of-products expressions. |
| `layout.rs` | `LayoutStep` enum and `compute_witness_layout()`. Determines the allocation order of witness variables in the Hyrax grid. |
| `witness.rs` | `BlindFoldWitness` with `assign()` and `assign_with_u()`. Populates the Z vector from `ZkStageData` collected during `prove_zk`. |
| `r1cs.rs` | `VerifierR1CS` and `VerifierR1CSBuilder`. Builds sparse A, B, C matrices encoding sumcheck round consistency, claim binding, and PCS evaluation constraints. |
| `relaxed_r1cs.rs` | `RelaxedR1CSInstance` and `RelaxedR1CSWitness`. Supports Nova-style folding where $AZ \circ BZ = u \cdot CZ + E$. |
| `folding.rs` | `compute_cross_term()`, `sample_random_satisfying_pair()`, `commit_cross_term_rows()`. The Nova folding step that hides the witness. |
| `spartan.rs` | Spartan outer sumcheck (degree 3, proves folded R1CS) and inner sumcheck (degree 2, reduces to witness evaluation). Implements Atlas's `SumcheckInstanceProver`/`Verifier` traits. |
| `protocol.rs` | `BlindFoldProver` and `BlindFoldVerifier`. Orchestrates: build R1CS, construct real instance, sample random instance, fold, Spartan outer + inner, Hyrax openings. |

### ZK sumcheck (`joltworks/src/subprotocols/sumcheck.rs`)

`BatchedSumcheck::prove_zk` and `BatchedSumcheck::verify_zk` are added alongside the existing `prove`/`verify` methods. They are not modifications of the standard methods; they are new, cfg-gated functions.

`prove_zk` replaces cleartext transcript appends with Pedersen commitments:

```rust
// Standard mode: append raw coefficients to transcript
compressed_poly.append_to_transcript(transcript);

// ZK mode: append Pedersen commitment
let commitment = pedersen_gens.commit(&batched_univariate_poly.coeffs, &blinding);
transcript.append_serializable(&commitment);
```

After all rounds, `prove_zk` collects `InputClaimConstraint` and `OutputClaimConstraint` from each sumcheck instance and pushes a `ZkStageData` into the `BlindFoldAccumulator`.

`verify_zk` absorbs commitments from the proof, derives the same challenges, but skips the output claim equality check (BlindFold's R1CS handles it).

The proof type is `ZkSumcheckProof<F, C, T>`, a separate struct from the existing `SumcheckInstanceProof<F, T>`. Unifying them into a single enum (as Jolt does) is deferred to the integration step.

### Constraint trait methods (`sumcheck_verifier.rs`)

`SumcheckInstanceParams<F>` gains four cfg-gated methods:

```rust
#[cfg(feature = "zk")]
fn input_claim_constraint(&self) -> InputClaimConstraint;
fn input_constraint_challenge_values(&self, accumulator: &dyn OpeningAccumulator<F>) -> Vec<F>;
fn output_claim_constraint(&self) -> Option<OutputClaimConstraint>;
fn output_constraint_challenge_values(&self, sumcheck_challenges: &[F::Challenge]) -> Vec<F>;
```

All four have `todo!()` default implementations. They compile, but calling them panics until per-operator constraints are implemented.

### Opening accumulator changes (`poly/opening_proof.rs`)

`ProverOpeningAccumulator` and `VerifierOpeningAccumulator` gain cfg-gated `pending_claims` / `pending_claim_ids` fields with `take_pending_claims()` / `take_pending_claim_ids()` accessors. These are populated by `cache_openings` during ZK proving and consumed by `prove_zk` to commit output claims via Pedersen.

---

## What is Not Yet Implemented

The subprotocol machinery is complete, but it is not yet wired into the proof pipeline. The following items remain before an end-to-end ZK proof can be generated:

### 1. Per-operator `InputClaimConstraint` / `OutputClaimConstraint`

Every operator's sumcheck instance (Add, Einsum, Softmax, Div, Rsqrt, ReLU, Gather, etc.) must implement the four constraint methods on `SumcheckInstanceParams`. Each constraint describes, as a sum-of-products over `ValueSource::{Opening, Challenge, Constant}`, the same formula that `input_claim()` and `expected_output_claim()` compute today. The constraint must stay in lockstep with the claim computation; any mismatch causes BlindFold R1CS unsatisfiability.

This is the largest remaining task. There are approximately 20 distinct sumcheck instance types across the operator implementations in `jolt-atlas-core/src/onnx_proof/ops/`.

### 2. ONNXProof integration

`ONNXProof::prove` needs a ZK code path that:

1. Creates a `BlindFoldAccumulator`.
2. Calls `prove_zk` instead of `prove` for each sumcheck stage (the IOP loop, the eval-reduction sumchecks, and the batch-opening sumcheck).
3. Builds `StageConfig`s and `BakedPublicInputs` from the Fiat-Shamir transcript.
4. Calls `BlindFoldProver::prove` to produce a `BlindFoldProof`.
5. Stores the `BlindFoldProof` in the `ONNXProof` (replacing `opening_claims: Claims<F>` with a cfg-gated field).

`ONNXProof::verify` needs the mirror: detect ZK mode from the proof, call `verify_zk` for each stage, then `BlindFoldVerifier::verify`.

### 3. `SumcheckInstanceProof` unification

The existing `SumcheckInstanceProof<F, T>` struct (used in ~40 files across `jolt-atlas-core`) needs to become an enum `Clear | Zk` or have cfg-gated fields. This is deferred because it touches every operator file. A type alias strategy (`type AtlasSumcheckProof<F, T> = SumcheckProof<F, Bn254Curve, T>`) can limit the blast radius.

### 4. ZK evaluation commitments (y_com) on HyperKZG

The HyperKZG opening proof currently sends polynomial evaluations in the clear. Under BlindFold, these become Pedersen commitments ($y_\text{com} = v \cdot G_0 + \rho \cdot H$) whose consistency is checked inside the BlindFold R1CS. This requires extending `HyperKZGProof` with a `y_com` field and modifying `prove`/`verify` to bind it.

### 5. End-to-end test

Once per-operator constraints are in place, a transformer end-to-end test under `--features zk` (analogous to Jolt's `muldiv` test) will catch claim/constraint desynchronization.

---

## Protocol Summary

For reference, the BlindFold protocol as implemented proceeds in the following stages:

```
Prover                                        Verifier
------                                        --------
For each sumcheck stage:
  For each round:
    Compute batched univariate poly
    C_j = Pedersen(coeffs, blinding)  ------>  absorb C_j into transcript
    derive r_j from transcript                 derive r_j from transcript
  Commit output claims via Pedersen   ------>  absorb output claim commitments

Collect StageConfigs + BakedPublicInputs       Reconstruct same StageConfigs + baked inputs
Build VerifierR1CS (A, B, C matrices)          Build same VerifierR1CS

Assign BlindFoldWitness (Z vector)
Construct real RelaxedR1CSInstance
Sample random satisfying instance
Compute cross-term T
Commit T rows                         ------>  absorb T commitments
Derive folding challenge r                     Derive same r
Fold: Z' = Z_real + r * Z_random              Fold instance commitments

Spartan outer sumcheck (degree 3)     ------>  Verify outer sumcheck
Spartan inner sumcheck (degree 2)     ------>  Verify inner sumcheck

Hyrax opening: W(ry), E(rx)          ------>  Check against folded row commitments
```

The random satisfying instance acts as a one-time pad: the folded witness $Z'$ reveals nothing about the real witness $Z_\text{real}$.

---

## File Index

| File | Lines | Feature-gated? |
|------|------:|:---:|
| `joltworks/src/curve.rs` | 299 | No |
| `joltworks/src/poly/commitment/pedersen.rs` | 254 | No |
| `joltworks/src/subprotocols/blindfold/mod.rs` | 533 | Module gated |
| `blindfold/output_constraint.rs` | 391 | Module gated |
| `blindfold/layout.rs` | 166 | Module gated |
| `blindfold/witness.rs` | 614 | Module gated |
| `blindfold/r1cs.rs` | 765 | Module gated |
| `blindfold/relaxed_r1cs.rs` | 292 | Module gated |
| `blindfold/folding.rs` | 308 | Module gated |
| `blindfold/spartan.rs` | 489 | `#[cfg(feature = "zk")]` |
| `blindfold/protocol.rs` | 519 | `#[cfg(feature = "zk")]` |
| `subprotocols/sumcheck.rs` (additions) | ~270 | `#[cfg(feature = "zk")]` |
| `subprotocols/sumcheck_verifier.rs` (additions) | ~20 | `#[cfg(feature = "zk")]` |
| `subprotocols/sumcheck_prover.rs` (additions) | ~5 | No |
| `poly/opening_proof.rs` (additions) | ~30 | `#[cfg(feature = "zk")]` |
| **Total new code** | **~4,600** | |
