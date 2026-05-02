# Dory

**Reference:** Jonathan Lee, "Dory: Efficient, Transparent Arguments for Generalised Inner Products and Polynomial Commitments" (TCC 2021). Presentation follows *Proofs, Arguments, and Zero-Knowledge*, Section 15.4.

## Overview

Dory is a transparent polynomial commitment scheme based on inner-product arguments over pairing groups. It achieves logarithmic proof size and verification time with compact ($O(1)$) commitments in the pairing target group $G_T$, at the cost of a linear-time transparent pre-processing phase (no toxic waste).

Compared to Bulletproofs (Section 14.4 of the book): both are transparent and have logarithmic proof sizes, but Bulletproofs have linear verification time while Dory achieves (poly)logarithmic verification. Concretely, Dory's proofs are larger by a significant constant factor.

## Building Blocks

### Inner-Pairing-Product Commitments (IPPCom)

The foundation of Dory is the ability to commit to vectors of *group elements* (not just field elements) using pairings.

Let $\mathbb{G}_1, \mathbb{G}_2, \mathbb{G}_T$ be a pairing-friendly triple with bilinear map $e : \mathbb{G}_1 \times \mathbb{G}_2 \to \mathbb{G}_T$. For a vector $w \in \mathbb{G}_1^n$ and a fixed public vector $\mathbf{g} = (g_1, \ldots, g_n) \in \mathbb{G}_2^n$, define

$$\text{IPPCom}(w) = \langle w, \mathbf{g} \rangle = \sum_{i=1}^{n} e(w_i, g_i).$$

This is a single element of $\mathbb{G}_T$. Intuitively, $e(w_i, g_i)$ acts as a Pedersen commitment to $w_i$ despite $w_i$ being a group element rather than a field element.

**Binding.** Under the Symmetric External Diffie-Hellman (SXDH) assumption, IPPCom is computationally binding. The book proves this by reducing a binding break to a DDH break in $\mathbb{G}_1$.

### Committing to Field Elements via Pairings

To commit to a vector of field elements $v \in \mathbb{F}_p^n$, first convert it to a vector of group elements. Let $h$ be any element of $\mathbb{G}_1$. Define $w(v)$ entry-wise as $w(v)_i = v_i \cdot h$. Then

$$\text{IPPCom}(v) := \text{IPPCom}(w(v)) = \sum_{i=1}^{n} e(v_i \cdot h, g_i).$$

Since $v \mapsto w(v)$ is bijective, this is a binding commitment to $v$.

**Efficiency.** Computing IPPCom($v$) directly requires $n$ pairings. But there is a faster way: compute a Pedersen-vector commitment $c = \sum_{i=1}^{n} v_i g_i$ in $\mathbb{G}_2$, then apply the bilinear map once: $e(h, c)$. By bilinearity these are equal. This reduces the cost to one $\mathbb{G}_2$ MSM plus one pairing.

## The VMV (Vector-Matrix-Vector) Structure

This is the key to understanding Dory's commitment and opening.

### Polynomial as a Matrix

A polynomial $q(X) = \sum_{i=0}^{n-1} a_i X^i$ of degree $n-1$ (where $n = 2^\ell$) is arranged as a $2^\nu \times 2^\sigma$ matrix $u$, with $\nu + \sigma = \ell$. Typically $\nu = \sigma = \ell/2$, giving a $\sqrt{n} \times \sqrt{n}$ layout.

### Evaluation as a Vector-Matrix-Vector Product

To evaluate $q(z)$ at a point $z \in \mathbb{F}_p$, we express the evaluation as

$$q(z) = \mathbf{b}^T \cdot u \cdot \mathbf{x},$$

