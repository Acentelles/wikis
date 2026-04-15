# Evaluation Reduction

Every node in the computation graph may have its output consumed by multiple downstream nodes. Each consumer's sumcheck requests an evaluation of the output polynomial at a different random point. The **evaluation reduction** protocol collapses these multiple requests into a single claim, avoiding the need for multiple independent polynomial openings per node.

---

## The Fan-Out Problem

Suppose node $v$ feeds into downstream nodes $u_1, u_2, u_3$. After the IOP loop processes $u_1, u_2, u_3$ (in reverse topological order, before $v$), each has registered a claim:

$$P_v(r_1) = c_1, \quad P_v(r_2) = c_2, \quad P_v(r_3) = c_3$$

where $P_v$ is the MLE of $v$'s output and $r_1, r_2, r_3$ are distinct random evaluation points drawn from the transcript.

Opening $P_v$ at three different points would require three independent HyperKZG opening proofs. The evaluation reduction avoids this cost.

---

## Line Restriction Reduction (PAZK-style)

`NodeEvalReduction::prove` in `jolt-atlas-core/src/onnx_proof/ops/eval_reduction.rs`:

1. **Construct the line.** The prover constructs the unique line $\ell(t)$ passing through all $N$ evaluation points:
   $$\ell(t) = r_1 + t \cdot (r_2 - r_1)$$
   (generalized to $N$ points using the Lagrange basis).

2. **Send the univariate restriction.** The prover computes and sends the univariate polynomial $h(t) = P_v(\ell(t))$, the restriction of $P_v$ to the line $\ell$.

3. **Verifier checks.** For each claimed evaluation:
   $$h(i) = P_v(\ell(i)) = P_v(r_i) = c_i \quad \text{for } i = 1, \ldots, N$$
   The verifier checks each equality.

4. **New challenge.** The verifier draws a fresh challenge $x'$ from the transcript and computes $r' = \ell(x')$.

5. **Reduced claim.** Both prover and verifier agree on the single claim:
   $$P_v(r') = h(x')$$

This reduces $N$ opening claims to one. The polynomial $h$ is sent as coefficients (one field element per coefficient), so the proof overhead is linear in the degree of $P_v \circ \ell$, which is at most the number of variables of $P_v$.

---

## Linking to the Committed Polynomial

The reduced claim $P_v(r') = h(x')$ is for the *virtual* polynomial `NodeOutput(v.idx)`. For the final HyperKZG batch opening to verify it, this claim must be linked to a committed polynomial.

For most simple operators, $P_v$ *is* directly committed as a `CommittedPolynomial::NodeOutput`. The accumulator call:

```rust
accumulator.append_dense(CommittedPolynomial::DivNodeQuotient(v.idx), r', h(x'));
```

registers the reduced claim against the pre-committed polynomial. The batch opening will later open `DivNodeQuotient` at $r'$ and check it equals $h(x')$.

---

## Verifier Side

The verifier replays the same reduction:

1. Read the $N$ consumer claims from the `VerifierOpeningAccumulator`.
2. Check $h(i) = c_i$ for each $i$ using the `h` polynomial from the `EvalReductionProof`.
3. Draw the same challenge $x'$ from the transcript.
4. Register the reduced claim $P_v(\ell(x')) = h(x')$ with the accumulator.

The verifier does not evaluate $P_v$ directly. The final HyperKZG check will confirm the claim.

---

## Why Not Just Batch with Random Coefficients?

An alternative would be to combine all $N$ claims into one via random linear combination: $\sum_i \lambda_i P_v(r_i) = \sum_i \lambda_i c_i$. But this requires a sumcheck to prove the linear combination (since the prover can't open a single polynomial at multiple points in one KZG step). The line restriction approach avoids this extra sumcheck by directly reducing to a single evaluation claim.
