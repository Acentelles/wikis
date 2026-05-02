# Speeding Up Sum-Check Proving

| | |
|---|---|
| **Authors** | Quang Dao, Zachary DeStefano, Suyash Bagad, Yuval Domb, Justin Thaler |
| **eprint** | 2026/587 |
| **Affiliations** | CMU, NYU, Ingonyama, Georgetown |

## Summary

Three complementary techniques for reducing sum-check proving time and memory, integrated into the Jolt zkVM with 1.7x-2.2x end-to-end speedups.

## Technique 1: High-Degree Products

When the sum-check polynomial $g$ is a product of $d$ multilinear polynomials in $v$ variables, the naive prover cost per round is $O(d^{v+1})$. This paper reduces it to $O(dv + d \cdot \log d)$ by:

- Computing partial products incrementally rather than recomputing from scratch.
- Using a structured evaluation schedule that shares intermediate results across degree positions.

## Technique 2: Small-Value Sum-Check

When the committed polynomials evaluate to small values (e.g., 64-bit or 32-bit integers), the sum-check prover can operate on native integers rather than full field elements for most of the computation:

- **Delayed reduction.** Intermediate sums are accumulated as machine-word integers, with modular reduction deferred until overflow threatens.
- **Streaming prover.** A time-space tradeoff that reduces memory from $O(2^\ell)$ to $O(\sqrt{2^\ell})$ or even $O(\ell)$ with multiple passes, while exploiting small values for faster arithmetic.

This is distinct from sparsity: the polynomial is dense, but its values fit in small machine words.

## Technique 3: Decomposable Polynomials

The equality polynomial $\widetilde{\text{eq}}(r, x) = \prod_{i=1}^{\ell}(r_i x_i + (1-r_i)(1-x_i))$ has tensor structure that can be exploited:

- Instead of treating $\widetilde{\text{eq}}$ as a black-box factor in the sum-check integrand, the prover uses its factored form to compute round polynomials incrementally.
- This nearly eliminates the overhead of including $\widetilde{\text{eq}}$ as a multiplicand.

## Impact

Integrated into Jolt:
- High-degree sum-check instances (e.g., memory checking with products of 3-4 MLEs) see the largest benefit.
- Small-value specialisation applies broadly, since Jolt's committed witness values are typically 32-bit or 64-bit.
- Combined speedup: 1.7x-2.2x on the sum-check prover phase.
