# Isogeny-based SNARK as a succinct wrapper

> Research sketch. Goal: construct a Groth16-style *constant-or-near-
> constant-size* post-quantum SNARK by wrapping a lattice-based SNARK
> (concretely, [Spartan](../papers/spartan.md) with
> [Hachi](../papers/hachi.md) as the multilinear PCS) in an
> isogeny-based zero-knowledge layer. The analogy to Groth16 is in the
> *tradeoff profile*, not the construction: we aim for very small
> proofs at the cost of a much heavier prover.
>
> This entry asks, specifically, whether the malleable commitments of
> [Chen-Lai-Laval-Marco-Petit (2023/1710)](../papers/malleable-commitments.md)
> can be leveraged to achieve this, and concludes that the direct
> approach does not work, but sketches partial wins and identifies
> what would need to change.

## The target

| Property                         | Groth16        | Hachi + Spartan       | Goal                           |
|----------------------------------|----------------|------------------------|--------------------------------|
| Proof size                       | ~200 bytes     | ~55 KB + $O(\log m)$   | ~1-5 KB                        |
| Post-quantum                     | no             | yes                    | yes                            |
| Trusted setup                    | yes (per-circuit) | transparent         | transparent                    |
| Prover cost                      | linear MSM     | linear (lattice ops)   | superlinear (isogeny evals)    |
| Verifier cost                    | $O(1)$ pairings| sublinear              | sublinear or $O(1)$            |

Compression target: shrink Hachi+Spartan's ~55 KB proof down toward
the single-digit-KB regime, paying in prover time, while preserving
post-quantum security and transparency.

## What an isogeny wrapper would look like

