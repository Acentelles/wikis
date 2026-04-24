# BlindFold Integration Plan

This page documents the plan for wiring the BlindFold subprotocol (already ported; see [BlindFold](./blindfold.md)) into the Jolt Atlas end-to-end proof pipeline. The goal is a working vertical slice: prove and verify a single operator (`Square`) under `--features zk`, exercising the full BlindFold stack on real inputs. Remaining operators are extended in follow-up PRs.

---

## Current State

The BlindFold subprotocol compiles behind `--features zk`, all 30 unit tests pass, and a latent `UniPoly::from_evals` truncation bug uncovered during the port has been fixed.

### Completed

1. **Square's BlindFold constraints are implemented and tested** (`jolt-atlas-core/src/onnx_proof/ops/square.rs`). The macro-generated `SquareParams` was replaced with a manual `SumcheckInstanceParams` impl that includes the four cfg-gated constraint methods:
   - `input_claim_constraint()`: returns `InputClaimConstraint::default()` (the input claim is the node's reduced evaluation, baked as a constant in the BlindFold R1CS).
   - `input_constraint_challenge_values()`: returns `vec![]` (no challenges needed).
   - `output_claim_constraint()`: returns `Challenge(0) * Opening(operand) * Opening(operand)`, matching the verifier's `eq_eval * operand^2` formula.
   - `output_constraint_challenge_values()`: computes `eq_eval = EqPolynomial::mle(r_node_output, r_node_output_prime)` from the sumcheck challenges.
2. **Constraint consistency test** (`test_square_blindfold_constraint_consistency`, cfg-gated under `--features zk`). Creates synthetic accumulator state, evaluates both the actual claim methods and the constraint formulas with the same inputs, and asserts they produce identical values. This validates the critical synchronization invariant that BlindFold depends on.
3. **`zk` feature propagation**: `jolt-atlas-core/Cargo.toml` now declares `zk = ["joltworks/zk"]`, allowing `cargo test -p jolt-atlas-core --features zk`.

### Remaining

1. ~~`SumcheckInstanceParams` constraint methods exist as `todo!()` defaults; no operator implements them.~~ Square, Add, Reshape, and Slice now implement them. Other operators still use the `todo!()` defaults.
2. ~~There is no test that exercises `--features zk` with BlindFold on real sumcheck data.~~ Four ZK e2e tests (`test_square_zk`, `test_add_zk`, `test_reshape_zk`, `test_slice_zk`) run the full BlindFold protocol on real sumcheck data.
3. ~~`ONNXProof::prove` / `verify` have no ZK branch.~~ The `onnx_proof::zk` module provides standalone `prove_zk` / `verify_zk` functions that run the full pipeline in a single pass.
4. ~~Batching coefficient not handled.~~ `prove_zk` in `sumcheck.rs` now wraps each output constraint with `scale_by_new_challenge()` and appends the batching coefficient.
5. ~~`y_com` for batch opening.~~ Infrastructure added in `prove_zk` (commits joint evaluation claim via Pedersen when `poly_map` is non-empty). Currently a no-op for Square/Add/Reshape/Slice (no committed polynomials).
6. `SumcheckInstanceProof<F, T>` is not yet a Clear/Zk enum.
7. Multi-stage operators (Erf, Cos, Sin, Tanh, Sigmoid, Rsqrt, ReLU, Div, Gather, Einsum, Softmax) are not yet wired.
8. Transcript consistency: the ZK prover uses its own transcript, not chained with a standard-mode transcript.

### Implemented operators

| Operator | Constraint formula | Challenges |
|---|---|---|
| **Square** | `batching * eq_eval * operand * operand` | `eq_eval`, `batching_coeff` |
| **Add** | `batching * eq_eval * left + batching * eq_eval * right` | `eq_eval`, `batching_coeff` |
| **Reshape** | `batching * selector_claim * input` | `selector_claim`, `batching_coeff` |
| **Slice** | `batching * selector_claim * input` | `selector_claim`, `batching_coeff` |

All four use an empty `InputClaimConstraint` (baked constant from eval reduction). The batching coefficient is added automatically by `prove_zk` via `scale_by_new_challenge()`.

### ZK pipeline architecture

The `onnx_proof::zk` module (cfg-gated behind `--features zk`) provides:

- `prove_zk()`: single-pass flow that traces the model once, runs setup (commit, output claim), then for each node runs eval reduction + `BatchedSumcheck::prove_zk`. Builds `BlindFoldWitness` / `VerifierR1CS` from accumulated `ZkStageData`, then runs `BlindFoldProver::prove` (Nova folding + Spartan + Hyrax).
- `verify_zk()`: verifies the BlindFold proof.
- `run_zk_sumcheck()`: factored helper that drains pending claims, creates Pedersen generators, and calls `prove_zk` for a single sumcheck instance.
- `ZkProofBundle`: carries the `BlindFoldProof`, verifier input, generators, stage configs, baked inputs, eval reduction proofs, and commitments.

### What the ZK tests cover

Each `test_<op>_zk` test:

- Creates a minimal model (single operator + Input node)
- Calls `prove_zk()` which runs the full single-pass pipeline
- Calls `verify_zk()` which verifies the BlindFold proof
- Any drift between `input_claim()` / `expected_output_claim()` and their constraint counterparts causes BlindFold R1CS unsatisfiability

### What the ZK tests do NOT cover

- **Multi-operator graphs**: each test uses a single operator. A model combining Square + Add + Reshape has not been tested.
- **Multi-stage operators**: Erf, lookup-based activations, einsum, softmax, etc.
- **Transcript chaining**: the ZK transcript is independent. Production use would chain it with the IOP transcript.
- **Formal ZK property**: tests validate completeness (honest prover accepted), not zero-knowledge (no simulator argument).

### Bugs found and fixed during integration

- **`append_virtual` never populated `pending_claims`** (`joltworks/src/poly/opening_proof.rs`). In Jolt upstream, `append_virtual` pushes to `pending_claims` under `#[cfg(feature = "zk")]`. Atlas's version was missing this. Without it, `prove_zk` collected zero output claims and the BlindFold witness had empty opening values.
- **`#[cfg(test)]` gates too restrictive** for cross-crate access. `PedersenGenerators::deterministic`, `BlindFoldWitness::new`, `BakedPublicInputs::from_witness`, and `VerifierR1CSBuilder::new` were gated with `#[cfg(test)]`, which only applies when joltworks itself is compiled in test mode. Changed to `#[cfg(any(test, feature = "test-feature", feature = "zk"))]`.
- **Batching coefficient mismatch**: `prove_zk` scales sumcheck polynomials by a batching coefficient, but output constraints evaluate to unbatched values. Fixed by adding `OutputClaimConstraint::scale_by_new_challenge()` which wraps each term with an additional challenge factor, and appending the batching coefficient to the challenge values.

### Performance (release, Square n=65536)

| | Standard | ZK | Overhead |
|---|---|---|---|
| **Prove** | 3.2ms | 24.3ms | 7.6x |
| **Verify** | 1.0ms | 3.3ms | 3.3x |

The overhead is dominated by curve operations (Pedersen commits, Hyrax row commitments, Nova folding, Spartan) on a small workload. At n=1M, prove overhead drops to 2.9x and verify is actually faster (3.1ms vs 7.8ms) because `verify_zk` only runs the tiny BlindFold verifier.

---

## Strategy: Vertical Slice over Horizontal Coverage

The vertical-slice approach is preferred over implementing all 26 operators' constraints first. The slice catches integration bugs early on a small tensor instead of a transformer; the remaining operators are then mechanical.

---

## Why Square as the Pilot Operator

An initial analysis suggested Erf as the pilot, but closer inspection revealed Erf runs **4 separate sumcheck stages** (neural teleportation division, lookup with gamma-folded claims, range checks, and one-hot checks), each with its own `SumcheckInstanceParams` implementation. This makes it far too complex for a first integration.

After auditing all operators for sumcheck complexity, **Square** is the simplest:

- **Exactly 1 sumcheck stage**: a single `Sumcheck::prove` call via the `impl_standard_sumcheck_proof_api!` macro.
- **1 SumcheckInstanceParams type**: just `SquareParams<F>`.
- **Simplest input_claim**: a single opening value (`accumulator.get_node_output_opening(node.idx).1`).
- **Simplest expected_output_claim**: `eq_eval * operand_claim * operand_claim`, a product of 3 polynomial evaluations with no challenge mixing.
- **Already has a unit test**: `test_square()` in `ops/square.rs`.
- **No advice polynomials, no range checks, no multi-stage reductions, no lookup tables**.

Runners-up (`Reshape`, `Slice`) were nearly as simple but involve selector polynomial construction. `Add` is also single-stage but combines two operands.

---

## Comparison with Jolt Upstream

**What matches Jolt:**

- **Compile-time cfg-gating on the prover** (`#[cfg(feature = "zk")]`). Jolt's prover selects `BatchedSumcheck::prove_zk` vs `prove` at compile time. Our plan uses the same branching.
- **BlindFold invoked at the very end.** Jolt's `BlindFoldAccumulator` collects `ZkStageData` throughout all sumcheck stages, then `BlindFoldProver::prove` runs once after stage 8. Not incremental. Our plan matches.
- **All constraints are hand-written** per sumcheck instance (23 in Jolt). No macro or code-generation. Our plan to hand-write Square's constraint and use it as a template is consistent.
- **`BlindFoldProof` stored in a cfg-gated field** on the proof struct (`JoltProof` has `#[cfg(feature = "zk")] blindfold_proof` vs `#[cfg(not(feature = "zk"))] opening_claims`). Our plan proposes the same for `ONNXProof`.

**Where this plan diverges from Jolt (and corrections applied):**

1. **Verifier uses runtime detection, not cfg-gating.** In Jolt, the prover is cfg-gated but the verifier detects ZK mode at runtime via `proof.stage1_sumcheck_proof.is_zk()`. This lets the verifier accept both proof types without recompilation. Our plan adopts this pattern on the verifier side.

2. **Evaluation secrecy in the PCS is load-bearing from the start.** In Jolt, Dory's `y_com` commits the evaluation scalar so `bind_opening_inputs_zk()` absorbs a group element instead of a raw scalar. Atlas uses HyperKZG instead of Dory, which presents a different challenge (see [HyperKZG evaluation secrecy](#hyperkzg-evaluation-secrecy-analysis) below).

3. **UniSkip handling.** Jolt's `BlindFoldAccumulator` has a separate `uniskip_data: Vec<UniSkipStageData>` alongside `stage_data`. Square does not use uni-skip, so this is safe to defer.

---

## Implementation Steps (completed)

### Step 1: Implement Square's BlindFold Constraints (done)

**Files:**
- `jolt-atlas-core/src/onnx_proof/ops/square.rs`

Replaced `impl_standard_params!(SquareParams, 3)` with a manual `SumcheckInstanceParams` impl that includes the four cfg-gated constraint methods:

- `input_claim_constraint()`: returns `InputClaimConstraint::default()`. The input claim is the node's reduced evaluation from eval reduction, baked as a constant in the BlindFold R1CS (no witness variable, no constraint).
- `input_constraint_challenge_values()`: returns `vec![]`.
- `output_claim_constraint()`: returns `ProductTerm::scaled(Challenge(0), [Opening(operand), Opening(operand)])`. This encodes the verifier's `eq_eval * operand^2` formula as a sum-of-products expression. The operand opening appears twice (squared). `prove_zk` then wraps this with `scale_by_new_challenge()` to add the batching coefficient, producing the final constraint: `batching_coeff * eq_eval * operand * operand`.
- `output_constraint_challenge_values()`: computes `eq_eval = EqPolynomial::mle(r_node_output, r_node_output_prime)` from the sumcheck challenges. The batching coefficient is appended automatically by `prove_zk`.

The `operand_opening_id()` helper (cfg-gated) returns `OpeningId::Virtual(NodeOutput(inputs[0]), NodeExecution(node.idx))`, matching the opening registered by `cache_openings`.

### Step 2: ZK prove/verify pipeline (done)

Rather than adding `prove_zk` to the `OperatorProofTrait` (which would require threading curve type parameters through the generic trait), a standalone `onnx_proof::zk` module was created with:

- `run_zk_sumcheck()`: factored helper that drains `pending_claims`, creates Pedersen generators, wraps the sumcheck prover in a single-element vector, and calls `BatchedSumcheck::prove_zk`. Pushes a `StageConfig` to the configs list.
- `prove_zk()`: single-pass entry point. Traces the model once, runs setup (commit, output claim), then for each node: eval reduction + operator dispatch to `run_zk_sumcheck`. Supported operators are dispatched by `match &node.operator`; unsupported operators panic.
- `verify_zk()`: rebuilds the `VerifierR1CS` from the `ZkProofBundle`'s stage configs and baked inputs, then calls `BlindFoldVerifier::verify`.

This approach avoids modifying `ONNXProof::prove` / `verify` or the `OperatorProofTrait`. The ZK flow is entirely parallel to the standard flow.

### Step 3: `y_com` transcript binding (done)

Infrastructure added in `prove_zk`: when `poly_map` is non-empty, replays the batch opening reduction, commits the joint claim via Pedersen, and appends `y_com` to the transcript. Currently a no-op for Square/Add/Reshape/Slice (no committed polynomials). Activates when operators with committed polynomials are added.

### Step 4: Batching coefficient fix (done)

`prove_zk` in `sumcheck.rs` now wraps each output constraint with `OutputClaimConstraint::scale_by_new_challenge()` (new method) and appends the batching coefficient as the final challenge value. This ensures the R1CS checks `final_claim == batching_coeff * raw_constraint_output`.

### Step 5: ZK e2e tests (done)

- `test_square_zk` in `e2e_tests.rs`: Square-only model, 16 elements.
- `test_square_blindfold_e2e` in `square.rs`: standalone BlindFold test that manually constructs the R1CS and runs the full Nova + Spartan + Hyrax pipeline.
- `test_square_blindfold_constraint_consistency` in `square.rs`: validates constraint formulas match actual claim computations with synthetic accumulator data.
- `test_add_zk`, `test_reshape_zk`, `test_slice_zk` in `e2e_tests.rs`: same pattern for the other three operators.

### Step 6: Add, Reshape, Slice constraints (done)

Same pattern as Square:
- **Add** (`add.rs`): manual `AddParams` replacing `impl_standard_params!`. Output constraint: `Challenge(0) * Opening(left) + Challenge(0) * Opening(right)` where `Challenge(0) = eq_eval`.
- **Reshape** (`reshape.rs`): constraint methods added to existing `ReshapeSumcheckParams`. Output constraint: `Challenge(0) * Opening(input)` where `Challenge(0) = selector_claim` (deterministic from public shapes + `r_output`).
- **Slice** (`slice.rs`): same pattern as Reshape. Output constraint: `Challenge(0) * Opening(input)` where `Challenge(0) = selector_claim`.

All three wired into `prove_zk` via the operator `match` dispatch.

---

## HyperKZG Evaluation Secrecy Analysis

### Revised understanding (corrects earlier analysis)

An earlier version of this section concluded that HyperKZG's intermediate evaluation matrix `v` (evaluations at `{r, -r, r^2}`) leaks witness information that BlindFold cannot protect. This was overly conservative. After studying how Jolt handles evaluation secrecy with Dory, the picture is clearer.

### What Jolt actually does

Jolt hides only the **final evaluation scalar** (the joint RLC claim from the batch opening reduction), not intermediate PCS-internal evaluations. The flow:

1. `PCS::prove` (Dory in ZK mode) internally generates `y_com = eval * G + blinding * H`.
2. The prover calls `bind_opening_inputs_zk`, which appends `y_com` (a group element) to the transcript instead of the raw evaluation scalar.
3. `y_blinding` is passed to `BlindFoldAccumulator::set_opening_proof_data`.
4. BlindFold's R1CS constrains that the committed evaluation equals the expected claim.

Dory's internal protocol structure does not reveal additional evaluations. The `y_com` is used purely for Fiat-Shamir transcript binding. The actual ZK guarantee comes from BlindFold's R1CS constraints.

### Why HyperKZG's `v` matrix is acceptable

HyperKZG's `v[i][j]` values are evaluations of intermediate folded polynomials at random challenge points `{r, -r, r^2}` sampled inside the PCS protocol. Under honest-verifier ZK (which Fiat-Shamir provides):

- The challenges are derived from the transcript, so a simulator can choose them.
- The evaluations are at random points of high-degree polynomials; they do not reveal the polynomial coefficients.
- After RLC batching, individual witness polynomial evaluations cannot be recovered from the batched values.
- These are verification artifacts, not meaningful witness queries.

The `v` values do NOT need to be hidden. They play the same role as intermediate prover messages in any interactive proof: determined by the witness but not directly revealing it, and simulatable given control of the challenges.

### What does need hiding

Only the **final evaluation scalar** from the batch opening reduction sumcheck. This is the value that directly corresponds to a sumcheck output claim and gets appended to the Fiat-Shamir transcript. In standard mode, the raw scalar is appended. In ZK mode, a Pedersen commitment `y_com` should be appended instead.

### Implementation

The fix is at the protocol layer (in `prove_reduced_openings` / the ZK prove flow), not inside HyperKZG:

1. After the batch opening reduction sumcheck produces the joint evaluation claim, commit it: `y_com = claim * G + blinding * H`.
2. Append `y_com` to the transcript instead of the raw claim.
3. Pass `y_blinding` to `BlindFoldAccumulator::set_opening_proof_data`.
4. On the verify side, read `y_com` from the proof, append it to the transcript, and let BlindFold's R1CS verify consistency.

No changes to `HyperKZGProof`, `kzg_open_batch`, or `kzg_verify_batch` are needed. Estimated ~30 lines of changes in the ZK prove/verify flow.

---

## Deferred to Follow-up PRs

- **Remaining ~22 operator constraints.** Next candidates: `Concat` (1 sumcheck, sum over inputs), `Iff` (1 sumcheck, conditional). Then multi-stage operators: unary lookup ops (`Erf`, `Cos`, `Sin`, `Tanh`, `Sigmoid`, `Rsqrt`, each 4 stages), einsum (6 variants), softmax (5 sub-stages).
- **Multi-operator graph tests.** Current tests use single-operator models. A model combining e.g. Add + Reshape + Square has not been tested, though the `prove_zk` flow iterates all nodes and should work.
- **`SumcheckInstanceProof` unification.** The ZK pipeline uses a parallel `ZkSumcheckProof` type and standalone `prove_zk` / `verify_zk`. Unifying into the main `ONNXProof::prove` / `verify` path requires an enum or cfg-gated fields on `SumcheckInstanceProof`.
- **Transcript chaining.** The ZK prover creates its own transcript. In production the transcripts must be unified.
- **UniSkip stage data.** Jolt's `BlindFoldAccumulator` separates uni-skip first-round data from regular stage data. None of the currently implemented operators use uni-skip.

---

## Operator Inventory

| Status | Operators |
|---|---|
| **Done** | Square, Add, Reshape, Slice |
| **No sumcheck (handled)** | Input, Identity, Broadcast, MoveAxis, Constant |
| **Single-stage, next** | Concat, Iff, Neg, Sub, Mul, Cube, And, IsNan, Sum |
| **Multi-stage lookup** | Erf, Cos, Sin, Tanh, Sigmoid, Rsqrt, ReLU |
| **Multi-stage division** | Div, ScalarConstDiv |
| **Multi-stage other** | Gather (3 stages) |
| **Einsum** | 6 variants (degree 3, 2-3 input polys) |
| **Softmax** | 5 sub-stages (degree 2-4) |

---

## Verification Plan

End-to-end (all ZK tests):

```bash
cargo test -p jolt-atlas-core --features zk "_zk"
```

Standard regression:

```bash
cargo test -p joltworks --features zk          # BlindFold + UniPoly tests
cargo test -p joltworks                        # standard-mode tests
cargo test -p jolt-atlas-core test_square      # cleartext Square path
cargo test -p jolt-atlas-core test_add         # cleartext Add path
cargo test -p jolt-atlas-core test_reshape     # cleartext Reshape path
cargo test -p jolt-atlas-core test_slice       # cleartext Slice path
```

CI parity:

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets
```
