# IPA and Bulletproofs

**References:**
- Bootle, Cerulli, Chaidos, Groth, and Petit, "Efficient Zero-Knowledge Arguments for Arithmetic Circuits in the Discrete Log Setting" (EUROCRYPT 2016).
- Bunz, Bootle, Boneh, Poelstra, Wuille, and Maxwell, "Bulletproofs: Short Proofs for Confidential Transactions and More" (IEEE S&P 2018).

Presentation follows *Proofs, Arguments, and Zero-Knowledge*, Section 14.4.

## Overview

Bulletproofs is a polynomial commitment scheme in which the commitment size is **constant** (one group element) and the evaluation proof length is **logarithmic** ($O(\log n)$ group elements). The verifier's runtime, however, is **linear** ($O(n)$ group operations). This is the fundamental tradeoff relative to [Hyrax](./hyrax.md) (which trades constant commitments for $O(\sqrt{n})$ commitments to get $O(\sqrt{n})$ verification) and [Dory](./dory.md) (which uses pairings to achieve polylogarithmic verification).

The scheme is transparent: no trusted setup, no toxic waste.

## Warm-Up: Proof of Knowledge of a Vector Commitment Opening

The core building block is a protocol for proving knowledge of an opening $u \in \mathbb{F}_p^n$ of a generalised Pedersen commitment:

$$c_u = \text{Com}(u) = \langle u, \mathbf{g} \rangle = \sum_{i=1}^{n} u_i \cdot g_i,$$

where $\mathbf{g} = (g_1, \ldots, g_n) \in \mathbb{G}^n$ are public generators and $\mathbb{G}$ is written additively.

### The Recursive Halving Protocol

The protocol proceeds in $\log_2 n$ rounds:

1. Write $u = u_L \circ u_R$ and $\mathbf{g} = \mathbf{g}_L \circ \mathbf{g}_R$ (left and right halves), so $\langle u, \mathbf{g} \rangle = \langle u_L, \mathbf{g}_L \rangle + \langle u_R, \mathbf{g}_R \rangle$.

2. The prover sends **cross-terms** $v_L, v_R \in \mathbb{G}$ claimed to equal $\langle u_L, \mathbf{g}_R \rangle$ and $\langle u_R, \mathbf{g}_L \rangle$.

3. The verifier sends a random challenge $\alpha \in \mathbb{F}_p$.

4. Both sides fold:
$$u' = \alpha u_L + \alpha^{-1} u_R, \qquad \mathbf{g}' = \alpha^{-1} \mathbf{g}_L + \alpha \mathbf{g}_R.$$

