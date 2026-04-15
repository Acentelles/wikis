# Phase 3: Sumcheck DAG

Phase 3 is where the actual ZK proof is generated. The `Trace` (a collection of `i32` tensors) is transformed into a collection of sumcheck proofs.

---

## The ONNXProof Structure

The proof is a flat collection of sumcheck instances:

```rust
pub struct ONNXProof<F, T, PCS> {
    /// Opening claims for committed polynomials
    pub opening_claims: Claims<F>,
    /// Sumcheck proofs keyed by (node_idx, ProofType)
    pub proofs: BTreeMap<ProofId, SumcheckInstanceProof<F, T>>,
    /// HyperKZG commitments to witness polynomials
    pub commitments: Vec<PCS::Commitment>,
    /// Evaluation reduction proofs (one per node)
    pub eval_reduction_proofs: BTreeMap<usize, EvalReductionProof<F>>,
    // Batched HyperKZG opening proof (private)
}
```

There is no explicit "DAG of sumchecks" object. The DAG is implicit: the topology is encoded in the `BTreeMap<ProofId, SumcheckInstanceProof>` and the `ProverOpeningAccumulator` that threads claims across node visits.

---

## ProofId

Each sumcheck instance is uniquely identified by a `ProofId`:

```rust
pub type ProofId = (usize, ProofType);
//                  node_idx  ^^^
```

`ProofType` is an enum with variants:

| Variant | Meaning |
|---------|---------|
| `Execution` | Main computation sumcheck (Add, Mul, Div, Einsum, …) |
| `RangeCheck` | Proves a remainder is in range (Div, Rsqrt) |
| `RaOneHotChecks` | Proves lookup address polynomial is valid one-hot |
| `NeuralTeleport` | Proves activation function via bounded-domain lookup |
| `SoftmaxDivSumMax` | Softmax division + sum + max combined proof |
| `SoftmaxExponentiationReadRaf` | Softmax exponentiation lookup |
| `SoftmaxExponentiationRaOneHot` | Softmax exponentiation one-hot check |

A single node visit can insert multiple `ProofId` entries. For example, a `Div` node inserts `(idx, Execution)`, `(idx, RangeCheck)`, and `(idx, RaOneHotChecks)`.

---

## CommittedPolynomial

Before any sumcheck challenge is drawn, certain auxiliary polynomials must be committed. These are polynomials that the prover introduces as part of a proof but that are not constrained by a prior node's output; they are *chosen* by the prover, not derived from the computation graph. For example, a Div node's quotient polynomial is not the output of any upstream node; the prover computes it and claims it satisfies $a = b \cdot q + R$. Without a commitment, a cheating prover could adapt $q$ after seeing the sumcheck challenges to make a false statement verify. Committing these polynomials to the Fiat-Shamir transcript before any challenge is drawn binds the prover to specific values.

Whether a commitment is needed depends on whether the sumcheck references any polynomial that the prover *invents*, as opposed to one that is simply another node's output.

**Simple operators need no commitments.** An `Add` node proving $C = A + B$ only references three polynomials (A, B, C), and all three are outputs of nodes in the computation graph. They are already pinned down by the eval-reduction chain that links each node's output to its consumers, so the prover has no freedom to substitute different values.

**Complex operators do.** A `Div` node proving $a = b \cdot q + R$ must introduce $q$ (the quotient) and $R$ (the remainder), which are not outputs of any upstream node. Without a commitment, a cheating prover could wait to see the random sumcheck challenges and then pick whatever $q$ and $R$ happen to make the check pass for an incorrect result. Committing these polynomials to the transcript before any challenge is drawn locks the prover in. The same applies to lookup address polynomials (one-hot encodings) and neural teleportation quotients.

The `CommittedPolynomial` enum in `common/src/lib.rs` names these:

```rust
pub enum CommittedPolynomial {
    NodeOutputRaD(usize, usize),          // one-hot lookup address, per node, per chunk
    DivNodeQuotient(usize),               // quotient polynomial for Div
    DivRangeCheckRaD(usize, usize),       // range-check address, per chunk
    TeleportNodeQuotient(usize),          // Neural Teleport quotient (Tanh/Erf/Sigmoid/…)
    TeleportRangeCheckRaD(usize, usize),  // Neural Teleport range-check address
    SoftmaxRemainder(usize, usize),       // Softmax division remainder
    SoftmaxExponentiationRaD(usize, usize), // Softmax exponentiation address
    GatherRa(usize),                      // Gather embedding address
    // ...
}
```

---

## VirtualPolynomial

A virtual polynomial is a polynomial that participates in a sumcheck relation but is never directly committed to. Instead, its evaluation at the sumcheck output point becomes an *opening claim* that is resolved later, either by another sumcheck instance or by the batch opening proof.

This concept originates from the [Binius](https://eprint.iacr.org/2023/1784) paper (Diamond and Posen, 2023), which introduces virtual polynomials as a way to compose sumcheck instances without committing to every intermediate value. A sumcheck over $f(x) \cdot g(x)$ does not require commitments to both $f$ and $g$ upfront; it only requires that, once the sumcheck binds all variables to a random point $r$, the verifier can obtain $f(r)$ and $g(r)$ from somewhere (a commitment opening, another sumcheck's output, or a public computation). This deferred resolution is what makes virtual polynomials useful: they allow the prover to chain sumcheck instances together, passing claims forward without paying commitment costs at each step.

In Jolt Atlas, virtual polynomials are the node output tensors and other intermediate values (division remainders, teleportation quotients) that flow between sumcheck instances during the IOP loop:

```rust
pub enum VirtualPolynomial {
    NodeOutput(usize),        // output of node idx
    DivRemainder(usize),      // remainder of node idx
    TeleportQuotient(usize),  // Neural Teleport quotient
    SoftmaxExponentiation(usize, usize),
    // ...
}
```

The `ProverOpeningAccumulator` maps `(VirtualPolynomial, SumcheckId)` to `(OpeningPoint, Claim)` to track which claims are outstanding and which sumcheck instance will produce the opening. When a sumcheck for node $i$ references the output of node $j$, the claim on `NodeOutput(j)` is not opened immediately; it is accumulated and resolved when node $j$'s own sumcheck runs (in reverse topological order). At the end of the IOP loop, all remaining claims are settled by the batch HyperKZG opening proof.

---

## The Three Phases of Phase 3

Phase 3 itself has four sequential sub-steps:

1. **[Witness commitment](./witness-commitment.md)**: commit all `CommittedPolynomial` values to the transcript.
2. **[IOP loop](./iop-loop.md)**: generate sumcheck proofs in reverse topological order.
3. **[Evaluation reduction](./eval-reduction.md)**: collapse fan-out opening requests per node.
4. **[Batch opening](./batch-opening.md)**: reduce all claims to a single HyperKZG opening.
