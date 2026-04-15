# Einsum and MatMul

All matrix multiplication operations in Jolt Atlas (including attention score computation, feedforward projection, and embedding mixing) are handled by the `Einsum` operator with a subscript string.

---

## What Einsum Proves

An Einsum contraction is a generalized tensor product. The canonical example, matrix-matrix multiply $C = A \cdot B$:

$$C[m, n] = \sum_k A[m, k] \cdot B[k, n]$$

In sumcheck form, this becomes a single sumcheck over the contraction index $k$:

$$\sum_{k \in \{0,1\}^{\log K}} \widetilde{eq}_{m}(r_m, k_m) \cdot A(r_m, k) \cdot B(k, r_n) = C(r_m, r_n)$$

where $r_m$ and $r_n$ are random evaluation points from the transcript and $k$ is the free variable being summed. This is a degree-3 sumcheck (degree 1 from each operand polynomial, degree 1 from the equality polynomial bound to the output dimensions).

---

## Supported Patterns

The `EINSUM_REGISTRY` dispatch table covers six patterns:

| Subscript | Operation | Use in transformers |
|-----------|-----------|---------------------|
| `mk,kn->mn` | Standard matrix-matrix multiply | Q·K^T, V projection, FFN |
| `k,nk->n` | Vector-matrix multiply | Bias addition (1D × 2D) |
| `bmk,bkn->mbn` | Batched matmul (variant 1) | Attention: batch × head |
| `bmk,kbn->mbn` | Batched matmul (variant 2) | Cross-attention |
| `mbk,bnk->bmn` | Batched matmul (variant 3) | Value mixing |
| `mbk,nbk->bmn` | Batched matmul (variant 4) | Attention output projection |

The correct `EinsumVariant` is selected automatically from the ONNX subscript string at model-load time. Unsupported subscripts produce a runtime error at preprocessing.

---

## GEMM and MatMul Normalization

ONNX models may use `Gemm` (generalized matrix multiply: $Y = \alpha A B + \beta C$) or `MatMul` nodes instead of `Einsum`. During parsing in `atlas-onnx-tracer`, both are normalized to `Einsum("mk,kn->mn")` with the GEMM parameters folded in.

---

## Role in Transformer Models

In a standard transformer attention block:

1. **Q, K, V projections:** `Einsum("mk,kn->mn")`, multiplying the input by the Q, K, V weight matrices.
2. **Attention scores:** `Einsum("bmk,bkn->mbn")` or similar, computing $Q K^\top$.
3. **Attention output:** `Einsum("mbk,bnk->bmn")`, applying attention weights to values.
4. **Feedforward:** `Einsum("mk,kn->mn")`, two-layer MLP.

Einsum is therefore the most frequently occurring operator in GPT-style models. The sumcheck for each Einsum is the dominant cost within the IOP loop.

---

## No Auxiliary Polynomials

Like all linear operators, Einsum has `get_committed_polynomials() = vec![]`. The proof is a single `ProofType::Execution` sumcheck with no pre-committed auxiliary data.

The operand MLEs (the two input tensors) are virtual polynomials opened via the eval-reduction chain. The output MLE is linked to the next node's sumcheck as a `NodeOutput` virtual polynomial.
