# Hyrax

**Reference:** Wahby, Tzialla, Shelat, Thaler, and Walfish, "Doubly-Efficient zkSNARKs without Trusted Setup" (IEEE S&P 2018). Presentation follows *Proofs, Arguments, and Zero-Knowledge*, Section 14.3.

## Overview

Hyrax is a transparent polynomial commitment scheme that trades off commitment size for verification time. Compared to the constant-commitment / linear-verification scheme of Section 14.2 and to [Bulletproofs](./ipa.md) (constant commitment, logarithmic proof, linear verification), Hyrax sets both commitment size and verification time to $O(\sqrt{n})$ group elements and operations respectively.

The key idea is to exploit the **vector-matrix-vector** (VMV) structure of polynomial evaluation (see [Dory, VMV](./dory.md#the-vmv-vector-matrix-vector-structure)) at the commitment level, committing to each column of the coefficient matrix separately.

## Construction

### Commitment Phase

Let $u \in \mathbb{F}_p^n$ be the coefficient vector of a polynomial $q$, and assume $n = m^2$ is a perfect square. View $u$ as an $m \times m$ matrix. The committer sends $m$ Pedersen vector commitments, one per column:

$$c_1 = \text{Com}(u_1, r_1), \ldots, c_m = \text{Com}(u_m, r_m),$$

where $u_j \in \mathbb{F}_p^m$ is the $j$-th column and each commitment is computed as

$$\text{Com}(u_j, r_j) = h^{r_j} \cdot \prod_{k=1}^{m} g_k^{u_{j,k}}$$

for public generators $g_1, \ldots, g_m, h \in \mathbb{G}$.

The commitment size is $m = \sqrt{n}$ group elements (compared to 1 for Bulletproofs or Section 14.2).

### Evaluation Phase

Recall from the [VMV structure](./dory.md#the-vmv-vector-matrix-vector-structure) that for any evaluation point $z$, the evaluation $q(z) = \langle u, y \rangle$ can be expressed as

$$q(z) = \mathbf{b}^T \cdot u \cdot \mathbf{a},$$

where $\mathbf{a}, \mathbf{b} \in \mathbb{F}_p^m$ are $m$-dimensional vectors derived from $z$.

The prover sends a commitment $c^*$ to the intermediate vector $u \cdot \mathbf{a}$ (the matrix-vector product). Using the additive homomorphism of Pedersen commitments, the verifier can on its own compute a commitment to $u \cdot \mathbf{a}$ from the column commitments:

$$\prod_{j=1}^{m} \text{Com}(u_j)^{a_j} = \text{Com}(u \cdot \mathbf{a}).$$

The verifier then needs to verify that $c^*$ is a commitment to $\mathbf{b}^T \cdot (u \cdot \mathbf{a})$, i.e., that the inner product $\langle u \cdot \mathbf{a}, \mathbf{b} \rangle = v$. This is exactly the kind of opening problem that the [Bulletproofs protocol](./ipa.md) (Section 14.4) was designed to solve, applied to vectors of length $m = \sqrt{n}$.

### Why the Verifier Is $O(\sqrt{n})$

In Bulletproofs applied to length-$n$ vectors, the verifier performs $O(n)$ group operations. In Hyrax, Bulletproofs is applied to length-$\sqrt{n}$ vectors, so the verifier performs $O(\sqrt{n})$ group operations for the opening proof. The verifier also performs an $O(\sqrt{n})$-sized MSM to homomorphically derive the commitment to $u \cdot \mathbf{a}$ from the column commitments. Total: $O(\sqrt{n})$.

## Costs

| | Cost |
|---|---|
| Commitment size | $O(\sqrt{n})$ group elements |
| Proof size | $O(\sqrt{n})$ field elements |
| Prover | $O(n)$ group operations (computing column commitments) |
| Verifier | $O(\sqrt{n})$ group operations (two MSMs of size $\sqrt{n}$) |
| Setup | Transparent (random generators) |

## Role in Jolt Recursion

Hyrax over Grumpkin is the natural choice for the recursion layer in Jolt's [Efficient Recursion](../papers/efficient-recursion.md) architecture, for three reasons:

1. **No pairings.** Grumpkin does not support pairings, ruling out Dory and KZG. Hyrax requires only Pedersen commitments and scalar multiplications.

2. **Curve cycle compatibility.** Grumpkin's scalar field is $\mathbb{F}_q$ (the base field of BN254). The auxiliary SNARK proving Dory verification steps operates natively over $\mathbb{F}_q$, so Hyrax over Grumpkin is the natural fit. See [Field Arithmetic](../fundamentals/field-arithmetic.md) for why this matters.

3. **$\sqrt{n}$ verification.** The Hyrax opening check consists of two $O(\sqrt{n})$-sized MSMs. For the Jolt recursion sizing ($n_{\text{pack}} = 21$ variables, $\sim$2.1M coefficients), each MSM has size $\sim 2^{10.5} \approx 1448$. This dominates the extended verifier's cycle budget at ~62%, but is manageable (around 109M RISC-V cycles).

### Concrete Sizing in the Recursion Paper

After family packing and [prefix packing](../papers/efficient-recursion.md#3-prefix-packing), all committed polynomials across all operation instances collapse into a single $n_{\text{pack}} = 21$-variable dense polynomial ($\sim$2.1M coefficients). The Hyrax commitment is a vector of $2^{n_{\text{pack}}/2}$ Pedersen commitments over Grumpkin, each computed as an MSM of size $2^{n_{\text{pack}}/2}$.

The prover's commitment cost ($\sim$1s) dominates the recursion prover's total overhead of 1.5-1.9s. The opening proof itself is negligible (< 5ms).

## Relationship to Other Schemes

| Scheme | Commitment | Proof | Verifier | Pairings |
|---|---|---|---|---|
| Section 14.1 (linear) | $O(n)$ | $O(1)$ | $O(n)$ | No |
| Section 14.2 (constant) | $O(1)$ | $O(n)$ | $O(n)$ | No |
| Hyrax (Section 14.3) | $O(\sqrt{n})$ | $O(\sqrt{n})$ | $O(\sqrt{n})$ | No |
| Bulletproofs (Section 14.4) | $O(1)$ | $O(\log n)$ | $O(n)$ | No |
| Dory (Section 15.4) | $O(1)$ | $O(\log^2 n)$ | $O(\log n)$ | Yes |
| KZG (Section 15.2) | $O(1)$ | $O(1)$ | $O(1)$ | Yes (trusted) |

Hyrax occupies a "sweet spot" for settings where pairings are unavailable (e.g., over Grumpkin in the recursion layer) and $O(n)$ verification is too expensive (e.g., when the verifier runs inside a zkVM).

(*Proofs, Arguments, and Zero-Knowledge*, Sections 14.3, 14.4; Efficient Recursion for the Jolt zkVM, Sections 4.3, 4.5)
