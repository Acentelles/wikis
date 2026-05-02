# HyperKZG

**References:**
- Kate, Zaverucha, and Goldberg, "Constant-Size Commitments to Polynomials and Their Applications" (ASIACRYPT 2010) [KZG10], for the univariate foundation.
- Papamanthou, Shi, and Tamassia, "Signatures of Correct Computation" (PST13), for the multilinear extension.
- Bootle, Chiesa, Hu, and Orru, "Gemini: Elastic SNARKs for Diverse Environments" (IACR ePrint 2022/420), for the HyperKZG instantiation used in practice.

Presentation of the univariate and multilinear KZG follows *Proofs, Arguments, and Zero-Knowledge*, Sections 15.2 and 15.3.

## KZG: The Univariate Foundation

### Trusted Setup

The structured reference string (SRS) consists of encodings in $\mathbb{G}$ of all powers of a random secret $\tau \in \mathbb{F}_p$:

$$\text{SRS} = (g, g^\tau, g^{\tau^2}, \ldots, g^{\tau^D}).$$

The value $\tau$ is **toxic waste**: anyone who knows it can break the binding property. It must be discarded after generation, typically via a multi-party ceremony.

### Commitment

To commit to a polynomial $q(X) = \sum_{i=0}^{D} c_i X^i$ over $\mathbb{F}_p$, the committer computes

$$c = g^{q(\tau)} = \prod_{i=0}^{D} \left(g^{\tau^i}\right)^{c_i}.$$

This is a single group element. Note that the committer does not know $\tau$, but can compute $g^{q(\tau)}$ from the SRS via a multi-exponentiation.

### Opening

To open at input $z \in \mathbb{F}_p$ to value $v$ (i.e., to prove $q(z) = v$), the committer computes a **witness polynomial**

$$w(X) := \frac{q(X) - v}{X - z},$$

which is a polynomial of degree at most $D - 1$ (it divides evenly because $q(z) = v$ implies $(X - z) | (q(X) - v)$). The committer sends $y = g^{w(\tau)}$.

### Verification

The verifier checks:

$$e(c \cdot g^{-v}, g) = e(y, g^\tau \cdot g^{-z}).$$

This is a single pairing equation. Correctness follows from bilinearity:

$$e(c \cdot g^{-v}, g) = e(g^{q(\tau) - v}, g) = e(g^{w(\tau) \cdot (\tau - z)}, g) = e(g^{w(\tau)}, g^{\tau - z}) = e(y, g^\tau \cdot g^{-z}).$$

### Binding

Binding rests on the **$D$-Strong Diffie-Hellman (SDH) assumption**: given the SRS $(g, g^\tau, \ldots, g^{\tau^D})$, no efficient algorithm can compute a pair $(z, g^{1/(\tau - z)})$. Intuitively, the SRS lets the prover "evaluate in the exponent" but not "divide in the exponent."

If one could open $c$ to two different values $v, v'$ at the same point $z$, one could recover $g^{1/(\tau - z)}$, breaking SDH.

### Extractability

The basic scheme is binding but not necessarily extractable (the prover is bound to *some* function, but not provably to a degree-$D$ polynomial). Extractability can be achieved by doubling the SRS to include powers of $\alpha \tau$:

$$\{(g, g^\alpha), (g^\tau, g^{\alpha\tau}), (g^{\tau^2}, g^{\alpha\tau^2}), \ldots, (g^{\tau^D}, g^{\alpha\tau^D})\},$$

and requiring the commitment to be a pair $(U, V) = (g^{q(\tau)}, g^{\alpha q(\tau)})$. The **Power Knowledge of Exponent (PKoE)** assumption then guarantees that any efficient committer "knows" the coefficients $c_0, \ldots, c_D$.

### Key Properties

- **Constant-size commitments**: 1 group element (or 2 for the extractable variant).
- **Constant-size proofs**: 1 group element.
- **Constant-time verification**: 1 pairing equation (2 pairings).
- **Trusted setup**: SRS of size $D + 1$, with toxic waste.

## Extension to Multilinear Polynomials (Section 15.3)

KZG extends to multilinear polynomials $q : \mathbb{F}_p^\ell \to \mathbb{F}_p$ (where $n = 2^\ell$ is the number of coefficients) via the **witness polynomial decomposition** of Papamanthou, Shi, and Tamassia.

### The SRS

The SRS consists of encodings in $\mathbb{G}$ of all powers of $r$ applied to all $2^\ell$ Lagrange basis polynomials $\chi_1, \ldots, \chi_{2^\ell}$ evaluated at a random point $r \in \mathbb{F}_p^\ell$:

$$\text{SRS} = (g^{\chi_1(r)}, \ldots, g^{\chi_{2^\ell}(r)}).$$

### Commitment

For a multilinear polynomial $q(X) = \sum_{i=0}^{2^\ell} c_i \chi_i(X)$, the commitment is

$$c = g^{q(r)} = \prod_{i=0}^{2^\ell} \left(g^{\chi_i(r)}\right)^{c_i},$$

