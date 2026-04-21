# Project idea: mLIP ↔ abelian variety isogeny correspondence

> Paper 100 at CiC 2026.1
> (my/presentations/mlip-correspondence/cic2026_1-paper100.pdf) establishes, for
> primes $p \equiv 3 \pmod 4$, a bijective correspondence between rank-2
> torsion-free module lattices over $\mathbb{Q}(\sqrt{-p})$ and supersingular
> elliptic curves over $\mathbb{F}_{p^2}$, and shows that isogeny finding
> reduces to recovering the module lattice corresponding to a curve. The
> correspondence passes through the same quaternion algebra
> $A = \left(\tfrac{-1, -p}{\mathbb{Q}}\right)$ on both sides, and the
> accompanying presentation (presentation.tex / .pdf) extends the picture
> to abelian surfaces via the Ibukiyama-Katsura-Oort (IKO) correspondence
> and the recent KLPT$^2$ algorithm. The open problem is whether the
> correspondence generalises to **rank-$n$ module lattices ↔
> higher-dimensional abelian varieties**, and what cryptanalytic leverage
> that buys on both sides.

## Setup: the two hard problems the paper connects

### mLIP side

A module lattice $M$ over a number field $K$ is a torsion-free
finitely-generated $\mathcal{O}_K$-module equipped with a pseudo-basis
$\mathbf{B} = (B, \{\mathfrak{a}_i\}_i)$ and pseudo-Gram matrix
$\mathbf{G} = (B^\ast B, \{\mathfrak{a}_i\}_i)$. A **module lattice
isomorphism** between $M, M'$ is $U \in \mathrm{GL}_n(K)$ with
$U_{i,j} \in \mathfrak{a}_i \mathfrak{b}_j^{-1}$ and
$(U^{-1})_{i,j} \in \mathfrak{b}_i \mathfrak{a}_j^{-1}$ such that
$G' = U^\ast G U$.

**wc-smodLIP$_K^{\mathbf{B}}$** (mLIP for short in these notes): given
$\mathbf{G}$ and $\mathbf{G}'$ promised congruent, find $U$.

Known reductions:
- Unstructured LIP → SVP ([HR13]).
- Rank-2 mLIP over totally real $K$ → Principal Ideal Problem (PIP) in
  $\mathcal{O}_K$ ([MPMPW24]).
- A single real embedding of $K$ is enough for the above to apply
  ([APMvW25]).
- Rank-2 mLIP over a CM extension $K/F$ → reduced-norm PIP in a quaternion
  algebra $\mathcal{A} = \left(\tfrac{a, -1}{F}\right)$, i.e.
  **$\mathcal{O}$-nrdPIP** ([CME+25]).
- For Hawk (cyclotomic, totally imaginary) this reduction is *uniform*:
  breaking Hawk is no harder than one SVP call in a lattice of dimension
  $2d$ ([CME+25, Thm 1.1]). This was folklore until CME+25 made it explicit.

Hawk ([DPPvW22]) is the representative Module-LIP signature: it is
submitted to NIST's post-quantum onramp.

### Isogeny side

The **Deuring correspondence** identifies isomorphism classes of
supersingular elliptic curves over $\overline{\mathbb{F}_p}$ with
isomorphism classes of maximal orders in the quaternion algebra
$B_{p,\infty}$ (ramified only at $p$ and $\infty$), and identifies
isogenies $\varphi : E_1 \to E_2$ with left ideals $I_\varphi$
connecting $\mathcal{O}_\ell(I_\varphi) \cong \mathrm{End}(E_1)$ to
$\mathcal{O}_r(I_\varphi) \cong \mathrm{End}(E_2)$ with
$\deg \varphi = \mathrm{nrd}(I_\varphi)$.

Known reductions:
- $\ell$-IsogenyPath ≡ EndRing ≡ MaxOrder, all polynomial-time
  interreducible ([Wes21], unconditional at [PW23]).
- Quaternion $\ell$-IsogenyPath is polynomial-time via KLPT
  ([KLPT14]), with subsequent improvements ([FLLW22], [BSE+25] "Qlapoti"
  for ideal-to-isogeny translation).
