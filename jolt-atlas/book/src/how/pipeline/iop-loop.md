# The IOP Loop

After witness commitment, the prover runs the interactive oracle proof (IOP) loop. This is where sumcheck instances for each network node are generated.

---

## Reverse Topological Order

```rust
// In prover.rs: iop()
for (_, node) in computation_nodes.iter().rev() {
    let (eval_reduction_proof, execution_proofs) = OperatorProver::prove(node, prover);
    eval_reduction_proofs.insert(node.idx, eval_reduction_proof);
    proofs.extend(execution_proofs);
}
```

The BTreeMap is iterated in reverse. Since the BTreeMap is approximately topologically ordered (parents before children), reversing it visits output nodes first and input nodes last.

**Why output-first?** This mirrors the standard GKR / sumcheck propagation pattern. To prove a claim about node $v$'s output MLE at a random point $r$, the sumcheck reduces it to claims about $v$'s *input* MLEs at some derived points. Those derived claims are then consumed when the upstream input nodes are visited. Processing output-first ensures that by the time an upstream node is visited, all of its downstream consumers have already registered their opening requests.

Consider this example fragment of a transformer computation graph:

```
  COMPUTATION GRAPH                    IOP VISIT ORDER
  (forward / data flow)                (reverse topological)

  Gather [0]                           Step 5:  Gather [0]
    │                                     ▲  consumes claim on Gather output
    ▼                                     │
  Einsum [1]  (Q*K^T)                 Step 4:  Einsum [1]
    │                                     ▲  consumes claim on Einsum output
    ▼                                     │
  Softmax [2]                          Step 3:  Softmax [2]
    │                                     ▲  consumes claim on Softmax output
    ▼                                     │
  Einsum [3]  (attn * V)              Step 2:  Einsum [3]
    │                                     ▲  consumes claim on Einsum output
    ▼                                     │
  Add [4]  (residual)                  Step 1:  Add [4]
    │                                     ▲  consumes claim on Add output
    ▼                                     │
  Output                               START:  seed claim on Output MLE at random r
```

The loop begins at the output and works backward:

1. **Seed.** The verifier draws a random point $r$ and the prover claims $\text{Output}(r) = v$. This is the initial opening claim.
2. **Step 1 (Add [4]).** The eval-reduction proof collapses any fan-out claims on Add's output into one. The Add sumcheck then proves $\text{Add}(r') = \text{Einsum}_3(r'_1) + \text{Residual}(r'_2)$, producing new claims on its inputs.
3. **Step 2 (Einsum [3]).** Consumes the claim on $\text{Einsum}_3(r'_1)$ registered by Step 1. Its sumcheck proves the matmul relation and produces claims on Softmax's output and V's weight polynomial.
4. **Steps 3-5** continue the same pattern: each node consumes the claims registered by its downstream consumers and produces new claims on its own inputs.

At the end, all claims have propagated to either committed polynomials (weights, quotients, lookup addresses) or input tensors. The [batch opening](./batch-opening.md) step settles them all in one shot.

---

## Operator Dispatch

`OperatorProver::prove(node, prover)` dispatches to the operator-specific implementation via `dispatch_operator!`:

```rust
dispatch_operator!(node.operator,
    Add => AddProver::prove_and_reduce_node(node, prover),
    Div => DivProver::prove_with_reduction(node, prover),
    Tanh => TanhProver::prove_neural_teleport(node, prover),
    Softmax => SoftmaxProver::prove_composite(node, prover),
    // ... one arm per Operator variant
)
```

Each arm calls the `OperatorProofTrait` implementation for that operator.

---

## Per-Node Output

Each call to `OperatorProver::prove` returns:

1. `EvalReductionProof`: a line-restriction proof collapsing multiple consumer opening requests into one (see [Evaluation Reduction](./eval-reduction.md)).
2. `Vec<(ProofId, SumcheckInstanceProof)>`: one or more sumcheck proofs. For simple operators (Add, Mul) this is one entry; for complex operators (Div, Softmax, Neural Teleport) this is two to four entries.

All `SumcheckInstanceProof` entries are accumulated into the global `BTreeMap<ProofId, SumcheckInstanceProof>` that will be stored in the final `ONNXProof`.

---

## The ProverOpeningAccumulator

The accumulator is the mechanism that links node visits together:

```rust
pub struct ProverOpeningAccumulator<F> {
    // Claims from downstream consumers waiting to be satisfied
    openings: BTreeMap<(VirtualPolynomial, SumcheckId), (OpeningPoint, Claim)>,
    // Reduced evaluations after eval-reduction proofs
    reduced_evaluations: BTreeMap<usize, F>,
}
```

When a downstream node's sumcheck produces a claim on `NodeOutput(upstream_idx)`, it calls:

```rust
accumulator.append_virtual(transcript,
    VirtualPolynomial::NodeOutput(upstream_idx),
    SumcheckId::NodeExecution(current_node_idx),
    opening_point,
    claimed_value,
);
```

When the upstream node is later visited, its `NodeEvalReduction::prove` reads all outstanding `NodeOutput(upstream_idx)` claims from the accumulator and generates an eval-reduction proof that collapses them.

---

## Transcript State Machine

The Fiat-Shamir transcript evolves continuously throughout the IOP loop. Every sumcheck round message is appended before the next challenge is derived. The ordering is deterministic and reproducible: the verifier reconstructs the same transcript by replaying the same operator visits with the same proof bytes.

The transcript at any point encodes the entire history of messages up to that point. This is what makes the proof non-interactive: a verifier can recompute all challenges from the proof bytes and the model's public inputs.

---

## Loop Termination

The loop terminates after visiting all nodes. At this point:

- All `EvalReductionProof` entries have been generated and inserted into `eval_reduction_proofs`.
- All `SumcheckInstanceProof` entries have been generated and inserted into `proofs`.
- The `ProverOpeningAccumulator` holds all outstanding scalar claims at their respective evaluation points.
- No claim remains unresolved (every `NodeOutput` virtual polynomial has been consumed by its node's eval-reduction proof).

The next step is [Batch Opening](./batch-opening.md).
