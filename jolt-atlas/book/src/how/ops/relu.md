# ReLU and Lookup-Based Activations

ReLU cannot be expressed as a low-degree polynomial; it is piecewise linear with a kink at zero. Jolt Atlas proves ReLU (and similar comparison-based operations) using **lookup arguments** rather than arithmetic constraints.

---

## The Lookup Approach

Instead of proving $\text{out}(x) = \max(0, \text{in}(x))$ arithmetically, the prover proves:

> For each input value $v_i$, there exists a row $j_i$ in the ReLU lookup table such that $\text{table}[j_i] = \text{out}_i$.

The lookup argument verifies that every output value was correctly read from a pre-defined table.

---

## The ReLU Lookup Table

The ReLU table maps each signed integer input (as a bit pattern) to its ReLU output:
$$\text{table}[j] = \max(0, j)$$

The table has $2^{64}$ entries in principle (for 64-bit integers), but it is never materialized in full. Instead, it is decomposed into prefix and suffix sub-tables via the Shout framework (see [Prefix-Suffix Decomposition](../lookups/prefix-suffix.md)).

---

## Committed Polynomial: NodeOutputRaD

The **read-address** polynomial encodes, for each cycle $t$ (i.e., each input element), which row of the lookup table was read:

```
address[t] = lookup_table_index(input[t])
```

This polynomial is committed as `NodeOutputRaD(node_idx, d)` where $d$ indexes the 4-bit chunk of the address (since the full 64-bit address is split into 16 chunks of 4 bits each via the prefix-suffix decomposition).

`NodeOutputRaD` is a **one-hot polynomial**: for each cycle $t$, exactly one entry of the 4-bit chunk is 1 (indicating the selected table row for that chunk), and all others are 0.

---

## Three ProofType Entries

A ReLU node generates:

1. **`ProofType::Execution`**: a standard degree-2 sumcheck proving the output satisfies $\text{out}(x) = \text{ReLU}(\text{in}(x))$ via the lookup table value.

2. **`ProofType::RaOneHotChecks`**: three sumchecks (run as a `BatchedSumcheck`) that together prove the read-address polynomial `NodeOutputRaD` is a valid one-hot encoding:
   - **Booleanity:** $\text{addr}(x) \in \{0, 1\}$ for each entry.
   - **Hamming weight:** exactly one entry per cycle is 1.
   - **Hamming booleanity:** the product of any two distinct entries is 0.

These three checks together constitute a complete one-hot validity proof.

---

## OpLookupProvider

The lookup argument is implemented by `OpLookupProvider` in `jolt-atlas-core/src/onnx_proof/op_lookups/mod.rs`. It implements `PrefixSuffixShoutProvider` for ReLU:

- Breaks the 64-bit lookup address into prefix and suffix 4-bit chunks.
- For each chunk, runs a Shout-style sumcheck over the corresponding sub-table.
- Batches all chunk sumchecks via `gamma` coefficients derived from the transcript.

The result is a single `SumcheckInstanceProof` stored as `(node_idx, ProofType::RaOneHotChecks)`.

---

## BatchedSumcheck

The three one-hot validity checks are run as a `BatchedSumcheck`:

```rust
BatchedSumcheck::prove(
    &[booleanity_prover, hamming_weight_prover, hamming_booleanity_prover],
    transcript,
)
```

Front-loaded batching: all three input claims are appended to the transcript first, a batching coefficient vector is derived, then the combined univariate message at each round is $\sum_i \lambda_i \cdot h_i(X)$.

---

## Other Lookup-Based Operators

The same pattern applies to:

- **`And`**: bitwise AND via lookup table.
- **`IsNaN`**: NaN detection via lookup table.
- **`Abs`**: absolute value via lookup table.
- **`Clamp`**: min/max clipping via lookup table.

Each uses `NodeOutputRaD` committed polynomials and `RaOneHotChecks` sumchecks.