- IsogenyPath itself is the main hard problem underlying CSIDH ([CLM+18]),
  SQIsign ([AAA+25]), and the CGL hash function ([CGL06]). SIDH was
  specifically broken using *torsion point leakage*, not by attacking
  IsogenyPath directly ([CD22]).

For $p \equiv 3 \pmod 4$ the Deuring-side quaternion algebra is
$B_{p,\infty} \cong \left(\tfrac{-1, -p}{\mathbb{Q}}\right)$ ([Piz80],
[Wes21]).

## The paper's main results (dim 1, rank 2)

The two sides share their quaternion algebra for $p \equiv 3 \pmod 4$:

**Theorem 9.** Both supersingular isogeny finding over $\mathbb{F}_{p^2}$
and rank-2 mLIP over $\mathbb{Q}(\sqrt{-p})$ reduce to problems over
$A = \left(\tfrac{-1, -p}{\mathbb{Q}}\right)$.

Spelling this out leads to a chain of bijections:

**Theorem 12 → 13 → 14 (Curve-Lattice correspondence).**

$$
\underbrace{\left\{
\begin{array}{c}
\text{rank-2 torsion-free} \\
\mathcal{O}_{\mathbb{Q}(\sqrt{-p})}\text{-module lattices up to iso}
\end{array}
\right\}}_{\text{lattice side}}
\;\longleftrightarrow\;
\underbrace{\left\{
\begin{array}{c}
\text{maximal orders of } A \\
\text{up to iso}
\end{array}
\right\}}_{\text{quaternion side}}
\;\longleftrightarrow\;
\underbrace{\left\{
\begin{array}{c}
\text{supersingular} \\
E(\mathbb{F}_{p^2}) \text{ up to iso}
\end{array}
\right\}}_{\text{geometric side}}.
$$

The left bijection is Theorem 13, the right is Deuring. The composite is
Theorem 14.

**Theorem 15 (mLIP ↔ fixed-degree endomorphism finding).** Fix $M$ and
$G'$ congruent to $G_M$, and let $E$ be the curve corresponding to
$\mathcal{O} = \mathcal{O}_r(I_M)$ under Deuring. Then
wc-smodLIP$_K^{\mathbf{B}}$ reduces to finding $\alpha \in \mathrm{End}(E)$
of a prescribed degree.

**Theorem 16 (Isogeny finding ↔ lattice recovery).** Given an oracle that
on input a curve returns its corresponding rank-2 module lattice over
$\mathbb{Q}(\sqrt{-p})$, there is a polynomial-time algorithm for
$\ell$-IsogenyPath using two oracle calls.

Note the asymmetry: rank-2 mLIP over imaginary-quadratic $K$ is *easy*
(Theorem 4 in the paper reduces it to a rank-2 SVP via
Kirschmer-Voight + Lagrange-Gauss, since
$|\mathbb{Z}_{\mathbb{Q}, +}^\times / \mathbb{Z}_{\mathbb{Q}}^{\times 2}| = 1$).
The content of Theorem 16 is therefore not a cryptanalytic attack but a
**reduction shape**: if you could read off a module lattice from a curve
*without going through its endomorphism ring*, you would attack isogeny
finding.

## The presentation's extension: rank-$n$ ↔ higher-dimensional abelian
varieties

The presentation (Ibukiyama-Katsura-Oort slide onward) pushes the picture
one dimension up and turns the paper's Theorem 14 into the dim-1
instance of a conjectural family. The key ingredients are:

### Ibukiyama-Katsura-Oort (IKO, 1986)

[IKO86] established, for $p \equiv 3 \pmod 4$, a bijection

$$
\left\{
\begin{array}{c}
\text{superspecial principally} \\
\text{polarised abelian surfaces} \\
(A, \lambda_A)/\overline{\mathbb{F}_p} \\
\text{up to polarised iso}
\end{array}
\right\}
\;\longleftrightarrow\;
\left\{
\begin{array}{c}
\text{principal polarisations} \\
\lambda \in \mathrm{PPol}(A_0) \\
\text{up to equivalence}
\end{array}
\right\}
\;\overset{\mu}{\longleftrightarrow}\;
\left\{
\begin{array}{c}
\text{matrices } g \in \mathrm{Mat}(A_0) \\
\text{up to congruence}
\end{array}
\right\},
$$

