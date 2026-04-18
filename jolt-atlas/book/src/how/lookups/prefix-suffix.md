# Prefix-Suffix Decomposition

Large lookup tables (like a 64-bit ReLU table) cannot be committed as a monolithic polynomial, since the commitment cost would be $O(2^{64})$. The prefix-suffix decomposition breaks every large table into small sub-tables that can be committed and evaluated efficiently.

---

## Constants

```rust
pub const XLEN: usize = 32;           // element bit width
pub const LOG_K_CHUNK: usize = 4;     // bits per chunk
pub const K_CHUNK: usize = 16;        // = 2^LOG_K_CHUNK, entries per sub-table
pub const LOG_K: usize = XLEN * 2;   // = 64, total address bits
```

Every lookup address is 64 bits, decomposed into 16 chunks of 4 bits each.

---

## How Decomposition Works

A 64-bit address $a$ is split into 16 nibbles $a = (c_0, c_1, \ldots, c_{15})$, where $c_i \in [0, 16)$.

For each chunk position $i$:
- The **prefix sub-table** encodes the function value on the most significant bits up to position $i$.
- The **suffix sub-table** encodes the function value on the least significant bits from position $i+1$.

The full table value at address $a$ is reconstructed by combining the relevant prefix and suffix sub-table entries via XOR or addition (depending on the function).

This decomposition is defined by the `PrefixSuffixDecompositionTrait` in `joltworks/src/lookup_tables/`.

---

## JoltLookupTable Trait

Every lookup table implements this trait:

```rust
pub trait JoltLookupTable: Sized {
    fn materialize(memory_bits: usize) -> Vec<u64>;
    fn combine_lookups(vals: &[u64], C: usize, M: usize) -> u64;
    fn g_poly_degree(C: usize) -> usize;
}
```

- `materialize`: builds the full table (used only at setup time for the subtable commitment).
- `combine_lookups`: reconstructs the full table value from the chunk values. For ReLU, this is a sign-extension and max operation.
- `g_poly_degree`: the degree of the combining polynomial (determines the sumcheck degree for the address phase).

---

## Prefix Evaluation Caching

During the address phase of the Shout sumcheck, the prover needs to evaluate the prefix sub-table contribution for each round. Recomputing this from scratch each round would be expensive.

`PrefixCheckpoint` and `PrefixEval` structures cache intermediate prefix evaluations:

```rust
pub struct PrefixCheckpoint<F> {
    // Cached evaluations of prefix polynomials at challenge points seen so far
    checkpoint: Vec<F>,
}
```

When challenge $r_j$ is ingested, the cache is updated in $O(K_\text{CHUNK})$ time rather than $O(2^j)$. This makes the address phase $O(\text{LOG\_K} \cdot K_\text{CHUNK})$ per sumcheck rather than exponential.

---

## Pay-Per-Bit Commitment

The one-hot address polynomial `NodeOutputRaD(node_idx, d)` has $N$ nonzero entries (one per cycle) out of $2^{\text{LOG\_K\_CHUNK}} = 16$ possible positions. HyperKZG commitment cost is proportional to the number of nonzeros.

For $N$ cycles and 16 chunks of 4 bits each, the total commitment cost is:
$$O(N \cdot 16) = O(N \cdot \text{LOG\_K} / \text{LOG\_K\_CHUNK})$$

This is **linear in $N$** (the number of elements being looked up), regardless of the full table size $2^{64}$. Hence "pay-per-bit."

---

## Subtable Commitments

The prefix and suffix sub-tables (each of size 16) are committed once at preprocessing time and shared across all operators using the same lookup function. This amortizes the commitment cost over the entire model.
