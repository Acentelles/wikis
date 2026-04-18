# Isogeny Wrapper SNARK from an Isogeny PCS

> Research sketch. Goal: construct a general-purpose, post-quantum
> SNARK whose *polynomial commitment scheme* is isogeny-based, aimed
> at the same design point [Groth16](https://eprint.iacr.org/2016/260)
> occupies in the elliptic-curve world: very small proofs, heavy
> prover, sublinear (ideally constant) verifier. If such a SNARK
> exists, it can play Groth16's role as an outer *wrapper* around
> larger post-quantum SNARKs (Hachi+Spartan, STARKs, Jolt-Atlas) to
> compress their proofs to the single-digit-KB regime.
>
> This entry is distinct from the sibling
> [isogeny-snark-wrapper](./isogeny-snark-wrapper.md) sketch, which
> wraps a lattice SNARK in an outer isogeny *ZK shim* using CLLMP23's
> malleable commitments. Here the question is sharper: can we build
> the *PCS itself* from isogenies, and therefore a fully
> isogeny-native SNARK via any standard PIOP+PCS template (Spartan,
> HyperPlonk, Marlin)? The open technical question is whether
> malleable commitments suffice as the PCS primitive, and, if not,
> whether a genuinely *homomorphic* commitment scheme can be
> constructed from isogenies at all.

## Motivation: a post-quantum Groth16

Modern SNARKs are PIOP + PCS compositions. In the elliptic-curve
world, KZG (pairing-based) and IPA (DL-based) give succinct PCS with
homomorphic openings, and Groth16 distils the result to a
constant-size proof by exploiting bilinear pairings directly. In the
post-quantum world the two viable families have opposing tradeoff
profiles:

| Family              | Proof size       | Prover cost          | Assumption |
|---------------------|------------------|----------------------|------------|
| Lattice SNARKs      | tens of KB       | linear, GPU-friendly | SIS / RSIS |
| Hash-based STARKs   | hundreds of KB   | linear, hash-heavy   | collision  |
| Isogeny signatures  | hundreds of B    | heavy per-op         | SIDH / CSIDH / SQIsign |

Isogenies sit on a fundamentally different point of the tradeoff
curve from lattices or hashes. Supersingular $j$-invariants are a
couple of field elements; commitment to a curve in the isogeny graph
is one group element. The prover cost is high per operation, but
small objects propagate through the proof. The *shape* of this
profile (small objects, slow proving, tight verifier) is much closer
to Groth16's profile (small proof, MSM-heavy prover, $O(1)$ pairings)
than to anything the lattice world currently produces. That is the
motivation: an isogeny-native SNARK should, in principle, reach a
Groth16-like point on the size axis.

The ARIA [unified proposal](../../../../my/papers/aria/unified-aria-proposal.md)
makes this explicit: the Lattice-Jolt track targets fast proving with
~tens of KB proofs, and a parallel exploration asks whether an
isogeny wrapper can further compress to single-digit KB for wrapping
scenarios. Current state of art (SPRINT, ePrint [2026/364](https://eprint.iacr.org/2026/364))
sits at ~80 KB, roughly an order of magnitude above the target and
two orders of magnitude above Groth16.

## The PCS-first formulation

Rather than inventing a bespoke isogeny proof system from scratch,
the cleanest route is to build a *multilinear* (or univariate)
polynomial commitment scheme from isogenies, then plug it into a
standard PIOP. The reasons are operational:

- Any PIOP+PCS template (Spartan-over-multilinears, HyperPlonk,
  Marlin-over-univariates) gives a general-purpose SNARK for
  R1CS/CCS/Plonkish with no further design work, once the PCS is
  available.
- The proof size of a PIOP+PCS SNARK is dominated by the PCS proof
  length. A log-time PCS (like KZG) gives polylog proofs; a
  sublinear PCS (like Hachi's $\sqrt{\cdot}$) gives tens of KB. If
  isogenies can deliver a log-time or *constant-size* PCS, the
  surrounding PIOP cost stays negligible.
- The PCS primitive is well-understood; reducing the "isogeny SNARK"
  question to an "isogeny PCS" question makes the cryptographic
  target precise.

What a multilinear PCS needs to provide, abstractly:

1. **Commit.** A short digest $\mathsf{com}(f)$ binding a multilinear
   polynomial $f : \{0,1\}^\ell \to \mathbb{F}$.
2. **Open.** A short proof that $f(r) = y$ for a verifier-chosen
   $r \in \mathbb{F}^\ell$.
3. **Binding and hiding.** Computational binding under a hardness
   assumption; optionally hiding for ZK.

The key structural property every existing efficient PCS uses is
some form of *homomorphism*: KZG openings rely on group homomorphism
in $\mathbb{G}_1, \mathbb{G}_2$; Dory/IPA on homomorphism of
Pedersen-style commitments; Bulletproofs on the same; Hachi on
Ajtai-homomorphism mod $q$; Ligero/Brakedown on the linearity of
error-correcting codes. The opening protocol in each case is, at its
core, a check that a homomorphic linear combination of committed
values equals a claimed value. Without homomorphism one either pays
Merkle-tree sized proofs per opening (Brakedown regime), or has to
rerandomise each opening by committing from scratch, which kills
succinctness.

So the central question for an isogeny PCS is:

> **Can an isogeny-based commitment scheme support the linear
> combinations required by a PCS opening protocol?**

There are two possible answers. Either (a) the malleable-commitment
primitive of [CLLMP23](../papers/malleable-commitments.md) already
supports enough linearity to run a PCS opening, or (b) it does not
and we need a stronger primitive, a genuinely *homomorphic*
commitment from isogenies, which is currently not known to exist.

## Attempt 1: malleable commitments as a PCS

CLLMP23 gives a commitment scheme from a known-order effective
group action (EGA) where the message space and randomness space are
abelian groups, and commitments support a restricted form of
malleability: for a relation $R$ on $M$, there is an efficient
algorithm that transforms $\mathsf{Com}(m)$ into $\mathsf{Com}(m')$
with $R(m, m')$, without knowing $m$. This is not full homomorphism;
it is the ability to apply structured transformations to committed
messages.

Concretely, CLLMP23 splits gates into two protocols:

- **Protocol B (cheap).** Subgroup-respecting operations; addition
  in $M$ or any linear map with codomain a subgroup. Per-gate cost
  is roughly one group-action evaluation.
- **Protocol A (expensive).** General multiplicative gates; cost is
  $O(|M'|^3)$ per gate in the target subgroup $M'$.

To use this as a PCS, we would commit to the coefficients of the
multilinear polynomial $f$ as a vector of messages, and open at $r$
by the standard *inner-product opening* trick: compute
$\langle c, \chi(r)\rangle$, where $\chi(r)$ is the
multilinear-Lagrange basis evaluated at $r$, and prove in
zero-knowledge that this inner product equals $y$.

The opening protocol reduces to:

1. A **linear combination** of committed coefficients with
   verifier-chosen scalars. This is exactly a Protocol B operation
   *if* the scalars live in the subgroup $M' \subseteq M$.
2. A **consistency check** against the claimed value $y$. This is a
   single equation on committed data, provable with a sigma protocol
   over the malleable commitment.

Two things go wrong:

**Problem 1: scalars from the full field, not a subgroup.** The
challenge $r \in \mathbb{F}^\ell$ and the Lagrange scalars
$\chi(r) \in \mathbb{F}^{2^\ell}$ are arbitrary field elements. For
Protocol B to apply, every scalar used in the opening must live in
the subgroup $M'$. If $M' = M$, we collapse to plain homomorphism;
if $M'$ is a proper subgroup, we cannot open at a generic $r$. This
is the same obstruction that kills the aggregation use in the
sibling [wrapper](./isogeny-snark-wrapper.md) sketch: arbitrary
scalar multiplications are not in the malleable regime.

**Problem 2: polynomial evaluation is multiplication-heavy.** The
multilinear evaluation $f(r) = \sum_{x \in \{0,1\}^\ell} f(x)
\prod_i (r_i x_i + (1-r_i)(1-x_i))$ is an inner product with
nonlinear (in $r$) weights. Even if we precompute $\chi(r)$ for the
verifier, the scaling step involves $2^\ell$ multiplicative gates
between committed coefficients and verifier-chosen scalars. In
CLLMP23, each such multiplication pays Protocol A cost
$O(|M'|^3)$, so the opening is $O(2^\ell |M'|^3)$, catastrophically
non-succinct.

Restricting the challenge space to $M'$ does not help: sum-check
soundness requires $|M'| \ge \lambda \cdot \ell$ for reasonable
error, so $M'$ must be large, and the $|M'|^3$ blowup dominates.

**Verdict: CLLMP23 malleable commitments do not give a useful PCS.**
The opening protocol wants arbitrary-scalar linear combinations
(full additive homomorphism with field-sized scalars), which is
strictly stronger than what malleability provides. The gap is not a
parameter-tuning issue; it is structural.

## Attempt 2: homomorphic commitments from isogenies

If malleability is not enough, the next question is whether
isogenies can support *full* homomorphism, meaning a commitment
scheme with

$$
\mathsf{Com}(m_1) \boxplus \mathsf{Com}(m_2) = \mathsf{Com}(m_1 + m_2), \qquad
\alpha \boxdot \mathsf{Com}(m) = \mathsf{Com}(\alpha \cdot m)
$$

for arbitrary $\alpha \in \mathbb{F}$, where $\boxplus, \boxdot$
are efficient operations on commitments. This is what KZG, Pedersen,
and Ajtai give in their respective worlds.

The CLLMP23 authors explicitly note that full additive homomorphism
over isogenies seems implausible because the commitment space $C$ is
*not* a group; it is a set (the supersingular isogeny graph) on
which a group $M$ acts. There is no natural addition on $C$. This
is the same reason [Couveignes-Rostovtsev-Stolbunov](https://eprint.iacr.org/2006/145)
group-action crypto gives key exchange but no ElGamal-style public
key encryption with additive homomorphism on ciphertexts.

Two escape hatches are worth investigating:

**(H1) Use $M$ itself as the commitment space.** If the commitment
is the *group element* that acts, rather than the resulting curve,
we have additive structure on $M$ for free. The binding argument
then depends on vectorization/group-action-DL style assumptions on
$(M, X, \star)$. This is essentially the "structured-message"
regime of AGAMC, but used differently: the commitment reveals a
group element, not a curve. The question is whether it still hides
$m$. In CSIDH/SCALLOP the group $M$ is the class group and elements
are efficiently represented; the hiding argument would need the
"given $m \cdot X$, recover $m$" problem (group-action DL) to be
hard, *without* the group-element itself being published. That is
the wrong direction: publishing $m$ breaks hiding trivially.

A refinement: commit by $c = (g^r, m + r \cdot h)$ in an additive
group $M$, where $g, h$ are structured elements and $r$ is random.
Binding reduces to discrete-log-style problems in $M$. But the
hiding requires a hash of $r \cdot X$ (for $X$ a fixed curve),
which breaks the homomorphism on $c$. This is Schnorr-Pedersen on
the class group, and it seems to work *if* the class group has a
useful DL-style assumption (it does, for CSIDH-ish sizes). **This
is probably the right technical direction to push.** It is a
Pedersen commitment over the *structured group* $M = \mathrm{Cl}(O)$
of a CSIDH/SCALLOP-style quadratic order. The resulting commitment
is additively homomorphic in $M$, with binding under
group-action-DL and hiding under the blinding $r$.

**(H2) Use pairings on isogenies if they exist.** There are no
bilinear-pairing analogues on the supersingular-isogeny graph
usable as SNARK primitives; this is folklore (the Weil and Tate
pairings exist on individual curves but do not lift to the graph
structure). SQIsign-style proofs use the endomorphism ring as the
"pairing analogue," but this is not a commitment operation.
Ruling this out leaves (H1) as the only concrete route.

## A concrete proposed construction: Pedersen-over-class-group PCS

Sketch, to be validated in detail:

1. Fix a known-order EGA $(M, C, \star)$ with $M$ abelian of order
   $\ge 2^{2\lambda}$. Candidates: CSIDH-2048 (classical $\lambda=128$,
   post-quantum $\lambda \approx 64$, too small), SCALLOP ($\lambda=128$
   plausibly, larger parameters than CSIDH but class group
   efficiently computable), SQIsignHD-ring ($\lambda=128$ at practical
   sizes, but non-commutative endomorphism structure may break the
   homomorphism).

2. Public parameters: two independent generators $g, h \in M$, and
   a base curve $X \in C$.

3. **Commit** to a vector $c = (c_1, \ldots, c_n) \in M^n$ of
   messages (the PCS takes a polynomial, encoded as coefficient
   vector in $M$): draw $r \leftarrow M$, output
   $$
   \mathsf{com}(c, r) = \left( \left( \sum_i c_i g_i \right) + r \cdot h \right) \star X,
   $$
   where $\{g_i\}$ is a vector of independent generators.
   This is Pedersen in the class group, published as a curve.

4. **Open** at $r \in M^\ell$ (the challenge point) by providing
   $y = \langle c, \chi(r) \rangle$ and a sigma protocol proof of
   knowledge of $c, r$ such that $\mathsf{com}(c, r)$ is well-formed
   and $y$ is the correct inner product.

The critical step is step 4. If the inner product
$\langle c, \chi(r) \rangle$ lives in $M$ and commits
additively-homomorphically under $\boxplus$, then a logarithmic-size
IPA-style opening (halving-protocol, a la Bulletproofs) should
port: each halving step folds two commitments into one via a
verifier-chosen scalar in $M$. This requires $\alpha \boxdot \mathsf{com}$
for $\alpha \in \mathbb{F}$, which is *scalar multiplication in
$M$*. In CSIDH-land, scalar multiplication on the class-group side
is exactly repeated group-action composition, which is efficient
(no $|M'|^3$ blowup). In CLLMP23 terms this is a Protocol-B-like
cost, but because we are scalar-multiplying in $M$ itself (not
applying a group action to a message in a subgroup), the cost is
linear in the bit-length of $\alpha$, not cubic in $|M|$.

If this goes through, the resulting PCS has:

- **Commit size:** one curve, $2 \log p$ bits. At typical PQ
  parameters $\sim 500$ bytes per commitment.
- **Opening proof:** $O(\log n)$ curves, via halving protocol.
  For $n = 2^{20}$ this is $\sim 20$ curves, $\sim 10$ KB.
- **Prover time:** $O(n)$ class-group compositions plus the
  halving-protocol transcript. Class-group compositions are the
  slow step (seconds to minutes at PQ parameters), which is the
  Groth16-like tradeoff.
- **Verifier time:** $O(\log n)$ curve operations.

$\sim 10$ KB is still above Groth16's 256 bits and above the
single-digit-KB target. Further compression requires either a
constant-size pairing-analogue (not available) or a recursive
wrapper (possible: run the PCS on its own verifier, à la
recursive SNARKs). Recursion in the class-group setting is itself
an open problem.

## What needs to hold for this to work

1. **Class-group DL is hard at $\lambda = 128$ post-quantum.** For
   CSIDH this requires impractical class-group sizes (NIST-level
   parameters near 4096 bits); SCALLOP is the right candidate and
   its concrete assumptions are active research.

2. **Scalar multiplication in $M$ is efficient.** In CSIDH the class
   group is commutative and scalar multiplication is repeated small
   prime ideal evaluation; SCALLOP preserves this with a tradeoff
   on parameter size. SQIsign-style non-commutative endomorphism
   rings would break the homomorphism argument; this route is
   restricted to commutative EGAs.

3. **The Pedersen-to-class-group reduction is sound.** The standard
   Pedersen security proof uses the DL assumption on a group
   $\mathbb{G}$; the class-group analogue needs group-action DL
   *with public base points* $g, h$. This is essentially the
   CSIDH/CSI-FiSh hardness assumption. The novelty is using it
   *additively* rather than as a key-exchange.

4. **IPA halving protocol ports to the class group.** The halving
   step folds $\mathsf{com}_L, \mathsf{com}_R$ into
   $\mathsf{com}_L + x \cdot \mathsf{com}_R$ for verifier
   scalar $x$. This requires scalar multiplication *of a
   commitment* by a field element. In a class-group Pedersen, this
   is well-defined because the commitment is a curve produced by a
   class-group action, and scalar-multiplying the class-group
   element and re-acting on $X$ gives the desired identity.

5. **Soundness of the sum-check over class-group scalars.** If the
   PIOP wants challenges in $\mathbb{F}$, but the commitment's
   natural scalar ring is $\mathrm{Cl}(O)$, there is a mismatch.
   One fix: pick the PIOP field $\mathbb{F}$ to be isomorphic to (a
   subring of) $\mathrm{End}(E)$ for a special curve $E$, so that
   PIOP scalars and commitment scalars are the same object. This is
   a nontrivial number-theoretic design choice; it is essentially
   the SQIsign "matching endomorphism ring to proof arithmetic"
   trick, transposed.

## Relationship to adjacent ideas

- **Sibling [isogeny-snark-wrapper](./isogeny-snark-wrapper.md).** That
  sketch uses CLLMP23 as an *outer ZK shim* around a lattice SNARK,
  accepting the linear-in-circuit-size proof length. This entry
  instead asks for an isogeny PCS that powers a *native*
  general-purpose SNARK, targeting log-size proofs. The two are
  complementary: if the isogeny-Pedersen PCS works, it obsoletes
  the CLLMP23-wrapper approach; if it does not, the wrapper
  approach remains the best fallback.
- **[Malleable commitments paper](../papers/malleable-commitments.md).**
  CLLMP23 proves malleability suffices for general-NP ZK at linear
  cost. The current sketch argues malleability is *not* enough for
  a succinct PCS, and that a strictly stronger primitive
  (group-action-Pedersen) is needed.
- **SPRINT (ePrint [2026/364](https://eprint.iacr.org/2026/364)).**
  Recent isogeny-based SNARK achieving ~80 KB proofs. Comparing the
  SPRINT construction against a class-group-Pedersen PCS + Spartan
  PIOP is the right empirical benchmark; SPRINT's size suggests
  there is room below their construction.
- **SQIsign / SQIsign2D.** Achieves ~200-byte signatures for the
  specific relation "I know a $2^k$-isogeny." An isogeny PCS
  piggybacking on SQIsign's signature-shape opening protocol is a
  possible shortcut, but the opening relation for a PCS is
  multilinear evaluation, which does not fit the SQIsign template
  directly.

## Open questions

- Is the class-group-Pedersen commitment *binding* under
  group-action-DL when the class group has non-trivial torsion? The
  Pedersen argument uses that the group has no small-order
  elements; class groups often do. This may force working in a
  torsion-free subgroup, which complicates parameter selection.
- Does the IPA halving protocol's soundness proof go through with
  scalars living in $\mathrm{Cl}(O)$ rather than a prime field?
  Forking-lemma extraction uses a bound on the number of distinct
  scalar choices; the class-group analogue is a bound on distinct
  ideal classes, which is $|\mathrm{Cl}(O)|$. This is fine if the
  class group is large enough.
- Can we recover a *constant-size* proof (real Groth16 analogue)
  via a further recursive wrapper, where the verifier of the
  log-size class-group PCS is itself proven with the same PCS?
  Recursion-friendly fields are another open problem for isogenies.
- How does the class-group-Pedersen PCS compose with a ZK lattice
  SNARK as its inner proof (the ARIA wrapper scenario)? The
  straight answer: use the lattice SNARK as the inner, produce its
  (non-ZK) verifier circuit, and commit the witness via the
  class-group PCS. This is essentially "Groth16-wraps-STARK" with
  Groth16 replaced by the isogeny-PCS SNARK. The compatibility
  cost is a circuit translation from $\mathbb{F}_q$ (lattice
  field) to $\mathrm{Cl}(O)$ (isogeny scalar ring), which
  probably requires a non-native-arithmetic trick.

## Bottom line

The structural reason an "isogeny Groth16" has not emerged from
existing isogeny commitments is that CLLMP23 malleable commitments
are strictly weaker than homomorphic commitments and cannot
instantiate a PCS opening protocol. The natural next step is to
construct a genuinely additively-homomorphic commitment from
isogenies by committing *in the class group* rather than *on the
curve set*, using a Pedersen-style construction with security
reducing to group-action DL. If this works, an off-the-shelf
Spartan-over-multilinears PIOP gives a general-purpose isogeny
SNARK with log-size proofs and heavy class-group prover cost. This
matches the Groth16 tradeoff profile, though not yet its
constant-size point.

The concrete technical deliverable of this line of work, before
claiming any SNARK construction, is a Pedersen-style commitment
scheme on a SCALLOP-scale class group with a proof of binding under
group-action DL and a working IPA-halving opening protocol. Sizing
and prover-time benchmarks would follow. This is a well-defined
sub-question with a yes/no answer and modest research scope.

## References

- Chen, Lai, Laval, Marco, Petit, *Malleable Commitments from Group
  Actions and Zero-Knowledge Proofs for Circuits based on
  Isogenies*, ePrint [2023/1710](https://eprint.iacr.org/2023/1710).
  See [wiki entry](../papers/malleable-commitments.md).
- Groth, *On the Size of Pairing-Based Non-interactive Arguments*,
  EUROCRYPT 2016, ePrint [2016/260](https://eprint.iacr.org/2016/260).
- Bünz, Bootle, Boneh, Poelstra, Wuille, Maxwell, *Bulletproofs:
  Short Proofs for Confidential Transactions and More*, 2018. The
  IPA halving template.
- De Feo, Kohel, Leroux, Petit, Wesolowski, *SQIsign: Compact
  Post-Quantum Signatures from Quaternions and Isogenies*,
  ASIACRYPT 2020.
- De Feo, Delpech de Saint Guilhem, Fouotsa, Kutas, Leroux, Petit,
  Silva, Wesolowski, *SCALLOP: scaling the CSI-FiSh*, PKC 2023.
  The candidate EGA for $\lambda = 128$ parameters.
- Castryck, Lange, Martindale, Panny, Renes, *CSIDH: An Efficient
  Post-Quantum Commutative Group Action*, ASIACRYPT 2018.
- Nguyen, Seiler, *Greyhound: Fast Polynomial Commitments from
  Lattices*, ePrint [2024/1293](https://eprint.iacr.org/2024/1293).
  The lattice analogue of what this sketch targets in isogeny-land.
- Nguyen, O'Rourke, Zhang, *Hachi*, ePrint [2026/156](https://eprint.iacr.org/2026/156),
  see [wiki entry](../papers/hachi.md). The current PQ PCS baseline.
- Setty, *Spartan*, ePrint [2019/550](https://eprint.iacr.org/2019/550),
  see [wiki entry](../papers/spartan.md). The PIOP the proposed PCS
  would slot into.
- SPRINT, ePrint [2026/364](https://eprint.iacr.org/2026/364). The
  most recent isogeny-based SNARK; ~80 KB proofs, the concrete
  benchmark to beat.
- Sibling entry: [Isogeny-based SNARK as a succinct wrapper](./isogeny-snark-wrapper.md).
