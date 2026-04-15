# Hachi: Lattice Multilinear Polynomial Commitments

> Ngoc Khanh Nguyen, George O'Rourke, Jiapeng Zhang
> *Hachi: Efficient Lattice-Based Multilinear Polynomial Commitments over Extension Fields*
> ePrint [2026/156](https://eprint.iacr.org/2026/156)

The PDF and clipping live in
`raw/lattices/2026-156-hachi-lattice-multilinear-polynomial-commitments/`.
Background entry; like [Spartan](./spartan.md), this lives in the
isogenies wiki only because other entries here reference it as a cost
model, not because it has anything to do with isogenies.

## TL;DR

Hachi is a **multilinear polynomial commitment scheme (MLE PCS)**, not a
full SNARK. It slots into Spartan (or HyperPlonk, Jolt, LatticeFold,
Nova) in the place where those schemes commit to the witness MLE
$\widetilde z$ and later prove an opening at a random point. The
headline claims:

- **Post-quantum**, under Module-SIS.
- **Proof size** $\approx 55$ KB, succinct and polynomial in the number
  of variables $\ell$ and the security parameter $\lambda$.
- **Verifier time** $\tilde O(\sqrt{2^\ell \cdot \lambda})$, a
  **12.5$\times$ practical speedup** over Greyhound (CRYPTO 2024),
  which was the previous state of the art in the same slot.
- Transparent (no trusted setup).

Two technical contributions drive the speedup:

1. A **ring-switching integration with Greyhound** based on Huang, Mao,
   Zhang (ePrint 2025). Running the sum-check protocol over a power-of-
   two cyclotomic ring $R_q = \mathbb{Z}_q[X]/(X^d + 1)$ is usually
   painful because lattice operations live in $R_q$ but sum-check
   soundness wants a large field. Ring switching lets the verifier
   perform *zero* multiplications in $R_q$; all the heavy arithmetic is
   pushed to the prover or to small-field scalar work.

2. A **generic reduction from extension-field PCSs to cyclotomic-ring
   PCSs.** Given a statement of the form "I know a multilinear over
   $\mathbb{F}_{q^k}$ that evaluates to $y$ at $x$," Hachi's reduction
   expresses it as an equivalent $R_q$-statement that can be discharged
   by any existing lattice PCS. This piece is separable from the main
   construction and is usable as a drop-in enhancement for Greyhound or
   similar schemes when you need large-field soundness without paying
   for it in the commitment.

## Problem statement

A **multilinear polynomial commitment** lets a prover:

- Commit to a multilinear polynomial $\widetilde f: \mathbb{F}^\ell \to
  \mathbb{F}$, producing a short commitment $c$.
- Later, for any evaluation point $x \in \mathbb{F}^\ell$ and claimed
  value $y$, produce a short proof that $\widetilde f(x) = y$.

The scheme must be **binding** (the prover cannot open $c$ to two
different polynomials), **evaluation-binding** (a single commitment
opens consistently at every point), and, for zkSNARK applications,
**hiding** (the commitment reveals nothing about $\widetilde f$).

Sum-check-based SNARKs use this primitive to discharge the final
evaluation claim the sum-check reduces to. In Spartan specifically:

- Outer sum-check reduces "all R1CS constraints hold" to
  "$\widetilde F(\tau) = 0$" for random $\tau$.
- Inner sum-check reduces the matrix-vector claim to an evaluation of
  $\widetilde z$ at a random point $r_y$.
- The PCS is what actually proves $\widetilde z(r_y) = v$.

Swap out the PCS and you swap out the scheme's (assumption, proof
size, verifier time, prover time) tradeoff, without touching the
sum-check skeleton.

## Why "over extension fields" matters

Lattice-based PCSs are fastest when both the committed polynomial and
the evaluation point live in $R_q$ for a small modulus $q$ and a small
ring degree $d$. But sum-check soundness requires the random evaluation
point to come from a large domain: soundness error per round is
$\approx d / |\text{domain}|$. Sampling $\tau$ from $R_q$ with small $q$
gives a lousy $\lambda$ unless you repeat, which kills the concrete
performance.

The standard workaround is to sample challenges from an extension field
$\mathbb{F}_{q^k}$ where $k$ is chosen so $q^k \ge 2^\lambda$. But then
the PCS needs to support evaluation over $\mathbb{F}_{q^k}$, not just
over $R_q$. Hachi's second contribution is the generic reduction that
makes this compatible: you can run sum-check over the large extension
field for good soundness, while the commitment still lives in
$R_q$ for cheap lattice operations.

This is the piece most directly relevant to LIP-style arithmetisations.

## Cost model cheat sheet

Let $\ell$ be the number of variables of the committed multilinear
(so the witness size is $2^\ell$), $\lambda$ the security parameter.

