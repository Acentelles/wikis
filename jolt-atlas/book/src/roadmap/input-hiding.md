# Hiding the Inputs with BlindFold

This page is the implementation spec for **input zero-knowledge** in Jolt Atlas: how to remove the plaintext input tensors from the verifier's view, and how to upgrade the sumcheck pipeline so the residual leakage from round messages and polynomial openings is also hidden. The mechanism is the BlindFold protocol used by Jolt's main repo (`code/jolt`), specialised to the Jolt Atlas DAG-of-sumchecks.

The discussion in [Hiding the Model](./model-hiding.md), and in particular the section "Do We Need NovaBlindFold for Input Zero-Knowledge?", concluded that:

1. **Moving inputs out of `ModelExecutionIO`** into committed witness polynomials is necessary but not sufficient: the activation trace is a function of the inputs, and sumcheck round messages still leak information at the field-element level.
2. **A formal ZK guarantee on the inputs** therefore requires a BlindFold-style commitment-then-deferred-verification mechanism, the same one Jolt uses to make its sumcheck proofs zero-knowledge.

This section gives the concrete implementation plan, grounded in the current `jolt-atlas-core` codebase.

---

## What the Verifier Sees Today

The relevant entry points are in `jolt-atlas-core/src/onnx_proof/`:

```rust
// jolt-atlas-core/src/onnx_proof/mod.rs
pub fn prove(
    pp: &AtlasProverPreprocessing<F, PCS>,
    inputs: &[Tensor<i32>],
) -> (Self, ModelExecutionIO, Option<ProverDebugInfo<F, T>>) { ... }

pub fn verify(
    &self,
    pp: &AtlasVerifierPreprocessing<F, PCS>,
    io: &ModelExecutionIO,        // ← inputs visible to verifier
    _debug_info: Option<ProverDebugInfo<F, T>>,
) -> Result<(), ProofVerifyError> { ... }
```

`ModelExecutionIO` (defined in `atlas-onnx-tracer/src/model/trace.rs`) carries the plaintext input and output tensors:

```rust
pub struct ModelExecutionIO {
    pub inputs: Vec<Tensor<i32>>,
    pub outputs: Vec<Tensor<i32>>,
    pub input_indices: Vec<usize>,
    pub output_indices: Vec<usize>,
}
```

There is exactly one place where the verifier *uses* `io.inputs`: the `Input` operator's `verify` (`onnx_proof/ops/input.rs`):

```rust
fn verify(
    &self,
    node: &ComputationNode,
    verifier: &mut Verifier<'_, F, T>,
) -> Result<(), ProofVerifyError> {
    let (r_node_input, input_claim) = verifier.accumulator.get_node_output_opening(node.idx);
    let input = verifier.io.inputs[verifier
        .io
        .input_indices
        .iter()
        .position(|&idx| idx == node.idx)
        .unwrap()]
    .padded_next_power_of_two();
    let expected_claim = MultilinearPolynomial::from(input).evaluate(&r_node_input.r);
    if expected_claim != input_claim {
        return Err(ProofVerifyError::InvalidOpeningProof(
            "Input claim does not match expected claim".to_string(),
        ));
    }
    Ok(())
}
```

This is the **input claim binding** check: the prover writes a claim into the opening accumulator for the MLE of the input tensor at some sumcheck challenge point, and the verifier reconciles it by re-evaluating the MLE of the public input tensor. Removing access to `io.inputs` therefore breaks exactly one operator-verifier path, plus the obvious "the verifier no longer learns the input" property elsewhere.

Sumcheck round messages and polynomial openings carry residual leakage as discussed in [Security Considerations](../appendix/security.md): activation MLEs are committed (`NodeOutputRaD`, `GatherRa`, etc.), but their evaluations at challenge points are sent in the clear, and round polynomials of every sumcheck instance are sent as field-element coefficients. Both are functions of the input tensor.

---

## Goal

After the changes described below, the following should hold:

