# Isogeny-based SNARKs as Wrappers

> Research sketch. Goal: construct a post-quantum SNARK occupying the
> [Groth16](https://eprint.iacr.org/2016/260) design point (very small
> proofs, heavy prover, tight verifier) by leveraging isogeny
> primitives, and use it as an outer wrapper around larger lattice
> SNARKs ([Hachi](../papers/hachi.md) + [Spartan](../papers/spartan.md),
> STARKs, Jolt-Atlas) to compress their proofs to the single-digit-KB
> regime.
>
> Two architectural approaches share motivation, obstacles and
> benchmarks; they are treated in parallel here because the fundamental
> questions (is malleability enough? is full homomorphism achievable?
> can arbitrary-scalar linear combinations be realised?) recur in both.
>
> - **Approach A.** Wrap a lattice SNARK in an outer isogeny *ZK shim*
>   built from the malleable commitments of
>   [Chen-Lai-Laval-Marco-Petit (CLLMP23)](../papers/malleable-commitments.md).
>   Conclusion: direct Groth16-tightness does not drop out; a
>   CLLMP23-based ZK shim over non-ZK Spartan plausibly reaches the
>   single-digit-KB regime.
> - **Approach B.** Build an isogeny-native *multilinear polynomial
>   commitment scheme* and plug it into a standard PIOP. Conclusion:
>   CLLMP23 is not strong enough for a PCS opening; a Pedersen-style
>   commitment *in the class group* (rather than *on the curve set*)
>   appears to be the right primitive.

## Motivation: a post-quantum Groth16

Modern SNARKs are PIOP + PCS compositions. In the elliptic-curve world,
KZG (pairing-based) and IPA (DL-based) give succinct PCS with
homomorphic openings, and Groth16 distils the result to a constant-size
proof by exploiting bilinear pairings directly. In the post-quantum
world the viable families have opposing profiles:

| Family              | Proof size       | Prover cost          | Assumption |
|---------------------|------------------|----------------------|------------|
| Lattice SNARKs      | tens of KB       | linear, GPU-friendly | SIS / RSIS |
| Hash-based STARKs   | hundreds of KB   | linear, hash-heavy   | collision  |
| Isogeny signatures  | hundreds of B    | heavy per-op         | SIDH / CSIDH / SQIsign |

Isogenies sit on a fundamentally different point of the tradeoff curve
from lattices or hashes. Supersingular $j$-invariants are a couple of
field elements; commitment to a curve in the isogeny graph is one group
element. The prover cost is high per operation, but small objects
propagate through the proof. The *shape* of this profile (small
objects, slow proving, tight verifier) is much closer to Groth16's
profile than to anything the lattice world currently produces. That is
the motivation: an isogeny-native (or isogeny-wrapped) SNARK should, in
principle, reach a Groth16-like point on the size axis.

