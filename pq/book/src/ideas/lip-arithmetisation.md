# Project idea: arithmetising LIP as R1CS

> A direct analogue of the isogeny-arithmetisation line (see
> [Atkin/Weber](../papers/atkin-weber-modular-polynomials.md),
> and the preceding [CLL23], [dHKMS25], [LP25])
> for the **Lattice Isomorphism Problem** (LIP).

## Motivation

One of the main reasons to build SNARKs at all is to prove *knowledge of a
witness to a hard relation*, i.e., that one knows the secret underlying
some cryptographic scheme. For this to be useful, the hard relation has to
be arithmetised into whatever proof system is at hand (R1CS, Plonkish,
AIR, …).

The isogeny community has been busy doing exactly this: [CLL23] encoded
$\ell^n$-isogeny walks in R1CS via classical modular polynomials; follow-up
works (canonical modular polynomials [dHKMS25], Atkin and Weber modular
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

### Dropping the inverse block: Gram plus lift already proves unimodularity

The auxiliary-inverse trick above is the general-purpose answer to
"how do I prove a matrix is invertible in R1CS": commit a candidate
inverse $V$, enforce $U V = I_n$, pay $n^2$ quadratic constraints.
That is essentially tight for a *generic* matrix, since the witness
carries $n^2$ degrees of freedom and each constraint fixes $O(1)$ of
them.

For LIP specifically, we can do better. The argument has three
pieces glued together: a public-input invariant, a determinant
identity on the Gram equation, and an integer-only extraction of
$\det(U) = \pm 1$.

**Piece 1: $\det(Q_0) = \det(Q_1)$ is automatic for any well-posed
LIP instance.** A Gram matrix of a lattice $\Lambda$ with basis $B$
(columns = basis vectors) is $G = B^\top B$, and
$$
\det(G) \;=\; \det(B)^2 \;=\; \mathrm{covol}(\Lambda)^2.
$$
That is, $\det(G)$ is the squared covolume of the lattice, which is
a *lattice invariant* independent of the chosen basis. Two lattices
are equivalent in the LIP sense exactly when they are isometric,
which in particular preserves covolume, so any pair $(Q_0, Q_1)$
that actually comes from equivalent lattices satisfies
$\det(Q_0) = \det(Q_1)$. Contrapositive: if $\det(Q_0) \ne \det(Q_1)$
then no unimodular $U$ can satisfy $U^\top Q_0 U = Q_1$, so the
instance is vacuous and the verifier should reject. Since $Q_0$
and $Q_1$ are public inputs, this check is an $O(n^3)$ one-time
computation outside the circuit; we never pay for it in R1CS.

**Piece 2: the determinant identity.** The determinant is a
multiplicative map on matrix multiplication, so applying $\det$ to
both sides of the Gram equation $U^\top Q_0 U = Q_1$ (viewing it as
an identity of integer matrices) gives
$$
\det(U^\top) \cdot \det(Q_0) \cdot \det(U) \;=\; \det(Q_1).
$$
Since $\det(U^\top) = \det(U)$ for any square matrix, this collapses
to
$$
\det(U)^2 \cdot \det(Q_0) \;=\; \det(Q_1).
$$
Combining with $\det(Q_0) = \det(Q_1) \ne 0$ from Piece 1, we can
divide both sides by $\det(Q_0)$ and conclude
$$
\det(U)^2 \;=\; 1.
$$

**Piece 3: the only integer solutions are $\pm 1$.** Over the
integers, the equation $x^2 = 1$ has exactly two solutions,
$x = +1$ and $x = -1$, so $\det(U) \in \{+1, -1\}$, i.e.,
$U \in \mathrm{GL}_n(\mathbb{Z})$ is unimodular. This is the
property we wanted to force.

Unimodularity falls out as a *consequence* of the Gram block. No
inverse witness is required. The Gram equation, public equality of
determinants, and integrality of $U$ together do all the work.

**Why the argument has to live over $\mathbb{Z}$, not $\mathbb{F}_p$.**
Piece 2's identity uses $\det$ on integer matrices and Piece 3 uses
$x^2 = 1 \Rightarrow x = \pm 1$ over $\mathbb{Z}$. Both steps fail
over $\mathbb{F}_p$: modulo $p$, the equation $x^2 = 1$ has two
residue-class solutions ($x \equiv \pm 1$) but those residue classes
contain infinitely many integers ($\ldots, -p-1, -1, p-1, 2p-1,
\ldots$ and $\ldots, -p+1, 1, p+1, 2p+1, \ldots$). So a cheating
prover could, in principle, pick a witness whose true integer
determinant is some large value $\equiv \pm 1 \pmod p$ yet not in
$\{\pm 1\}$, and the $\mathbb{F}_p$-only argument would accept.

To rule this out we enforce $\|U\|_\infty \le B$ as a range check on
the witness, and pick $p$ large enough that the $\mathbb{F}_p$
version of the Gram equation forces the $\mathbb{Z}$ version
coordinate-wise. Each entry
$(U^\top Q_0 U)_{ij} = \sum_{k, l} U_{ki}\, Q_0[k, l]\, U_{lj}$ is a
sum of $n^2$ products of three numbers each of absolute value at
most $\max(B, M)$ (where $M := \|Q_0\|_\infty$), so
$\bigl|(U^\top Q_0 U)_{ij}\bigr| \le n^2 B^2 M$. Picking
$p > 2 n^2 B^2 M$ ensures each true integer value fits in
$[-p/2, p/2)$, so $\mathbb{F}_p$-equality forces integer equality.
Once the Gram equation holds over $\mathbb{Z}$, Pieces 2 and 3 run
and deliver $\det(U) = \pm 1$ over $\mathbb{Z}$.

For HAWK-scale parameters ($n \approx 2^9$, $B \approx 2^{30}$,
$M \approx 2^{30}$), the bound is $\approx 2^{108}$, comfortably
below a BN254-scale prime.

**Revised constraint count.**

| Block                           | Shape           | Count         |
|---------------------------------|-----------------|---------------|
| Gram form $U^\top Q_0 U = Q_1$  | quadratic       | $n(n+1)/2$    |
| Range check on $U$              | linear + lookup | $O(n^2 \log B)$ bits, amortisable |

Total quadratic constraints: $\tfrac{1}{2} n^2 + O(n)$, a 3x
reduction relative to the auxiliary-inverse arithmetisation. The
lookup load also halves (range on $U$ only; no $V$). The subtlety
is that the argument relies on a tight range bound on $U$ so the
Gram equation lifts; if one uses a looser bound (or works modulo a
smaller prime), the auxiliary-inverse arithmetisation remains the
right fallback.

This is specific to LIP: the Gram block is doing double duty as
"equivalence of quadratic forms" and as "unimodularity certificate."
Without the extra quadratic constraint the LIP setup gives us for
free, $\Omega(n^2)$ constraints for invertibility is information-
theoretically tight.

**Caveat.** If a later part of the protocol wants $V = U^{-1}$ as a
committed object (e.g., a zero-knowledge extractor that rewinds on
$V$, or a sibling relation that references the inverse directly),
committing $V$ anyway may be the right engineering choice even
though the Gram-plus-lift argument makes it redundant for the
soundness of unimodularity alone. Worth making this call explicit in
the write-up rather than defaulting to one option.

### Aside: structured-witness alternatives (and why Gram-plus-lift dominates)

A natural meta-question: instead of witnessing the full inverse
$V$, can we commit to a *structured* object that makes invertibility
a local (row-by-row or piece-by-piece) check rather than a global
determinant check? A few candidates worth naming, mostly because
they are easy to misreach for:

- **Reduced row echelon form (RREF).** The RREF of any invertible
  $n \times n$ matrix is $I_n$, so constraining $U$ to be in RREF
  collapses the witness set to the single matrix $I_n$ and changes
  the relation. Not usable.
- **Plain row echelon form / upper triangular.** Most unimodular
  matrices are not upper triangular, so restricting to this
  structure changes the relation rather than arithmetising it. Also
  not usable.
- **LU / PLU / Bruhat decomposition.** Commit $U = P L U'$ with $P$
  a permutation (free $\det = \pm 1$), $L$ unit-lower-triangular
  (free $\det = 1$), and $U'$ upper-triangular with diagonal entries
  in $\{\pm 1\}$ (determinant checkable by inspecting $n$ diagonal
  entries). Cost: $O(n^2)$ structure constraints (zero entries below
  / above pivots, permutation one-hot rows) plus $O(n^2)$ quadratic
  constraints to enforce $P L U' = U$. Same asymptotic cost as
  $U V = I_n$, with worse constants.
- **Word in elementary matrices.** Commit a sequence of elementary
  row operations (swap / add integer multiple / $\pm 1$ scale)
  realising $U$; each elementary has $\det = \pm 1$ by inspection,
  so $\det(U) = \pm 1$ combinatorially. Cost: for a word of length
  $k$, $\Theta(k)$ matrix multiplications. A generic unimodular
  matrix requires $k = \Theta(n^2)$ elementary operations (Gaussian
  elimination), giving $\Theta(n^4)$ constraints. Strictly worse.

None of these beat $U V = I_n$ on a general matrix. The reason is
the same information-theoretic argument as before: invertibility is
a *global* property, and any certificate that is locally checkable
still has to hold against the $n^2$ degrees of freedom in $U$,
forcing $\Omega(n^2)$ constraints.

**Why Gram-plus-lift dominates all of these for LIP.** The Gram
block $U^\top Q_0 U = Q_1$ with $Q_0, Q_1$ invertible has a
closed-form consequence: since
$$
U^\top Q_0 U = Q_1
\;\;\Longrightarrow\;\;
U^{-1} = Q_1^{-1} \, U^\top \, Q_0
$$
(multiply on the left by $Q_1^{-1}$ and on the right by $U^{-1}$,
then transpose as needed), the inverse is not just witnessable; it
is *computable from $U$ by a public linear map*. Invertibility mod
$p$ follows by inspection of this formula, and unimodularity over
$\mathbb{Z}$ follows from the determinant lift. We pay zero extra
constraints; the Gram block is doing all the work.

No structured-witness arithmetisation beats "zero constraints for
invertibility." The lesson is that REF / LU / elementary-word
approaches are the right reflex when you are certifying
invertibility of a matrix *in isolation*, but LIP is not that
setting: the hard relation itself already contains an invertibility
certificate.

### Aside: characteristic polynomial and eigenvalue approaches

A different reflex is to commit spectral data about $U$ (eigenvalues
or the characteristic polynomial) and reduce unimodularity to a
check on that data. It is worth recording why this does not help,
since the idea is natural enough to come up twice.

**Committing eigenvalues directly does not type-check.** For a
generic integer matrix, eigenvalues are algebraic numbers of degree
up to $n$ over $\mathbb{Q}$, and over $\mathbb{F}_p$ they live in an
extension $\mathbb{F}_{p^k}$ that depends on the matrix. Unimodular
matrices have eigenvalues that are algebraic integers whose product
is $\pm 1$, but the individual values are usually irrational:
$\begin{pmatrix} 2 & 1 \\ 1 & 1 \end{pmatrix}$ has eigenvalues
$(3 \pm \sqrt{5})/2$. So "commit the eigenvalues as field elements"
is not a well-typed move unless the witness class is restricted to
matrices with spectrum in $\mathbb{F}_p$ (finite-order, roots of
unity, etc.), which changes the LIP relation.

**Committing the characteristic polynomial does type-check, but is
expensive.** The cleaner version is to commit
$$
\chi_U(x) \;=\; x^n + c_{n-1} x^{n-1} + \ldots + c_1 x + c_0,
\qquad c_i \in \mathbb{F}_p,
$$
and verify Cayley-Hamilton: $\chi_U(U) = 0$. Unimodularity then
reduces to $c_0 = \pm 1$, one scalar check. But evaluating
$\chi_U(U)$ requires computing $U, U^2, \ldots, U^n$, i.e., $n - 1$
matrix-matrix products, each costing $\Theta(n^2)$ quadratic
constraints (with $\Theta(n^3)$ auxiliary witnesses for intermediate
products). Total: $\Theta(n^3)$ quadratic constraints. Strictly
worse than both $U V = I_n$ (at $n^2$) and Gram-plus-lift (at $0$).

**The Cayley-Hamilton-derived inverse is also dominated.** Once
$\chi_U$ is committed, one gets
$$
U^{-1} \;=\; -\frac{1}{c_0} \bigl( U^{n-1} + c_{n-1} U^{n-2} + \ldots + c_1 I \bigr),
$$
which is a polynomial expression for $U^{-1}$ of degree $n - 1$ in
$U$. Evaluating it is again $\Theta(n^3)$. In the LIP setting we
already have the *linear* closed form $U^{-1} = Q_1^{-1} U^\top Q_0$
with public coefficients, so the Cayley-Hamilton route pays more to
reach a weaker version of the same object.

**When spectral checks do pay off.** Two niche regimes:

- **Finite-order unimodular matrices** (orthogonal integer matrices,
  torsion in $\mathrm{GL}_n(\mathbb{Z})$): eigenvalues are roots of
  unity, conjugacy classes are finite, and $\chi_U$ has $O(1)$
  possibilities per class. One can commit the class index, look up
  $\chi_U$ from a small table, and verify $\chi_U(U) = 0$. Useful if
  the LIP instance is constrained to torsion witnesses, e.g., for
  specific lattice automorphism groups.
- **Sanity-checking the public Gram matrices.** $Q_0$ and $Q_1$ are
  symmetric positive-definite with eigenvalues that are real
  algebraic integers; one can commit bounds on their spectra and
  range-check. This is a check on the *instance*, not on the secret
  witness $U$, and is orthogonal to the unimodularity question.

Neither case applies to generic LIP witnesses.

**Ranking summary.** Pecking order on invertibility arithmetisations
for LIP:

| Approach                                   | Quadratic constraints          | Notes                                  |
|--------------------------------------------|--------------------------------|----------------------------------------|
| Gram-plus-lift                             | $0$ extra                      | LIP-specific, needs tight range bound  |
| Auxiliary inverse $UV = I_n$               | $n^2$                          | general-purpose baseline               |
| LU / PLU / Bruhat decomposition            | $\Theta(n^2)$                  | same asymptotic, worse constants       |
| Cayley-Hamilton on committed $\chi_U$      | $\Theta(n^3)$                  | dominated                              |
| Leibniz determinant                        | $\Theta(n^3)$ via elimination  | strictly worse                         |
| Eigenvalues committed directly             | does not type-check            | no-go for general $U$                  |

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
   - Enforce unimodularity of $U$. Two options discussed above:
     (a) auxiliary-inverse matrix $V$ with $UV = I_n$ and range
     checks on both $U$ and $V$ ($\tfrac{3}{2} n^2$ quadratic,
     general-purpose); (b) exploit the Gram block directly, relying
     on $\det(Q_0) = \det(Q_1)$ and a tight range bound on $U$ to
     lift the Gram equation to $\mathbb{Z}$, whence $\det(U)^2 = 1$
     and the inverse block is redundant ($\tfrac{1}{2} n^2$
     quadratic, LIP-specific).
   - Pick the range bound $B$ accordingly. Under option (a), $B$ must
     cover the cofactors of $U$ (entries of $V = U^{-1}$), which are
     materially larger than $U$'s entries. Under option (b), $B$
     must satisfy $n^2 B^2 \|Q_0\|_\infty < p/2$ so the Gram equation
     lifts from $\mathbb{F}_p$ to $\mathbb{Z}$.
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

Idea only, not yet written up.

## References

### Isogeny arithmetisation line (prior art)

| Tag | Paper | ePrint | Raw |
|-----|-------|--------|-----|
| [CLL23] | Cong, Lai, Levin, *Efficient Isogeny Proofs Using Generic Techniques* (ACNS 2023) | [2023/037](https://eprint.iacr.org/2023/037) | `raw/isogenies/2023-037-efficient-isogeny-proofs-generic-techniques/` |
| [dHKMS25] | den Hollander, Kleine, Mula, Slamanig, Spindler, *More Efficient Isogeny Proofs of Knowledge via Canonical Modular Polynomials* (CRYPTO 2025) | [2024/1738](https://eprint.iacr.org/2024/1738) | `raw/isogenies/2024-1738-efficient-isogeny-proofs-canonical-modular-polynomials/` |
| [LP25] | Levin, Pedersen, *Faster Proofs and VRFs from Isogenies* (ASIACRYPT 2025) | [2024/1626](https://eprint.iacr.org/2024/1626) | `raw/isogenies/2024-1626-faster-proofs-vrfs-isogenies/` |
| | den Hollander, Mula, Slamanig, Spindler, *On the Use of Atkin and Weber Modular Polynomials in Isogeny Proofs of Knowledge* (PQCrypto 2026) | [2026/193](https://eprint.iacr.org/2026/193) | `raw/isogenies/2026-193-atkin-weber-modular-polynomials-isogeny-pok/` |

See also [this wiki's Atkin/Weber entry](../papers/atkin-weber-modular-polynomials.md).

### LIP / Hawk

- [DPPvW22] Ducas, Postlethwaite, Pulles, van Woerden, *Hawk: Module LIP
  makes lattice signatures fast, compact and simple*,
  ePrint [2022/1155](https://eprint.iacr.org/2022/1155).
  HAWK and related LIP-based signatures provide the canonical source of
  concrete $(Q_0, Q_1)$ instances for benchmarking.

### Proof systems referenced

- [Spartan](../papers/spartan.md) (Setty, CRYPTO 2020): sum-check-based
  SNARK, the cost model used in the aggregation analysis above.