where $\mathrm{Mat}(A_0) \subset \mathrm{GL}_2(\mathcal{O}_0)$ is the
Hermitian-like set of matrices
$\begin{pmatrix} s & r \\ \bar{r} & t \end{pmatrix}$
with $s, t \in \mathbb{Z}_{>0}$, $r \in \mathcal{O}_0$, $st - r\bar{r} = 1$.
The Rosati involution (depending on the polarisation) is realised as
conjugate-transpose $(\cdot)^\ast$ in the quaternion world, and
polarised isogenies of degree $N$ become
$\gamma \in M_2(\mathcal{O}_0)$ with $\gamma^\ast g_2 \gamma = N g_1$.

This is already strikingly close to the Hawk / mLIP form $G' = U^\ast G U$.

### KLPT$^2$ (2025)

[KLPT$^2$, ePrint 2025/372] gives a heuristic polynomial-time algorithm
that, on input $g_1, g_2 \in \mathrm{Mat}(A_0)$, finds a connecting matrix
$\gamma \in M_2(\mathcal{O}_0)$ with $\gamma^\ast g_2 \gamma = \ell^e g_1$
for small $\ell$ and bounded $e$, running in time $O(p^{25 + o(1)})$.
Applications include:

- A constructive IKO correspondence: realising a PPSAS as a Jacobian of a
  genus-2 curve.
- Attack on CGL-style hash functions over PPSAS with untrusted setup
  ([CDS19] class).
- The dim-2 analog of the constructive Deuring correspondence.

### Equivalent computational problems for PPSAS ([Mon26])

[ePrint 2026/120, Montessinos] proves the dim-2 analog of
[PW23] for *products of supersingular elliptic curves*: under GRH + the
KLPT$^2$ assumptions, the following are polynomial-time equivalent:

- Computing the IKO matrix $g$ of $A$;
- Computing an effective representation of $\mathrm{End}(A)$ as a
  $\mathbb{Z}$-module;
- Computing an unpolarised isomorphism from $E_0^2$ to $A$.

For Jacobian PPSAS the equivalence is partial (IKO ↔ good-representation
End; unpolarised-iso and End reduce *to* IKO, not necessarily from it).

### The conjecture

The presentation's final slide states a three-way conjectural
correspondence:

$$
\left\{
\begin{array}{c}
\text{superspecial PP} \\
\text{abelian surfaces}
\end{array}
\right\}
\;\longleftrightarrow\;
\left\{
\begin{array}{c}
\text{matrices } g \in \mathrm{Mat}(A_0) \\
\text{up to congruence}
\end{array}
\right\}
\;\overset{?}{\longleftrightarrow}\;
\left\{
\begin{array}{c}
\text{rank-}n \text{ module lattices} \\
\text{over some number field}
\end{array}
\right\}.
$$

The left correspondence is IKO. The right correspondence is the
conjectural extension: what rank $n$ and what number field $K$ give a
faithful mLIP dictionary for **abelian surfaces**? And inductively, for
abelian $d$-folds?

Several things make the dim-2 case look tractable today in a way that
was not true a year ago:

1. The KLPT$^2$ algorithm gives a polynomial-time quaternion-side solver,
   the missing analog of KLPT that paper 100 already uses in dim 1.
2. [Mon26] gives the dim-2 analog of PW23, closing the
   geometric ↔ quaternion gap.
3. [LMP25] studies quaternion ideal-SVP and its reductions, providing the
   lattice-reduction-side infrastructure that would mate with mLIP at
   rank $\geq 2$.

Remaining ingredient: the mLIP ↔ IKO-matrix step. In dim 1 this is
[CME+25] (rank-2 mLIP → nrdPIP). In dim 2 the analogous statement is
speculative; Conjecture 1 in paper 100 is its formalisation.

## What a research programme looks like

### Milestone 1: formalise the conjecture for abelian surfaces

Pin down the "right" combination of $(n, K)$ such that rank-$n$ mLIP
over $K$ is bijectively correspondent to superspecial PPSAS over
$\mathbb{F}_{p^2}$. Candidate starting point:

