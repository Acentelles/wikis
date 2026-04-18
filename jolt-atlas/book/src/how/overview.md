# Architecture Overview

Jolt Atlas proves neural network inference in three phases. This page gives the full picture before diving into each phase individually.

---

## The Full Pipeline

```
ONNX file (.onnx)
    │
    ▼  atlas-onnx-tracer :: Model::load()
ComputationGraph
    BTreeMap<node_idx, ComputationNode>
    - operator: Add | ReLU | Einsum | Softmax | ...
    - inputs: Vec<usize>   ← DAG edges
    - output_dims: Vec<usize>  ← padded to power-of-two
    │
    ▼  atlas-onnx-tracer :: Model::execute_graph()
Trace
    BTreeMap<node_idx, Tensor<i32>>
    - all intermediate tensors in integer arithmetic
    │
    ▼  jolt-atlas-core :: ONNXProof::prove()
    │
    ├─ commit_witness_polynomials()
    │    NodeCommittedPolynomials::get_committed_polynomials(node)
    │    → Vec<CommittedPolynomial>  (quotients, lookup addresses, ...)
    │    WitnessGenerator::generate_witness()
    │    → MultilinearPolynomial<F>  (dense Vec<F> or sparse OneHotPolynomial)
    │    PCS::commit()  → Commitment
    │    transcript.append_serializable(commitment)
    │
    ├─ output_claim()
    │    evaluate output MLE at random τ drawn from transcript
    │    seed ProverOpeningAccumulator
    │
    ├─ iop()  [reverse topological order over ComputationGraph.nodes]
    │    for each node:
    │      NodeEvalReduction::prove()    → EvalReductionProof
    │      OperatorProofTrait::prove()   → SumcheckInstanceProof (Execution)
    │      [if lookup node]
    │        OpLookupProvider::prove()   → SumcheckInstanceProof (RaOneHotChecks)
    │        RangeCheckProvider::prove() → SumcheckInstanceProof (RangeCheck)
    │        NeuralTeleport::prove()     → SumcheckInstanceProof (NeuralTeleport)
    │
    └─ prove_reduced_openings()
         accumulator.prove_batch_opening_sumcheck()
         → ReducedOpeningProof (batch sumcheck + HyperKZG joint opening)
    │
    ▼
ONNXProof {
    proofs: BTreeMap<ProofId, SumcheckInstanceProof>,
    commitments: Vec<PCS::Commitment>,
    eval_reduction_proofs: BTreeMap<usize, EvalReductionProof>,
    reduced_opening_proof,
}
```

---

## Three Phases

### Phase 1: ONNX → ComputationGraph

Handled by `atlas-onnx-tracer`. The ONNX protobuf is parsed into a typed node graph (`ComputationGraph`). All tensor dimensions are padded to the next power-of-two. Weights are quantized to 8-bit fixed-point integers.

The `ComputationGraph` is a `BTreeMap<usize, ComputationNode>`. Each node holds its operator type, the indices of its input nodes (the DAG edges), and the padded output shape.

### Phase 2: Forward Pass → Trace

`Model::execute_graph()` runs the model in integer arithmetic, visiting nodes in BTreeMap insertion order (topological). The result is a `Trace`: a `BTreeMap<usize, Tensor<i32>>` mapping every node index to its output tensor. All 45+ operators have pure `i32` evaluation functions; no ZK machinery is involved.

### Phase 3: Trace → Proof

`ONNXProof::prove()` in `jolt-atlas-core` has four sub-steps:

1. **Witness commitment**: for each node, the operator declares which auxiliary polynomials it needs committed (quotients, lookup addresses, etc.). These are materialized from the trace, committed via HyperKZG, and appended to the Fiat-Shamir transcript. *All commitments precede all challenges.*

2. **Output claim**: the output node's multilinear extension (MLE) is evaluated at a random point `τ` drawn from the transcript. This seeds the `ProverOpeningAccumulator`.

3. **IOP loop**: nodes are visited in reverse topological order (output first, input last). For each node, an evaluation reduction proof collapses fan-out opening requests, then the operator's sumcheck(s) prove the computation is correct.

4. **Batch opening**: all accumulated opening claims are reduced to a single joint claim via a final sumcheck, then opened jointly by HyperKZG.

---

## Crate Responsibilities

| Crate | Phase | Key types |
|-------|-------|-----------|
| `atlas-onnx-tracer` | 1, 2 | `Model`, `ComputationGraph`, `ComputationNode`, `Trace`, `Tensor` |
| `jolt-atlas-core` | 3 | `ONNXProof`, `Prover`, `Verifier`, `OperatorProofTrait`, `WitnessGenerator` |
| `joltworks` | 3 | `MultilinearPolynomial`, `HyperKZG`, `SumcheckInstanceProver`, `ProverOpeningAccumulator` |
| `common` | all | `CommittedPolynomial`, `VirtualPolynomial`, constants |
