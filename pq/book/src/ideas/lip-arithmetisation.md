# Project idea: arithmetising LIP as R1CS

> A direct analogue of the isogeny-arithmetisation line (see
> [Atkin/Weber](../papers/atkin-weber-modular-polynomials.md),
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

So the *Gram-form* part of LIP is about as arithmetisation-friendly a
hard relation as you can hope for: it is natively quadratic, it has a
clean block structure, and the number of constraints is $n(n+1)/2$
without any clever tricks.

But the Gram-form equations alone do **not** fully capture LIP. We
also need to prove that $U$ is actually unimodular.

## The missing constraint: $U$ must be unimodular

The LIP relation asks for $U \in \mathrm{GL}_n(\mathbb{Z})$, i.e.,
$\det(U) = \pm 1$. The quadratic system $U^\top Q_0 U = Q_1$ on its own
does not force this inside a SNARK, for two reasons:

1. **Invertibility is not implied mod $p$.** Over $\mathbb{Z}$, when
   $Q_0$ and $Q_1$ are positive definite with equal determinant,
   $U^\top Q_0 U = Q_1$ forces $\det(U)^2 = 1$, hence $\det(U) = \pm 1$,
   and $U$ is unimodular. But the SNARK works over a prime field
   $\mathbb{F}_p$ where the witness entries $x_{ij}$ are formal scalars.
   A cheating prover can pick $U \in \mathbb{F}_p^{n \times n}$ whose
   determinant over $\mathbb{Z}$ is an unrelated large integer that
   merely happens to be $\equiv \pm 1 \pmod p$, and satisfies the
   quadratic system modulo $p$ without corresponding to any genuine
   unimodular integer witness.
2. **Integer-lift is not implied either.** Even if the field equations
   are satisfied, each $x_{ij} \in \mathbb{F}_p$ is just a field element
   and need not lift to a bounded integer. LIP witnesses are integer
   matrices with entries controlled by the short-basis norm; arbitrary
   field elements are not valid witnesses.

A complete arithmetisation therefore needs two additional pieces on
top of the Gram-form system.

### Piece 1: an auxiliary inverse witness $V$

Commit to a second matrix $V = (y_{ij}) \in \mathbb{F}_p^{n \times n}$
and enforce
$$
U V \;=\; I_n \qquad\Longleftrightarrow\qquad
\sum_{k=1}^n x_{i,k}\, y_{k,j} \;=\; \delta_{i,j}
\quad\text{for all } (i,j).
$$
This is another $n^2$ quadratic constraints in the combined witness
$(U, V)$, one per entry of the product $U V - I_n$. Over a commutative
ring, $UV = I$ implies $VU = I$, so we only need one side.

Why this suffices (together with Piece 2): over $\mathbb{Z}$,
$\det(U) \det(V) = \det(I) = 1$, so both determinants are units in
$\mathbb{Z}$, i.e., $\pm 1$. So if we additionally know that $U$ and
$V$ are genuine integer matrices (Piece 2), we have proved $U$ is
unimodular and $V = U^{-1}$ is its integer inverse.

### Piece 2: range checks on $U$ and $V$

Bound each entry: $|x_{ij}|, |y_{ij}| \le B$ for some prescribed
$B \in \mathbb{Z}$. Any standard range-check gadget works:

- **Bit decomposition** (generic, R1CS-friendly): commit to the
  $\log_2(2B + 1)$ bits of each entry; costs $O(n^2 \log B)$
  constraints total across both matrices.
- **Lookup arguments** (Lasso, Plookup, cq): check each entry against
  a precomputed table $[-B, B]$; costs are amortised and typically
  much cheaper than bit decomposition for large $B$.

Concretely, $B$ must be large enough to accommodate:

- Entries of $U$, bounded by the short-basis norm that defines the
  HAWK-style instance (tens of bits for typical parameters).
- Entries of $V = U^{-1}$, bounded by the signed cofactors of $U$,
  which are degree-$(n-1)$ polynomials in $U$'s entries. For short
  unimodular $U$ these stay polynomially bounded but can be
  noticeably larger than $U$'s own entries; the tightness of the
  bound matters for the cost of the range check.

A careful write-up would pick $p$ and $B$ such that (i) no wrap-around
can occur in the field equations, and (ii) no false witness exists:
any $(U, V)$ satisfying the field equations with entries in $[-B, B]$
necessarily satisfies the integer equations.

### Constraint count with unimodularity

| Block                           | Shape          | Count         |
|---------------------------------|----------------|---------------|
| Gram form $U^\top Q_0 U = Q_1$  | quadratic      | $n(n+1)/2$    |
| Inverse $U V = I_n$             | quadratic      | $n^2$         |
| Range checks on $U, V$          | linear + lookup| $O(n^2 \log B)$ bits, amortisable |

Total quadratic constraints: $\tfrac{3}{2} n^2 + O(n)$. The headline
"LIP is a clean quadratic system" is preserved; we just enforce two
quadratic systems instead of one, and add range checks to bridge the
field-vs-integer gap.

### Sanity check via determinants

If you want a belt-and-braces check, you can additionally have the
prover commit to $d := \det(U) \in \{-1, +1\}$ and arithmetise the
Leibniz formula as an extra constraint. This is expensive (degree $n$
in the entries) and redundant given the auxiliary-inverse approach
above, so it is not recommended; mentioning it only to note that the
auxiliary-inverse trick is the cheap alternative to doing this the
obvious way.

## Aggregating the residuals: when it helps, when it does not

Even $n(n+1)/2$ can be a lot of equations once $n$ gets into the HAWK
range ($n \approx 512$, so $\sim 130{,}000$ quadratic residuals). A
standard move is to batch them by a random linear combination:

- Let the verifier (or a Fiat–Shamir hash) sample $N := n(n+1)/2$ scalars
  $c_{i,k}$ over a suitably large extension field.
- Prove the single aggregated identity
  $g(x) := \sum_{i \le k} c_{i,k} \, f_{i,k}(x) = 0$.

By Schwartz–Zippel, if any $f_{i,k}(x^*) \ne 0$ then $g(x^*) = 0$ with
probability at most $1/|\mathbb{F}|$, so the aggregated check is as sound
as checking all $N$ residuals individually (soundness error $\approx
2^{-128}$ over a 128-bit extension field).

It is tempting to then say "and now we only have one constraint," but this
is wrong in general. Whether aggregation actually pays off depends
entirely on what the target proof system charges per.

### Not a win: R1CS-QAP SNARKs (Groth16, Marlin, Plonk-without-sum-check)

R1CS counts **rank-1 quadratic** constraints
$\langle a, z\rangle \cdot \langle b, z\rangle = \langle c, z\rangle$. The
aggregated $g(x)$ is a general quadratic form in the entries of $U$,
which in general has rank $\Theta(N)$. Writing a rank-$r$ quadratic form
in R1CS needs $r$ constraints, so aggregation just trades $N$ *sparse*
rank-1 constraints for $\Theta(N)$ *dense* ones; often a wash, sometimes
a loss. In the QAP-based prover (Groth16 & co.), cost is dominated by
the number of non-zero R1CS entries and by one or two MSMs, and the
proof size is already constant, so there is nothing for the trick to
shrink. For these systems the natural description is the
$n(n+1)/2$-constraint one, period.

### A real win: sum-check-based SNARKs (Spartan, HyperPlonk, Jolt)

Sum-check SNARKs charge per **polynomial identity the verifier reduces
via sum-check**, not per scalar constraint. The sum-check protocol
(Lund–Fortnow–Karloff–Nisan) is an interactive proof for a claim of the
form
$$
S = \sum_{b \in \{0,1\}^\mu} p(b_1, \ldots, b_\mu),
$$
with cost $O(\mu \cdot d)$ rounds, $O(2^\mu \cdot d)$ prover field ops,
and $O(\mu \cdot d)$ verifier ops, where $d$ is the individual degree of
$p$. *One* sum-check instance can certify *one* polynomial identity,
regardless of how many scalar equations that identity packages.

For LIP, index the residuals by $(i, k)$ with $i \le k$, and let
$\nu := \lceil \log_2 N \rceil$. Identify $(i, k)$ with a bit-string
$b \in \{0,1\}^\nu$ and build the multilinear extension
$$
\widetilde F(b, x) \;=\; \text{multilinear interpolation of }
\bigl(f_{i,k}(x)\bigr)_{(i,k)} \text{ in the first argument}.
$$
"$U$ is a LIP witness" is now equivalent to "$\widetilde F(b, x) = 0$
for every $b \in \{0,1\}^\nu$." Apply the standard Spartan reduction:

1. Verifier picks random $\tau \in \mathbb{F}^\nu$.
2. Prover and verifier run sum-check on
   $$
   \sum_{b \in \{0,1\}^\nu} \widetilde{\mathrm{eq}}(\tau, b) \cdot
   \widetilde F(b, x) \;=\; 0,
   $$
   where $\widetilde{\mathrm{eq}}(\tau, \cdot)$ is the multilinear
   extension of the indicator $[b = \tau]$.
3. Soundness of step 2 reduces "residuals vanish on the cube" to
   "$\widetilde F(\tau, x) = 0$," again by Schwartz–Zippel.

The informal "sample challenges $c_{i,k}$ and prove
$\sum c_{i,k} f_{i,k} = 0$" is exactly this sum-check in disguise: the
$c_{i,k}$ are the values $\widetilde{\mathrm{eq}}(\tau, b)$ on the
Boolean cube.

**What aggregation buys here, concretely.** Without it, a naive "one
sum-check per residual" would run $N = n(n+1)/2$ sum-checks, each of
size $O(n^2)$; total prover work $\Theta(N \cdot n^2) = \Theta(n^4)$,
proof size $\Theta(N \log N)$. With aggregation you run one sum-check of
size $O(n^2)$; prover work $\Theta(n^2)$, proof size $\Theta(\log N)$.
That is a genuine $N$-fold reduction in prover work and a $\log N$-fold
reduction in proof size, directly attributable to the packaging step.
This is the setting where the LIP arithmetisation is most attractive.

### A middle ground: Aurora / Ligero and batched low-degree tests

FRI-based R1CS SNARKs like Aurora and Ligero commit to the witness via
a Reed–Solomon encoding and run a batched low-degree test over the
constraint-residual polynomials $A z \circ B z - C z$. The prover cost
still scales with the number of non-zero R1CS entries (so the
$n(n+1)/2$-constraint description is the right baseline for prover
work), but proof size scales with the number of aggregated polynomial
identities the verifier reduces, so aggregation shrinks proof size
without helping prover time. This is a modest win in practice.

### Summary table

| Proof system                      | Per-unit cost                         | Aggregation wins? |
|-----------------------------------|---------------------------------------|-------------------|
| Groth16 / Marlin / Plonk (QAP)    | non-zero R1CS entries, MSM size       | no                |
| Aurora / Ligero (FRI-over-R1CS)   | non-zeros (prover), identities (size) | proof size only   |
| Spartan / HyperPlonk / Jolt       | polynomial identities per sum-check   | yes, $N \to 1$    |
| Lattice SNARKs with batched opens | batched polynomial openings           | yes               |

### Takeaway for a LIP write-up

The aggregation step is not a "free constraint reduction." It is a
soundness-amplification + batching trick whose concrete payoff is
proof-system-specific. A careful write-up should:

1. State the per-equation R1CS description as the arithmetisation
   baseline (the one Groth16 and friends want).
2. State the aggregated multilinear-extension description as the
   sum-check SNARK description, and quote the $n^4 \to n^2$ prover and
   $N \to \log N$ proof-size reductions it enables.
3. Benchmark *both* on the target SNARK, since which description is
   cheaper is determined by the SNARK's cost model, not by the
   arithmetisation itself.

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
4. **Integer vs field subtleties and unimodularity.** LIP lives over
   $\mathbb{Z}$, but R1CS lives over a prime field $\mathbb{F}_p$. The
   write-up must:
   - Argue that working mod a sufficiently large prime $p$ (or a few
     primes via CRT) does not introduce false witnesses.
   - Enforce unimodularity of $U$ via the auxiliary-inverse matrix
     $V$ with $UV = I_n$ and range checks on both $U$ and $V$, as
     described in the "missing constraint" section above. Without
     this, the arithmetisation admits witnesses that satisfy the
     Gram-form equations mod $p$ without being valid LIP solutions.
   - Pick the range bound $B$ for $V = U^{-1}$'s entries carefully;
     it can be materially larger than the bound on $U$'s entries.
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
  [this wiki's entry](../papers/atkin-weber-modular-polynomials.md).
- HAWK and related LIP-based signatures, for the canonical source of
  concrete $(Q_0, Q_1)$ instances.