1. **`ONNXProof::verify` no longer takes a `&ModelExecutionIO` containing plaintext inputs.** The verifier receives only commitments and an output statement (or a commitment to the outputs, depending on whether output hiding is also wanted).
2. **The input MLEs are committed** before any Fiat–Shamir challenge is drawn, exactly the way the existing auxiliary witness polynomials are committed in `commit_witness_polynomials`.
3. **All sumcheck round polynomial coefficients become Pedersen commitments**, and the algebraic checks the verifier currently performs in the clear are deferred to a single small R1CS instance, folded against a random satisfying instance via Nova and proved with Spartan; this is BlindFold proper.
4. **All polynomial evaluations** ($W(r_y)$, weight openings if model hiding is also active, the input opening from step 2) become **ZK evaluation commitments** rather than field elements in the clear.

What we are *not* changing: the per-operator IOP, the `CommittedPolynomial`/`VirtualPolynomial` taxonomy, the evaluation-reduction chain, the batch-opening sumcheck, or HyperKZG. BlindFold sits at the boundary between "round messages produced by sumcheck" and "verifier checks them"; everything inside the IOP loop stays as it is today.

---

## Step 1: Move Inputs into Committed Witness Polynomials

This step alone gives "the verifier doesn't see the plaintext input tensors", but **without** a formal ZK guarantee; the same caveat that applies to today's weight-hiding plan applies here. If you stop after this step, you inherit the "single MLE evaluation at a uniformly random point is computationally indistinguishable from random under DL" heuristic. Step 2 (BlindFold proper) is what closes that gap.

### 1.1 New `CommittedPolynomial` variant

Extend `common::CommittedPolynomial` (in `common/src/lib.rs`) with an input variant keyed by the input node's index in the graph:

```rust
pub enum CommittedPolynomial {
    // ... existing variants ...
    /// MLE of the input tensor at the given `Operator::Input` node.
    InputTensor(usize),
}
```

Update the `CanonicalSerialize` / `CanonicalDeserialize` impl for `CommittedPolynomial` to include the new tag, mirroring how the other variants are tagged in `common/src/lib.rs`. The discriminant is part of the wire format, so add the new tag at the *end* of the existing list (do not reorder).

### 1.2 Witness generation for `InputTensor`

Extend the `WitnessGenerator<F> for CommittedPolynomial` impl in `jolt-atlas-core/src/onnx_proof/witness.rs` with a match arm for `InputTensor(node_idx)`. The witness is just the MLE built from the tensor that lives in `trace.node_outputs[&node_idx]` (which is where `Trace::store_inputs` puts the input tensor; see `atlas-onnx-tracer/src/model/execute.rs`):

```rust
CommittedPolynomial::InputTensor(node_idx) => {
    let tensor = trace.node_outputs.get(node_idx).expect("input node missing from trace");
    let padded = tensor.padded_next_power_of_two();
    MultilinearPolynomial::from(padded)
}
```

This is the only witness path for input tensors. From the prover's perspective, the input MLE is now indistinguishable from any other committed polynomial (`DivNodeQuotient`, `RsqrtNodeInv`, etc.).

### 1.3 Register input commitments in `commit_witness_polynomials`

`ONNXProof::commit_witness_polynomials` (in `onnx_proof/prover.rs`) builds the polynomial map by iterating over each node's `NodeCommittedPolynomials::get_committed_polynomials`. Extend the dispatch in `ops/input.rs` so that the `Input` operator advertises `CommittedPolynomial::InputTensor(node.idx)` as one of its committed polynomials. Concretely, the `Input` operator currently exposes no committed polynomials (it has nothing to commit because the verifier reconstructs the MLE from `io.inputs`). After the change it exposes exactly one: its own input tensor.

This automatically routes the input MLE through the existing infrastructure:

- It is committed by `Self::commit_to_polynomials`.
- Its commitment is appended to the Fiat–Shamir transcript (this is the binding step; it must happen before any challenge is drawn).
- It enters `prepare_for_sumcheck` and the batch-opening sumcheck on the same footing as every other committed polynomial.
- Its evaluation at the joint batch challenge point is opened as part of the joint HyperKZG opening.

No new commit-then-open machinery is needed; the existing commit-before-challenge ordering is preserved by reusing `commit_witness_polynomials`.

