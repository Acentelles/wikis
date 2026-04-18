# The Verifier

The Jolt Atlas verifier checks an `ONNXProof` against a public `ModelExecutionIO` (the model's inputs and outputs) and the verifier's preprocessing data. It is structurally a mirror of the prover: every step the prover took to *produce* a sumcheck round, opening claim, or commitment has a corresponding step on the verifier side that *consumes* it.

The public entry point is `ONNXProof::verify` in `jolt-atlas-core/src/onnx_proof/mod.rs`. The internal helpers live in `jolt-atlas-core/src/onnx_proof/verifier.rs`.

---

## What the Verifier Receives

```rust
pub fn verify(
    &self,
    pp: &AtlasVerifierPreprocessing<F, PCS>,
    io: &ModelExecutionIO,
    _debug_info: Option<ProverDebugInfo<F, T>>,
) -> Result<(), ProofVerifyError>
```

The verifier is given three things:

1. **The proof** (`&self: &ONNXProof`), which contains:
   - `opening_claims: Claims<F>`: every scalar claim the prover made about a polynomial at some opening point.
   - `proofs: BTreeMap<ProofId, SumcheckInstanceProof>`: one or more sumcheck proofs per node.
   - `commitments: Vec<PCS::Commitment>`: the HyperKZG commitments to the witness polynomials.
   - `eval_reduction_proofs: BTreeMap<usize, EvalReductionProof>`: line-restriction proofs that collapse multi-consumer claims into one (see [Evaluation Reduction](./eval-reduction.md)).
   - `reduced_opening_proof: Option<ReducedOpeningProof>`: the batched opening sumcheck plus the joint HyperKZG opening proof.

2. **The verifier preprocessing** (`pp`), which holds the public model structure (operator DAG, shapes, scales) and the HyperKZG verifier generators. Only the *structure* of the model is needed; none of the witness data.

3. **The public IO** (`io`), which contains the input tensors and the claimed output tensors. The verifier ultimately checks that the output the prover *committed* to matches the output the verifier *expects*.

The verifier returns `Ok(())` if the proof is valid and a `ProofVerifyError` otherwise.

---

## Verifier State

All in-memory state lives in a single `Verifier` struct:

```rust
pub struct Verifier<'a, F: JoltField, T: Transcript> {
    pub preprocessing: &'a AtlasSharedPreprocessing,
    pub accumulator:    VerifierOpeningAccumulator<F>,
    pub transcript:     T,
    pub proofs:         &'a BTreeMap<ProofId, SumcheckInstanceProof<F, T>>,
    pub io:             &'a ModelExecutionIO,
}
```

The two pieces that *change* during verification are the `accumulator` and the `transcript`. The accumulator is the verifier-side mirror of the `ProverOpeningAccumulator`; it stores opening claims at evaluation points and is the channel through which sumchecks pass scalar claims to one another. The transcript is the Fiat-Shamir transcript: every commitment, sumcheck round message, and operator output is appended to it, and every challenge the verifier draws is derived from its current state.

---

## The Four Phases of Verification

`verify` runs four phases in order. If any phase fails, verification aborts and the error is returned.

```
            ┌─────────────────────────────┐
            │ 1. populate_accumulator     │  load claims & commitments
            └─────────────┬───────────────┘
                          │
            ┌─────────────▼───────────────┐
            │ 2. verify_output_claim      │  pin output MLE to public IO
            └─────────────┬───────────────┘
                          │
            ┌─────────────▼───────────────┐
            │ 3. verify_iop               │  per-node sumcheck verification
            └─────────────┬───────────────┘     (reverse topological order)
                          │
            ┌─────────────▼───────────────┐
            │ 4. verify_reduced_openings  │  batch sumcheck + HyperKZG
            └─────────────────────────────┘
```

### Phase 1: `populate_accumulator`

Before any sumcheck is checked, the verifier seeds its accumulator with every opening claim that the prover stored in `opening_claims`. These are scalar values like *"polynomial $P$ evaluates to $v$ at some point"*; the point itself will be filled in later when the corresponding sumcheck runs.

In the same pass, every PCS commitment in `self.commitments` is appended to the transcript. This must happen before any challenge is drawn so that subsequent challenges are bound to the witness commitments, exactly mirroring what the prover did at the end of `commit_witness_polynomials`.

```rust
for (key, (_, claim)) in &self.opening_claims.0 {
    verifier.accumulator.openings
        .insert(*key, (OpeningPoint::default(), *claim));
}
for commitment in &self.commitments {
    verifier.transcript.append_serializable(commitment);
}
```

### Phase 2: `verify_output_claim`

This is the *seed* of the entire IOP. The verifier draws a random point $\tau$ from the transcript (whose dimension is the log of the padded output size) and computes the expected MLE evaluation of the public output tensor at $\tau$:

$$
\widehat{\text{Output}}(\tau) \;=\; \widetilde{\text{IO.outputs}[0]}(\tau)
$$

It then registers a `NodeOutput(output_idx)` virtual claim at $\tau$ in the accumulator. The prover must have produced *exactly* this same value during proving and stored it in `opening_claims`. The verifier reads back the prover's stored claim and checks they agree:

```rust
if expected_output_claim != output_claim {
    return Err(ProofVerifyError::InvalidOpeningProof(...));
}
```

If they match, the entire chain of sumchecks downstream of the output is anchored to a value the verifier *itself* computed from the public IO. From this point on, the verifier never has to "trust" the prover about the output; every subsequent claim is reduced back to this anchor.

### Phase 3: `verify_iop`, the per-node verification loop

```rust
for (_, node) in model.graph.nodes.iter().rev() {
    OperatorVerifier::verify(node, verifier, &self.eval_reduction_proofs)?;
}
```

The verifier walks the computation graph in **reverse topological order**, exactly the same order the prover used in [The IOP Loop](./iop-loop.md). At each node:

1. **Eval reduction.** If the node has multiple downstream consumers, several `NodeOutput(node.idx)` claims at *different* opening points will have been registered in the accumulator by previously visited downstream nodes. `NodeEvalReduction::verify` checks the line-restriction proof in `eval_reduction_proofs[node.idx]` and collapses these into a single claim at one point. (See [Evaluation Reduction](./eval-reduction.md).)

2. **Operator-specific verification.** The `OperatorVerifier::verify` dispatcher routes to the operator's `OperatorProofTrait::verify` implementation. This typically:
   - Reads the now-collapsed output claim from the accumulator.
   - Replays the sumcheck protocol against `proofs[ProofId::for_node(node.idx)]`, drawing the same challenges from the transcript that the prover did.
   - Checks the final round equation, the polynomial identity that defines the operator (e.g. for `Add`: $\widehat{\text{out}}(r) = \widehat{\text{lhs}}(r) + \widehat{\text{rhs}}(r)$).
   - Registers new opening claims on the operator's *inputs* at the sumcheck challenge point. These get consumed when the upstream nodes are visited later in the loop.

Concretely, an operator's verifier looks like this in the simple case:

```rust
fn verify(&self, node: &ComputationNode, verifier: &mut Verifier<F, T>) -> Result<(), _> {
    let output_claim = verifier.accumulator
        .get_virtual_polynomial_opening(NodeOutput(node.idx), ...);

    let (final_claim, r) = self.proofs[node.idx]
        .verify(output_claim, num_rounds, degree, &mut verifier.transcript)?;

    let lhs_eval = ...; // from final round message
    let rhs_eval = ...;
    if final_claim != self.identity(lhs_eval, rhs_eval) {
        return Err(ProofVerifyError::InvalidSumcheck);
    }

    verifier.accumulator.append_virtual(/* NodeOutput(lhs_idx) at r */);
    verifier.accumulator.append_virtual(/* NodeOutput(rhs_idx) at r */);
    Ok(())
}
```

The complex operators (Div, Rsqrt, Softmax, Neural Teleport, Einsum) consume *multiple* sumcheck proofs per node and check several identities, but the pattern is identical: read claims, replay sumcheck, check the round equation, register new claims on inputs.

By the time the loop terminates, every claim in the accumulator has been pushed all the way back to either a **committed polynomial** (a model weight, a quotient polynomial, a lookup address polynomial) or a **public input tensor**. The latter the verifier can evaluate itself and check directly. The former are settled in Phase 4.

### Phase 4: `verify_reduced_openings`

After the IOP loop, the accumulator contains a set of opening claims of the form *"committed polynomial $P_i$ evaluates to $v_i$ at point $r_i$"*. There may be dozens or hundreds of these, at potentially many different points. Verifying each one with a separate HyperKZG opening would be expensive.

The prover has already collapsed these into a single joint opening using the batch-opening sumcheck (see [Batched Opening and HyperKZG](./batch-opening.md)). The verifier now mirrors that process:

```rust
verifier.accumulator.prepare_for_sumcheck(&reduced_opening_proof.sumcheck_claims);

// 1. Verify the batching sumcheck.
let r_sumcheck = verifier.accumulator.verify_batch_opening_sumcheck(
    &reduced_opening_proof.sumcheck_proof,
    &mut verifier.transcript,
)?;

// 2. Finalize: derive gamma powers, the joint claim, and the joint point.
let verifier_state = verifier.accumulator.finalize_batch_opening_sumcheck(
    r_sumcheck, &reduced_opening_proof.sumcheck_claims, &mut verifier.transcript,
);

// 3. Combine the individual commitments into a single joint commitment.
let joint_commitment = PCS::combine_commitments(
    &self.commitments, &verifier_state.gamma_powers,
);

// 4. Verify the joint HyperKZG opening at the sumcheck point.
verifier.accumulator.verify_joint_opening::<_, PCS>(
    &pp.generators, &reduced_opening_proof.joint_opening_proof,
    &joint_commitment, &verifier_state, &mut verifier.transcript,
)?;
```

This phase is the only one that calls into the polynomial commitment scheme (HyperKZG). Everything else in verification is field arithmetic and transcript bookkeeping.

If the model has no committed polynomials (a pathological corner case), `reduced_opening_proof` is `None` and the verifier just checks that this is consistent with `pp.shared.get_models_committed_polynomials()` being empty.

---

## Why the Verifier is Fast

Verification of nanoGPT takes ~0.7 s, of which the dominant cost is the joint HyperKZG opening check (a constant-size pairing check plus a few MSMs). The per-node sumcheck verification is cheap because:

- Each sumcheck round is $O(d)$ field operations to evaluate the round polynomial at the challenge, where $d$ is the operator's sumcheck degree (typically 2-4).
- The total number of rounds across the whole IOP is $O(\sum_v \log |v.\text{output}|)$, which for a 250k-parameter model is a few thousand rounds.
- Operator dispatch and accumulator updates are $O(1)$ per node.
- The eval-reduction proof for a multi-consumer node is a single MLE evaluation along a line, also $O(1)$.

In practice, **the verifier's runtime is dominated by the batch sumcheck and the HyperKZG pairing check, not by the per-node IOP loop.** This is what enables Jolt Atlas to verify models the size of GPT-2 in a few seconds even though proving takes more than a minute.

---

## Soundness Anchors

It is worth being explicit about *what makes the verifier sound*. There are exactly three anchors connecting the proof to the public statement:

1. **Output anchor.** Phase 2 forces the output MLE evaluation to match what the verifier computes from `io.outputs`. If the prover lies about the output, this fails immediately.
2. **Input anchors.** During Phase 3, when an `Input` node is visited, its operator-specific verifier checks the registered claim against the MLE evaluation of the corresponding public `io.inputs[i]` tensor. If the prover used the wrong inputs, the sumcheck chain that flows back to that input cannot satisfy this check.
3. **Commitment anchors.** All claims about model weights, quotients, and lookup addresses are settled by the joint HyperKZG opening in Phase 4 against commitments that were appended to the transcript *before any challenges were drawn*. The Fiat-Shamir binding prevents the prover from adapting the witness to the challenges after the fact.

Every other check inside the verifier is a sumcheck-soundness reduction; collectively they push every prover claim back to one of these three anchors. If all three anchors hold and every sumcheck round equation passes, the proof is accepted.
