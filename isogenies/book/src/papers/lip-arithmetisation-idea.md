# Project idea: arithmetising LIP as R1CS

> A direct analogue of the isogeny-arithmetisation line (see
> [Atkin/Weber](./atkin-weber-modular-polynomials.md),
> and the preceding [CLL23], [dHKM+25], [LP25])
> for the **Lattice Isomorphism Problem** (LIP).

## Motivation

One of the main reasons to build SNARKs at all is to prove *knowledge of a
witness to a hard relation*, i.e., that one knows the secret underlying
some cryptographic scheme. For this to be useful, the hard relation has to
be arithmetised into whatever proof system is at hand (R1CS, Plonkish,
AIR, …).

The isogeny community has been busy doing exactly this: [CLL23] encoded
$\ell^n$-isogeny walks in R1CS via classical modular polynomials; follow-up
works (canonical modular polynomials [dHKM+25], Atkin and Weber modular
polynomials [ePrint 2026/193](https://eprint.iacr.org/2026/193), radical
isogenies [LP25]) squeezed out better and better constraint counts.

As far as I know, **no one has done the analogous exercise for LIP**, the
natural hard relation underlying lattice-isomorphism-based signatures (e.g.,
HAWK). This note sketches why that is easy, and what a project to write it
up would look like.

## The LIP relation

Fix two symmetric positive-definite integer matrices
$Q_0, Q_1 \in \mathbb{Z}^{n \times n}$. The **Lattice Isomorphism Problem**
asks for a unimodular $U \in \mathrm{GL}_n(\mathbb{Z})$ with
$$
U^\top Q_0 \, U = Q_1.
$$
Equivalently, $U$ realises an isometry between the lattices defined by
$Q_0$ and $Q_1$. The hard-relation version is: given $(Q_0, Q_1)$, prove
knowledge of such a $U$.

## The key observation

Let $U = (x_{ij})_{i,j=1}^n$, i.e., treat the $n^2$ entries of $U$ as
formal variables. Expanding the matrix equation
$U^\top Q_0 U = Q_1$ entry-by-entry gives, for each $(i, k)$,
$$
f_{i,k}(x_{11}, \ldots, x_{nn})
\; := \;
\sum_{s=1}^n \sum_{r=1}^n Q_0[s, r] \, x_{r,i} \, x_{s,k} \; - \; Q_1[i, k]
\; = \; 0.
$$
This is a system of $n^2$ **quadratic** equations in the $n^2$ unknowns
$x_{ij}$. Two easy symmetry remarks:

1. Because $Q_0$ and $Q_1$ are symmetric, so is the matrix
   $U^\top Q_0 U$, hence $f_{i,k} = f_{k,i}$. We only need to enforce the
   equations for $i \le k$, i.e., the $\binom{n+1}{2} = n(n+1)/2$ on-or-
   above-diagonal entries. (The user-sketched count $n(n-1)/2$ counts the
   *strictly* off-diagonal ones; whether the diagonal entries need their
   own equations or can be absorbed into the off-diagonal ones by a change
   of basis is a small bookkeeping question worth nailing down in the
   write-up.)

2. The system is uniformly quadratic: every $f_{i,k}$ has total degree
   exactly 2. This means each $f_{i,k}$ translates to a **single R1CS
   constraint**, of the form $\langle a, z \rangle \cdot \langle b, z
   \rangle = \langle c, z \rangle$, where $z$ is the witness vector and
   $a, b, c$ are the structure-constant rows. No auxiliary witnesses or
   multiplication gates are needed to expand the equation.

So LIP is about as arithmetisation-friendly a hard relation as you can hope
for: it is *natively* quadratic, it has a clean block structure, and the
number of constraints is $n(n+1)/2$ without any clever tricks.

## A cheap random-linear-combination optimisation

Even $n(n+1)/2$ can be a lot of constraints once $n$ gets into the HAWK
range ($n \approx 512$, so $\sim 130{,}000$ quadratic constraints). A
standard move in SNARK-land:

- Let the verifier (or a Fiat–Shamir hash) sample $n(n+1)/2$ challenges
  $c_{i,k}$ over a suitably large extension field.
- Prove the single aggregated equation
  $\sum_{i \le k} c_{i,k} \, f_{i,k}(x) = 0$.

By Schwartz–Zippel, this is sound whp as long as the challenge domain is
large enough (soundness error $\le 1/|\mathbb{F}|$). The aggregated
equation is still a quadratic form in the $x_{ij}$, hence still a *single*
R1CS-style quadratic constraint in the $x_{ij}$, albeit a dense one.

Whether this aggregation is actually a win depends on the target proof
system: systems whose prover cost is dominated by the number of non-zero
entries (Aurora, Ligero) may prefer the structured sparse version of
$n(n+1)/2$ constraints over a single dense one; systems with per-constraint
overheads (some Plonkish variants) may prefer the aggregated form. A
careful write-up should benchmark both.

## What the project looks like

Concretely, to turn this observation into a short paper:

1. **Formal statement.** Define the LIP PoK relation; write out the
   quadratic system and the symmetry reduction; state the soundness of
   the random-linear-combination aggregation.
2. **R1CS description.** Give the structure-constant matrices $A, B, C$
   for the per-equation constraints; give the aggregated dense variant.
   Count constraints, variables, and non-zero entries as a function of
   $n$ and compare with alternatives (e.g., committing to $U$ and
   verifying in the clear).
3. **Integration.** Plug the constraint system into at least one concrete
   zk-SNARK (Aurora / Ligero / Jolt / a lattice-friendly SNARK) and
   report prover/verifier times for HAWK-scale parameters. The
   [`isogeny-walk-constraint-counter`](https://github.com/QuSAC/isogeny-walk-constraint-counter)
   framework from ePrint 2026/193 is a good model for how to present the
   numbers; an LIP-flavoured version of it would make follow-up
   comparisons easy.
4. **Integer vs field subtleties.** LIP lives over $\mathbb{Z}$, but
   R1CS lives over a prime field $\mathbb{F}_p$. The write-up needs to
   argue that working mod a sufficiently large prime (or a few primes via
   CRT) does not introduce false witnesses, using standard bounds on the
   entries of $U$ for the regime of interest.
5. **Zero-knowledge of $U$.** LIP is a search problem whose secret is
   $U$ itself, so the proof must be zero-knowledge for $U$ (not merely
   argument-of-knowledge). This comes for free from any ZK-R1CS SNARK
   but is worth spelling out.

## Why it is worth doing even if easy

The isogeny analogues of this idea ([CLL23] and its descendants) are
conceptually simple too. Their value is not in technical difficulty, but
in (i) unlocking generic SNARKs for a post-quantum hard relation, and
(ii) establishing a concrete performance baseline that future more-clever
encodings (canonical / Atkin / Weber / radical) can be measured against.
A LIP arithmetisation paper plays the same role: it gives the community
a clean baseline for "SNARK over HAWK-style LIP," against which future
optimisations (smarter witness decompositions, avoiding the integer-to-
field lift, exploiting structure of specific $Q_0$ like the HAWK
construction) can be compared.

## Status

Idea only, not yet written up. Prior art to cite at the very least:

- [CLL23] Cong, Lai, Levin, modular-polynomial R1CS for isogenies
  (ACNS'23).
- [dHKM+25] den Hollander, Kutas, Mula, Slamanig, Spindler, canonical
  modular polynomials (CRYPTO'25).
- [ePrint 2026/193](https://eprint.iacr.org/2026/193), Atkin / Weber
  variants (PQCrypto'26); see
  [this wiki's entry](./atkin-weber-modular-polynomials.md).
- HAWK and related LIP-based signatures, for the canonical source of
  concrete $(Q_0, Q_1)$ instances.
