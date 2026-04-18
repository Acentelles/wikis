# Witness Generation and Commitment

Before any sumcheck challenge is drawn, Jolt Atlas commits all auxiliary polynomials to the Fiat-Shamir transcript. This is the **binding step**: it prevents a cheating prover from adapting polynomial values after seeing challenge points.

---

## Overview

```
ComputationGraph.nodes  (forward order)
        │
        ▼  NodeCommittedPolynomials::get_committed_polynomials(node)
        │  → Vec<CommittedPolynomial>
        │
        ▼  WitnessGenerator::generate_witness(model, trace)
        │  reads i32 values from Trace
        │  → MultilinearPolynomial<F>  (dense or sparse OneHotPolynomial)
        │
        ▼  PCS::commit(poly)  [HyperKZG]
        │  → Commitment
        │
        ▼  transcript.append_serializable(commitment)
        │  binds into Fiat-Shamir state
        │
        ├──→ BTreeMap<CommittedPolynomial, MultilinearPolynomial<F>>
        │    held by prover throughout the IOP loop
        └──→ Vec<Commitment>   written into ONNXProof, sent to verifier
```

---

## Step 1: Which Polynomials Are Needed?

```rust
// In prover.rs: polynomial_map()
model.graph.nodes.values()
    .flat_map(|node| NodeCommittedPolynomials::get_committed_polynomials(node))
    .map(|cp| (cp.clone(), cp.generate_witness(model, trace)))
    .collect::<BTreeMap<_, _>>()
```

`NodeCommittedPolynomials::get_committed_polynomials` dispatches to each operator's `OperatorProofTrait::get_committed_polynomials` via the `dispatch_operator!` macro.

**Per-operator committed polynomials:**

| Operator | Committed polynomials |
|----------|----------------------|
| Add, Sub, Mul, Einsum, Reshape, … | *(none)* |
| ReLU, And | `NodeOutputRaD(node_idx, d)` for each chunk `d` |
| Div | `DivNodeQuotient(node_idx)`, `DivRangeCheckRaD(node_idx, d)` for each `d` |
| Rsqrt | `RsqrtNodeInv(node_idx)`, `RsqrtRangeCheckRaD(node_idx, d)` |
| ScalarConstDiv | `ScalarConstDivNodeRemainder(node_idx)` |
| Tanh, Erf, Sigmoid, Sin, Cos | `TeleportNodeQuotient(node_idx)`, `TeleportRangeCheckRaD(node_idx, d)` |
| Softmax | `SoftmaxRemainder(node_idx, i)`, `SoftmaxExponentiationRaD(node_idx, d)` |
| Gather | `GatherRa(node_idx)` |

The index `d` in the `RaD` variants refers to the **chunk index** from the [prefix-suffix decomposition](../lookups/prefix-suffix.md). A 64-bit lookup address is split into 16 chunks of 4 bits each (nibbles), and each chunk gets its own one-hot polynomial. So `NodeOutputRaD(node_idx, d)` with `d` ranging from 0 to 15 means there are 16 separate one-hot committed polynomials per lookup node, one for each nibble of the address. Each one-hot polynomial encodes which of the 16 possible values that nibble takes at each cycle.

---

## Step 2: Materialise from Trace

`WitnessGenerator::generate_witness` (in `jolt-atlas-core/src/onnx_proof/witness.rs`) reads the `Trace` and builds each polynomial. Two representative cases:

### Dense polynomial: DivNodeQuotient

```rust
// The output of the Div node is the quotient. Just wrap it.
let layer_data = Trace::layer_data(trace, node);
MultilinearPolynomial::from(layer_data.output.clone())
// i32 values → Vec<F>, zero-padded to 2^k
```

### Sparse one-hot: NodeOutputRaD

```rust
// Compute which lookup table row each input maps to.
let lookup_indices = compute_lookup_indices_from_operands(&padded_operands, is_interleaved);
// Build a one-hot polynomial for the d-th address chunk.
build_one_hot_rad_witness(&lookup_indices, d)
// → MultilinearPolynomial::OneHot(OneHotPolynomial { indices: Vec<u16>, ... })
```

### Neural Teleport: TeleportNodeQuotient

```rust
// Divide input by τ (periodicity constant) to get the fractional residue.
let (quotient, _) = compute_division(input, inner.tau);
// Map quotient entries to table indices, build one-hot for chunk d.
build_teleport_activation_rad_witness(input, inner.tau, inner.log_table, d_idx)
```

---

## Step 3: Commit via HyperKZG

```rust
poly_map.values().map(|poly| PCS::commit(poly, pcs).0).collect::<Vec<_>>()
```

`PCS` is instantiated as `HyperKZG<Bn254>`. For a dense polynomial of $2^k$ elements, this requires $O(2^k)$ group operations. For a one-hot polynomial with $N$ nonzeros, it requires only $O(N)$ (the "pay-per-bit" property).

---

## Step 4: Append to Transcript

```rust
for commitment in &commitments {
    transcript.append_serializable(commitment);
}
```

After this loop, the Fiat-Shamir state includes all polynomial commitments. Only then does the prover draw the first challenge (`output_claim`). This ordering is the critical security invariant: the prover cannot influence its polynomial values based on the challenges.

---

## Why the BTreeMap is Kept Alive

The `BTreeMap<CommittedPolynomial, MultilinearPolynomial<F>>` is held in memory throughout the entire proving session. At the end of the IOP loop, `prove_reduced_openings` needs to evaluate each committed polynomial at the batch challenge point and generate the HyperKZG opening proof. The polynomial evaluations cannot be recovered from the commitments alone.