### 1.4 Replace the Input verifier check with an opening claim

`Input::verify` in `onnx_proof/ops/input.rs` is the only place that reads `io.inputs`. Replace its body with an opening-claim binding:

```rust
fn verify(
    &self,
    node: &ComputationNode,
    verifier: &mut Verifier<'_, F, T>,
) -> Result<(), ProofVerifyError> {
    // The prover wrote an opening claim for VirtualPolynomial::NodeOutput(node.idx)
    // (the "output" of an Input node is the input tensor itself) into the accumulator.
    // The corresponding committed polynomial is CommittedPolynomial::InputTensor(node.idx),
    // which the batch-opening sumcheck will reconcile against the HyperKZG commitment
    // included in `self.commitments`.
    //
    // The check that the input MLE evaluation is consistent with the committed
    // input polynomial therefore happens in `verify_reduced_openings`; there is
    // nothing operator-specific to do here.
    let _ = verifier.accumulator.get_node_output_opening(node.idx);
    Ok(())
}
```

The actual binding (that the value the operator IOP claimed for the input MLE matches the *committed* input polynomial) is what `prove_reduced_openings` / `verify_reduced_openings` already do for every other committed polynomial in the model. The input is now just one more entry in the joint HyperKZG opening.

### 1.5 Drop `io.inputs` from `ModelExecutionIO` (verifier-facing)

`ModelExecutionIO` is the public statement passed to `verify`. After step 1.4 the verifier never reads `io.inputs`, so it can be removed from the verifier-facing struct. The simplest concrete shape is to split:

```rust
// atlas-onnx-tracer/src/model/trace.rs
pub struct ModelExecutionIO {
    pub inputs: Vec<Tensor<i32>>,        // prover-only
    pub outputs: Vec<Tensor<i32>>,
    pub input_indices: Vec<usize>,
    pub output_indices: Vec<usize>,
}

pub struct PublicModelIO {
    pub outputs: Vec<Tensor<i32>>,       // or commitments to outputs, see below
    pub input_indices: Vec<usize>,
    pub output_indices: Vec<usize>,
}
```

`ONNXProof::prove` keeps producing the full `ModelExecutionIO` for prover bookkeeping but only emits `PublicModelIO` to the verifier. Update `ONNXProof::verify` to take `&PublicModelIO` instead, and update the `Verifier::new` constructor and the verifier's `io` field type accordingly. The output-claim check in `verify_output_claim` (in `verifier.rs`) is unchanged; it only reads `io.outputs[0]`.

If you also want to hide outputs, the symmetric construction applies: commit the output MLE, drop `outputs` from `PublicModelIO`, and replace `verify_output_claim` with an opening-claim binding. Outputs are typically smaller, so this is essentially free, but it is conceptually orthogonal to input hiding and is left as an extension.

### 1.6 What this gives you

After step 1 alone:

- The verifier no longer receives the plaintext input tensors. The proof carries one extra HyperKZG commitment per `Operator::Input` node and one extra entry in the joint opening.
- The Fiat–Shamir ordering invariant is preserved: input commitments are appended to the transcript inside `commit_witness_polynomials`, before any operator IOP runs.
- **Caveat (intentional):** sumcheck round polynomial coefficients are still sent in the clear, and `verify_reduced_openings` still receives a plaintext field-element evaluation of the input MLE. Under the discrete-log assumption a single MLE evaluation at a uniformly random point is computationally indistinguishable from a random field element (the same heuristic argument the [model-hiding](./model-hiding.md) plan uses for hidden weights), so step 1 is "good enough" if you accept that argument. If you do not, proceed to step 2.

---

## Step 2: BlindFold for Sumcheck Round Messages and Openings

Step 2 closes the residual leakage by integrating BlindFold. The protocol is described in detail on the Jolt book's [BlindFold page](https://jolt.a16zcrypto.com/how/blindfold.html); the version below is a translation to the Jolt Atlas DAG-of-sumchecks pipeline.

### 2.1 What changes in the sumcheck path

Today, every sumcheck instance run inside `iop()` (in `prover.rs`) sends round polynomial coefficients in the clear. With BlindFold, they are replaced by Pedersen commitments. Concretely, **per round** the prover:

1. Computes the batched univariate round polynomial as today.
2. Commits to its coefficients via Pedersen: $C_j = \sum_i c_{j,i} \cdot G_i + \rho_j \cdot H$.
3. Appends $C_j$ (not the coefficients) to the Fiat–Shamir transcript and derives the round challenge $r_j$ from the transcript.
4. Caches $(c_{j,i}, \rho_j, C_j)$ in the `ProverOpeningAccumulator` for later reuse when constructing the BlindFold witness.

The verifier receives only $C_j$. It derives the same $r_j$ (because it hashes the same commitments) but **cannot perform the algebraic check $g_j(0) + g_j(1) = \text{claim}_j$ or the chaining $g_j(r_j) = \text{claim}_{j+1}$** because it never sees the coefficients. Both checks are deferred to BlindFold's verifier R1CS.

This is exactly the change Jolt makes between its standard mode and `--features zk` mode: every sumcheck prover gains a `prove_zk` variant that commits round polys instead of sending them, and every sumcheck verifier becomes a no-op for the round-consistency checks (which are now baked into a separate R1CS instance).

In Jolt Atlas, the relevant call sites are:

- The sumcheck instances built inside `OperatorProver::prove(node, prover)` for each node, in `iop()` (`onnx_proof/prover.rs`).
- The evaluation-reduction sumchecks emitted as `EvalReductionProof<F>` per node.
- The batch-opening sumcheck inside `prove_reduced_openings`.

Each of these needs a `*_zk` variant that swaps cleartext round polynomials for Pedersen commitments. The `joltworks::subprotocols::sumcheck` module already exposes the `SumcheckInstanceProof` type that all of these flow through; the equivalent in Jolt's main repo is `subprotocols/sumcheck.rs`, which gained `prove_zk`/`verify_zk` paired with `BlindFoldProver` / `BlindFoldVerifier` in the `subprotocols/blindfold/` module.

### 2.2 `StageConfig` for the Jolt Atlas IOP

BlindFold's verifier R1CS is built from a list of `StageConfig`s, one per sumcheck "stage". A stage is a contiguous group of sumcheck instances that share a transcript section and produce claims feeding into a subsequent stage. In Jolt the stages are the eight phases listed on the BlindFold page (Spartan outer, instruction lookups, RAM read-write, etc.).

In Jolt Atlas the natural stage decomposition is:

| Stage | Sumcheck instances | Notes |
|---|---|---|
| 1 | Output-claim sampling + per-node execution sumchecks (one per node, in reverse topological order) | Each node currently runs through `OperatorProver::prove`; the per-stage config records the round count, polynomial degree, and the chain that links each node's output claim to the next node's input claims. |
| 2 | Per-node evaluation-reduction proofs (`EvalReductionProof`) | Linear in the number of nodes; shares the transcript state with stage 1. |
| 3 | Lookup sub-protocols emitted by neural-teleport / softmax / range-check operators (`RaOneHotChecks`, `RaHammingWeight`, `SoftmaxDivSumMax`, `SoftmaxExponentiationReadRaf`, …) | Already enumerated in `ProofType` (`onnx_proof/types.rs`); each proof type is a stage entry. |
| 4 | Batch-opening sumcheck inside `prove_reduced_openings` | Reduces all polynomial openings to a single point; exactly one stage. |
| 5 | HyperKZG joint opening | Not a sumcheck; contributes a ZK evaluation commitment (see 2.5). |

Each `StageConfig` records, per round:

- The polynomial **degree** of the round polynomial (e.g., 3 for the standard execution sumcheck, 4 for `Operator::Cube`).
- Whether the round is a **uni-skip** round (Jolt Atlas does not currently use uni-skip; flag set to `false`).
- The **input claim source**: typically the chained output of the previous sumcheck stage, expressed as a sum-of-products over `ValueSource::{Opening, Challenge, Constant}`. For the first stage this is the output-claim sampling at $\tau$.
- The **output claim sink**: the next-stage input claim or, for stage 5, the final HyperKZG evaluation claim.

