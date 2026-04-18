# BlindFold

This page documents the current state of the BlindFold zero-knowledge integration into Jolt Atlas. BlindFold is the protocol used by Jolt's main repo to make sumcheck proofs zero-knowledge: instead of sending cleartext round polynomial coefficients, the prover sends Pedersen commitments, and the sumcheck verification equations are encoded into a small R1CS circuit proved via Nova folding + Spartan.

The implementation is ported from Jolt (`jolt-core/src/subprotocols/blindfold/`) and lives entirely in `joltworks/`, behind the `zk` Cargo feature flag. Jolt Atlas compiles and passes the full test suite in both standard and ZK modes (164 lib tests with `zk`, 134 without), including the 22-test BlindFold port. Porting the suite exposed and allowed us to fix a latent `UniPoly` normalisation bug (see [Resolved issue: `UniPoly::from_evals` stripped trailing zeros](#resolved-issue-unipolyfrom_evals-stripped-trailing-zeros)).

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

## Resolved issue: `UniPoly::from_evals` stripped trailing zeros

**Status:** Fixed in `joltworks/src/poly/unipoly.rs`. All 30 BlindFold tests now pass. This section is kept as a record of the bug and the reasoning behind the fix.

After porting the full BlindFold test suite from Jolt into `joltworks`, four `protocol.rs` tests initially failed while the other 18 passed. All four failed with the same verifier error:

```text
Err(DegreeBoundExceeded { expected: 3, got: 1 })   // single-round tests
Err(DegreeBoundExceeded { expected: 3, got: 2 })   // multi-round test
```

Failing tests:

- `test_blindfold_protocol_completeness`
- `test_blindfold_protocol_multi_round`
- `test_blindfold_soundness_tampered_w_combined_row`
- `test_blindfold_soundness_tampered_e_combined_row`

Under the same test inputs, Jolt's upstream suite passes. The divergence is in `UniPoly`, not in any BlindFold file.

### Root cause

The Spartan outer sumcheck has degree bound 3 (`SPARTAN_DEGREE_BOUND = 3`). Each round the prover computes four evaluations $(p(0), p(1), p(2), p(3))$, interpolates them into a `UniPoly`, then compresses the polynomial for the transcript:

```rust
let poly = spartan_prover.compute_round_polynomial(claim);  // UniPoly
let compressed = poly.compress();                           // CompressedUniPoly
```

The verifier then strict-equality-checks the compressed length against the degree bound:

```rust
if compressed_poly.coeffs_except_linear_term.len() != SPARTAN_DEGREE_BOUND {
    return Err(BlindFoldVerifyError::DegreeBoundExceeded { .. });
}
```

`compress()` returns `coeffs_except_linear_term.len() == self.coeffs.len() - 1`, so a degree-3 polynomial yields length 3. The bug is that `coeffs.len()` is not always 4.

In Jolt Atlas, `UniPoly::from_coeff` normalises the coefficient vector:

```rust
// joltworks/src/poly/unipoly.rs:38
pub fn from_coeff(mut coeffs: Vec<F>) -> Self {
    while Some(&F::zero()) == coeffs.last() {
        coeffs.pop();
    }
    ...
}
```

And `from_evals` routes through this path:

```rust
// joltworks/src/poly/unipoly.rs:54
pub fn from_evals(evals: &[F]) -> Self {
    let coeffs = Self::vandermonde_interpolation(evals);
    Self::from_coeff(coeffs)      // <- strips trailing zeros
}
```

When the round polynomial happens to have a zero leading coefficient (for example, the degree-3 term vanishes on a particular witness), `from_coeff` drops it. `compress()` then emits a length-2 vector instead of length 3, and the verifier's strict equality rejects.

In Jolt's upstream `UniPoly`, `from_coeff` is the trivial constructor (it does not strip zeros), and `from_evals` uses hard-coded degree-2 / degree-3 interpolation routines (`from_evals_degree3`) that always produce a 4-element coefficient vector. Zero leading coefficients are preserved, so `compress()` always emits the expected length.

### Why the tests expose it but nothing else does

The BlindFold protocol tests run a full end-to-end prove-and-verify with the existing Spartan prover/verifier on small, structured witnesses where the top coefficient of the Spartan round polynomial can, and empirically does, happen to be zero. All other `joltworks` tests either:

- test `UniPoly` arithmetic without round-tripping through `compress()` + strict verifier checks, or
- do not exercise `BlindFoldVerifier::verify` at all.

The two soundness tests that pass (`tampered_az_r`, `wrong_proof_length`) tamper with fields that are rejected *before* the strict-length check runs.

### Fix options

1. **Port Jolt's `from_evals` fast paths** (`from_evals_degree2`, `from_evals_degree3`). Always produce a fixed-length coefficient vector. This is the minimal change and matches upstream semantics.
2. **Skip normalisation in `from_evals` only**: call the raw constructor with the vandermonde output, leaving `from_coeff`'s normalisation for other call sites. Less invasive but creates two constructors with subtly different semantics.
3. **Pad the compressed polynomial** on the prover side up to `SPARTAN_DEGREE_BOUND`. Masks the symptom; callers elsewhere may still depend on length.
4. **Relax the verifier check** to `len() <= SPARTAN_DEGREE_BOUND`. Weakens the degree-bound guarantee; not recommended.

Option 1 is the closest to upstream and preserves the invariant that a degree-`d` interpolation yields a length-`(d+1)` coefficient vector, which is what the rest of the BlindFold code assumes.

### Fix applied

Option 1. `from_evals_degree2` and `from_evals_degree3` were added to `joltworks/src/poly/unipoly.rs`, matching Jolt upstream byte-for-byte. `from_evals` now dispatches to these fast paths for the 3- and 4-evaluation cases (which cover both the Spartan outer sumcheck, degree 3, and the inner sumcheck, degree 2). The fallback `vandermonde_interpolation` + `from_coeff` path is retained for other evaluation lengths.

`from_coeff` was left unchanged: its zero-stripping behaviour is relied on elsewhere (degree queries, polynomial arithmetic). The invariant we now carry is "`from_evals` preserves `evals.len()` coefficients", which matches what every `compress()` caller assumes.

The `test_from_evals_edge_cases` unit test in `unipoly.rs` was updated to reflect the new invariant: four evaluations of $x^2$ now yield `[0, 0, 1, 0]`, not `[0, 0, 1]`.

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
