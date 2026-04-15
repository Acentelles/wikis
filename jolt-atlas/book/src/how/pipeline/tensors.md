# Tensor Representation

`Tensor<T>` is the generic multi-dimensional array used throughout Jolt Atlas. It appears as `Tensor<i32>` in the tracer (execution) and `Tensor<F>` in some proof utilities.

---

## The Tensor Type

```rust
pub struct Tensor<T> {
    data: Vec<T>,          // flat row-major storage
    dims: Vec<usize>,      // shape (e.g. [batch, seq, hidden])
    scale: Scale,          // fixed-point scale metadata
}
```

`Tensor::new(data, dims)` constructs a tensor from a flat `Vec<T>` and a shape. The total number of elements must equal the product of the dimensions.

---

## Indexing and Iteration

Tensors use row-major (C-order) flat storage. A 3D tensor of shape `[a, b, c]` stores element `[i, j, k]` at flat index `i*b*c + j*c + k`.

Parallel iteration is available via Rayon:

```rust
tensor.par_enum_map(|idx, val| transform(idx, val))
```

This maps a function over all elements in parallel, producing a new `Tensor<U>` with the same shape.

---

## From Tensor to Multilinear Polynomial

When the prover encodes a tensor as a cryptographic polynomial, `Tensor<i32>` is converted to `MultilinearPolynomial<F>`:

1. Each `i32` element is cast to a field element `F`.
2. If the tensor has $N$ elements and $k = \lceil \log_2 N \rceil$, the flat data is zero-padded to length $2^k$.
3. The result is a `MultilinearPolynomial::Dense(Vec<F>)`, a dense evaluation table over the Boolean hypercube $\{0,1\}^k$.

This polynomial uniquely represents the tensor: its evaluation at the Boolean point corresponding to index $i$ equals the $i$-th tensor element (as a field element).

**Note:** This is why all tensor dimensions are padded to powers-of-two at load time. The zero-padded elements contribute zero to every sumcheck over the hypercube, so they do not affect proof correctness.

---

## One-Hot Polynomials (Sparse)

For lookup addresses, a different representation is used: `MultilinearPolynomial::OneHot(OneHotPolynomial)`.

A one-hot polynomial stores only the *indices* where the polynomial evaluates to 1, rather than storing all $2^k$ evaluations. For a lookup address polynomial, each element maps to exactly one table row, so exactly one entry per cycle is nonzero.

```rust
pub struct OneHotPolynomial {
    indices: Vec<u16>,     // index of the '1' entry per cycle
    num_vars: usize,       // log2 of the domain size
}
```

**Why sparse?** HyperKZG commitment cost is proportional to the number of group operations, which is proportional to the number of nonzero evaluations. For a one-hot polynomial with $N$ cycles out of $2^k$ possible positions, the commitment costs $O(N)$ rather than $O(2^k)$. This is the "pay-per-bit" property.

---

## Scale Metadata

The `scale` field on `Tensor<i32>` carries the fixed-point quantization exponent (see [Quantization](./quantization.md)). It is used by:

- The tracer to track precision through arithmetic operations.
- The proof system to correctly interpret tensor values when constructing lookup table indices.

Scale is **not** part of the ZK proof; it is a metadata convention agreed upon by prover and verifier based on the model preprocessing.