| Quantity              | Cost                                 |
|-----------------------|--------------------------------------|
| Commitment size       | $O(\lambda)$                         |
| Opening proof size    | $\mathrm{poly}(\ell, \lambda) \approx 55$ KB |
| Prover time           | $O(2^\ell \cdot \mathrm{poly}(\lambda))$ field ops |
| Verifier time         | $\tilde O(\sqrt{2^\ell \cdot \lambda})$ |
| Setup                 | transparent                          |
| Assumption            | Module-SIS over $R_q$                |

The 12.5× verifier speedup over Greyhound comes from the ring-switching
trick; the asymptotic claim is an $\tilde O(\lambda)$ improvement in
verifier time.

## Hachi in a Spartan-over-LIP pipeline

Putting the pieces together for the
[LIP arithmetisation idea](../ideas/lip-arithmetisation.md):

1. **R1CS arithmetisation.** $N = n(n+1)/2$ rank-1 quadratic constraints
   encoding $U^\top Q_0 U = Q_1$, witness $z \in \mathbb{F}_p^{n^2 + \mathrm{aux}}$.
2. **Spartan outer sum-check.** Certifies all $N$ constraints as a
   single polynomial identity; runs in $O(\log m)$ rounds.
3. **Spartan inner sum-check.** Reduces the matrix-vector claims to one
   evaluation of $\widetilde z$ at a random point $r_y$.
4. **Hachi opens $\widetilde z(r_y)$.** Commit to $\widetilde z$ once at
   the start of the proof; discharge the final evaluation claim with a
   ~55 KB Hachi opening.
5. **Soundness comes from extension-field challenges.** $\tau$ and the
   sum-check challenges are sampled from $\mathbb{F}_{p^k}$ with
   $p^k \ge 2^{128}$; Hachi's generic reduction makes this compatible
   with an $R_q$-native commitment.

End-to-end this gives a **transparent, post-quantum, lattice-native
SNARK for LIP** with:

- Prover time $O(n^2)$ plus the lattice commitment work (one-shot).
- Verifier time $O(\log m + \sqrt{n^2 \cdot \lambda}) = \tilde O(n
  \sqrt{\lambda})$ via Spartan's sum-check plus Hachi's sublinear
  verifier.
- Proof size dominated by Hachi's ~55 KB, plus $O(\log m)$ field
  elements from the sum-check transcript.

## When Hachi is the right choice (and when it is not)

**Use Hachi** when:

- You need a **post-quantum** multilinear commitment with a **succinct,
  concretely small** proof (~55 KB).
- Your statement is lattice-adjacent, so the $R_q$-native commitment
  composes cleanly with whatever else is going on (HAWK, Dilithium,
  BFV/BGV, …).
- You want a **transparent** setup; no trusted setup, no per-circuit
  preprocessing beyond the usual Spartan matrix commitment.
- Verifier time is a first-order cost (signatures verified often, not
  just proved once).

**Don't use Hachi** when:

- You need **constant-size proofs** (for on-chain verifiers); Hachi is
  sublinear, not constant. Use KZG-based PLONK or Groth16 if PQ is not
  a requirement.
- You are in **small-characteristic** land where Binius-style PCSes
  (tower fields, $\mathbb{F}_{2^k}$) win on prover throughput.
- You want **aggressive proof-size minimisation** and are happy with a
  bigger verifier; FRI-based schemes over large fields (Basefold,
  WHIR, Brakedown variants) may edge out Hachi on certain parameter
  regimes, though currently Hachi looks better on the combined
  (size, verifier, PQ) axis.
- You expect to fold many proofs; in that case compose **LatticeFold**
  with Hachi as the final PCS, rather than running Spartan-with-Hachi
  per instance.

## Relation to the Greyhound / LaBRADOR / LatticeFold lineage

- **LaBRADOR** (Eurocrypt 2024 / ePrint 2024/1245): the foundational
  lattice-based proof system; introduces the recursive amortisation
  machinery for proving large relations.
- **Greyhound** (CRYPTO 2024): a multilinear polynomial commitment on
  top of LaBRADOR-style tools; the direct predecessor Hachi improves
  on.
- **LatticeFold** (ePrint 2024/257): a folding scheme for lattice-based
  SNARKs, the natural "recursion / batching" layer that composes with
  Hachi.
- **Neo** (ePrint 2024/1615): a more recent lattice SNARK in the same
  author lineage, oriented toward folding-centric arithmetisations.

Hachi is the **commitment layer** in this stack; the other four are
proof-system layers. Picking a SNARK for LIP amounts to picking one of
those proof systems plus Hachi as the PCS.

## Things to read next

- Greyhound (the scheme Hachi improves on), for the baseline analysis
  it inherits.
- Huang, Mao, Zhang (ePrint 2025) for the **ring-switching** technique,
  which is the conceptual core of Hachi's verifier speedup and is
  independently useful for other lattice SNARK constructions.
- LatticeFold, for how you would amortise a LIP PoK across many
  instances if LIP-via-Spartan-with-Hachi is the inner step.
