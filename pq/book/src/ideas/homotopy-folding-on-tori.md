# Homotopy folding on tori

> Research sketch by Luca Dall'Ava (ICME Labs), *Homotopy and Geodesics on
> Tori, Folding*. The goal is to replace Nova-style linear folding of
> lattice witnesses by a *homotopy* on the torus $\mathbb{R}^d/(2B+1)\mathbb{Z}^d$,
> with the hope of keeping the folded witness inside the norm-bound disk
> $D_B$ for free. Status: speculative; soundness is not argued in the
> memo, and the concrete "knight's tour" instantiation reduces to Nova
> folding with modular wrap.

## Motivation

Standard folding schemes (Nova, HyperNova/Neo, LatticeFold, ...) fold two
witnesses $\mathbf{v}_P, \mathbf{v}_Q$ into $\mathbf{v}_P + r \cdot \mathbf{v}_Q$ for a verifier
challenge $r$. In the lattice setting, the sup-norm of this linear
combination grows with $r$, so every folding step either burns norm
budget or forces an explicit norm-decomposition step that adds prover
work. This is the central pain point LatticeFold and Neo each pay for in
their own way.

The memo asks whether a different folding rule can avoid the norm
blow-up at the source by confining the folded witness to
$D_B := \{x \in \mathcal{R} : \|x\|_\infty \le B\}$ by construction,
using the geometry of the torus $\mathbb{R}^d/(2B+1)\mathbb{Z}^d$ rather
than linear arithmetic in $\mathbb{Z}^d$.

## The proposal in a nutshell

Identify $D_B$ with the discrete torus
$\mathbb{T}_L := \mathbb{Z}_L^d$, $L := 2B+1$, via the bijection between
$\{-B, \ldots, B\}$ and $\{0, \ldots, 2B\}$. Given witnesses
$\mathbf{v}_P, \mathbf{v}_Q \in \mathbb{T}_L$ and a verifier challenge $t$, the prover
outputs a third point $P_t \in \mathbb{T}_L$ that lies on a path
$\ell_{P, Q, r} \subset \mathbb{T}_L$ uniquely determined by $P, Q$ and
some randomness $r$. The folded witness is $P_t$.

Three families of paths are sketched:

1. **Knight's tour on the discrete torus.** Set
   $\mathbf{s} := \mathbf{v}_Q - \mathbf{v}_P \bmod L$ and define
   $P_t := \mathbf{v}_P + t \cdot \mathbf{s} \bmod L$. When $L$ is prime and
   $\mathbf{s} \ne 0$, the orbit is Hamiltonian on $\mathbb{T}_L^d$ (for
   $d = 1$) or on the one-dimensional affine line through
   $(\mathbf{v}_P, \mathbf{s})$ (for $d > 1$). Cycle length is $L$.
2. **Interpolation by $m$ points.** Sample $m$ points $P_1, \ldots, P_m$
   in $\mathbb{T}_L^d$ and use the affine sub-variety
   $V_{P_1, \ldots, P_m} : P_1 + \sum_{i=2}^m t_i (P_i - P_1)$ as the
   sub-variety along which to fold. This generalises the knight's tour
   to higher-degree objects and decouples the randomness from $Q - P$.
3. **Tile-and-shift bookkeeping.** Tile $\mathbb{R}^d$ by copies of
   $D_B$ and, when summing two witnesses, track the translation vector
   $\tau \in \mathbb{Z}^d$ such that
   $\mathbf{v}_P + \mathbf{v}_Q = (\mathbf{v}_P + \mathbf{v}_Q \bmod L) + L \cdot \tau$ as an
   auxiliary committed quantity that is folded alongside the witness.

## Reading the knight's-tour path

A concrete $d = 2$, $L = 17$ example from the memo: starting from
$P = (2, 5)$ with step vector $\mathbf{s} = (5, 3)$, iterating
$P_t := P + t \mathbf{s} \bmod 17$ produces a Hamiltonian orbit of length
17 that visits every column exactly once. In $d = 3$ with $L = 17$ and
$\mathbf{s} = Q - P$, the orbit is still a single cycle of length 17 (not
Hamiltonian on the whole $17^3$ torus, only on its one-dimensional
affine subtorus). For composite $L = 2049 = 3 \cdot 683$ and a step
whose first coordinate is divisible by 3, the cycle collapses to
length 683 and visits only a "triple-staircase" sublattice.

## Commentary

The motivation is right: lattice folding wants a norm-preserving fold,
and thinking geometrically about the witness space as a torus is a
legitimate reframing. But the concrete knight's-tour proposal, as
written, collapses to Nova-style folding modulo a coordinatewise
reduction. Three observations sharpen the picture.

### The knight's tour *is* Nova folding, modulo wrap

The path $P_t = P + t(Q - P) \bmod L$ equals $(1-t)P + tQ \bmod L$,
i.e., the standard linear combination with a coordinatewise reduction.
So the novelty has to come from the modular wrap, not from "homotopy."
The memo's later "interpolation by $m$ points" construction is
analogously the Nova batching rule with wrap.