which is again a single group element.

### Opening via Witness Polynomials

To prove $q(z) = v$ for $z = (z_1, \ldots, z_\ell) \in \mathbb{F}_p^\ell$, the committer uses the multilinear analogue of the univariate division identity:

$$q(X) - v = \sum_{i=1}^{\ell} (X_i - z_i) \cdot w_i(X_1, \ldots, X_\ell),$$

where $w_1, \ldots, w_\ell$ are multilinear **witness polynomials** that exist and are unique (Fact 15.1 in the book). The committer sends $y_i = g^{w_i(r)}$ for each $i$.

### Verification

The verifier checks:

$$e(c \cdot g^{-v}, g) = \prod_{i=1}^{\ell} e(y_i, g^{r_i} \cdot g^{-z_i}).$$

This requires $\ell$ pairings plus access to the verification key $(g^{r_1}, \ldots, g^{r_\ell})$, which is a subset of the SRS (since each "dictator function" $(X_1, \ldots, X_\ell) \mapsto X_i$ is a Lagrange basis polynomial).

### Costs

| | Cost |
|---|---|
| Commitment size | $O(1)$ group elements |
| Proof size | $\ell = \log_2 n$ group elements |
| Prover | $O(2^\ell)$ field operations (computing witness polynomials) + $O(\ell)$ group exponentiations |
| Verifier | $O(\ell)$ pairings |
| SRS size | $O(2^\ell) = O(n)$ |

## HyperKZG: Practical Multilinear KZG

HyperKZG (from the Gemini paper, IACR 2022/420) is the practical instantiation of multilinear KZG used in deployed systems like Nova and, in the Jolt recursion architecture, as the wrapper PCS.

The key optimisation relative to the textbook Section 15.3 presentation is to reduce the $\ell$ witness polynomial commitments to a constant number via batching and recursive variable elimination:

1. **Variable elimination.** Instead of computing all $\ell$ witness polynomials upfront, HyperKZG processes one variable at a time. In each round, fixing variable $X_i$ at challenge $r_i$ produces a quotient polynomial commitment and reduces the polynomial's number of variables by one.

2. **Batching.** The $\ell$ quotient commitments are batched into a constant number of pairing checks using random linear combinations from the Fiat-Shamir transcript.

The result is a scheme with:

| | Cost |
|---|---|
| Commitment size | $O(1)$ group elements in $\mathbb{G}_1$ |
| Proof size | $O(\ell)$ group elements, batched to $O(1)$ pairing checks |
| Prover | $O(n)$ group operations |
| Verifier | $O(1)$ pairing checks |
| SRS | Trusted, size $O(n)$ |

## Role in Wrapping

HyperKZG combined with Spartan is the proposed fast wrapper in the Jolt [Efficient Recursion](../papers/efficient-recursion.md) architecture:

1. **Spartan** handles the R1CS constraint system produced by the extended Jolt verifier, reducing it to polynomial evaluation claims via sum-check.

2. **HyperKZG** verifies these evaluation claims with constant-size proofs.

3. The wrapper produces proofs of $\sim$5-6 KB, with expected proving time under a second on a laptop.

This is the middle ground between Groth16 ($\sim$128-byte proofs, per-circuit trusted setup, $\sim$10s proving) and unwrapped proofs ($\sim$80-100 KB, no setup). Spartan can be over 100x faster than Groth16 for proving.

| Wrapper | Proof size | Proving time | Setup |
|---|---|---|---|
| Groth16 | $\sim$128 bytes | $\sim$10s | Per-circuit trusted setup |
| Spartan + HyperKZG | $\sim$5 KB | < 1s | Universal SRS |
| None (unwrapped) | $\sim$80-100 KB | N/A | Transparent |

## Comparison with Other PCS

| | KZG (univariate) | KZG (multilinear) | HyperKZG | Dory | Hyrax | Bulletproofs |
|---|---|---|---|---|---|---|
| Commitment | $O(1)$ | $O(1)$ | $O(1)$ | $O(1)$ | $O(\sqrt{n})$ | $O(1)$ |
| Proof | $O(1)$ | $O(\ell)$ | $O(1)$ batched | $O(\log^2 n)$ | $O(\sqrt{n})$ | $O(\log n)$ |
| Verifier | $O(1)$ pairings | $O(\ell)$ pairings | $O(1)$ pairings | $O(\log n)$ | $O(\sqrt{n})$ | $O(n)$ |
| Setup | Trusted | Trusted | Trusted | Transparent | Transparent | Transparent |

The tradeoff is clear: KZG-family schemes offer the smallest proofs and fastest verification, at the cost of a trusted setup. For on-chain verification where proof size and verifier gas cost dominate, this is compelling. For recursion layers where transparency matters, [Hyrax](./hyrax.md) or [Dory](./dory.md) are preferred.

(*Proofs, Arguments, and Zero-Knowledge*, Sections 15.2, 15.3; Gemini, IACR 2022/420; Efficient Recursion for the Jolt zkVM, Section 5.2)