The architectural pattern, borrowed from
[NovaBlindFold++](../papers/spartan.md) and
[Vega](https://eprint.iacr.org/2025/2094):

1. Run Hachi + Spartan as an *inner* proof system on the circuit of
   interest. Treat the inner proof as a string $\pi_{\mathrm{inner}}$
   together with a small *verifier* $V_{\mathrm{inner}}$.
2. Encode "$V_{\mathrm{inner}}(\mathrm{vk}, x, \pi_{\mathrm{inner}})
   = 1$" as an NP relation. Its witness is
   $\pi_{\mathrm{inner}}$; its instance is $(\mathrm{vk}, x)$.
3. Prove this NP relation using an *outer*, more succinct proof
   system, revealing nothing about $\pi_{\mathrm{inner}}$.
4. The final proof the verifier sees is just the outer proof; it is
   only valid if the prover genuinely held a satisfying
   $\pi_{\mathrm{inner}}$.

Plugging isogeny machinery into the outer role is attractive because:

- **Base objects are small**: a supersingular $j$-invariant is
  $\sim 2 \log p$ bits; commitments to points in the isogeny graph
  are typically a handful of field elements.
- **Concrete sigma-protocol-style isogeny proofs already achieve
  compact signatures**: SQIsign, SQIsign2D, CSI-FiSh produce
  signatures in the hundreds-of-bytes to low-KB range for the
  specific relation "I know a $2^k$-isogeny between two curves."

The question is whether the *general-purpose* isogeny NP-proof
machinery, specifically the malleable-commitments NIZK of
Chen-Lai-Laval-Marco-Petit (CLLMP23), can do the outer role.

## The reality check: CLLMP23 is linear in circuit size

The headline property of CLLMP23's NIZK is that the proof length is
**linear in the circuit size**, with a constant per-gate overhead
determined by whether the gate can be handled by their Protocol B
(cheap, subgroup-structured operations like addition) or their
Protocol A (expensive, $O(|M'|^3)$ per multiplication gate, forcing
the message space $|M'|$ to be small).

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

## Where malleable commitments might specifically help

The malleable-commitment primitive of CLLMP23 is *not* fully
homomorphic; it supports structured transformations of committed
messages for relations that live in a subgroup (Protocol B regime) or
in a small subset (Protocol A regime). Three places where these
transformations are potentially useful for a SNARK wrapper:

**(W1) Aggregating sum-check rounds into one malleable claim.**
A sum-check transcript is a sequence of $\log m$ polynomial commits
and $\log m$ verifier challenges. Each round's consistency check is a
linear equation on committed coefficients. If the coefficient
commitments live in a group where *linear combinations* can be
performed malleably (Protocol B), one could hope to fold the $\log m$
separate checks into a single aggregated check provable in
constant isogeny overhead. The catch: CLLMP23's linear-combination
malleability requires the transformation to respect the *group*
structure of $M$, and the sum-check round polynomials' coefficients
live in a large field, so we would need the group action to act on a
very large ring. Protocol B requires $M'$ to be a subgroup, so the
cheap case is *addition*, which works. The multiplicative steps in
the sum-check chain binding (Horner evaluation
$g(r) = c_0 + c_1 r + c_2 r^2 + \cdots$) fall into Protocol A and pay
$O(|M'|^3)$. This is the dominant cost and kills succinctness.

**(W2) ZK shim over a non-ZK SNARK.** If we run Spartan + a log-time
PCS *non-ZK* (fastest prover) and use the isogeny wrapper purely to
hide the inner witness, the outer proof only needs to attest to "I
know a valid non-ZK transcript," which is a fixed, small NP
statement. Structurally this matches how NovaBlindFold and Vega use
Nova folding plus Spartan. An isogeny analogue would replace the
Nova folding step with an isogeny sigma protocol that
re-randomises the committed witness. This does *not* reduce proof
size below the inner SNARK's size (the inner SNARK's proof is still
part of the bundle); it only adds ZK. Not a win for the "compress
proof size" axis, but it is the most concretely achievable use of
isogeny-land ZK.

**(W3) Malleable aggregation of evaluation claims.** In a Spartan
pipeline, the outer sum-check reduces to several evaluation claims
$\widetilde{Az}(r), \widetilde{Bz}(r), \widetilde{Cz}(r)$. The
verifier batches them by a random linear combination before the
inner sum-check. If those evaluation claims were *committed in an
isogeny-friendly form*, a single malleable operation could perform
the batching inside the commitment. This is similar in spirit to
homomorphic-commitment batching in lattice SNARKs, but restricted to
the Protocol B regime (linear combinations of committed values).
This is the most plausible concrete use of malleable commitments in
this design: a small batching shim, not a proof-system wrapper.

## The fundamental obstacles

Three structural reasons why "isogeny Groth16" does not drop out of
malleable commitments alone:

1. **No pairing-analogue on isogenies.** Groth16 is tight because it
   leverages bilinear pairings to verify quadratic constraints with
   $O(1)$ pairings. There is no isogeny analogue of a pairing usable
   as an algebraic verifier (this has been studied; negative results
   are folklore). Without such a primitive, any isogeny-based
   verifier of a non-trivial relation must either run a circuit
   (CLLMP23-style) or reduce to a signature-shape relation (SQIsign-
   style). Neither gives Groth16-tightness.

2. **Malleable is weaker than homomorphic.** The standard "batch many
   checks into one" trick in SNARK compression relies on *full*
   additive homomorphism (take a random linear combination of
   commitments, open one). Malleable commitments support this only
   when the target relation is a subgroup; arbitrary linear
   combinations over the message ring work, but *multiplicative*
   combinations (needed for polynomial-evaluation checks at random
   points) force Protocol A and the $O(|M'|^3)$ blowup. The outer
   verifier circuit of any modern SNARK involves polynomial
   evaluations and is therefore multiplication-heavy.

3. **Prover cost scales with isogeny-graph walks, not MSMs.** Groth16
   accepts a heavy prover because the heavy step is a few large
   MSMs, which are highly parallel and GPU-friendly. Isogeny
   evaluations at SQIsign-scale parameters are notably slower
   per-operation, so "just make the prover slower" does not buy as
   much as it does in pairing-land. Per-gate costs in CLLMP23
   include hashing a Merkle tree over group-action orbits, and
   multiplicative gates cost $O(|M'|^3)$ such operations. At typical
   CSIDH parameters with $|M'| \approx 2^{20}$, a Protocol-A gate
   costs $\sim 2^{60}$ group-action operations, which is
   astronomical. The authors mitigate this by restricting $|M'|$
   sharply, and that is exactly the restriction that kills a
   general-purpose SNARK wrapper.

Together these three suggest that the Groth16-tightness goal is not
reachable with current isogeny primitives plus CLLMP23. A weaker
goal, "use isogeny machinery as a *constant-overhead ZK shim* on top
of a lattice SNARK" (application (W2) above), is reachable.

## A more realistic framing

Given the obstacles, the most useful reframing is to lower the
ambition: *not* Groth16-style constant-size, but "add
zero-knowledge to a non-ZK lattice SNARK using isogeny machinery."
Concretely:

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
4. The verifier rerun Spartan's sum-check transcripts in the clear
   (these are information-theoretically zero-knowledge under Fiat-
   Shamir plus masking; see the
   [zk-neo-folding](./zk-neo-folding.md) analysis), and check the
   sigma protocol.

Proof size breakdown:

- Sum-check transcripts, masked: $O(\log m \cdot d_Q)$ field
  elements = few KB.
- $O(kt)$ CLLMP23 commitments to Spartan's σ, θ values, each of
  size $O(\lambda)$ bits = $O(kt \lambda)$ bits, at most a few KB
  for realistic $(k, t)$.
- Sigma-protocol transcripts on those commitments: $O(kt)$
  challenges and responses, another few KB.
- **No inner Hachi opening proof is included**: the witness MLE
  opening is handled by the sigma protocol on committed σ, θ,
  removing the ~55 KB Hachi opening.

Rough total: $O(kt \lambda + \log m \cdot d_Q)$ bits, plausibly in
the single-digit-KB range. Not Groth16-tight, but *significantly
smaller than Hachi alone*, at the cost of an isogeny-heavy prover.

This is essentially the LNP22 / Vega template, translated to
isogeny-land: commit to verifier-witness-dependent openings in a
hiding additively-homomorphic (well, malleable) scheme, and discharge
the final check via a sigma protocol on the committed values. It is
the concrete thing worth attempting, and it is what the malleable
commitments primitive is actually built for.

## Sub-problems a full construction must solve

1. **Realise a malleable commitment to a ring element of the right
   size.** Spartan σ, θ live in the SNARK's prime field (BN254 or
   similar at $\lambda = 128$). CLLMP23 Protocol B wants $M'$ to be
   a subgroup of the group-action message space. Matching these up
   requires picking a CSIDH-style class group with suitable
   structure, or a known-order effective group action with a
   specific subgroup. This is a concrete parameter-selection
   exercise; it may or may not yield practical numbers.

2. **Sigma protocol over CLLMP23 commitments.** The standard Schnorr
   HVZK technique relies on the commitment scheme being additively
   homomorphic. Malleable Protocol-B gives you that for a subgroup
   relation; a Schnorr-style sigma for "$C = \mathrm{Com}(v)$ and $v$
   satisfies linear equation $L$" should work if $L$ lives in the
   subgroup. Multiplicative checks (the product term in Spartan's
   binding equation, $\prod_{j \in S_i} \theta_{j,k}$) need a
   Protocol-A-backed sigma, which is where the $O(|M'|^3)$ blowup
   may show up again.

3. **Soundness parameters for the group action.** CSIDH at classical
   $\lambda = 128$ needs very large parameter sets; post-quantum
   security argues for SQIsign-scale rather than CSIDH. The outer
   isogeny sigma protocol's soundness error per round must be
   controlled, which for a known-order EGA is $1/|G|$, requiring
   $|G| \ge 2^\lambda$. This is compatible with SQIsign-scale
   parameters but not CSIDH-scale small-class-group parameters.

4. **Integration with the inner SNARK's ZK.** If the inner Spartan
   is run non-ZK, the sum-check transcripts must be separately
   masked (e.g., via the Chiesa-Forbes-Spielman masking-polynomial
   trick). This adds one pre-sumcheck commitment (to the mask) and
   one extra field-element challenge. The mask can itself be
   committed using the CLLMP23 malleable scheme for consistency,
   but a lattice commitment is likely cheaper.

5. **End-to-end soundness proof.** Composition of a masked sum-check
   with an isogeny sigma protocol on committed evaluations has not
   been formally analysed in the literature; the obvious reduction
   should go through but needs a careful write-up (analogous to the
   NovaBlindFold++ analysis).

## Open questions

- Is there a Protocol-B-structured commitment scheme over
  isogeny-graph objects for which the Spartan binding equation falls
  into the subgroup regime throughout? This is the make-or-break
  question for getting rid of the multiplicative-gate blowup.
- Does any known-order effective group action other than the
  ideal-class-group-of-CSIDH candidates support the size of message
  space needed for typical SNARK field elements? Replacing CSIDH
  with SCALLOP-family EGAs (class group computable in heuristic
  polynomial time) may be the right answer.
- Can the "isogeny pairing" obstacle be worked around at the
  verifier-circuit level by using an isogeny-based SNARK *for a
  specific, structured verifier language* (e.g., "I know a valid
  Spartan sum-check transcript") rather than for general NP? This
  is the SQIsign-style move and might give genuinely small proofs
  for this *one* language, sidestepping the linear-in-circuit-size
  problem of CLLMP23.
- Does the "known-order EGA" assumption compose cleanly with the
  SIS/RSIS assumption underpinning Hachi? Mixing assumption
  families in a single proof is usually fine but should be
  validated.

## Bottom line

A direct "Groth16 analogue over isogenies, wrapping a lattice SNARK"
does not drop out of CLLMP23. The fundamental obstacle is that
isogeny-based general-NP proof systems are linear in circuit size,
not succinct, and no isogeny analogue of pairings is known. The
Groth16 design point (constant-size proofs, heavy prover) requires
primitives that isogeny-land does not currently provide.

What *does* drop out is a CLLMP23-based ZK shim on top of a non-ZK
lattice SNARK, roughly matching the LNP22 / Vega template but with
the Pedersen or Nova folding step replaced by malleable commitments
over an EGA. This does not shrink proofs to Groth16 size, but it
can plausibly reach the single-digit-KB regime, significantly below
Hachi's ~55 KB, at the cost of an isogeny-heavy prover. The concrete
cost and exact proof size depend on parameter choices that have not
been computed and are the right next technical step if this line is
pursued.

## References

- Chen, Lai, Laval, Marco, Petit, *Malleable Commitments from Group
  Actions and Zero-Knowledge Proofs for Circuits based on Isogenies*,
  ePrint [2023/1710](https://eprint.iacr.org/2023/1710). See the
  [wiki entry](../papers/malleable-commitments.md) for the
  Protocol A / Protocol B distinction.
- Groth, *On the Size of Pairing-Based Non-interactive Arguments*,
  EUROCRYPT 2016. The target design point.
- Kaviani, Setty, *Vega: Low-Latency Zero-Knowledge Proofs over
  Existing Credentials*, ePrint
  [2025/2094](https://eprint.iacr.org/2025/2094). Template for the ZK
  shim.
- Kothapalli, Setty, *HyperNova: Recursive arguments for
  customizable constraint systems*, ePrint
  [2023/573](https://eprint.iacr.org/2023/573), Section 7
  (NovaBlindFold).
- Nguyen, O'Rourke, Zhang, *Hachi*, ePrint
  [2026/156](https://eprint.iacr.org/2026/156), see
  [wiki entry](../papers/hachi.md).
- Setty, *Spartan*, ePrint
  [2019/550](https://eprint.iacr.org/2019/550), see
  [wiki entry](../papers/spartan.md).
- [Zero-knowledge for Neo without Nova](./zk-neo-folding.md): the
  sister entry for adding ZK to lattice folding, from which much of
  the "mask sum-check + commit-and-prove" template is borrowed.
