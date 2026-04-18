# BlindFold Integration Plan

This page documents the plan for wiring the BlindFold subprotocol (already ported; see [BlindFold](./blindfold.md)) into the Jolt Atlas end-to-end proof pipeline. The goal is a working vertical slice: prove and verify a single operator (`Square`) under `--features zk`, exercising the full BlindFold stack on real inputs. Remaining operators are extended in follow-up PRs.

---

## Current State

The BlindFold subprotocol compiles behind `--features zk`, all 30 unit tests pass, and a latent `UniPoly::from_evals` truncation bug uncovered during the port has been fixed. What is **not** yet wired:

1. `SumcheckInstanceParams` `input_claim_constraint` / `output_claim_constraint` methods exist as `todo!()` defaults; no operator implements them.
2. `ONNXProof::prove` / `verify` have no `#[cfg(feature = "zk")]` branch; they always run the cleartext path.
3. `SumcheckInstanceProof<F, T>` is not yet a Clear/Zk enum.
4. HyperKZG has no `y_com` (committed-evaluation) field.
5. There is no end-to-end test that exercises `--features zk` on any model.

These form a dependency chain: nothing downstream works until at least one operator implements its claim constraint and the prove/verify pipeline has a ZK branch.

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

2. **`y_com` is load-bearing from the start.** In Jolt, `y_com` (committed evaluation) is baked into Dory. When `bind_opening_inputs_zk()` runs at stage 8, it absorbs a Pedersen commitment to the evaluation value instead of the raw scalar. Without `y_com`, the batch opening stage would reveal cleartext evaluations, breaking the ZK property. An earlier draft of this plan deferred `y_com`, but it must be part of the pilot. Atlas uses HyperKZG instead of Dory, so an optional `y_com: Option<G1Affine>` field and a `bind_opening_inputs_zk` path must be added to `HyperKZGProof`.

3. **UniSkip handling.** Jolt's `BlindFoldAccumulator` has a separate `uniskip_data: Vec<UniSkipStageData>` alongside `stage_data`. Square does not use uni-skip, so this is safe to defer.

---

## Implementation Steps

### Step 1: Implement Square's BlindFold Constraints

**Files:**
- `jolt-atlas-core/src/onnx_proof/ops/square.rs`

Implement the four cfg-gated trait methods on `SquareParams`'s `SumcheckInstanceParams` impl:

- `input_claim_constraint() -> InputClaimConstraint`
- `input_constraint_challenge_values(&dyn OpeningAccumulator<F>) -> Vec<F>`
- `output_claim_constraint() -> Option<OutputClaimConstraint>`
- `output_constraint_challenge_values(&[F::Challenge]) -> Vec<F>`

The input claim is a single node-output opening. The output constraint is `eq_eval * operand * operand`, expressed as a single `ProductTerm` over three `ValueSource::Opening` factors. Reuse `ProductTerm::product()` from `joltworks/src/subprotocols/blindfold/output_constraint.rs`.

### Step 2: Add `prove_zk` to the OperatorProver Trait

**Files:**
- `jolt-atlas-core/src/onnx_proof/ops/mod.rs` -- add a cfg-gated trait method `prove_zk(...)` with a `todo!()` default so other ops still compile.
- `jolt-atlas-core/src/onnx_proof/ops/square.rs` -- override the default, reusing `BatchedSumcheck::prove_zk` from `joltworks/src/subprotocols/sumcheck.rs`. The body is structurally identical to `prove` but routes the round polynomials through the Pedersen committer that `prove_zk` already provides.

### Step 3: Add `y_com` to HyperKZG

**Files:**
- `joltworks/src/poly/commitment/hyperkzg/mod.rs`

Add an optional `y_com: Option<G1Affine>` field to `HyperKZGProof`. Implement `bind_opening_inputs_zk()` which, under `#[cfg(feature = "zk")]`, commits the evaluation value via Pedersen ($y_\text{com} = v \cdot G_0 + \rho \cdot H$) and absorbs the commitment into the transcript instead of the raw evaluation scalar. The standard-mode `bind_opening_inputs()` is unchanged.

This is required for the ZK property: without it, the batch opening stage reveals cleartext polynomial evaluations.

### Step 4: Add a ZK Branch to ONNXProof prove/verify

**Files:**
- `jolt-atlas-core/src/onnx_proof/mod.rs` (prove orchestration)
- `jolt-atlas-core/src/onnx_proof/prover.rs` (IOP loop)
- `jolt-atlas-core/src/onnx_proof/verifier.rs` (verification)

