# Lookup Arguments

Lookup arguments are the central design choice in Jolt Atlas. They enable the proof system to handle non-linear operators efficiently without arithmetic circuit constraints.

---

## Why Lookups?

Non-linear functions like ReLU, Tanh, and exponentiation cannot be expressed as low-degree polynomials. In circuit-based zkML systems (ezkl, zkml), they require expensive gadgets: byte decomposition, range checks, lookup tables with grand products and permutation arguments.

Jolt Atlas avoids all of that. Instead, for each non-linear operator, the prover:

1. Pre-commits a **read-address polynomial** that maps each input element to its lookup table row.
2. Runs a **Shout-style sumcheck** that proves the output was correctly read from the table at those addresses.

The Shout protocol has no grand products and no permutation checks. It reduces to a standard polynomial evaluation sumcheck.

---

## Two Lookup Providers

### OpLookupProvider

Located in `jolt-atlas-core/src/onnx_proof/op_lookups/mod.rs`.

Used for: ReLU, And, IsNaN, and other operators with small per-element lookup tables.

Implements `PrefixSuffixShoutProvider`, the Shout protocol with prefix-suffix decomposition of the lookup address (see [Prefix-Suffix Decomposition](./prefix-suffix.md)).

Returns `ProofType::RaOneHotChecks` sumcheck(s).

### RangeCheckProvider

Located in `jolt-atlas-core/src/onnx_proof/range_checking/mod.rs`.

Used for: remainder validation in Div, Rsqrt, Neural Teleport, and Softmax.

Proves $0 \le R < b$ using an `UnsignedLessThan` lookup table. Returns `ProofType::RangeCheck` sumcheck.

---

## Integration into the IOP Loop

Lookup proofs are generated alongside the main execution sumcheck within a single node visit:

```rust
// In OperatorProver::prove() for Div:
let execution_proof = prove_execution_sumcheck(node, prover);
let range_proof = RangeCheckProvider::read_raf_prove(node, prover);
let onehot_proof = shout::ra_onehot_provers(node, prover);

proofs.extend([
    (ProofId(node.idx, Execution),      execution_proof),
    (ProofId(node.idx, RangeCheck),     range_proof),
    (ProofId(node.idx, RaOneHotChecks), onehot_proof),
]);
```

All three proofs share the same Fiat-Shamir transcript state and reference the same pre-committed polynomials.

---

## Lookup Tables in joltworks

The actual lookup tables are defined in `joltworks/src/lookup_tables/`. They implement the `JoltLookupTable` trait:

```rust
pub trait JoltLookupTable {
    fn evaluate(index: u64) -> u64;
    fn subtable_indices() -> Vec<usize>;
}
```

Implemented tables include:
- `ReluTable`: ReLU function
- `UnsignedLessThan`: range comparison
- `SignedExponentiation`: exp for Softmax
- And others for Abs, Sin, Cos, etc.

---

## Lookup Proof Size

Each lookup argument generates one `SumcheckInstanceProof` with $k$ rounds (where $k = \log(\text{tensor size})$), plus an additional `BatchedSumcheck` with three instances for the one-hot checks. The total proof overhead per operator is small: a handful of field elements per round, times log of the domain size.