- $K = \mathbb{Q}(\sqrt{-p})$, rank $n = 4$. Heuristic reason: IKO
  matrices live in $M_2(\mathcal{O}_0)$ with $\mathcal{O}_0$ a maximal
  order in $B_{p,\infty}$, hence $8$ quaternion basis elements over
  $\mathbb{Z}$; Theorem 14 in paper 100 gave a rank-2 lattice over an
  imaginary quadratic field for a $4$-dim quaternion space, so the
  natural guess is rank doubles when the matrix dimension doubles.
- Alternative: rank $n = 2$ over a biquadratic or quartic CM field. This
  matches the fact that, for Hawk's cyclotomic field $\mathbb{Q}(\zeta)$
  of degree $d$, rank-2 mLIP already encodes a dim-$d$ geometric object
  on the quaternion side via CME+25. The map to superspecial abelian
  $d$-folds remains to be written down.

### Milestone 2: port the IKO-matrix ↔ rank-$n$ mLIP step

This is the open conjecture. Concretely we want a dim-2 analog of CME+25
Theorem 1.1:

> Given an IKO matrix $g \in \mathrm{Mat}(A_0)$, compute (in polynomial
> time) a pseudo-Gram matrix $\mathbf{G}$ of a rank-$n$ module lattice
> $M$ over $K$ such that IKO-congruence $\gamma^\ast g_2 \gamma = g_1$
> corresponds to module-lattice isomorphism $G_2 = U^\ast G_1 U$.

Two technical obstacles:

1. **Non-commutativity.** CME+25's dim-1 reduction artificially embeds
   $K^2$ into the quaternion algebra via $(x, y) \mapsto x + yi$, a
   *$K$-vector-space* isomorphism between $K^2$ and $K \oplus Ki$. For
   dim-2 the analog would need to embed $K^4$ (or similar) into
   $M_2(B_{p,\infty})$, which has a richer non-commutative structure. The
   Luo-Jiang-Pan-Wang dim-1 reduction ([CME+25, footnote 8]) bypassed the
   non-commutativity via a symplectic automorphism; it is worth checking
   whether their approach extends.
2. **Reduced norm.** The dim-1 reduction uses the reduced norm of the
   quaternion algebra. In dim 2 the natural analog is
   $\det \circ \mathrm{nrd}$ acting on $M_2(B_{p,\infty})$, but the
   algebraic invariants this exposes (e.g. Rosati involution, principal
   polarisation equivalence) are substantially more subtle.

### Milestone 3: the cryptanalytic payoff

Two directions, symmetric to Problem 1 and Problem 2 of paper 100 §4:

- **Abelian-variety tools for mLIP / Hawk.** If
  Hawk's rank-2 mLIP over $\mathbb{Q}(\zeta_{2d})$ corresponds to a
  $d$-dim superspecial abelian variety, then IKO-style matrix reduction,
  equidistribution on the variety's isogeny graph, or dim-$d$ constructive
  Deuring might give new attacks. Paper 100 already flags this: even a
  partial correspondence to modular forms / abelian varieties "gives a
  geometric insight to both problems which may be beneficial for
  cryptanalysis". The threat to Hawk is that cryptanalysis done in the
  dim-$d$ geometric world does not go through any module-lattice
  reduction; it would be a new angle of attack.
- **Module-lattice tools for isogeny finding.** Conversely, if PPSAS
  isogeny finding corresponds to rank-$n$ mLIP over some field, then
  module-lattice reduction ([LPMSW19], [DEdP25], [PL21]) and the
  dimension-reducing ideal-SVP algorithms ([LMP25]) port to attack dim-2
  isogeny-based primitives: SQIsign2D variants, genus-2 CGL hash
  functions, and constructions built on PPSAS. Paper 100's Problem 1
  already asks for a polynomial-time method to recover a module lattice
  from a curve bypassing the endomorphism ring; this would become strictly
  more impactful at dim 2, where no generic polynomial-time EndRing
  algorithm is known.

### Milestone 4: the rank-2 over totally-imaginary closed case

Paper 100 §4.3 (Conjecture 2) asks for the reverse reduction
$\mathcal{O}$-nrdPIP → mLIP. In the presentation this becomes: is there a
geometric *problem-preserving* reduction from "find a fixed-degree
endomorphism of a supersingular curve" back to "find a module lattice
isomorphism"? For dim-1 imaginary-quadratic this is vacuous (both easy);
for Hawk's cyclotomic (totally imaginary non-CM-over-$\mathbb{Q}$ setting)
it is substantive. The dim-2 version would ask whether KLPT$^2$ admits a
direct mLIP formulation.