### The wrap breaks the commitment homomorphism

Ajtai commitments satisfy
$\mathsf{Com}(a\mathbf{v}_P + b\mathbf{v}_Q) = a \cdot \mathsf{Com}(\mathbf{v}_P) + b \cdot \mathsf{Com}(\mathbf{v}_Q)$,
but this fails under coordinatewise reduction mod $L$: the reduced
witness differs from the linear combination by a lattice vector
$L \cdot \tau$ whose commitment is not zero. For the verifier to check
the fold by linearly combining commitments (the reason folding is cheap
at all), the prover has to send side information describing the wrap,
i.e., commit to $\tau$ and fold it alongside the witness. This is
precisely what the memo's "tile-and-shift" idea does, and it is also
essentially what LatticeFold's garbage-term decomposition already does.
In other words, once you make the knight-tour construction sound, it
converges to the existing lattice-folding template.

### The challenge space is small, and extraction is not obvious

For $B = 2^{10}$, the challenge $t$ lives in $\{0, \ldots, 2B\}$ of
size $\approx 2^{11}$, giving soundness error of roughly $2^{-11}$ per
round. Standard Schwartz-Zippel amplification in an extension ring does
not apply because the wrap is a pointwise operation on
$\mathbb{Z}_L^d$, not a polynomial identity over a field extension.
Extraction from two accepting transcripts $P_{t_1}, P_{t_2}$ requires
inverting a $2 \times 2$ linear system modulo $L$, together with
guessing the per-coordinate wrap vector, so the standard Nova-style
two-transcript extractor does not directly apply either. Without a
cleaner extraction strategy, soundness is open.

## Where the construction might still pay off

Two directions from the memo that are not immediate consequences of
Nova folding and that might be worth pursuing on their own:

1. **Tile-and-shift as an alternative to Neo's decomposition.**
   Tracking $\tau$ explicitly is a different way to absorb norm growth
   than Neo's radix-$b$ decomposition. A comparison against Neo's
   decomposition (one extra witness of size $d$ per fold, but
   amortisable across folds) and against LatticeFold's garbage
   commitment would say whether this is a real cost improvement.
2. **Interpolation by $m$ points with a higher-degree curve.** If the
   path is replaced by a higher-degree curve (say degree $m$) over
   $\mathbb{Z}_L^d$, the challenge lives in a larger set and the
   memo's "sub-variety" framing might plausibly recover Schwartz-Zippel-
   style soundness. This then starts looking like a lattice analogue of
   sumcheck folding, which is what Neo does, but parameterised by the
   curve rather than the Boolean cube. Whether anything is gained is
   unclear.

Neither direction is worked out in the memo, and the speculative
"homotopy"/"non-archimedean"/"modular forms" framings that the memo
also lists are separate research threads.

## Suggested next step

Before developing the knight-path formalism further, write down the
two-transcript extractor for the simplest $d = 2$ case and see where
it breaks. If extraction fails, the failure mode identifies exactly
what side information (most likely: the wrap vector $\tau$) the prover
must commit to, which is probably the real shape of any sound protocol
on this track. If extraction succeeds, that by itself is the headline
result and deserves to be written up in isolation before layering on
the higher-degree-curve or tile-and-shift variants.

## Cross-reference

- The [zero-knowledge for Neo without Nova](./zk-neo-folding.md) entry
  discusses the *orthogonal* question of how to blind Neo's folding
  transcript without recursing into a second scheme. The two notes
  attack different problems (norm preservation vs. transcript
  blinding) but both live in the "can we modify Neo's folding from
  inside?" design space.
- [LatticeFold](https://eprint.iacr.org/2024/257) and
  [Neo](https://eprint.iacr.org/2025/294) are the natural baselines
  for "what does existing lattice folding pay for norm control?"

## Open questions

1. **Soundness of the knight's-tour fold.** Construct or rule out a
   two-transcript extractor. If an extractor exists, make the
   side-information it consumes explicit; if not, diagnose the
   obstruction (most likely the $L$-small challenge space or the
   non-linearity of coordinatewise wrap).
2. **Commitment scheme compatibility.** Any sound instantiation needs
   the prover to commit to the wrap vector $\tau$ (or equivalent). Which
   commitment scheme supports the needed linear checks cheaply?
   Ajtai/Neo-matrix is plausible because $\tau$ is short, but the
   analysis has not been done.
3. **Relation to Neo's decomposition.** Does the tile-and-shift
   bookkeeping give a genuinely different cost profile than Neo's
   radix-$b$ witness decomposition, or does it reduce to it with
   different constants?
4. **Higher-degree curves.** If the path is replaced by a
   degree-$m$ curve in $\mathbb{Z}_L^d$, does the challenge space
   enlarge to $L^m$ in a way that gives Schwartz-Zippel-style
   soundness, or does the wrap still collapse the effective challenge
   space to something linear in $L$?
5. **Speculative directions in the memo.** The memo also lists
   non-archimedean norms, modular forms, and isogenies of abelian
   varieties as possible reframings. These are not pursued here;
   each would warrant its own note.
