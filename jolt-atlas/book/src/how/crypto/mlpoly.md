# Multilinear Polynomials

Multilinear polynomials are the algebraic objects that represent tensors in the Jolt Atlas proof system. Every tensor value (activations, weights, quotients, lookup addresses) is encoded as a multilinear polynomial (MLE) over a Boolean hypercube.

---

## Definition

A **multilinear polynomial** in $k$ variables is a polynomial $P: \mathbb{F}^k \to \mathbb{F}$ that has degree at most 1 in each variable. Equivalently, it is uniquely determined by its $2^k$ evaluations at the Boolean hypercube $\{0,1\}^k$.

Given a flat array of values $(v_0, v_1, \ldots, v_{2^k - 1})$, the **multilinear extension** is the unique multilinear polynomial $\widetilde{v}$ such that:
$$\widetilde{v}(b_1, \ldots, b_k) = v_{\text{idx}(b_1, \ldots, b_k)} \quad \forall (b_1, \ldots, b_k) \in \{0,1\}^k$$

where $\text{idx}(b_1, \ldots, b_k)$ is the binary index.

---

## MultilinearPolynomial Enum

```rust
pub enum MultilinearPolynomial<F: JoltField> {
    Dense(DensePolynomial<F>),
    OneHot(OneHotPolynomial),
}
```

### Dense

`DensePolynomial<F>` stores all $2^k$ evaluations as a flat `Vec<F>`. This is the general-purpose representation used for tensors with arbitrary values (activations, weights, quotients).

### OneHot

`OneHotPolynomial` stores only the indices where the polynomial evaluates to 1:

```rust
pub struct OneHotPolynomial {
    indices: Vec<u16>,   // index of the '1' per cycle (one entry per cycle)
    num_vars: usize,     // log2 of the domain
}
```

This is used for lookup address polynomials where each cycle selects exactly one table row. Commitment and evaluation are $O(N)$ rather than $O(2^k)$.

---

## The Bind Operation

The key operation in sumcheck is **binding**: fixing one variable to a challenge value $r$ and folding the polynomial:

$$P'(x_2, \ldots, x_k) = P(r, x_2, \ldots, x_k) = (1 - r) P(0, x_2, \ldots, x_k) + r \cdot P(1, x_2, \ldots, x_k)$$

In code:
```rust
poly.bind_parallel(r, BindingOrder::LowToHigh);
```

After $k$ bindings, the polynomial has been reduced to a single scalar: the evaluation at the full challenge point $(r_1, \ldots, r_k)$.

`bind_parallel` uses Rayon for parallel execution across the $2^{k-1}$ pairs of adjacent evaluations.

---

## Equality Polynomial

The equality polynomial $\widetilde{eq}(r, x)$ is defined as:
$$\widetilde{eq}(r, x) = \prod_{i=1}^{k} (r_i x_i + (1 - r_i)(1 - x_i))$$

It evaluates to 1 if $x = r$ on the Boolean hypercube and 0 otherwise. It is used in every sumcheck as the "selector" that restricts the sum to look like a polynomial evaluation at $r$.

`GruenSplitEqPolynomial` is an optimized lazy representation that avoids materializing the full $2^k$-element equality polynomial. It uses the Gruen split:

$$\widetilde{eq}(r, x) = \widetilde{eq}(r_\text{low}, x_\text{low}) \cdot \widetilde{eq}(r_\text{high}, x_\text{high})$$

where $r_\text{low}, r_\text{high}$ are the lower and upper halves of $r$. This reduces the binding cost per round from $O(2^k)$ to $O(2^{k/2})$ in many cases.

---

## UniPoly and CompressedUniPoly

Each sumcheck round produces a univariate polynomial message:

```rust
pub struct UniPoly<F> {
    coeffs: Vec<F>,  // coefficients of h(X) = c_0 + c_1 X + c_2 X^2 + ...
}

pub struct CompressedUniPoly<F> {
    // Stores all coefficients EXCEPT the linear term c_1
    // c_1 can be recovered from the claimed sum and h(0), h(1)
    coeffs_except_linear_term: Vec<F>,
}
```

The linear term is omitted from the proof to save one field element per round. The verifier reconstructs it from the round constraint $h(0) + h(1) = \text{previous\_claim}$.