The ARIA [unified proposal](../../../../my/papers/aria/unified-aria-proposal.md)
makes this explicit: the Lattice-Jolt track targets fast proving with
~tens-of-KB proofs, and a parallel exploration asks whether an isogeny
wrapper can further compress to single-digit KB for wrapping scenarios.
Current state of art (SPRINT, ePrint [2026/364](https://eprint.iacr.org/2026/364))
sits at ~80 KB, roughly an order of magnitude above the target and two
orders of magnitude above Groth16.

## The target

| Property        | Groth16          | Hachi + Spartan       | Goal                           |
|-----------------|------------------|------------------------|--------------------------------|
| Proof size      | ~200 bytes       | ~55 KB + $O(\log m)$   | ~1-5 KB                        |
| Post-quantum    | no               | yes                    | yes                            |
| Trusted setup   | yes (per-circuit)| transparent            | transparent                    |
| Prover cost     | linear MSM       | linear (lattice ops)   | superlinear (isogeny evals)    |
| Verifier cost   | $O(1)$ pairings  | sublinear              | sublinear or $O(1)$            |

Compression target: shrink Hachi+Spartan's ~55 KB proof down toward the
single-digit-KB regime, paying in prover time, while preserving
post-quantum security and transparency.

## Approach A: outer ZK shim wrapping a lattice SNARK

The architectural pattern, borrowed from
[NovaBlindFold++](../papers/spartan.md) and
[Vega](https://eprint.iacr.org/2025/2094):

1. Run Hachi + Spartan as an *inner* proof system on the circuit of
   interest. Treat the inner proof as a string $\pi_{\mathrm{inner}}$
   together with a small *verifier* $V_{\mathrm{inner}}$.
2. Encode "$V_{\mathrm{inner}}(\mathrm{vk}, x, \pi_{\mathrm{inner}})
   = 1$" as an NP relation. Its witness is $\pi_{\mathrm{inner}}$; its
   instance is $(\mathrm{vk}, x)$.
3. Prove this NP relation using an *outer*, more succinct proof system,
   revealing nothing about $\pi_{\mathrm{inner}}$.
4. The final proof the verifier sees is just the outer proof; it is
   only valid if the prover genuinely held a satisfying
   $\pi_{\mathrm{inner}}$.

Plugging isogeny machinery into the outer role is attractive because
base objects are small ($\sim 2 \log p$ bits per $j$-invariant,
handfuls of field elements per commitment) and because concrete
sigma-protocol-style isogeny proofs (SQIsign, SQIsign2D, CSI-FiSh)
already achieve hundreds-of-bytes to low-KB signatures for the specific
relation "I know a $2^k$-isogeny between two curves." The question is
whether the *general-purpose* isogeny NP-proof machinery, specifically
CLLMP23, can do the outer role.

### Reality check: CLLMP23 is linear in circuit size

The headline property of CLLMP23's NIZK is that the proof length is
**linear in the circuit size**, with a constant per-gate overhead
determined by whether the gate can be handled by their Protocol B
(cheap, subgroup-structured operations like addition) or their Protocol
A (expensive, $O(|M'|^3)$ per multiplication gate, forcing the message
space $|M'|$ to be small).

This rules out the Groth16-style goal of *constant-size* proofs from
CLLMP23 alone. For any circuit $C$, the outer proof has size
$\Omega(|C|)$. To come close to Groth16, we would need $|C| = O(1)$,
i.e., the verification circuit of the inner proof would have to be of
size $O(1)$ in the statement size. That is not true for any existing
lattice SNARK: Hachi + Spartan's verifier is sublinear but not
constant.

Concretely, the Hachi + Spartan verifier circuit consists of:

- The sum-check verifier's $O(\log m)$ univariate polynomial checks
  (linear in $\log m$, small constant per round).
- The PCS opening verification at a random point: for Hachi this is
  $\tilde O(\sqrt{2^\ell \lambda})$ field operations; for a log-time
  multilinear PCS like Dory or Zeromorph it would be $O(\log m)$.

So the inner verifier circuit is of size $O(\mathrm{poly}\log m)$ at
best, and $\tilde O(\sqrt m)$ for Hachi specifically. Wrapping that
inside CLLMP23 gives an outer proof of size
$O(\lambda \cdot \mathrm{poly}\log m)$ or $\tilde O(\lambda \sqrt m)$
respectively.

For $\lambda = 128$ and $m = 2^{20}$:

- Log-time PCS wrapper: $128 \cdot (\log 2^{20})^k$ for small $k$. At
  $k = 2$ this is $\approx 6$ KB. Plausible target.
- Hachi wrapper: $128 \cdot \sqrt{2^{20}} = 128 \cdot 1024 \approx 16$
  KB. Worse than just shipping Hachi directly.

**So the first concrete finding is that Hachi is the wrong inner PCS
for this wrapping strategy.** If you want to use CLLMP23 as the outer
layer, you want a *log-time* multilinear PCS inside (KZG, Dory,
Zeromorph, Ligero++), not a $\sqrt{\cdot}$ one. But KZG and Dory are
*not* post-quantum, which defeats the whole point. The "log-time PQ
multilinear PCS" slot is currently occupied by Basefold-family and
Brakedown-family schemes, whose verifier circuits are larger than KZG
but still (in principle) log-time; whether one of them fits under
CLLMP23 with acceptable concrete numbers is an open computation.

### Where malleable commitments might specifically help

The malleable-commitment primitive of CLLMP23 is *not* fully
homomorphic; it supports structured transformations of committed
messages for relations that live in a subgroup (Protocol B) or in a
small subset (Protocol A). Three places where these transformations
are potentially useful for a SNARK wrapper:

**(W1) Aggregating sum-check rounds into one malleable claim.**
A sum-check transcript is a sequence of $\log m$ polynomial commits and
$\log m$ verifier challenges. Each round's consistency check is a
linear equation on committed coefficients. If the coefficient
commitments live in a group where *linear combinations* can be
performed malleably (Protocol B), one could hope to fold the $\log m$
separate checks into a single aggregated check provable in constant
isogeny overhead. The catch: CLLMP23's linear-combination malleability
requires the transformation to respect the *group* structure of $M$,
and the sum-check round polynomials' coefficients live in a large
field, so we would need the group action to act on a very large ring.
Protocol B requires $M'$ to be a subgroup, so the cheap case is
*addition*, which works. The multiplicative steps in the sum-check
chain binding (Horner evaluation $g(r) = c_0 + c_1 r + c_2 r^2 +
\cdots$) fall into Protocol A and pay $O(|M'|^3)$. This is the
dominant cost and kills succinctness.

**(W2) ZK shim over a non-ZK SNARK.** If we run Spartan + a log-time
PCS *non-ZK* (fastest prover) and use the isogeny wrapper purely to
hide the inner witness, the outer proof only needs to attest to "I know
a valid non-ZK transcript," which is a fixed, small NP statement.
Structurally this matches how NovaBlindFold and Vega use Nova folding
plus Spartan. An isogeny analogue would replace the Nova folding step
with an isogeny sigma protocol that re-randomises the committed
witness. This does *not* reduce proof size below the inner SNARK's
size (the inner SNARK's proof is still part of the bundle); it only
adds ZK. Not a win for the "compress proof size" axis, but it is the
most concretely achievable use of isogeny-land ZK.

**(W3) Malleable aggregation of evaluation claims.** In a Spartan
pipeline, the outer sum-check reduces to several evaluation claims
$\widetilde{Az}(r), \widetilde{Bz}(r), \widetilde{Cz}(r)$. The verifier
batches them by a random linear combination before the inner sum-check.
If those evaluation claims were *committed in an isogeny-friendly
form*, a single malleable operation could perform the batching inside
the commitment. This is similar in spirit to homomorphic-commitment
batching in lattice SNARKs, but restricted to the Protocol B regime
(linear combinations of committed values). This is the most plausible
concrete use of malleable commitments in this design: a small batching
shim, not a proof-system wrapper.

### A more realistic framing (CLLMP23-based ZK shim)

Given the obstacles, the most useful reframing is to lower the
ambition: *not* Groth16-style constant-size, but "add zero-knowledge to
a non-ZK lattice SNARK using isogeny machinery." Concretely:

1. Prover runs Hachi + Spartan *non-ZK*. Gets a standard transcript
   with some witness-dependent openings (the Spartan σ, θ
   equivalents).
2. Prover commits to each witness-dependent opening using CLLMP23's
   malleable commitment. This is a small number of commits (order
   $kt$, a constant in the size of the statement for a fixed CCS).
3. Prover produces a constant-round isogeny sigma protocol proving
   that the committed openings satisfy the Spartan verifier's final
   binding equation. The equation is a sum (Protocol B, cheap) of
   products (Protocol A, costly but *not* inside a multiplicative
   evaluation chain: the products are already fixed by the protocol,
   so each one pays $O(|M'|^3)$ only once).
4. The verifier reruns Spartan's sum-check transcripts in the clear
   (these are information-theoretically zero-knowledge under Fiat-
   Shamir plus masking; see the
   [zk-neo-folding](./zk-neo-folding.md) analysis), and checks the
   sigma protocol.

Proof size breakdown:

- Sum-check transcripts, masked: $O(\log m \cdot d_Q)$ field elements
  = few KB.
- $O(kt)$ CLLMP23 commitments to Spartan's σ, θ values, each of size
  $O(\lambda)$ bits = $O(kt \lambda)$ bits, at most a few KB for
  realistic $(k, t)$.
- Sigma-protocol transcripts on those commitments: $O(kt)$ challenges
  and responses, another few KB.
- **No inner Hachi opening proof is included**: the witness MLE
  opening is handled by the sigma protocol on committed σ, θ,
  removing the ~55 KB Hachi opening.

Rough total: $O(kt \lambda + \log m \cdot d_Q)$ bits, plausibly in the
single-digit-KB range. Not Groth16-tight, but *significantly smaller
than Hachi alone*, at the cost of an isogeny-heavy prover. This is
essentially the LNP22 / Vega template, translated to isogeny-land:
commit to verifier-witness-dependent openings in a hiding
additively-homomorphic (well, malleable) scheme, and discharge the
final check via a sigma protocol on the committed values.

### Sub-problems specific to Approach A

1. **Realise a malleable commitment to a ring element of the right
   size.** Spartan σ, θ live in the SNARK's prime field (BN254 or
   similar at $\lambda = 128$). CLLMP23 Protocol B wants $M'$ to be a
   subgroup of the group-action message space. Matching these up
   requires picking a CSIDH-style class group with suitable structure,
   or a known-order effective group action with a specific subgroup.
2. **Sigma protocol over CLLMP23 commitments.** The standard Schnorr
   HVZK technique relies on the commitment scheme being additively
   homomorphic. Malleable Protocol-B gives you that for a subgroup
   relation; a Schnorr-style sigma for "$C = \mathrm{Com}(v)$ and $v$
   satisfies linear equation $L$" should work if $L$ lives in the
   subgroup. Multiplicative checks (the product term in Spartan's
   binding equation, $\prod_{j \in S_i} \theta_{j,k}$) need a
   Protocol-A-backed sigma, where the $O(|M'|^3)$ blowup may reappear.
3. **Soundness parameters for the group action.** CSIDH at classical
   $\lambda = 128$ needs very large parameter sets; post-quantum
   security argues for SQIsign-scale rather than CSIDH. The outer
   isogeny sigma protocol's soundness error per round for a known-order
   EGA is $1/|G|$, requiring $|G| \ge 2^\lambda$.
4. **Integration with the inner SNARK's ZK.** If inner Spartan is run
   non-ZK, sum-check transcripts must be separately masked (e.g., via
   the Chiesa-Forbes-Spielman masking-polynomial trick).
5. **End-to-end soundness proof.** Composition of a masked sum-check
   with an isogeny sigma protocol on committed evaluations has not
   been formally analysed in the literature; the obvious reduction
   should go through but needs a careful write-up (analogous to the
   NovaBlindFold++ analysis).

## Approach B: isogeny-native PCS for a general-purpose SNARK

Rather than wrapping a lattice SNARK, the cleanest route to a genuinely
isogeny-native design is to build a *multilinear* (or univariate)
polynomial commitment scheme from isogenies, then plug it into a
standard PIOP (Spartan-over-multilinears, HyperPlonk,
Marlin-over-univariates) to obtain a general-purpose SNARK for
R1CS/CCS/Plonkish with no further design work.

The proof size of a PIOP+PCS SNARK is dominated by the PCS proof
length. A log-time PCS (like KZG) gives polylog proofs; a sublinear PCS
(like Hachi's $\sqrt{\cdot}$) gives tens of KB. If isogenies can
deliver a log-time or *constant-size* PCS, the surrounding PIOP cost
stays negligible.

A multilinear PCS must provide:

1. **Commit.** A short digest $\mathsf{com}(f)$ binding a multilinear
   polynomial $f : \{0,1\}^\ell \to \mathbb{F}$.
2. **Open.** A short proof that $f(r) = y$ for a verifier-chosen
   $r \in \mathbb{F}^\ell$.
3. **Binding and hiding.** Computational binding under a hardness
   assumption; optionally hiding for ZK.

The key structural property every existing efficient PCS uses is some
form of *homomorphism*: KZG relies on group homomorphism in
$\mathbb{G}_1, \mathbb{G}_2$; Dory/IPA on homomorphism of
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

### Attempt 1: malleable commitments as a PCS (fails)

To use CLLMP23 as a PCS, we would commit to the coefficients of the
multilinear polynomial $f$ as a vector of messages, and open at $r$ by
the standard *inner-product opening* trick: compute
$\langle c, \chi(r)\rangle$, where $\chi(r)$ is the multilinear-
Lagrange basis evaluated at $r$, and prove in zero-knowledge that this
inner product equals $y$. Two things go wrong:

**Problem 1: scalars from the full field, not a subgroup.** The
challenge $r \in \mathbb{F}^\ell$ and the Lagrange scalars
$\chi(r) \in \mathbb{F}^{2^\ell}$ are arbitrary field elements. For
Protocol B to apply, every scalar used in the opening must live in the
subgroup $M'$. If $M' = M$, we collapse to plain homomorphism; if $M'$
is a proper subgroup, we cannot open at a generic $r$. Same structural
obstruction that kills (W1) in Approach A.

**Problem 2: polynomial evaluation is multiplication-heavy.** The
multilinear evaluation
$f(r) = \sum_{x \in \{0,1\}^\ell} f(x) \prod_i (r_i x_i + (1-r_i)(1-x_i))$
is an inner product with nonlinear (in $r$) weights. Even if we
precompute $\chi(r)$ for the verifier, the scaling step involves
$2^\ell$ multiplicative gates between committed coefficients and
verifier-chosen scalars. In CLLMP23, each such multiplication pays
Protocol A cost $O(|M'|^3)$, so the opening is $O(2^\ell |M'|^3)$,
catastrophically non-succinct. Restricting the challenge space to $M'$
does not help: sum-check soundness requires
$|M'| \ge \lambda \cdot \ell$ for reasonable error, so $M'$ must be
large, and the $|M'|^3$ blowup dominates.

**Verdict: CLLMP23 malleable commitments do not give a useful PCS.**
The opening protocol wants arbitrary-scalar linear combinations (full
additive homomorphism with field-sized scalars), which is strictly
stronger than what malleability provides. The gap is not a
parameter-tuning issue; it is structural.

### Attempt 2: homomorphic commitments from isogenies

The question is whether isogenies can support *full* homomorphism:

$$
\mathsf{Com}(m_1) \boxplus \mathsf{Com}(m_2) = \mathsf{Com}(m_1 + m_2), \qquad
\alpha \boxdot \mathsf{Com}(m) = \mathsf{Com}(\alpha \cdot m)
$$

for arbitrary $\alpha \in \mathbb{F}$. This is what KZG, Pedersen and
Ajtai give in their respective worlds. The CLLMP23 authors explicitly
note that full additive homomorphism over isogenies seems implausible
because the commitment space $C$ is *not* a group; it is a set (the
supersingular isogeny graph) on which a group $M$ acts. There is no
natural addition on $C$. This is the same reason
[Couveignes-Rostovtsev-Stolbunov](https://eprint.iacr.org/2006/145)
group-action crypto gives key exchange but no ElGamal-style PKE with
additive homomorphism on ciphertexts.

Two escape hatches are worth investigating:

**(H1) Commit *in the class group*, not on the curve set.** If the
commitment is the group element that acts, rather than the resulting
curve, we have additive structure on $M$ for free. Concretely:
$c = (g^r, m + r \cdot h)$ in an additive group $M$, where $g, h$ are
structured elements and $r$ is random; binding reduces to
discrete-log-style problems in $M$. This is Schnorr-Pedersen on the
class group, and it seems to work *if* the class group has a useful
DL-style assumption (it does, for CSIDH-ish sizes). **This is probably
the right technical direction to push.** The resulting commitment is
additively homomorphic in $M$, with binding under group-action-DL and
hiding under the blinding $r$.

**(H2) Pairings on isogenies.** There are no bilinear-pairing analogues
on the supersingular-isogeny graph usable as SNARK primitives; this is
folklore (the Weil and Tate pairings exist on individual curves but do
not lift to the graph structure). SQIsign-style proofs use the
endomorphism ring as the "pairing analogue," but this is not a
commitment operation. Ruling this out leaves (H1) as the only concrete
route.

### A concrete proposal: Pedersen-over-class-group PCS

Sketch, to be validated in detail:

1. Fix a known-order EGA $(M, C, \star)$ with $M$ abelian of order
   $\ge 2^{2\lambda}$. Candidates: CSIDH-2048 (classical $\lambda=128$,
   post-quantum $\lambda \approx 64$, too small), SCALLOP
   ($\lambda=128$ plausibly, larger parameters than CSIDH but class
   group efficiently computable), SQIsignHD-ring ($\lambda=128$ at
   practical sizes, but non-commutative endomorphism structure may
   break the homomorphism).

2. Public parameters: two independent generators $g, h \in M$, and a
   base curve $X \in C$.

3. **Commit** to a vector $c = (c_1, \ldots, c_n) \in M^n$ of messages
   (the PCS takes a polynomial, encoded as coefficient vector in $M$):
   draw $r \leftarrow M$, output
   $$
   \mathsf{com}(c, r) = \left( \left( \sum_i c_i g_i \right) + r \cdot h \right) \star X,
   $$
   where $\{g_i\}$ is a vector of independent generators. This is
   Pedersen in the class group, published as a curve.

4. **Open** at $r \in M^\ell$ by providing
   $y = \langle c, \chi(r) \rangle$ and a sigma protocol proof of
   knowledge of $c, r$ such that $\mathsf{com}(c, r)$ is well-formed
   and $y$ is the correct inner product.

The critical step is step 4. If the inner product
$\langle c, \chi(r) \rangle$ lives in $M$ and commits
additively-homomorphically under $\boxplus$, then a logarithmic-size
IPA-style opening (halving protocol, à la Bulletproofs) should port:
each halving step folds two commitments into one via a verifier-chosen
scalar in $M$. This requires $\alpha \boxdot \mathsf{com}$ for
$\alpha \in \mathbb{F}$, which is *scalar multiplication in $M$*. In
CSIDH-land, scalar multiplication on the class-group side is exactly
repeated group-action composition, which is efficient (no $|M'|^3$
blowup). In CLLMP23 terms this is a Protocol-B-like cost, but because
we are scalar-multiplying in $M$ itself (not applying a group action to
a message in a subgroup), the cost is linear in the bit-length of
$\alpha$, not cubic in $|M|$.

If this goes through, the resulting PCS has:

- **Commit size:** one curve, $2 \log p$ bits. At typical PQ parameters
  $\sim 500$ bytes per commitment.
- **Opening proof:** $O(\log n)$ curves, via halving protocol. For
  $n = 2^{20}$ this is $\sim 20$ curves, $\sim 10$ KB.
- **Prover time:** $O(n)$ class-group compositions plus the
  halving-protocol transcript. Class-group compositions are the slow
  step (seconds to minutes at PQ parameters), which is the
  Groth16-like tradeoff.
- **Verifier time:** $O(\log n)$ curve operations.

$\sim 10$ KB is still above Groth16's 256 bits and above the
single-digit-KB target. Further compression requires either a
constant-size pairing-analogue (not available) or a recursive wrapper
(possible: run the PCS on its own verifier, à la recursive SNARKs).
Recursion in the class-group setting is itself an open problem.

### What needs to hold for Approach B

1. **Class-group DL is hard at $\lambda = 128$ post-quantum.** For
   CSIDH this requires impractical class-group sizes (NIST-level
   parameters near 4096 bits); SCALLOP is the right candidate and its
   concrete assumptions are active research.
2. **Scalar multiplication in $M$ is efficient.** CSIDH's class group
   is commutative and scalar multiplication is repeated small prime
   ideal evaluation; SCALLOP preserves this with a tradeoff on
   parameter size. SQIsign-style non-commutative endomorphism rings
   would break the homomorphism argument; this route is restricted to
   commutative EGAs.
3. **The Pedersen-to-class-group reduction is sound.** Needs
   group-action DL *with public base points* $g, h$. Essentially the
   CSIDH/CSI-FiSh hardness assumption, used additively rather than as
   key-exchange.
4. **IPA halving protocol ports to the class group.** The halving step
   folds $\mathsf{com}_L, \mathsf{com}_R$ into
   $\mathsf{com}_L + x \cdot \mathsf{com}_R$ for verifier scalar $x$.
5. **Soundness of the sum-check over class-group scalars.** If the
   PIOP wants challenges in $\mathbb{F}$ but the commitment's natural
   scalar ring is $\mathrm{Cl}(O)$, there is a mismatch. One fix: pick
   $\mathbb{F}$ isomorphic to (a subring of) $\mathrm{End}(E)$ for a
   special curve $E$, so that PIOP scalars and commitment scalars are
   the same object (a transposition of the SQIsign
   "matching endomorphism ring to proof arithmetic" trick).

## Shared obstacles

Three structural reasons why an "isogeny Groth16" does not drop out of
CLLMP23 alone, regardless of the architectural approach:

1. **No pairing-analogue on isogenies.** Groth16 is tight because it
   leverages bilinear pairings to verify quadratic constraints with
   $O(1)$ pairings. There is no isogeny analogue of a pairing usable
   as an algebraic verifier (this has been studied; negative results
   are folklore). Without such a primitive, any isogeny-based verifier
   of a non-trivial relation must either run a circuit (CLLMP23-style)
   or reduce to a signature-shape relation (SQIsign-style). Neither
   gives Groth16-tightness.
2. **Malleable is weaker than homomorphic.** The standard "batch many
   checks into one" trick in SNARK compression relies on *full*
   additive homomorphism (random linear combination of commitments,
   open one). Malleable commitments support this only when the target
   relation is a subgroup; arbitrary linear combinations over the
   message ring work, but *multiplicative* combinations (needed for
   polynomial-evaluation checks at random points) force Protocol A and
   the $O(|M'|^3)$ blowup. The outer verifier circuit of any modern
   SNARK involves polynomial evaluations and is therefore
   multiplication-heavy.
3. **Prover cost scales with isogeny-graph walks, not MSMs.** Groth16
   accepts a heavy prover because the heavy step is a few large MSMs,
   which are highly parallel and GPU-friendly. Isogeny evaluations at
   SQIsign-scale parameters are notably slower per-operation, so "just
   make the prover slower" does not buy as much as it does in
   pairing-land. Per-gate costs in CLLMP23 include hashing a Merkle
   tree over group-action orbits, and multiplicative gates cost
   $O(|M'|^3)$ such operations. At typical CSIDH parameters with
   $|M'| \approx 2^{20}$, a Protocol-A gate costs $\sim 2^{60}$
   group-action operations, which is astronomical.

Together these three suggest the Groth16-tightness goal is not
reachable with current isogeny primitives plus CLLMP23. A weaker goal,
"use isogeny machinery as a *constant-overhead ZK shim* on top of a
lattice SNARK" (Approach A, variant (W2)), is reachable;
Approach B aims higher but depends on the unvalidated
Pedersen-over-class-group construction.

## Comparison of the two approaches

| Axis               | Approach A (outer ZK shim)    | Approach B (isogeny PCS)               |
|--------------------|-------------------------------|-----------------------------------------|
| Proof size         | single-digit KB               | $\sim 10$ KB (log halving) or better via recursion |
| Prover cost        | isogeny-heavy on shim only    | $O(n)$ class-group compositions         |
| Primitive needed   | CLLMP23 (exists)              | Pedersen-over-class-group (unvalidated) |
| Dependencies       | any lattice SNARK as inner    | PIOP only; no lattice SNARK             |
| Concrete status    | write-up pending, parameters not computed | binding proof + IPA opening pending |
| Obsoletes the other| no                            | yes, if it works                        |

If the class-group-Pedersen PCS works, it obsoletes the CLLMP23-
wrapper approach; if it does not, the wrapper approach remains the
best fallback.

## Open questions

**Shared / Approach A.**

- Is there a Protocol-B-structured commitment scheme over
  isogeny-graph objects for which the Spartan binding equation falls
  into the subgroup regime throughout? This is the make-or-break
  question for getting rid of the multiplicative-gate blowup.
- Does any known-order effective group action other than the
  ideal-class-group-of-CSIDH candidates support the size of message
  space needed for typical SNARK field elements? Replacing CSIDH with
  SCALLOP-family EGAs (class group computable in heuristic polynomial
  time) may be the right answer.
- Can the "isogeny pairing" obstacle be worked around at the
  verifier-circuit level by using an isogeny-based SNARK *for a
  specific, structured verifier language* (e.g., "I know a valid
  Spartan sum-check transcript") rather than for general NP? This is
  the SQIsign-style move and might give genuinely small proofs for
  this *one* language, sidestepping the linear-in-circuit-size problem
  of CLLMP23.
- Does the "known-order EGA" assumption compose cleanly with the
  SIS/RSIS assumption underpinning Hachi? Mixing assumption families
  in a single proof is usually fine but should be validated.

**Approach B.**

- Is the class-group-Pedersen commitment *binding* under
  group-action-DL when the class group has non-trivial torsion? The
  Pedersen argument uses that the group has no small-order elements;
  class groups often do. This may force working in a torsion-free
  subgroup, which complicates parameter selection.
- Does the IPA halving protocol's soundness proof go through with
  scalars living in $\mathrm{Cl}(O)$ rather than a prime field?
  Forking-lemma extraction uses a bound on the number of distinct
  scalar choices; the class-group analogue is a bound on distinct
  ideal classes, which is $|\mathrm{Cl}(O)|$. This is fine if the
  class group is large enough.
- Can we recover a *constant-size* proof (real Groth16 analogue) via a
  further recursive wrapper, where the verifier of the log-size
  class-group PCS is itself proven with the same PCS? Recursion-
  friendly fields are another open problem for isogenies.
- How does the class-group-Pedersen PCS compose with a ZK lattice
  SNARK as its inner proof (the ARIA wrapper scenario)? The straight
  answer: use the lattice SNARK as the inner, produce its (non-ZK)
  verifier circuit, and commit the witness via the class-group PCS.
  This is essentially "Groth16-wraps-STARK" with Groth16 replaced by
  the isogeny-PCS SNARK. The compatibility cost is a circuit
  translation from $\mathbb{F}_q$ (lattice field) to $\mathrm{Cl}(O)$
  (isogeny scalar ring), which probably requires a non-native-
  arithmetic trick.

## Bottom line

A direct "Groth16 analogue over isogenies" does not drop out of
existing isogeny commitments. The fundamental obstacle is that
isogeny-based general-NP proof systems are linear in circuit size, not
succinct, and no isogeny analogue of pairings is known. The Groth16
design point (constant-size proofs, heavy prover) requires primitives
that isogeny-land does not currently provide.

What *does* drop out, in decreasing order of certainty:

- **Approach A (certain).** A CLLMP23-based ZK shim on top of a non-ZK
  lattice SNARK, roughly matching the LNP22 / Vega template but with
  the Pedersen or Nova folding step replaced by malleable commitments
  over an EGA. Reaches single-digit KB, significantly below Hachi's
  ~55 KB, at the cost of an isogeny-heavy prover. Concrete cost and
  exact proof size depend on parameter choices not yet computed.
- **Approach B (plausible).** A Pedersen-style commitment in the class
  group, rather than on the curve set, powering a multilinear PCS and
  therefore an off-the-shelf Spartan-over-multilinears SNARK. Reaches
  $\sim 10$ KB via an IPA halving protocol, assuming binding under
  group-action DL and a clean port of the halving soundness argument
  to $\mathrm{Cl}(O)$. Neither is yet established. If both hold, the
  result matches the Groth16 tradeoff profile but not its constant-size
  point; a further recursive wrapper would be needed to reach that.

The concrete technical deliverable for Approach B, before claiming any
SNARK construction, is a Pedersen-style commitment scheme on a
SCALLOP-scale class group with a proof of binding under group-action DL
and a working IPA-halving opening protocol. Approach A's deliverable is
a parameterised cost model for the CLLMP23 ZK shim instantiated on a
log-time PQ PCS (Basefold or Brakedown family). Either is a
well-defined sub-question of modest research scope.

## Relationship to adjacent ideas

- **[Malleable commitments paper](../papers/malleable-commitments.md).**
  CLLMP23 proves malleability suffices for general-NP ZK at linear
  cost. Approach A uses it as-is; Approach B argues it is *not* enough
  for a succinct PCS, motivating the strictly stronger
  Pedersen-over-class-group primitive.
- **[Zero-knowledge for Neo without Nova](./zk-neo-folding.md).** The
  sister entry for adding ZK to lattice folding, from which much of
  the "mask sum-check + commit-and-prove" template in Approach A is
  borrowed.
- **SPRINT (ePrint [2026/364](https://eprint.iacr.org/2026/364)).**
  Recent isogeny-based SNARK achieving ~80 KB proofs. Comparing the
  SPRINT construction against both approaches above is the right
  empirical benchmark; SPRINT's size suggests room below their
  construction.
- **SQIsign / SQIsign2D.** Achieves ~200-byte signatures for the
  specific relation "I know a $2^k$-isogeny." An isogeny PCS
  piggybacking on SQIsign's signature-shape opening protocol is a
  possible shortcut, but the opening relation for a PCS is multilinear
  evaluation, which does not fit the SQIsign template directly.

## References

- Chen, Lai, Laval, Marco, Petit, *Malleable Commitments from Group
  Actions and Zero-Knowledge Proofs for Circuits based on Isogenies*,
  ePrint [2023/1710](https://eprint.iacr.org/2023/1710). See
  [wiki entry](../papers/malleable-commitments.md).
- Groth, *On the Size of Pairing-Based Non-interactive Arguments*,
  EUROCRYPT 2016, ePrint [2016/260](https://eprint.iacr.org/2016/260).
  The target design point.
- Kaviani, Setty, *Vega: Low-Latency Zero-Knowledge Proofs over
  Existing Credentials*, ePrint [2025/2094](https://eprint.iacr.org/2025/2094).
  Template for the ZK shim in Approach A.
- Kothapalli, Setty, *HyperNova: Recursive arguments for customizable
  constraint systems*, ePrint [2023/573](https://eprint.iacr.org/2023/573),
  Section 7 (NovaBlindFold).
- Bünz, Bootle, Boneh, Poelstra, Wuille, Maxwell, *Bulletproofs:
  Short Proofs for Confidential Transactions and More*, 2018. The IPA
  halving template reused in Approach B.
- De Feo, Kohel, Leroux, Petit, Wesolowski, *SQIsign: Compact
  Post-Quantum Signatures from Quaternions and Isogenies*,
  ASIACRYPT 2020.
- De Feo, Delpech de Saint Guilhem, Fouotsa, Kutas, Leroux, Petit,
  Silva, Wesolowski, *SCALLOP: scaling the CSI-FiSh*, PKC 2023. The
  candidate EGA for $\lambda = 128$ parameters in Approach B.
- Castryck, Lange, Martindale, Panny, Renes, *CSIDH: An Efficient
  Post-Quantum Commutative Group Action*, ASIACRYPT 2018.
- Couveignes; Rostovtsev, Stolbunov, *Group-action-based key exchange*,
  ePrint [2006/145](https://eprint.iacr.org/2006/145).
- Nguyen, Seiler, *Greyhound: Fast Polynomial Commitments from
  Lattices*, ePrint [2024/1293](https://eprint.iacr.org/2024/1293).
  The lattice analogue of what Approach B targets.
- Nguyen, O'Rourke, Zhang, *Hachi*, ePrint [2026/156](https://eprint.iacr.org/2026/156),
  see [wiki entry](../papers/hachi.md). The current PQ PCS baseline
  and the inner SNARK in Approach A.
- Setty, *Spartan*, ePrint [2019/550](https://eprint.iacr.org/2019/550),
  see [wiki entry](../papers/spartan.md). The PIOP used in both
  approaches.
- SPRINT, ePrint [2026/364](https://eprint.iacr.org/2026/364). Most
  recent isogeny-based SNARK; ~80 KB proofs, concrete benchmark to
  beat.