**Prover (cfg-gated):** Inside the `iop()` loop, branch on `#[cfg(feature = "zk")]` to call `prove_zk` and accumulate `ZkStageData` into a `BlindFoldAccumulator`. After the IOP loop, call `BlindFoldProver::prove` and store the resulting `BlindFoldProof` in a new cfg-gated field on `ONNXProof`. The non-ZK path keeps its existing `Claims<F>` field.

**Verifier (runtime detection):** Following Jolt's pattern, the verifier detects ZK mode from the proof struct at runtime (e.g. checking whether `blindfold_proof` is present or whether sumcheck proofs carry commitments). This lets a single compiled verifier accept both proof types. The verifier calls `BlindFoldVerifier::verify` after the sumcheck verifier loop when ZK mode is detected.

### Step 5: ZK End-to-End Test on Square

**File:** `jolt-atlas-core/src/onnx_proof/ops/square.rs` (unit test) or `e2e_tests.rs`

Add `test_square_zk()` that mirrors `test_square()` but with the cfg gate, asserting the proof contains a `BlindFoldProof` and that verify returns `Ok`. This test plays the same role as Jolt's `muldiv` e2e test: any drift between `input_claim()` and `input_claim_constraint()` manifests as BlindFold R1CS unsatisfiability and fails this single test.

---

## Deferred to Follow-up PRs

- **Remaining 25 operator constraints.** After Square, `Add` and `Reshape`/`Slice` are the next simplest single-stage operators. Unary lookup ops (`Erf`, `Cos`, `Sin`, `Tanh`, `Sigmoid`, `Rsqrt`) each involve 4 sumcheck stages and should be grouped. Einsum (6 variants) and Softmax (5 sub-stages) need dedicated design.
- **`SumcheckInstanceProof` unification.** The pilot can use the parallel `ZkSumcheckProof` (already defined in `joltworks/src/subprotocols/sumcheck.rs`) and a cfg-gated field on `ONNXProof` to avoid touching every operator file. Unification into a Clear/Zk enum is a subsequent refactor.
- **UniSkip stage data.** Jolt's `BlindFoldAccumulator` separates uni-skip first-round data from regular stage data. Square does not use uni-skip, so this channel can be added when operators that use it are wired up.

---

## Operator Inventory

For reference, here are the operators grouped by sumcheck complexity:

**Single-stage, simplest (3)** -- pilot candidates:
`Square` (1 sumcheck, `eq * op^2`), `Reshape` (1 sumcheck, `input * selector`), `Slice` (1 sumcheck, `input * selector`)

**Single-stage, slightly more complex (3):**
`Add` (1 sumcheck, `eq * (left + right)`), `Concat` (1 sumcheck, sum over inputs), `Iff` (1 sumcheck, conditional)

**No sumcheck (2):**
`Broadcast`, `Moveaxis` (direct openings, no sumcheck proof)

**Multi-stage lookup ops (7)** -- 3-4 sumcheck stages each:
`Erf`, `Cos`, `Sin`, `Tanh`, `Sigmoid`, `Rsqrt`, `ReLU`

**Multi-stage with range checks (2):**
`Div`, `ScalarConstDiv`

**Multi-stage with advice (1):**
`Gather` (3 sumcheck stages)

**Einsum variants (6)** -- sum-of-products with 2-3 input polys, degree 3:
`bmk_bkn_mbn`, `bmk_kbn_mbn`, `mbk_bnk_bmn`, `mbk_nbk_bmn`, `k_nk_n`, `mk_kn_mn`

**Softmax sub-stages (5)** -- complex, multi-stage, degree 2-4:
`MaxIndicator`, `RecipMult`, `SatDiff`, `ExpSum`, `Exponentiation::Mult`

---

## Verification Plan

End-to-end:

```bash
cargo test -p jolt-atlas-core --features zk test_square_zk
```

If the input-claim and input-claim-constraint formulas drift, this test fails with `BlindFoldVerifyError::FinalClaimMismatch` or `OuterClaimMismatch`.

Regression:

```bash
cargo test -p joltworks --features zk          # BlindFold + UniPoly tests
cargo test -p joltworks                        # standard-mode tests
cargo test -p jolt-atlas-core test_square      # cleartext Square path
```

CI parity:

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets
```
