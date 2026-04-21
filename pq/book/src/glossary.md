# Glossary

Short definitions for terms that recur across the notes. Entries are kept
deliberately terse; follow the links into individual papers for proper
treatments.

## Unimodular matrix

A square integer matrix $U \in \mathbb{Z}^{n \times n}$ with
$\det(U) \in \{-1, +1\}$. Equivalently, $U \in \mathrm{GL}_n(\mathbb{Z})$,
i.e., $U$ is invertible and its inverse also has integer entries.

Unimodular matrices are exactly the $\mathbb{Z}$-linear change-of-basis
transformations on the lattice $\mathbb{Z}^n$. They act on lattice bases
without changing the lattice itself, and on Gram matrices by
$Q \mapsto U^\top Q U$, which is the central operation in the Lattice
Isomorphism Problem (see the
[LIP arithmetisation idea](./ideas/lip-arithmetisation.md)).

Lattice term, included here because LIP shows up in the SNARK-arithmetisation
entries on this wiki.

## Cyclotomic conductor

The integer $n$ that indexes a cyclotomic field $\mathbb{Q}(\zeta_n)$ and
its cyclotomic polynomial $\Phi_n(X)$. The conductor determines:

- **Ring dimension**: $D = \varphi(n)$, the degree of $\Phi_n$.
- **Ring arithmetic**: the shape of $\Phi_n$. Power-of-two conductors give
  $\Phi_{2^k}(X) = X^{2^{k-1}} + 1$ (simple negacyclic reduction); other
  conductors give more complex polynomials (e.g.
  $\Phi_{81} = X^{54} + X^{27} + 1$).
- **Splitting of primes**: whether a modulus $q$ splits completely in
  $\mathbb{Z}[\zeta_n]$, which is needed for NTT-friendly arithmetic and
  for CRT decompositions such as $\Lambda_q \cong \prod M_2(\mathbb{F}_q)$
  in the [ABBA](./papers/abba.md) commitment scheme.
- **Galois structure**: the automorphism group
  $\mathrm{Gal}(\mathbb{Q}(\zeta_n)/\mathbb{Q}) \cong (\mathbb{Z}/n\mathbb{Z})^\times$,
  including complex conjugation $\theta : \zeta_n \mapsto \zeta_n^{-1}$.

Common conductors in lattice cryptography:

| Conductor | $\Phi_n$ | $D = \varphi(n)$ | Used by |
|---|---|---|---|
| $2^k$ (e.g. 256) | $X^{2^{k-1}} + 1$ | $2^{k-1}$ | CRYSTALS-Kyber/Dilithium |
| 128 | $X^{64} + 1$ | 64 | ABBA candidate for Neo |
| $3^k$ (e.g. 81) | $X^{54} + X^{27} + 1$ | 54 | Nightstream (Neo) |
| $p$ prime | $1 + X + \cdots + X^{p-1}$ | $p - 1$ | some NTRU variants |

The **parity of the conductor** matters for ABBA's Neo integration: the
identity $\dim(O_L) = \dim(\Lambda)$ holds if and only if $n$ is even
(see [the parity obstruction](./papers/abba.md#the-parity-obstruction)).
