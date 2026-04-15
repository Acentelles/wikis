# Atkin and Weber Modular Polynomials in Isogeny PoKs

> Thomas den Hollander, Marzio Mula, Daniel Slamanig, Sebastian A. Spindler
> *On the Use of Atkin and Weber Modular Polynomials in Isogeny Proofs of Knowledge*
> PQCrypto 2026, ePrint [2026/193](https://eprint.iacr.org/2026/193)

The PDF and full markdown of the paper live in
`raw/isogenies/2026-193-atkin-weber-modular-polynomials-isogeny-pok/`.

## TL;DR

The paper continues a line of work that arithmetises *knowledge of an isogeny*
as a rank-1 constraint system (R1CS), so that any R1CS-friendly zk-SNARK
(Aurora, Ligero, Limbo, …) can then be used as a black box to obtain an
isogeny proof of knowledge. Previous entries in the line picked different
*equations* to encode a single $\ell$-isogeny step between $j$-invariants:

- [CLL23] used the **classical** modular polynomial $\Phi_\ell(J_0, J_1)$;
- [dHKM+25] used **canonical** modular polynomials, cutting roughly 48% of
  the constraints over [CLL23];
- [LP25] dropped modular polynomials entirely for $\ell = 2$ in favour of
  **radical isogeny formulas**.

This paper studies two further candidates, **Atkin** modular polynomials
$\Phi_\ell^A(Y, J)$ and **Weber** modular polynomials $\Phi_\ell^W(F_0, F_1)$,
and shows that both give sparser R1CS descriptions of $\ell^n$-isogeny walks
than the current state of the art:

- Atkin: up to **27% fewer constraints** than [dHKM+25] for $\ell > 2$, and
  *new coverage* of primes $\ell$ not supported by canonical modular
  polynomials (in particular some $\ell \in \{41, 47, 59, 71\}$).
- Weber: up to **39% sparser constraint systems** (fewer non-zero entries in
  the R1CS matrices) than the canonical variant, for $\ell \ge 5$.

Along the way the authors provide a cleaner resultant-based proof machinery
that yields simpler proofs of the multiplicity theorems they need, and a Rust
framework for describing, optimising, and counting R1CS constraints for
isogeny walks.

## Why another modular polynomial?

The R1CS arithmetisations in this line all exploit the same skeleton:

1. Fix a modular equation $\Psi(\cdot, \cdot)$ with the property that
   $\Psi(a, b) = 0$ iff $a$ and $b$ label (via $j$-invariant or a suitable
   modular function) two curves linked by an $\ell$-isogeny.
2. Represent an $\ell^n$-isogeny walk $E_0 \to E_1 \to \cdots \to E_n$ as a
   chain of solutions $\{(a_i, a_{i+1})\}_{i=0}^{n-1}$ of $\Psi$.
3. Arithmetise $\Psi$ and the chaining constraint as R1CS; the prover is
   the party who can exhibit the chain.

The *cost* of such a proof is dominated by the number of constraints and the
sparsity of the constraint matrices, both of which depend heavily on the
*degree* and *structure* of $\Psi$. Classical $\Phi_\ell(X, Y)$ is symmetric
and has total degree $\ell+1$, but its coefficients are enormous. Canonical
and Atkin polynomials live on $X_0^+(N)$ rather than $X_0(N)$ and are more
compact; Weber polynomials live one level up in a tower that resolves
$j$-invariants via the Weber function $\mathfrak{f}$, and end up much
sparser still.

Concretely, the Atkin encoding uses a chain
$\{(t_i, j_i, j_{i+1})\}$ satisfying
$$
\Phi_\ell^A(Y, J_0) = 0 \quad\text{and}\quad
\frac{\Phi_\ell^A(Y, J_1) - \Phi_\ell^A(Y, J_0)}{J_1 - J_0} = 0,
$$
where $t_i$ is an auxiliary witness representing a root $Y = t_i$ of
$\Phi_\ell^A(Y, J_0)$ that is shared with $\Phi_\ell^A(Y, J_1)$. The second
equation is the *finite-difference* trick that encodes "$J_0$ and $J_1$ are
both roots of the same $Y$-fibre" without needing a second copy of
$\Phi_\ell^A$.

The Weber encoding uses a chain $\{(f_i, f_{i+1})\}$ satisfying
$\Phi_\ell^W(F_0, F_1) = 0$, plus a *boundary* constraint
$(F^{24} - 16)^3 - F^{24} J = 0$ that ties the Weber witnesses $f_1$ and
$f_n$ back to the $j$-invariants $j_1$ and $j_n$ of the endpoints (which are
what the verifier actually cares about).

## Technical contributions

1. **Atkin Multiplicity Theorem.** For $\ell \le 31$ or
   $\ell \in \{41, 47, 59, 71\}$, the authors show that the chain of
   solutions of the two-equation system above always sits inside
   $\mathbb{F}_{p^2}$ when the underlying $j$-invariants are supersingular,
   and that the fibre structure has the right multiplicities for the
   finite-difference trick to cut to *exactly* the $\ell$-isogenous
   neighbours.

2. **Weber Multiplicity Theorem.** Analogously, for $\ell \ge 5$, the Weber
   chain sits in $\mathbb{F}_{p^2}$ and the pair
   $\Phi_\ell^W(F_0, F_1) = 0$ plus the boundary conditions characterises
   $\ell^n$-isogeny walks on the nose.

3. **Generalised resultant machinery.** Building on [dHKM+25, §2.2], the
   authors upgrade their resultant techniques to account for roots with
   higher multiplicity (Prop. 2 plus Theorem 4), which makes the
   multiplicity proofs much shorter than their predecessors.

4. **Automated R1CS optimisation.** A Rust framework (companion repo
   `QuSAC/isogeny-walk-constraint-counter`) systematically applies the
   optimisations from [dHKM+25] (variable fusion, constant folding, dead
   constraint elimination, etc.) and measures the number of constraints,
   variables, and non-zero entries in $A, B, C$. This is the tool used to
   produce the 27% / 39% headline numbers, and is designed to support
   consistent comparisons between future candidate encodings as well.

## Where Atkin / Weber win and lose

- **Atkin wins on "primes you couldn't do before."** Canonical modular
  polynomials only support a limited set of primes; Atkin extends the list,
  giving flexibility in the choice of $\ell$ (and hence in the shape of the
  $\ell^n$-isogeny walk that defines the hard relation).
- **Weber wins on sparsity.** The Weber modular polynomials have far smaller
  coefficients and lower degree than the classical or canonical ones, so the
  resulting R1CS has up to 39% fewer non-zero entries. For proof systems
  whose prover cost is dominated by non-zero density (e.g. Aurora, Ligero),
  this is a direct concrete speed-up.
- **Both lose a bit on "bookkeeping."** Weber in particular needs a boundary
  constraint to tie the Weber-level witnesses back to the $j$-invariants the
  verifier actually sees. Atkin needs an auxiliary witness $t_i$ per step.
  These are cheap but not free.

## Where the ideas generalise

The resultant-based multiplicity technology is the most broadly reusable
contribution: any future proposal that wants to swap in yet another modular
equation (e.g. a polynomial for a different Atkin–Lehner quotient, or a
$\Gamma_1(N)$-level function) now has a clean, short template for proving
that the proposed equation *really does* cut out $\ell$-isogenous $j$-pairs
and nothing else. Together with the Rust counter, this makes the whole
"try a new modular equation and see" loop much tighter than it was in 2023.

## Open questions

- Pushing Atkin beyond $\ell = 71$ while keeping the multiplicity theorem
  clean; the current list reflects where the authors could close the
  argument, not a fundamental barrier.
- Combining Weber-level sparsity with the radical isogeny formulas of
  [LP25] for $\ell = 2$, to see whether the two reductions compose or
  merely overlap.
- Extending the framework to arithmetise isogenies of *abelian surfaces*
  (genus-2 analogue), where modular polynomials exist but are much larger.

## Related notes

- [Project idea: arithmetising LIP as R1CS](./lip-arithmetisation-idea.md):
  a direct analogue of this line of work for the Lattice Isomorphism
  Problem, where the "modular equation" is replaced by the quadratic system
  $U^\top Q_0 U = Q_1$.
