# Spartan

> Srinath Setty
> *Spartan: Efficient and general-purpose zkSNARKs without trusted setup*
> CRYPTO 2020, ePrint [2019/550](https://eprint.iacr.org/2019/550)

Background entry, referenced from the
[LIP arithmetisation idea](./lip-arithmetisation-idea.md) and anywhere else
on this wiki that invokes "sum-check-based SNARK" as a cost model.

## TL;DR

Spartan is a transparent (no trusted setup) zkSNARK for R1CS whose prover
is linear in the number of non-zero R1CS entries and whose verifier is
sublinear. The cryptographic core is **multilinear polynomial commitments
plus the sum-check protocol**, applied twice: an **outer** sum-check
certifies that all R1CS constraints are satisfied, and an **inner**
sum-check reduces the resulting evaluation claim about the sparse
matrices $A, B, C$ to one evaluation at a random point. Both sum-checks
batch all constraints into a single polynomial identity, which is why
Spartan (and its descendants: HyperPlonk, Jolt, Nova's later folding
composition, …) "pays per polynomial identity" rather than per
constraint.

## Problem statement

The relation Spartan proves is **R1CS satisfiability**:

> Given public matrices $A, B, C \in \mathbb{F}^{m \times n}$ and a public
> input $x \in \mathbb{F}^{|x|}$, does there exist a witness
> $w \in \mathbb{F}^{n - |x| - 1}$ such that the assembled vector
> $z = (1, x, w) \in \mathbb{F}^n$ satisfies
> $(A z) \circ (B z) = C z$?

The symbol $\circ$ is the Hadamard (entry-wise) product. Writing out the
$i$-th coordinate, $(Az)_i \cdot (Bz)_i = (Cz)_i$ is the familiar "one
rank-1 quadratic constraint per row" picture.

Spartan then arithmetises this relation via multilinear extensions.

## Multilinear extensions, briefly

Let $\mu := \lceil \log_2 m \rceil$. Any function $g: \{0,1\}^\mu \to
\mathbb{F}$ has a unique **multilinear extension (MLE)** $\widetilde g:
\mathbb{F}^\mu \to \mathbb{F}$: the unique polynomial of individual
degree 1 in each variable that agrees with $g$ on the Boolean cube.
Concretely
$$
\widetilde g(X_1, \ldots, X_\mu) \;=\; \sum_{b \in \{0,1\}^\mu}
g(b) \cdot \prod_{j=1}^\mu \bigl(X_j b_j + (1 - X_j)(1 - b_j)\bigr).
$$
The product factor is the Lagrange indicator; it equals 1 when $X = b$
and 0 elsewhere on the cube. Two properties matter:

1. $\widetilde g(b) = g(b)$ for all $b \in \{0,1\}^\mu$.
2. Two multilinear polynomials that disagree anywhere disagree at most
   at a $\mu / |\mathbb{F}|$ fraction of points (Schwartz–Zippel),
   so a random evaluation is an unforgeable fingerprint.

Indexing an $m$-dimensional vector $v \in \mathbb{F}^m$ by bit-strings
$b \in \{0,1\}^\mu$ gives a function $\{0,1\}^\mu \to \mathbb{F}$ whose
MLE $\widetilde v$ we write with the same letter. Similarly, an
$m \times n$ matrix is a function on $\{0,1\}^{\mu+\nu}$ with MLE
$\widetilde A, \widetilde B, \widetilde C$.

## The outer sum-check

Define the residual polynomial
$$
F(X) \;:=\; \widetilde{Az}(X) \cdot \widetilde{Bz}(X) \;-\; \widetilde{Cz}(X),
$$
where $\widetilde{Az}$ is the MLE of the vector $Az \in \mathbb{F}^m$ and
similarly for $B, C$. "All $m$ R1CS constraints hold" is equivalent to
"$F(b) = 0$ for every $b \in \{0,1\}^\mu$."

The verifier picks a random $\tau \in \mathbb{F}^\mu$ and the two parties
run the sum-check protocol on
$$
0 \;\stackrel{?}{=}\; \sum_{b \in \{0,1\}^\mu} \widetilde{\mathrm{eq}}(\tau, b)
\cdot F(b),
$$
where $\widetilde{\mathrm{eq}}(\tau, b) = \prod_j (\tau_j b_j + (1 - \tau_j)(1 - b_j))$
is the MLE of the indicator $[b = \tau]$.

Why the $\widetilde{\mathrm{eq}}$ factor: multiplying $F$ by
$\widetilde{\mathrm{eq}}(\tau, \cdot)$ and summing over the cube recovers
$F(\tau)$ (the MLE of $F$ evaluated at $\tau$), because
$\widetilde{\mathrm{eq}}$ *is* the Lagrange indicator of $\tau$. Proving
the sum equals 0 for a random $\tau$ therefore proves, whp, that $F$
vanishes on the whole cube, which is the R1CS statement.

The sum-check runs $\mu = \log_2 m$ rounds, with per-round prover work
$O(m)$ and polynomial degree 3 (the product of an MLE $\widetilde{Az}$,
an MLE $\widetilde{Bz}$, and the linear-in-each-variable
$\widetilde{\mathrm{eq}}$), ending with the verifier holding a claim
about the value of $F$ at a random point $r_x \in \mathbb{F}^\mu$:
$$
F(r_x) \;=\; \widetilde{Az}(r_x) \cdot \widetilde{Bz}(r_x) - \widetilde{Cz}(r_x).
$$

The verifier cannot compute the three MLEs $\widetilde{Az}, \widetilde{Bz},
\widetilde{Cz}$ at $r_x$ by itself without linear work, so another step
is needed.

## The inner sum-check

The vector $Az \in \mathbb{F}^m$ is the matrix-vector product of $A$ and
$z$, and for any index $i \in \{0,1\}^\mu$,
$$
(Az)_i \;=\; \sum_{j \in \{0,1\}^\nu} A[i, j] \cdot z_j.
$$
Taking multilinear extensions,
$$
\widetilde{Az}(r_x) \;=\; \sum_{j \in \{0,1\}^\nu}
\widetilde A(r_x, j) \cdot \widetilde z(j).
$$
This is another sum over a Boolean cube; Spartan batches the three such
claims (one per matrix $A, B, C$) via a random linear combination with
verifier-sampled coefficients $r_A, r_B, r_C$, and runs a second
sum-check, the **inner sum-check**, on
$$
r_A \widetilde{Az}(r_x) + r_B \widetilde{Bz}(r_x) + r_C \widetilde{Cz}(r_x)
\;=\; \sum_{j} \bigl(r_A \widetilde A(r_x, j) + r_B \widetilde B(r_x, j)
+ r_C \widetilde C(r_x, j)\bigr) \cdot \widetilde z(j).
$$
This sum-check runs $\nu = \log_2 n$ rounds and ends with the verifier
holding two evaluation claims:

- One about the witness MLE $\widetilde z$ at a random point $r_y$.
- One about the batched matrix MLE $r_A \widetilde A + r_B \widetilde B +
  r_C \widetilde C$ at $(r_x, r_y)$.

The witness claim is discharged by a **polynomial commitment opening**;
the matrix claim is discharged by an encoding of the sparse R1CS
matrices. Spartan's two headline instantiations differ in how each of
these is implemented.

## Polynomial commitments: Spartan$_{\text{DL}}$ and Spartan$_{\text{NIZK}}$

The paper instantiates the commitment to $\widetilde z$ two ways:

- **Hyrax-style commitment** (Spartan$_{\text{NIZK}}$): square-root
  proof size, prover time $O(n)$, verifier time $O(\sqrt{n})$, based on
  discrete-log. No trusted setup.
- **Linear-time commitment via a tensor-product code**
  (Spartan$_{\text{DL}}$): linear prover time for commitment, sublinear
  verifier, transparent.

Later follow-ups plug in KZG-style (Dory, Zeromorph, Ligero++) or
FRI-based (Brakedown, Basefold) multilinear commitments with different
tradeoffs; the sum-check skeleton is unchanged.

For the **sparse matrices** $A, B, C$, Spartan uses a dedicated
**SparseMLE commitment** scheme that commits once to the encoding of
the R1CS structure at preprocessing time, and later lets the verifier
check $\widetilde A(r_x, r_y)$ at an arbitrary point with $O(\log n)$
work. The scheme's own correctness reduces to yet another sum-check
(sometimes called the "memory-checking sum-check") run over the
sparse-matrix encoding.

## Why "pays per polynomial identity"

Spartan's entire verifier interaction with the R1CS side consists of
**two sum-checks and one polynomial evaluation**, regardless of how many
constraints $m$ there are. A new batch of constraints does not add a new
sum-check; it only makes the one sum-check bigger ($\mu = \log m$ rounds
instead of $\mu' = \log m'$). The proof size scales with $\log m$, not
$m$. The prover work scales with $m$ once, not $m$ times
"constraint-checking overhead."

This is the exact sense in which sum-check-based SNARKs "pay per
polynomial identity." In Spartan there are two polynomial identities
(outer: constraint satisfaction; inner: matrix–witness inner product),
each handled by one sum-check. Anything you can fold into *one* of
those identities is free, up to the cost of making its constituent MLEs
larger.

## What Spartan gave us, practically

1. **Transparent.** No trusted setup, unlike Groth16 / PLONK with KZG.
2. **Linear-time prover.** No FFTs, no MSMs over structured groups, no
   QAP. A direct R1CS-to-sum-check compiler.
3. **Sublinear verifier.** $O(\log m + \sqrt{n})$ or $O(\log m + \log n)$
   depending on the commitment.
4. **A reusable skeleton.** HyperPlonk (plonkish + sum-check), Jolt
   (CPU zkVMs via the lookup arguments + Spartan-style outer check),
   Nova / HyperNova / Protostar (folding schemes composed with a final
   Spartan-style proof), Lasso (lookup arguments whose correctness
   reduces to Spartan-compatible sum-checks).

## Cost model cheat sheet

Let $m$ be the number of R1CS constraints, $N$ the number of non-zero
entries of $A, B, C$, $n$ the witness size.

| Quantity        | Cost                                |
|-----------------|-------------------------------------|
| Prover time     | $O(N + n)$ field ops                |
| Prover memory   | $O(N + n)$                          |
| Verifier time   | $O(\log m + \sqrt n)$ (Hyrax) or $O(\log m + \log n)$ (KZG) |
| Proof size      | $O(\log m + \sqrt n)$ or $O(\log m + \log n)$ |
| Preprocessing   | $O(N)$ (once, to commit to $A, B, C$) |

The $O(\log m)$ term is the sum-check transcript; the $\sqrt n$ or
$\log n$ term is the multilinear commitment opening of $\widetilde z$.

## Why this matters for LIP, HAWK, and friends

For arithmetisations that produce many scalar residuals with a clean
polynomial structure, like the LIP residuals $\{f_{i,k}\}$ in
[this entry](./lip-arithmetisation-idea.md), Spartan (or any of its
descendants) lets you package all residuals into a single multilinear
extension and certify them with one sum-check. That is the mechanism by
which the LIP arithmetisation's $N = n(n+1)/2$ residuals collapse to a
$\log N$-sized proof and $O(n^2)$ prover work, rather than $n^4$.

Nothing about the Spartan framework is lattice- or isogeny-specific;
this entry lives in the isogenies wiki only because several other
entries here invoke it as a cost model.

## Things to read next

- HyperPlonk (Chen, Bünz, Boneh, Zhang; ePrint 2022/1355): plonkish
  constraints instead of R1CS, same sum-check skeleton.
- Jolt (Arun, Setty, Thaler; ePrint 2023/1217): a zkVM whose execution
  correctness is proved via Spartan-style sum-checks plus Lasso lookup
  arguments.
- Lasso (Setty, Thaler, Wahby; ePrint 2023/1216): lookup arguments
  reduced to sum-check.
- Basefold / Brakedown / Zeromorph / Dory: multilinear polynomial
  commitment schemes you can plug into the Spartan skeleton to change
  the (prover time, proof size, verifier time) tradeoff point.