5. The verifier updates the commitment:
$$c_{u'} := c_u + \alpha^2 v_L + \alpha^{-2} v_R.$$

6. Recurse on $\langle u', \mathbf{g}' \rangle = c_{u'}$ with vectors of half the length.

After $\log_2 n$ rounds, $u$ and $\mathbf{g}$ have length 1 and the prover reveals $u$ directly.

### Why This Is Sound

The actual inner product after folding is:

$$\langle u', \mathbf{g}' \rangle = \langle u, \mathbf{g} \rangle + \alpha^2 \langle u_L, \mathbf{g}_R \rangle + \alpha^{-2} \langle u_R, \mathbf{g}_L \rangle.$$

If the prover sends honest cross-terms ($v_L = \langle u_L, \mathbf{g}_R \rangle$, $v_R = \langle u_R, \mathbf{g}_L \rangle$), then $\langle u', \mathbf{g}' \rangle = c_{u'}$ exactly. If the cross-terms are dishonest, the commitment equation $c_{u'} = \langle u', \mathbf{g}' \rangle$ becomes a degree-4 polynomial in $\alpha$; two distinct polynomials can agree on at most 4 values, so with high probability over random $\alpha$, the prover is caught.

### Costs

| | Cost |
|---|---|
| Rounds | $\log_2 n$ |
| Prover messages | 2 group elements per round |
| Prover time | $O(n)$ group operations total |
| Verifier time | $O(n)$ group operations (recomputing $\mathbf{g}'$ each round) |

The $O(n)$ verifier cost comes from updating the generator vector: computing $\mathbf{g}' = \alpha^{-1} \mathbf{g}_L + \alpha \mathbf{g}_R$ requires a number of group exponentiations proportional to $|\mathbf{g}|$, and summing over all rounds gives $O(n + n/2 + n/4 + \cdots) = O(n)$.

### Knowledge Extraction via the Forking Lemma

Knowledge-soundness is proved via a generalised forking lemma. The extractor constructs a **3-transcript tree** $\mathcal{T}$ of depth $\log_2 n$: each non-leaf node has 3 children (corresponding to 3 different verifier challenges). From the tree, the extractor works bottom-up, using the three equations at each node to "cancel the cross-terms" and reconstruct the committed vector $u$.

Concretely, at each node with children having challenges $\alpha_1, \alpha_2, \alpha_3$, the matrix

$$A = \begin{bmatrix} 1 & \alpha_1^2 & \alpha_1^{-2} \\ 1 & \alpha_2^2 & \alpha_2^{-2} \\ 1 & \alpha_3^2 & \alpha_3^{-2} \end{bmatrix}$$

is invertible (its determinant is a Vandermonde-like expression, nonzero when the $\alpha_i$ are distinct). The first row of $A^{-1}$ gives coefficients $\beta_1, \beta_2, \beta_3$ such that $u = \sum_{i=1}^{3} (\beta_i \cdot \alpha_i^{-1} \cdot u_i) \circ (\beta_i \cdot \alpha_i \cdot u_i)$, recovering the opening.

(*Proofs, Arguments, and Zero-Knowledge*, Theorem 14.1)

## Extending to a Polynomial Commitment Scheme

Protocol 13 in the book extends the knowledge-of-opening protocol to a full polynomial commitment:

The prover must establish not only $\sum u_i g_i = c_u$ (knowledge of the coefficient vector) but also $\sum u_i y_i = v$ (correctness of the evaluation), where $y$ is the evaluation vector derived from the query point $z$.

The key observation is that both equations have the same form: they are inner products of $u$ with different vectors ($\mathbf{g}$ and $y$). So the protocol runs **two parallel instances** of the recursive halving, using the same verifier challenges in both, with $y$ playing the role of a second "generator vector" whose entries are field elements rather than group elements.

### Tensor Structure of $y$

For a univariate polynomial, $y = (1, z, z^2, \ldots, z^{n-1})$. For a multilinear polynomial at point $r = (r_1, \ldots, r_\ell)$, $y = (\chi_1(r), \ldots, \chi_{2^\ell}(r))$ where $\chi_i$ are the Lagrange basis polynomials. In both cases, $y$ has **tensor structure**: it can be written as a Kronecker product of $\ell$ two-element vectors. This means the verifier can compute the folded $y'$ in $O(\log n)$ time per round (just updating two scalars), rather than $O(n)$.

This tensor structure does not help with the generator vector $\mathbf{g}$, which is why the verifier runtime remains $O(n)$.

## Connection to Dory

The book (Section 14.4.2, "Dory: Reducing Verifier Time to Logarithmic") explains how Lee's Dory reduces the verifier's $O(n)$ runtime to $O(\log n)$ by:

1. Producing a logarithmic-sized **verification key** from the length-$n$ generator vector $\mathbf{g}$ via a one-time pre-processing phase.
2. Using the verification key so that the verifier never needs to touch $\mathbf{g}$ directly during the protocol.

This is covered in detail in the [Dory](./dory.md) page.

## IPA-Sumcheck Connection

Eagen and Gabizon (2025/1325) observe that the recursive halving in IPA is structurally identical to sum-check, but operating "in the exponent" over a group $G$. This perspective yields:

- A multilinear PCS with Halo-style accumulation for multilinear (not just univariate) evaluation claims.
- A polylogarithmic verifier by replacing the $O(n)$ MSM with a "group variant of BaseFold", reducing verifier time from $O(n)$ to $O(\lambda \cdot \log^2 n)$.
- The tradeoff: the prover performs an additional $4n$ scalar multiplications for deciding accumulated claims.

See [Revisiting the IPA-Sumcheck Connection](../papers/ipa-sumcheck-connection.md) for details.

## Summary

| | Bulletproofs | Hyrax | Dory |
|---|---|---|---|
| Commitment size | $O(1)$ | $O(\sqrt{n})$ | $O(1)$ |
| Proof size | $O(\log n)$ | $O(\sqrt{n})$ | $O(\log^2 n)$ |
| Verifier time | $O(n)$ | $O(\sqrt{n})$ | $O(\log^2 n)$ or $O(\log n)$ |
| Setup | Transparent | Transparent | Transparent |
| Pairings | No | No | Yes |

(*Proofs, Arguments, and Zero-Knowledge*, Sections 14.4, 14.4.2)