where $\mathbf{b} \in \mathbb{F}_p^{2^\nu}$ and $\mathbf{x} \in \mathbb{F}_p^{2^\sigma}$ are vectors derived from $z$ using the tensor structure of the Lagrange basis polynomials. Specifically, $\mathbf{x} = \bigotimes_{i=1}^{\sigma}(1, r^{2^i})$ and $\mathbf{b} = \bigotimes_{i=1}^{\nu}(1, r^{2^{\sigma+i}})$ (these are the "left" and "right" halves of the equality polynomial's evaluation vector).

This is why it is called VMV: the polynomial evaluation is literally a *vector-matrix-vector* product.

### Commitment

The commitment to the polynomial is an IPPCom of the matrix's columns (or rows) against the public generator vector. It is a single $\mathbb{G}_T$ element.

The prover computes column commitments $c_j = \sum_{i=1}^{m} u_{i,j} \cdot h_i \in \mathbb{G}_1$ for each column $j$, then the full commitment is $c^* = \sum_{j=1}^{m} e(c_j, g_j) \in \mathbb{G}_T$.

### Opening

To open an evaluation $q(z) = v$, the prover must convince the verifier that $\mathbf{b}^T \cdot u \cdot \mathbf{x} = v$. The prover sends the intermediate vector $\vec{w} = u \cdot \mathbf{x} \in \mathbb{F}_p^{2^\nu}$ (the matrix-vector product), and the protocol reduces to proving:

1. $c_q = \text{IPPCom}(w(u \cdot \mathbf{x}))$, i.e., $\vec{w}$ is consistent with the committed matrix.
2. $\langle \vec{w}, \mathbf{b} \rangle = v$, i.e., the inner product gives the claimed evaluation.

Claim (2) is a simple inner-product check. Claim (1) is the hard part, and this is where the Bulletproofs-style recursive halving protocol comes in.

## The Knowledge-of-Opening Protocol

### Connection to Bulletproofs

The core protocol proves knowledge of a vector $u^{(0)} \in \mathbb{G}^n$ satisfying

$$c_{u^{(0)}} = \langle u^{(0)}, \mathbf{g}^{(0)} \rangle,$$

where $\mathbf{g}^{(0)} \in \mathbb{G}^n$ is the public commitment key. This is the same problem Bulletproofs solves, but with a critical difference: the verifier does not know $\mathbf{g}^{(0)}$ explicitly (it is too large for a logarithmic-time verifier to read). Instead, the verifier only knows *pre-computed commitments* to the left and right halves of $\mathbf{g}^{(0)}$ under a sequence of commitment keys.

### Pre-Processing Phase

A transparent, linear-time pre-processing phase computes and publishes commitments to the halves of $\mathbf{g}^{(0)}$ at each level of the recursion. Concretely:

- Split $\mathbf{g}^{(0)} = (\mathbf{g}_L^{(0)}, \mathbf{g}_R^{(0)}) \in \mathbb{G}^{n/2} \times \mathbb{G}^{n/2}$.
- Compute $\Delta_L^{(1)} = \langle \mathbf{g}_L^{(0)}, \Gamma^{(1)} \rangle$ and $\Delta_R^{(1)} = \langle \mathbf{g}_R^{(0)}, \Gamma^{(1)} \rangle$ using a public, randomly chosen commitment key $\Gamma^{(1)} \in \mathbb{G}^{n/2}$.
- Recurse: write $\Gamma^{(1)} = (\Gamma_L^{(1)}, \Gamma_R^{(1)})$ and compute $\Delta_L^{(2)}, \Delta_R^{(2)}$ for both halves, using a new key $\Gamma^{(2)} \in \mathbb{G}^{n/4}$.
- Continue for $\log n$ iterations. Each iteration produces two AFGHO commitments ($\mathbb{G}_T$ elements).

Any entity can perform the pre-processing and publish the results. Anyone can also verify the published commitments, raising an alarm if a discrepancy is found. This is what makes Dory *transparent*: there is no trusted setup and no toxic waste.

A logarithmic-time verifier stores only the $2 \log n$ pre-processing commitments $\Delta_L^{(i)}, \Delta_R^{(i)}$ and the final (length-1) commitment key $\mathbf{g}^{(\log n)}$.

### Recursive Rounds

At the start of round $i$, the prover claims to know vectors $u^{(i-1)}, \mathbf{g}^{(i-1)} \in \mathbb{G}^{n/2^{i-1}}$ satisfying three equations:

$$\langle u^{(i-1)}, \mathbf{g}^{(i-1)} \rangle = c_1, \quad \langle u^{(i-1)}, \Gamma^{(i-1)} \rangle = c_2, \quad \langle \mathbf{g}^{(i-1)}, \Gamma^{(i-1)} \rangle = c_3.$$

Each round proceeds as follows:

1. The prover sends four AFGHO commitments $D_{1L}, D_{1R}, D_{2L}, D_{2R} \in \mathbb{G}_T$ (cross-terms for the left and right halves of $u^{(i-1)}$ and $\mathbf{g}^{(i-1)}$).

2. The verifier sends a random $\beta \in \mathbb{F}_p$. This collapses the three equations into a single equation via a random linear combination.

3. The prover and verifier set $w_1 = u^{(i-1)} + \beta \Gamma^{(i-1)}$ and $w_2 = \mathbf{g}^{(i-1)} + \beta^{-1} \Gamma^{(i-1)}$.

4. The verifier sends a random $\alpha \in \mathbb{F}_p$. The prover folds:
$$u^{(i)} = \alpha \cdot w_{1L} + \alpha^{-1} w_{1R}, \qquad \mathbf{g}^{(i)} = \alpha^{-1} \cdot w_{2L} + \alpha \cdot w_{2R}.$$

5. The verifier *homomorphically* updates $c_1', c_2', c_3'$ from the prover's messages and the pre-processing commitments $\Delta_L^{(i)}, \Delta_R^{(i)}$, without ever seeing $u^{(i)}$ or $\mathbf{g}^{(i)}$ explicitly.

After $\log n$ rounds, the vectors have length 1, and the prover reveals them for direct checking.

### Why Three Equations?

In standard Bulletproofs, the prover only needs to prove one inner-product relation $\langle u, \mathbf{g} \rangle = c$. But there, the verifier knows $\mathbf{g}$ explicitly and can recompute the folded key. In Dory, the verifier does *not* know $\mathbf{g}$, so it also needs to track $\langle u, \Gamma \rangle$ and $\langle \mathbf{g}, \Gamma \rangle$ to homomorphically derive the folded commitments from the pre-processing. The three equations enable the verifier to update all three commitments using only the $O(1)$ pre-processing outputs per round.

## Extending to a Polynomial Commitment

Section 15.4.4 of the book shows how the knowledge-of-opening protocol extends to a full polynomial commitment scheme:

1. The polynomial $q$ is committed via IPPCom of $w(a)$ (the coefficient vector embedded in $\mathbb{G}_1$), giving a single $\mathbb{G}_T$ element $c_q$.

2. To evaluate at $r \in \mathbb{F}_p$, the prover establishes two things in parallel:
   - $c_q = \text{IPPCom}(w(a))$, i.e., it knows an opening of the commitment (via the protocol above).
   - $q(r) = v$, i.e., the evaluation is correct (established in close analogy to Bulletproofs, exploiting the tensor structure of $y = (1, r, r^2, \ldots, r^{n-1})$).

3. The evaluation vector $y$ has tensor structure: $y = \bigotimes_{i=1}^{\log n} (\alpha_i + \alpha_i^{-1} r^{2^i})$, which the verifier can compute in $O(\log n)$ time.

## Reducing Pre-Processing to $O(\sqrt{n})$ via Matrix Commitments

Section 15.4.5 shows how to reduce the pre-processing time from $O(n)$ to $O(\sqrt{n})$ by combining with the VMV matrix structure:

- Instead of committing to the full coefficient vector, the prover commits to each column of the $\sqrt{n} \times \sqrt{n}$ coefficient matrix separately (as Pedersen-vector commitments in $\mathbb{G}_1$).
- The full commitment is an IPPCom of these column commitments.
- The evaluation query $q(z) = \mathbf{b}^T \cdot u \cdot \mathbf{x}$ is established by having the prover reveal $\vec{w} = u \cdot \mathbf{x}$, then opening a Pedersen commitment to $\vec{w}$ and checking $\langle \vec{w}, \mathbf{b} \rangle = v$.

This is the VMV structure: the prover sends the intermediate "matrix times right-vector" product, and the verifier checks it against the column commitments and the left vector.

## Costs

| | Cost |
|---|---|
| Commitment size | $O(1)$ in $\mathbb{G}_T$ |
| Proof size | $O(\log^2 n)$ scalar multiplications in $\mathbb{G}_T$ |
| Prover | $O(\sqrt{n})$ multi-pairings (dominating) |
| Verifier | $O(\log^2 n)$ scalar multiplications in $\mathbb{G}_T$ (Section 15.4.3), reducible to $O(\log n)$ (Section 15.4.6) |
| Pre-processing | $O(n)$ group operations (Section 15.4.3), reducible to $O(\sqrt{n})$ (Section 15.4.5) |
| Setup | Transparent (no toxic waste) |

## The Dory Verifier in the Efficient Recursion Paper

The Efficient Recursion paper (Section 4.1) instantiates the Dory verifier concretely and counts its operations for recursion purposes. Using the reduce-and-fold formulation, for $\sigma$ rounds:

| Group | Scalar Mul / Exp | Add / Mul |
|---|---|---|
| $G_1$ | $3\sigma + 4$ | $3\sigma + 3$ |
| $G_2$ | $3\sigma + 4$ | $3\sigma + 2$ |
| $G_T$ | $10\sigma + 4$ | $11\sigma + 5$ |
| Multi-pairing | 1 (size-3) | |

The verifier proceeds in three phases:

1. **VMV initialisation.** Extract the VMV message $(C, D_2, E_1)$ from the proof and compute $E_2 = [v]H_2$ where $v$ is the claimed evaluation.

2. **Reduce-and-fold.** For $\sigma$ rounds, update accumulators $(C, D_1, D_2, E_1, E_2)$ using prover messages, setup corrections, and transcript challenges $(\alpha, \beta)$. The $G_T$ accumulator update is:
$$C \leftarrow C \cdot \chi_i \cdot D_2^{\beta_i} \cdot D_1^{\beta_i^{-1}} \cdot C_+^{\alpha_i} \cdot C_-^{\alpha_i^{-1}}.$$

3. **Final check.** Verify one batched 3-pair multi-pairing. The VMV relation $D_2 = e(E_1, H_2)$ is deferred and folded into the final check using a fresh challenge $d$.

For a 1-billion-step trace ($n = 2^{38}$), $\sigma = 19$, yielding 649 group operations plus one 3-pairing.

## Recursion Bottleneck

Dory is the recursion bottleneck in Jolt. The verifier's $G_T$ exponentiations operate in $\mathbb{F}_{q^{12}}$, a degree-12 extension of the base field. When arithmetised inside a SNARK or compiled to RISC-V, each $G_T$ operation becomes extremely expensive due to non-native arithmetic (see [Field Arithmetic](../fundamentals/field-arithmetic.md)). This leads to billions of VM cycles for naive recursion.

The [Efficient Recursion](../papers/efficient-recursion.md) paper resolves this by proving all Dory verification steps in an auxiliary SNARK over Grumpkin (the cycle-curve partner of BN254), where $\mathbb{F}_q$ arithmetic is native. The $G_T$, $G_1$, and $G_2$ operations are proved via dedicated [group arithmetic PIOPs](../fundamentals/group-arithmetic.md), and the entire auxiliary SNARK's committed data fits in a single 21-variable polynomial (~2.1M coefficients).

(*Proofs, Arguments, and Zero-Knowledge*, Section 15.4; Efficient Recursion for the Jolt zkVM, Section 4.1)