The crucial invariant from the Jolt CLAUDE.md (the system reminder above) carries over verbatim: every claim derivation in the IOP must have a matching `InputClaimConstraint` / `OutputClaimConstraint` describing the same formula as a sum-of-products. **Any change to how a sumcheck's input or output claim is computed in `OperatorProver::prove` requires a matching update to that operator's BlindFold constraint.** This is the "claim/constraint synchronization" invariant that Jolt's `muldiv` e2e test catches; the equivalent test in Jolt Atlas would be a transformer end-to-end test under the new `zk` feature.

### 2.3 Verifier R1CS construction (Phase 2 in BlindFold terms)

Both prover and verifier deterministically construct the same `VerifierR1CS` from the `StageConfig`s plus a `BakedPublicInputs` bundle. The witness vector has the Hyrax grid layout described on the BlindFold page:

- **Coefficient rows**: one row per sumcheck round, holding that round polynomial's coefficients (zero-padded to $C$ columns). The round commitments cached in step 2.1 *are* the Pedersen commitments to these rows; no additional commitments are computed.
- **Non-coefficient rows**: hold next-round claims, Horner intermediates used to express $g(r_j)$, and the polynomial-evaluation values.

The constraints encoded per round are the standard sumcheck checks restated as R1CS:

- **Sum constraint**: $2c_0 + c_1 + \cdots + c_d = \text{claim}_j$.
- **Chain constraint**: $g(r_j) = \text{claim}_{j+1}$, computed via Horner with auxiliary variables.

Plus, at chain endpoints:

- **Final output binding**: the last round's evaluation matches a sum of polynomial evaluation variables (this is where the eval-reduction chain is "closed off" against the batch-opening claim).
- **Input claim binding**: at chain start, the initial claim is a sum-of-products over polynomial openings; for the per-node execution sumcheck this is exactly the output-claim formulae the operator currently computes inside `Operator::input_claim`.
- **PCS evaluation binding**: the joint HyperKZG opening's claimed evaluation matches the corresponding witness variable.

**Baked public inputs.** Fiat–Shamir-derived values (round challenges $r_j$, the output-claim challenge $\tau$, the batching coefficient $\gamma$ used in the eval-reduction RLC, etc.) are embedded directly into the R1CS matrix coefficients $A$, $B$, $C$ rather than as witness variables. Both prover and verifier hash the same Fiat–Shamir transcript, so they derive identical baked inputs and identical $A, B, C$ matrices. This keeps the witness vector minimal.

### 2.4 Nova folding + Spartan (Phases 3–5)

Once the verifier R1CS exists, the rest of BlindFold is unchanged from Jolt:

1. **Sample a random satisfying instance** $(Z_2, E_2)$ for the same R1CS.
2. **Compute the cross-term** $T = (AZ_1) \circ (BZ_2) + (AZ_2) \circ (BZ_1) - u_1 \cdot (CZ_2) - u_2 \cdot (CZ_1)$ and commit each row with Pedersen.
3. **Derive the folding challenge** $r$ from the transcript.
4. **Fold**: $Z' = Z_1 + r Z_2$, $E' = E_1 + r T + r^2 E_2$, $u' = u_1 + r u_2$. The folded instance satisfies relaxed R1CS $(A Z') \circ (B Z') = u' (C Z') + E'$.
5. **Spartan outer sumcheck** ($\log m$ rounds, degree 3) proves the folded relaxed R1CS is satisfied.
6. **Spartan inner sumcheck** ($\log |W|$ rounds, degree 2) reduces the witness contributions to a single $W(r_y)$.
7. **Hyrax-style openings** verify $W(r_y)$ and $E(r_x)$ against folded row commitments. Because the coefficient-row commitments are exactly the round-poly commitments from step 2.1, no extra group operations are needed for those rows.

The folded random instance acts as a one-time pad: knowing $Z'$ alone reveals nothing about $Z_1$. This is what gives the formal ZK guarantee on the original sumcheck witness, i.e. on the activation trace and, through it, on the inputs.

### 2.5 ZK evaluation commitments for HyperKZG

The HyperKZG opening at the end of `prove_reduced_openings` currently sends the polynomial evaluations as field elements (`sumcheck_claims: Vec<F>` in `ReducedOpeningProof`). Under BlindFold these become **ZK evaluation commitments**: instead of revealing $v = \tilde f(r)$ in the clear, the prover sends a Pedersen commitment $y_{\text{com}} = v \cdot G_0 + \rho \cdot H$ and proves consistency inside the HyperKZG opening proof.

The supporting changes that Jolt's BlindFold integration needed (`poly/commitment/dory/commitment_scheme.rs` adds `y_com`) translate to Jolt Atlas as a parallel change in `joltworks/src/poly/commitment/hyperkzg/`: extend the HyperKZG `Proof` type with a `y_com` field, modify `prove`/`verify` to bind it, and route the evaluation through the BlindFold witness so the verifier R1CS can constrain $y_{\text{com}}$ to equal the corresponding witness variable in $W$.

After this change, the input MLE evaluation that step 1 added to the joint opening is also a ZK evaluation commitment, closing the residual leakage that step 1 alone left open.

### 2.6 Where BlindFold sits in the prover pipeline

Jolt Atlas's `ONNXProof::prove` becomes:

```text
1. Trace + IO (unchanged)
2. commit_witness_polynomials                  // now also commits InputTensor MLEs
3. output_claim                                // unchanged; transcript appends τ
4. iop()                                        // each sumcheck uses prove_zk variant:
                                                //   - commits round polys with Pedersen
                                                //   - caches (coeffs, blinding, commitment)
                                                //     into ProverOpeningAccumulator
5. prove_reduced_openings                      // batch-opening sumcheck also prove_zk;
                                                // HyperKZG opening produces y_com instead of plaintext eval
6. (NEW) build VerifierR1CS from collected
   StageConfigs and BakedPublicInputs
7. (NEW) BlindFold: Nova fold + Spartan outer
   + Spartan inner + Hyrax openings
8. finalize_proof                              // emits ONNXProof with a BlindFoldProof
                                                // field instead of cleartext sumcheck claims
```

`ONNXProof::verify` mirrors this:

```text
1. populate_accumulator                        // unchanged
2. verify_output_claim                         // unchanged for outputs;
                                                //   no input check (Step 1.4 above)
3. verify_iop                                  // each operator's verify becomes a no-op
                                                // for round consistency; the only thing it
                                                // does is register InputClaimConstraint /
                                                // OutputClaimConstraint formulae for BlindFold
4. verify_reduced_openings                     // unchanged structurally;
                                                // but joint opening now binds y_com
5. (NEW) verify the BlindFoldProof             // reconstructs VerifierR1CS, absorbs random
                                                // instance, folds, runs Spartan outer/inner,
                                                // checks Hyrax openings
```

The verifier never sees the plaintext input tensors, never sees a sumcheck round polynomial coefficient, and never sees a polynomial evaluation in the clear; it only sees commitments, $y_{\text{com}}$ values, and the structured BlindFold proof.

---

## What the Verifier Learns

After Step 1 + Step 2, with input hiding (and inputs hidden as committed polynomials):

| Information | Visible to verifier? |
|---|---|
| Architecture (operators, shapes, connectivity) | Yes (unless [model architecture hiding](./model-hiding.md#hiding-the-architecture-via-recursive-verification) is also active) |
| Weight tensors | Yes today; **No** if model hiding is also active |
| **Plaintext input tensors** | **No** |
| **Plaintext output tensors** | Yes (unless symmetric output hiding from §1.5 is added) |
| Input MLE evaluation in the clear | **No**; replaced by Pedersen $y_{\text{com}}$ |
| Sumcheck round polynomial coefficients | **No**; replaced by Pedersen commitments |
| Activation tensor evaluations at challenge points | **No**; committed and bound through BlindFold's R1CS |
| HyperKZG polynomial evaluations | **No**; replaced by Pedersen $y_{\text{com}}$ |
| BlindFold proof artifacts (folded instance, Spartan outer/inner sumcheck transcript, Hyrax openings) | Yes, but they reveal nothing about $Z_1$ thanks to the random-instance fold |

This is a *formal* ZK guarantee on the inputs (and on the activation trace as a side benefit), not the heuristic argument of Step 1 alone.

---

## Per-Operator Impact

Most operators are unaffected by Step 1, because they already consume node-output openings rather than reading tensors directly. The relevant changes are:

| Operator | Step 1 (input hiding) | Step 2 (BlindFold) |
|---|---|---|
| `Input` | Replace `verify` body with an opening-claim binding (§1.4). Add `InputTensor(node.idx)` to `get_committed_polynomials`. | Sumcheck round polys committed; `input_claim_constraint()` added |
| `Constant` | No change | Same as model-hiding plan; weight commitments use `y_com` |
| `Einsum`, `Gather`, `Add` (with bias) | No change beyond what model hiding already requires | Round polys committed; constraint synchronization invariant applies |
| All other operators (`ReLU`, `Tanh`, `Erf`, `Sigmoid`, `Sin`, `Cos`, `Softmax`, `Div`, `Rsqrt`, `ScalarConstDiv`, …) | No change | Round polys committed; per-operator `input_claim_constraint()` / `output_claim_constraint()` formulae must match the existing `input_claim` / `output_claim` derivations |

The synchronization invariant deserves emphasis. Every operator that today implements `input_claim()` (typically as part of `OperatorProofTrait`) must additionally implement `input_claim_constraint()` returning the same sum-of-products as a `LinearCombination` over `ValueSource::{Opening, Challenge, Constant}`. The pairs must stay in lockstep; failing to update one when the other changes causes BlindFold R1CS unsatisfiability, which is silent until the recursive fold check fails. The recommended discipline (taken from Jolt's `CLAUDE.md`) is:

1. Never modify `input_claim()` without also modifying `input_claim_constraint()`.
2. Add a transformer end-to-end test under `--features zk` that exercises every operator in the test set, analogous to Jolt's `muldiv` test.
3. Decompose any value the verifier needs to reconstruct in non-ZK mode (e.g. for a debug build of the verifier) so that `input_claim()` recomposes to the same field element on both sides.

---

## Cargo Feature Gate

Match Jolt's structure: introduce a `zk` Cargo feature on `jolt-atlas-core` that selects the BlindFold path at compile time. Compile-time selection avoids a runtime `zk_mode` field in the prover and keeps the standard-mode hot path untouched.

| Aspect | Standard | `--features zk` |
|---|---|---|
| Sumcheck prover | `BatchedSumcheck::prove` (cleartext round polys) | `BatchedSumcheck::prove_zk` (Pedersen-committed) |
| `ONNXProof` field | `opening_claims: Claims<F>` | `blindfold_proof: BlindFoldProof` |
| `Input::verify` | Reads `io.inputs[i]`, evaluates MLE | No-op; binding via batch opening |
| `verify` signature | Takes `&ModelExecutionIO` | Takes `&PublicModelIO` (inputs removed) |
| HyperKZG opening | Plaintext `sumcheck_claims: Vec<F>` | `y_com` Pedersen commitments |

The verifier should detect mode from the proof at runtime, exactly as Jolt does (`stage1_sumcheck_proof.is_zk()` in Jolt). This avoids a verifier rebuild when switching modes.

---

## Implementation Path

In dependency order:

1. **`common`**: add `CommittedPolynomial::InputTensor(usize)` and update the CanonicalSerialize/Deserialize tag list.
2. **`onnx_proof::witness`**: add the `InputTensor` arm to `WitnessGenerator::generate_witness`.
3. **`onnx_proof::ops::input`**: extend `get_committed_polynomials` to include `InputTensor(node.idx)`; replace `Input::verify` body with the opening-claim binding.
4. **`atlas-onnx-tracer::model::trace`**: split `ModelExecutionIO` into prover-side (`ModelExecutionIO`) and verifier-side (`PublicModelIO`); update `Trace::io` to produce both.
5. **`onnx_proof::mod` / `verifier`**: change `ONNXProof::verify` to take `&PublicModelIO`; update `Verifier::new` and the `io` field type.
6. **e2e tests**: add a non-ZK test that exercises a model with hidden inputs (verifier never sees `inputs` field). This validates Step 1 in isolation.
7. **`joltworks::poly::commitment::pedersen`**: introduce a Pedersen commitment scheme for small vectors (round polynomials).
8. **`joltworks::subprotocols::sumcheck`**: add `BatchedSumcheck::prove_zk` / `verify_zk` that commit round polys instead of sending them, mirroring Jolt's `subprotocols/sumcheck.rs`.
9. **`joltworks::subprotocols::blindfold`** (new module): port the Jolt module structure (`r1cs.rs`, `protocol.rs`, `folding.rs`, `spartan.rs`, `relaxed_r1cs.rs`, `witness.rs`, `output_constraint.rs`, `layout.rs`).
10. **`joltworks::poly::commitment::hyperkzg`**: extend `Proof` and `prove`/`verify` with `y_com` ZK evaluation commitments.
11. **`onnx_proof::ops::*`**: implement `input_claim_constraint()` / `output_claim_constraint()` for each `OperatorProofTrait` impl, mirroring the existing `input_claim()` / `output_claim()` formulae.
12. **`onnx_proof::prover`**: thread `StageConfig` collection through `iop()` and `prove_reduced_openings`, build `VerifierR1CS`, run BlindFold, package as `BlindFoldProof`.
13. **`onnx_proof::verifier`**: detect mode from proof; verify `BlindFoldProof` after `verify_reduced_openings`.
14. **`zk` cargo feature**: cfg-gate the new prover/verifier paths and the `BlindFoldProof` field on `ONNXProof`.
15. **e2e tests**: add a transformer-scale test under `--features zk` that catches claim/constraint desynchronization (the Jolt Atlas analogue of the `muldiv` test).

Steps 1–6 give plaintext-input hiding with the same "computationally useless leakage" caveat that today's weight-hiding plan accepts. Steps 7–15 deliver the formal ZK guarantee.

---

## What Stays Out of Scope

- **Activation-only ZK** without input hiding. There is no good reason to do this: the BlindFold integration cost dominates the work, and once it is paid, hiding the inputs is a small additional change. Step 1 + Step 2 should be done together.
- **Hiding the architecture via BlindFold.** BlindFold hides the *witness* of the verifier R1CS, but the R1CS structure itself is built from `StageConfig`s that encode the model architecture. Hiding the architecture is a separate problem; see [model hiding via recursive verification](./model-hiding.md#hiding-the-architecture-via-recursive-verification).
- **Replacing HyperKZG with Dory.** Jolt's BlindFold integration uses Dory because Dory's evaluation commitments compose naturally with Pedersen `y_com`. Jolt Atlas can keep HyperKZG by implementing the analogous `y_com` extension on the HyperKZG opening proof (§2.5). If that proves more invasive than expected, switching the PCS to Dory is the alternative; it removes the trusted setup as a bonus.

---

## References

- [BlindFold](https://eprint.iacr.org/2025/2094), Kaviani, Setty (2025). The protocol Jolt Atlas would adopt verbatim, modulo PCS choice.
- [Jolt BlindFold documentation](https://jolt.a16zcrypto.com/how/blindfold.html): the reference implementation in Jolt's main repo (`code/jolt`), including the `BlindFoldProver` / `BlindFoldVerifier` API.
- [Hyrax](https://eprint.iacr.org/2017/1132.pdf), Wahby et al. Matrix-commitment-based polynomial evaluation; supplies the row-commitment trick used in BlindFold's openings.
- [Nova](https://eprint.iacr.org/2021/370), Kothapalli, Setty, Tzialla. The folding scheme that lets BlindFold hide the witness with a single random instance.
- [Spartan](https://eprint.iacr.org/2019/550), Setty. Sumcheck-based R1CS proving used to prove the folded relaxed R1CS.
- [Hiding the Model](./model-hiding.md): companion roadmap section. The "Do We Need NovaBlindFold for Input Zero-Knowledge?" subsection is the conceptual prelude to this page.
- [Security Considerations](../appendix/security.md): documents the existing "No Zero-Knowledge for Activation Values" caveat that BlindFold closes.