## Literature in `raw/`

All papers stored with both PDF and `markitdown` conversion for search.

### mLIP / Hawk / quaternion reductions (raw/lattices)

| Year | Paper | ePrint | Raw |
|------|-------|--------|-----|
| 2015 | Cramer, Ducas, Peikert, Regev, *Recovering Short Generators of Principal Ideals in Cyclotomic Rings* | [2015/313](https://eprint.iacr.org/2015/313) | `raw/lattices/2015-313-recovering-short-generators-principal-ideals/` |
| 2019 | Lee, Pellet-Mary, Stehle, Wallet, *An LLL Algorithm for Module Lattices* | [2019/1035](https://eprint.iacr.org/2019/1035) | `raw/lattices/2019-1035-lll-for-module-lattices/` |
| 2022 | Ducas, Postlethwaite, Pulles, van Woerden, *Hawk: Module LIP makes lattice signatures fast, compact and simple* | [2022/1155](https://eprint.iacr.org/2022/1155) | `raw/lattices/2022-1155-hawk-module-lip-signatures/` |
| 2024 | Mureau, Pellet-Mary, Pliatsok, Wallet, *Cryptanalysis of Rank-2 Module-LIP in Totally Real Number Fields* (Eurocrypt 2024) | [2024/441](https://eprint.iacr.org/2024/441) | `raw/lattices/2024-441-cryptanalysis-rank-2-module-lip-totally-real/` |
| 2025 | Allombert, Pellet-Mary, van Woerden, *Cryptanalysis of Rank-2 Module-LIP: a Single Real Embedding Is All It Takes* | [2025/280](https://eprint.iacr.org/2025/280) | `raw/lattices/2025-280-rank-2-module-lip-single-real-embedding/` |
| 2025 | Chevignard, Mureau, Espitau, Pellet-Mary, Pliatsok, Wallet, *A Reduction from Hawk to the Principal Ideal Problem in a Quaternion Algebra* | [2025/287](https://eprint.iacr.org/2025/287) | `raw/lattices/2025-287-hawk-reduction-to-principal-ideal-quaternion/` |
| 2025 | Ling, Mendelsohn, Porter, *Dimension-Reducing Algorithms for Quaternion Ideal-SVP* | [2025/1448](https://eprint.iacr.org/2025/1448) | `raw/lattices/2025-1448-dimension-reducing-quaternion-ideal-svp/` |

### Isogeny / Deuring / KLPT / abelian-surface (raw/isogenies)

| Year | Paper | ePrint | Raw |
|------|-------|--------|-----|
| 2014 | Kohel, Lauter, Petit, Tignol, *On the Quaternion $\ell$-Isogeny Path Problem* (KLPT) | [2014/505](https://eprint.iacr.org/2014/505) | `raw/isogenies/2014-505-klpt-quaternion-l-isogeny-path/` |
| 2016 | Galbraith, Petit, Shani, Ti, *On the Security of Supersingular Isogeny Cryptosystems* (GPST) | [2016/859](https://eprint.iacr.org/2016/859) | `raw/isogenies/2016-859-security-supersingular-isogeny-cryptosystems/` |
| 2018 | Castryck, Lange, Martindale, Panny, Renes, *CSIDH: Efficient Post-Quantum Commutative Group Action* | [2018/383](https://eprint.iacr.org/2018/383) | `raw/isogenies/2018-383-csidh-post-quantum-commutative-group-action/` |
| 2021 | Wesolowski, *The Supersingular Isogeny Path and Endomorphism Ring Problems Are Equivalent* | [2021/919](https://eprint.iacr.org/2021/919) | `raw/isogenies/2021-919-supersingular-isogeny-path-endo-ring-equivalent/` |
| 2022 | De Feo, Leroux, Longa, Wesolowski, *New Algorithms for the Deuring Correspondence* | [2022/234](https://eprint.iacr.org/2022/234) | `raw/isogenies/2022-234-new-algorithms-deuring-correspondence-sqisign/` |
| 2022 | Castryck, Decru, *An Efficient Key Recovery Attack on SIDH* | [2022/975](https://eprint.iacr.org/2022/975) | `raw/isogenies/2022-975-efficient-key-recovery-sidh/` |
| 2023 | Page, Wesolowski, *The Supersingular Endomorphism Ring and One Endomorphism Problems Are Equivalent* | [2023/1399](https://eprint.iacr.org/2023/1399) | `raw/isogenies/2023-1399-supersingular-endomorphism-ring-one-endo-equivalent/` |
| 2025 | Castryck, Decru, Kutas, Laval, Petit, Ti, *KLPT$^2$: Algebraic Pathfinding in Dimension Two and Applications* (Crypto 2025) | [2025/372](https://eprint.iacr.org/2025/372) | `raw/isogenies/2025-372-klpt2-algebraic-pathfinding-dimension-two/` |
| 2025 | Borin, Corte-Real Santos, Eriksen, Invernizzi, Mula, Schaeffler, Vercauteren, *Qlapoti: Simple and Efficient Translation of Quaternion Ideals to Isogenies* | [2025/1604](https://eprint.iacr.org/2025/1604) | `raw/isogenies/2025-1604-qlapoti-quaternion-ideals-to-isogenies/` |
| 2019 | Castryck, Decru, Smith, *Hash Functions from Superspecial Genus-2 Curves Using Richelot Isogenies* | [2019/296](https://eprint.iacr.org/2019/296) | `raw/isogenies/2019-296-hash-functions-superspecial-genus2-richelot/` |
| 2026 | Montessinos, *Equivalent Computational Problems for Superspecial Abelian Surfaces* | [2026/120](https://eprint.iacr.org/2026/120) | `raw/isogenies/2026-120-equivalent-problems-superspecial-abelian-surfaces/` |

### Background references (not in `raw/`, external)

- [HR13] Haviv, Regev, *On the Lattice Isomorphism Problem*, arXiv:1311.0366.
- [IKO86] Ibukiyama, Katsura, Oort, *Supersingular Curves of Genus Two and
  Class Numbers*, Compositio Math. 57 (1986).
- [PL21] Porter, Ling, *Reduction Theory of Algebraic Modules and Their
  Successive Minima*, arXiv:2111.06937.
- [DEdP25] Ducas, Engelberts, de Perthuis, *Predicting Module-Lattice
  Reduction*, arXiv:2510.10540.
- [Voi21] Voight, *Quaternion Algebras*, Springer 2021 (standard reference).
- [AAA+25] SQIsign NIST submission, https://sqisign.org.

## Open problems restated

Paper 100 §4 lists three. I add two more that the presentation brings into
focus:

1. (§4.1, Conjecture 1) **General correspondence.** For every rank-$n$
   mLIP over some number field $K$, is there a structured set $S$ of
   algebraic varieties such that mLIP$_K^n$ reduces to endomorphism finding
   in $X \in S$? And is the reverse reduction also in polynomial time?

2. (§4.2, Problem 1) **Isogeny-finding via lattice recovery.** Given
   $E / \mathbb{F}_{p^2}$ supersingular, is there a polynomial-time
   algorithm to recover a rank-2 module lattice $M$ corresponding to $E$
   under Theorem 14, bypassing the EndRing problem?

3. (§4.3, Conjecture 2) **Reverse reduction.** Does
   $\mathcal{O}$-nrdPIP reduce to wc-smodLIP$_K^{\mathbf{B}}$ in polynomial
   time, given pseudo-bases of $\mathcal{O}_\ell(I_M)$ and
   $\mathcal{O}_r(I_M)$?

4. **(New, presentation)** Give a dim-2 analog of Theorem 14: an explicit
   correspondence between IKO matrices $g \in \mathrm{Mat}(A_0)$ (up to
   congruence) and rank-$n$ module lattices over some field $K$. Candidate
   $(n, K)$ pairs worth testing numerically: $(4, \mathbb{Q}(\sqrt{-p}))$,
   $(2, \text{quartic CM})$.

5. **(New, presentation)** Port Hawk cryptanalysis through the dim-$d$
   geometric side: for cyclotomic $K = \mathbb{Q}(\zeta_{2d})$, does Hawk's
   mLIP correspond to isogeny finding between superspecial PPAVs of
   dimension $d$? If yes, can KLPT$^2$ (or its dim-$d$ analog, when it
   exists) be turned into an attack?

## References

- [paper100] (my/presentations/mlip-correspondence/cic2026_1-paper100.pdf),
  Anonymous Submission, *Lattice Isomorphism Problem and Elliptic Curve
  Isogeny Finding Correspondence*, CiC 2026.1.
- [CME+25] Chevignard, Mureau, Espitau, Pellet-Mary, Pliatsok, Wallet,
  ePrint [2025/287](https://eprint.iacr.org/2025/287).
- [MPMPW24] Mureau, Pellet-Mary, Pliatsok, Wallet,
  ePrint [2024/441](https://eprint.iacr.org/2024/441), Eurocrypt 2024.
- [APMvW25] Allombert, Pellet-Mary, van Woerden,
  ePrint [2025/280](https://eprint.iacr.org/2025/280).
- [LMP25] Ling, Mendelsohn, Porter,
  ePrint [2025/1448](https://eprint.iacr.org/2025/1448).
- [DPPvW22] Ducas, Postlethwaite, Pulles, van Woerden,
  ePrint [2022/1155](https://eprint.iacr.org/2022/1155), Hawk.
- [CDPR15] Cramer, Ducas, Peikert, Regev,
  ePrint [2015/313](https://eprint.iacr.org/2015/313), Eurocrypt 2016.
- [LPMSW19] Lee, Pellet-Mary, Stehle, Wallet,
  ePrint [2019/1035](https://eprint.iacr.org/2019/1035).
- [Wes21] Wesolowski, ePrint [2021/919](https://eprint.iacr.org/2021/919).
- [PW23] Page, Wesolowski,
  ePrint [2023/1399](https://eprint.iacr.org/2023/1399).
- [KLPT14] Kohel, Lauter, Petit, Tignol,
  ePrint [2014/505](https://eprint.iacr.org/2014/505).
- [FLLW22] De Feo, Leroux, Longa, Wesolowski,
  ePrint [2022/234](https://eprint.iacr.org/2022/234).
- [BSE+25] Borin, Corte-Real Santos, Eriksen, Invernizzi, Mula, Schaeffler,
  Vercauteren, ePrint [2025/1604](https://eprint.iacr.org/2025/1604).
- [KLPT$^2$] Castryck, Decru, Kutas, Laval, Petit, Ti,
  ePrint [2025/372](https://eprint.iacr.org/2025/372), Crypto 2025.
- [Mon26] Montessinos, *Equivalent Computational Problems for Superspecial
  Abelian Surfaces*, ePrint [2026/120](https://eprint.iacr.org/2026/120).
- [CD22] Castryck, Decru, *An Efficient Key Recovery Attack on SIDH*,
  ePrint [2022/975](https://eprint.iacr.org/2022/975), Eurocrypt 2023.
- [CDS19] Castryck, Decru, Smith, *Hash Functions from Superspecial
  Genus-2 Curves Using Richelot Isogenies*,
  ePrint [2019/296](https://eprint.iacr.org/2019/296), J. Math. Cryptol. 2020.
- [GPST16] Galbraith, Petit, Shani, Ti,
  ePrint [2016/859](https://eprint.iacr.org/2016/859).
- [CLM+18] Castryck, Lange, Martindale, Panny, Renes, *CSIDH*,
  ePrint [2018/383](https://eprint.iacr.org/2018/383).
- [CGL06] Charles, Goren, Lauter, *Cryptographic Hash Functions from
  Expander Graphs*, ePrint [2006/021](https://eprint.iacr.org/2006/021).
- [AAA+25] SQIsign team, https://sqisign.org.
- [HR13] Haviv, Regev, arXiv:1311.0366.
- [IKO86] Ibukiyama, Katsura, Oort, Compositio Math. 57 (1986), 127-152.
- [Piz80] Pizer, *An Algorithm for Computing Modular Forms on $\Gamma_0(N)$*,
  J. Algebra 64 (1980).
- [PL21] Porter, Ling, arXiv:2111.06937.
- [DEdP25] Ducas, Engelberts, de Perthuis, arXiv:2510.10540.
- [Voi21] Voight, *Quaternion Algebras*, Springer 2021.
