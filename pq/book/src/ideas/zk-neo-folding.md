# Adding zero-knowledge to Neo without Nova

> Project idea / research sketch. The goal is to make the Neo (lattice
> folding scheme for CCS, ePrint
> [2025/294](https://eprint.iacr.org/2025/294)) prover zero-knowledge
> using a Chiesa-Forbes-Spielman-style **masking polynomial** combined
> with hiding **commitments to the $\sigma_{j,k}$ and $\theta_{j,k}$
> messages** of the multi-folding sum-check, **without** invoking a
> separate Nova-style R1CS folding step (the "NovaBlindFold" trick used
> by the original HyperNova ZK layer and by Vega/BlindFold).

In the protocol descriptions below, plain text is the original Neo
protocol (Construction $\Pi_{\mathrm{CCS}}$, Section 4.4 of
[2025/294](https://eprint.iacr.org/2025/294), itself an adaptation of
HyperNova's Construction 1, Section 4 of
[2023/573](https://eprint.iacr.org/2023/573)). New ZK additions are in
<span style="color:#1f6feb">**blue**</span>; replacements of clear-text
messages by hiding commitments are in <span style="color:#cf222e">**red**</span>.

## TL;DR

1. The HyperNova/Neo multi-folding scheme (Construction 1) is **not
   blinding** as defined in Section 7 of HyperNova: the prover's
   transcript reveals (i) all $s$ sum-check round polynomials and (ii)
   the matrix-witness inner products $\sigma_{j,k} = \sum_y
   \widetilde{M}_j(r'_x, y)\, \widetilde{z}_{1,k}(y)$ and $\theta_{j,k} =
   \sum_y \widetilde{M}_j(r'_x, y)\, \widetilde{z}_{2,k}(y)$. Both are
   linear in the witness MLE.
2. HyperNova fixes this by composing the multi-folding step with a
   *separate* Nova R1CS folding step (Construction 3), whose blinding
   property (Lemma 5) hides the multi-folding output. This is the basis
   of NovaBlindFold and of Vega's BlindFold protocol used in Jolt.
3. **Neo lives in the lattice setting and has no Nova analogue to
   delegate blinding to.** Adding ZK by recursively folding into a
   second lattice scheme is possible (LatticeFold could play the role)
   but inherits all of LatticeFold's overheads (cyclotomic ring
   commitments, large field operations) that Neo was specifically
   designed to avoid.
4. We propose two compatible modifications that make Neo's
   $\Pi_{\mathrm{CCS}}$ blinding **on its own**:
   - **Round-polynomial masking** (Approach A): add a fresh random
     polynomial $h$ of the same degree profile as $Q$ to the sum-check
     polynomial; the prover commits to $h$ once (hidingly) at the start
     of the protocol, and the sum-check is run on $Q + \rho \cdot h$
     for a verifier challenge $\rho$. The round polynomials of
     $Q + \rho h$ are uniform conditioned on the transcript.
   - **Commit-and-prove for $\sigma, \theta$** (Approach B): instead
     of sending $\sigma_{j,k}, \theta_{j,k}$ in the clear, send hiding
     Ajtai/Pedersen commitments $C_{\sigma}, C_{\theta}$ and prove the
     final sum-check binding equation $c = \cdots$ holds inside the
     commitments via a sigma protocol over the Ajtai/Pedersen group.
5. Combining A and B yields a **standalone zero-knowledge variant of
   Neo's $\Pi_{\mathrm{CCS}}$** that does not require recursing into a
   second proof system. Soundness loss is one extra Schwartz-Zippel
   round; prover overhead is one extra commit to a sum-check-shaped
   masking polynomial and a small sigma protocol; verifier overhead is
   constant in the witness size.

## What we are protecting

Recall the Neo $\Pi_{\mathrm{CCS}}$ reduction's externally visible
prover messages:

- The **sum-check transcript**: $s = \log(d \cdot n)$ univariate round
  polynomials $s_i(X) \in K[X]$, each of degree at most $d_Q + 1$
  where $d_Q$ is the total degree of the sum-check polynomial $Q$.
- The **final claim opening**: per-matrix evaluations
  $y'_{(i,j)} \in K^d$ for $j \in [t]$, $i \in [k]$. In HyperNova
  notation these are exactly the $\sigma_{j,k}, \theta_{j,k}$ values
  written above. They are linear functions of the committed witnesses
  $z_{1,k}, z_{2,k}$.
- (After the sum-check, in $\Pi_{\mathrm{RLC}}$ and $\Pi_{\mathrm{DEC}}$,
  more values are sent. We focus on $\Pi_{\mathrm{CCS}}$ here; the
  same masking technique applies to the inner sum-check inside
  $\Pi_{\mathrm{RLC}}$, with no new ideas.)

Both classes of messages encode information about the witness:

1. **Round polynomials.** Each $s_i$ is a degree-$(d_Q+1)$ univariate
   polynomial whose evaluations at $0, 1, \ldots, d_Q + 1$ are partial
   sums of $Q$. By the structure of $Q$, these partial sums are linear
   combinations of products of $\widetilde{z}$-values restricted to a
   sub-cube. They are *not* uniformly random; an adversary that sees
   them learns linear information about the witness.
2. **Final $\sigma, \theta$.** The values $\sigma_{j,k} =
   \widetilde{M}_j(r'_x, \cdot)^\top \widetilde{z}_{1,k}$ and the
   analogous $\theta$ are inner products of a verifier-known vector
   with the witness. Even a single $\sigma$ leaks one linear function
   of the witness exactly.

A simulator without the witness cannot produce a transcript with the
same joint distribution of these values, because they are
deterministically tied to the witness via the matrices and verifier
challenges. This is the obstruction the user flagged: applying a
naive simulator to Neo's $\Pi_{\mathrm{CCS}}$ would require knowing
the witness in order to reproduce $s_i$ and $\sigma, \theta$.

## Original Neo protocol $\Pi_{\mathrm{CCS}}$ (clear-text version)

We restate the protocol in our notation. Let $Q(X_1, \ldots, X_{\log
dn})$ denote the multivariate polynomial constructed in Step 2 below,
$T$ its claimed sum over the Boolean cube, $s := \log(dn)$ the number
of sum-check rounds, and $d_Q$ the total degree of $Q$.

**Inputs.** A linearised committed CCS instance-witness pair
$(\mathbf{u}_1, \mathbf{w}_1)$ and a committed CCS instance-witness
pair $(\mathbf{u}_2, \mathbf{w}_2)$. Public parameters fix matrices
$M_1, \ldots, M_t$, predicate $F$, norm bound $b$, decomposition radix.

**Steps.**

1. $V \to P$: sample and send challenges $\alpha \in K^{\log d}$,
   $\beta \in K^{\log(dn)}$, $\gamma \in K$.
2. $V \leftrightarrow P$: define
   $$
   Q(X_{[1, \log dn]})
   \;:=\;
   \widetilde{\mathrm{eq}}(X, \beta) \cdot
   \Bigl( F(X_{[\log d + 1, \log dn]})
   + \sum_{i \in [k]} \gamma^i \cdot \mathrm{NC}_i(X) \Bigr)
   + \gamma^k \sum_{\substack{j \in [t] \\ i \in [2, k]}}
     \gamma^{i + (j-1)k - 1} \cdot \mathrm{Eval}_{(i,j)}(X),
   $$
   with the claimed sum
   $$
   T \;:=\; \gamma^k \sum_{\substack{j \in [t] \\ i \in [2, k]}}
     \gamma^{i + (j-1)k - 1} \cdot \widetilde{y}_{(i,j)}(\alpha).
   $$
   Run the sum-check protocol $\langle P, V \rangle (Q, s, d_Q + 1, T)$,
   reducing the claim to a single evaluation $v \stackrel{?}{=} Q(\alpha',
   r')$ for $(\alpha', r') \in K^{\log d} \times K^{\log n}$.
3. $P \to V$: for all $i \in [k]$, $j \in [t]$, send
   $$
   y'_{(i,j)} \;:=\; Z_i M_j^\top \widehat{r'} \;\in\; K^d,
   $$
   where $\widehat{r'}$ is the appropriate boolean-encoded form of the
   sum-check final challenge. (These are HyperNova's
   $\sigma_{j,k}, \theta_{j,k}$.)
4. $V$: check
   $$
   v \;\stackrel{?}{=}\;
   \widetilde{\mathrm{eq}}\bigl((\alpha', r'), \beta\bigr) \cdot
   \Bigl( F + \sum_{i \in [k]} \gamma^i N_i \Bigr)
   + \gamma^k \sum_{\substack{j \in [t] \\ i \in [2, k]}}
     \gamma^{i + (j-1)k - 1} \cdot E_{(i,j)},
   $$
   where $F$, $N_i$, $E_{(i,j)}$ are deterministic functions of
   $\alpha', r', \alpha, r$, and the prover's $y'_{(i,j)}$.
5. Output the folded instance-witness pair (folded $C$, $u$, $x$,
   $r'$, $\{y'_{(i,j)}\}_{j \in [t]}$, $Z_i$) for each $i \in [k]$.

## Modified protocol $\Pi^{\mathrm{ZK}}_{\mathrm{CCS}}$

Below is the same protocol with our two modifications. Read it
top-to-bottom as a single coherent ZK variant. <span style="color:#1f6feb">**Blue
text**</span> denotes additions; <span style="color:#cf222e">**red text**</span>
denotes replacements of clear-text messages by hiding commitments and
their associated proofs.

**Inputs.** As before, plus public parameters of a hiding,
additively-homomorphic commitment scheme $\mathrm{Com}$ over the
prover's working ring (Ajtai over $R_q$ in Neo's setting; Pedersen
suffices for the elliptic-curve HyperNova analogue).

**Steps.**

<span style="color:#1f6feb">**0. Masking-polynomial commitment (NEW).**
$P$ samples a uniformly random multivariate polynomial $h(X_{[1, s]})$
of the same individual-degree profile as $Q$, i.e., $\deg_{X_i} h \le
\deg_{X_i} Q$ for every $i$. $P$ computes
$$
T_h \;:=\; \sum_{x \in \{0,1\}^s} h(x) \;\in\; K
$$
and the per-round univariate slices needed to run sum-check on $h$
later. $P$ sends a hiding commitment to $h$, namely
$$
C_h \;:=\; \mathrm{Com}(h; \rho_h)
$$
(in practice, a commitment to the dense coefficient vector of $h$, or
to its evaluations on a sufficient grid; under Ajtai both are
homomorphically usable). $P$ also sends a clear-text $T_h$. The
soundness justification for revealing $T_h$ in the clear: it is the
sum of a uniformly random polynomial over the cube and is itself a
single field element with no information about the witness, but it is
needed so the verifier can reconstruct the masked claimed sum.</span>

1. $V \to P$: sample and send challenges $\alpha \in K^{\log d}$,
   $\beta \in K^{\log(dn)}$, $\gamma \in K$.
   <span style="color:#1f6feb">In addition, sample and send a fresh
   masking challenge $\rho \in K$ (used to randomise the sum-check
   polynomial as $Q + \rho \cdot h$).</span>

2. $V \leftrightarrow P$: define $Q$ and $T$ as before, and let
   <span style="color:#1f6feb">$\widetilde{Q} := Q + \rho \cdot h$ and
   $\widetilde{T} := T + \rho \cdot T_h$.</span> Run the sum-check
   protocol $\langle P, V \rangle$ on
   $(\widetilde{Q}, s, d_Q + 1, \widetilde{T})$ instead of on
   $(Q, s, d_Q + 1, T)$, reducing to the claim
   $$
   \widetilde{v} \;\stackrel{?}{=}\;
   \widetilde{Q}(\alpha', r')
   \;=\; Q(\alpha', r') + \rho \cdot h(\alpha', r').
   $$
   <span style="color:#1f6feb">In each sum-check round, the prover's
   message is the round polynomial of $\widetilde{Q}$. Because $h$ has
   the same degree profile as $Q$ and was sampled uniformly *before*
   the verifier's challenges, the round polynomials of $\widetilde{Q}$
   are uniformly distributed conditioned on the verifier's view; they
   leak no information about the witness.</span>

3. $P \to V$: <span style="color:#cf222e">**Replaced.** Instead of
   sending $y'_{(i,j)} = \sigma_{j,k}, \theta_{j,k}$ in the clear,
   the prover sends hiding commitments
   $$
   C^{(\sigma)}_{j,k} \;:=\; \mathrm{Com}(\sigma_{j,k}; \rho^{(\sigma)}_{j,k}),
   \qquad
   C^{(\theta)}_{j,k} \;:=\; \mathrm{Com}(\theta_{j,k}; \rho^{(\theta)}_{j,k})
   $$
   for all relevant $(j, k)$. The prover also sends an opening of
   the masking polynomial at $(\alpha', r')$, namely either a hiding
   evaluation proof from a multilinear PCS for $h$ (e.g. Hachi with
   the standard Gaussian-mask trick), or, equivalently and more
   cheaply, a sigma protocol on $C_h$ proving that $h(\alpha', r') =
   v_h$ for a committed scalar $v_h$ with commitment
   $C_{v_h} = \mathrm{Com}(v_h; \rho_{v_h})$.</span>

4. $V$: <span style="color:#cf222e">**Modified.** The verifier no
   longer evaluates the binding equation in the clear. Instead, the
   prover and verifier run a small sigma protocol on the homomorphic
   commitments to prove that the claimed binding holds.</span>
   Concretely, the verifier already knows the field constants
   $$
   e \;:=\; \widetilde{\mathrm{eq}}\bigl((\alpha', r'), \beta\bigr),
   \qquad
   F = f(m_1, \ldots, m_t),
   \qquad
   N_i, \;\;E_{(i,j)} \text{ functions of } \alpha', r'.
   $$
   In the original protocol the verifier checks
   $$
   v \;=\;
   e \cdot \Bigl(F + \sum_i \gamma^i N_i\Bigr)
   + \gamma^k \sum_{i, j} \gamma^{i + (j-1)k - 1} E_{(i,j)},
   $$
   where $F, N_i, E_{(i,j)}$ are linear in the prover's $y'_{(i,j)}$
   (and constants, plus a single product $f(m_1, \ldots, m_t)$ that
   the user can split off into a separate commit-and-prove if desired).
   <span style="color:#1f6feb">In the ZK variant the prover instead
   proves, via a standard sigma protocol over the homomorphic
   commitment scheme:
   - $C^{(F)} := \sum_{(\text{linear in } C^{(\sigma)})} = \mathrm{Com}(F; \rho^{(F)})$ (homomorphic linear combination).
   - $C^{(E_{(i,j)})}$ analogously, using $C^{(\sigma)}, C^{(\theta)}$.
   - $C^{(N_i)}$ is the only nonlinear piece; if $b = 1$ (the standard
     Neo regime where decomposition is into bits), $\mathrm{NC}_i$
     reduces to a *quadratic* function of $\widetilde{Z}_i$, which is a
     single bilinear sigma protocol on the existing commitments.
   - The verifier finally checks that $\widetilde{v} - \rho \cdot v_h$
     equals the homomorphically-combined commitment, using the
     committed $v_h$ from step 3 above and a final equality sigma
     protocol.
   This sigma protocol has $O(k \cdot t)$ commitments and runs in
   constant rounds (3 rounds, Schnorr-style); soundness error is
   $\Theta(1/|K|)$, matching the rest of the protocol.</span>

5. Output the folded instance-witness pair as before, where
   <span style="color:#cf222e">the folded prover-side commitments
   $\sigma_{j,k}, \theta_{j,k}$ are folded *as commitments* (using the
   homomorphism of $\mathrm{Com}$ and the random-linear-combination
   challenge $\rho$ of the outer multi-folding scheme); the folded
   instance therefore contains commitments to $\sigma, \theta$ rather
   than the values themselves, preserving hiding through the recursion.</span>

## Why the modifications work

We argue informally that $\Pi^{\mathrm{ZK}}_{\mathrm{CCS}}$ is
zero-knowledge against an honest-verifier; the standard
Cramer-Damgard transformation upgrades to malicious verifiers.

### Round polynomials are uniform conditioned on the transcript

Because $h$ is sampled uniformly from a polynomial space with the same
degree profile as $Q$, and the prover commits to $h$ before the
verifier's first challenge $\rho$, the polynomial $\widetilde{Q} = Q +
\rho \cdot h$ has the property that, for any fixed $Q$, the
distribution of $\widetilde{Q}$ over the choice of $h$ is uniform on
the same polynomial space. Therefore the sum-check round polynomials
$\widetilde{s}_i$ derived from $\widetilde{Q}$ are uniform conditioned
on the verifier's challenges $r'_1, \ldots, r'_{i-1}$. This is the
classical Chiesa-Forbes-Spielman ZK sum-check argument
([CFS17](https://arxiv.org/abs/1704.02086), Section 5; see also
Thaler's *Proofs, Arguments, and Zero-Knowledge*, Section 13.2). The
only delta from the standard ZK sum-check is that we use a homomorphic
*lattice* commitment for $h$ rather than a Pedersen commitment.

### $\sigma, \theta$ are hidden behind hiding commitments

By the hiding property of $\mathrm{Com}$, the commitments
$C^{(\sigma)}_{j,k}, C^{(\theta)}_{j,k}$ leak nothing about the
underlying values. The sigma protocol in step 4 is itself
zero-knowledge for an honest-verifier (Schnorr is HVZK; the
linear-combination check needed here is a direct generalisation).

### Folded commitments preserve hiding through the recursion

The output of $\Pi^{\mathrm{ZK}}_{\mathrm{CCS}}$ is a folded
linearised CCS instance whose components include commitments to
$\sigma, \theta$, not the values. Subsequent invocations of
$\Pi_{\mathrm{RLC}}$ and $\Pi_{\mathrm{DEC}}$, and of the outer
multi-folding into a fresh CCS instance, see only commitments. The
homomorphism of $\mathrm{Com}$ lets the standard Neo folding step
proceed without unhiding anything, modulo a small bookkeeping change
to compute the folded commitment as a linear combination of inputs.

### Simulator construction

Sketch:

1. The simulator samples a uniformly random masking polynomial
   $h^*$ and a uniformly random scalar $T_h^*$, commits to $h^*$ as
   $C_h^*$, and sends $(C_h^*, T_h^*)$.
2. Receives $\alpha, \beta, \gamma, \rho$ from the verifier.
3. Plays the sum-check protocol *honestly* with respect to a randomly
   chosen polynomial $\widetilde{Q}^*$ of the right degree profile and
   a random claimed sum $\widetilde{T}^*$. Crucially, this requires no
   witness because the simulator chooses $\widetilde{Q}^*$ freely.
4. Sends fresh hiding commitments to random values
   $\sigma_{j,k}^*, \theta_{j,k}^*$ as $C^{(\sigma)*}, C^{(\theta)*}$.
5. Runs the sigma protocol of step 4 by simulating its transcript:
   pick the verifier's challenge first, sample the prover's response,
   solve for the prover's commitment. This is the standard HVZK
   simulator for Schnorr-style protocols, and works because the sigma
   protocol verifies a *homomorphic* equality among committed values
   that the simulator can satisfy by choice of randomness, without
   knowing the underlying scalars.

The hiding of $\mathrm{Com}$ ensures the simulated transcript is
indistinguishable from an honest one. No witness is used at any step.

### Soundness and the role of $T_h$

Why does revealing $T_h$ in the clear not hurt soundness? The honest
verifier's check (equivalent to the original) is
$$
\sum_{x \in \{0,1\}^s} \widetilde{Q}(x) \;=\; \widetilde{T}
\;\Longleftrightarrow\;
\sum_x Q(x) + \rho \cdot \sum_x h(x) \;=\; T + \rho \cdot T_h.
$$
The right-hand-side is what the verifier reconstructs from the
prover's $T_h$ message. Soundness reduces to: a cheating prover who
knows no satisfying witness, hence whose $\sum_x Q(x) \ne T$, can
satisfy the masked equation with probability at most $\Pr_\rho [\rho =
(T_h - \sum_x h(x))/(\sum_x Q(x) - T)] = 1/|K|$ over the choice of
$\rho$, which is the standard one-extra-round Schwartz-Zippel cost.
This is the same cost the masking polynomial trick incurs in any
ZK-sum-check construction.

## Why this avoids the Nova-style fix

The HyperNova / Vega / Jolt-BlindFold approach to ZK is to:

1. Run the multi-folding sum-check **non-blindingly**, accepting that
   the transcript reveals witness information.
2. Encode the multi-folding *verifier's checks* as a small R1CS
   instance (size $O(\log m)$).
3. Apply Nova's R1CS folding scheme to fold the small verifier R1CS
   with a randomly sampled relaxed R1CS instance, and use the blinding
   property of Nova folding (Lemma 5 of HyperNova) to hide the
   resulting witness.
4. Optionally apply non-ZK Spartan to the folded R1CS for succinctness.

This is conceptually elegant but operationally heavy: it requires a
*second* recursive folding scheme (Nova) wrapping the first
(HyperNova/Neo). In the lattice setting:

- Nova's R1CS folding requires a *hiding additively-homomorphic
  commitment* over a finite field, instantiated in Vega/Jolt with
  Pedersen commitments over an elliptic curve. Translating this to
  Ajtai commitments inherits all of LatticeFold's overheads (cyclotomic
  rings, large parameters), eating into Neo's design goal of working
  over small prime fields like Goldilocks.
- The R1CS encoding of the verifier's checks is non-trivial to keep
  small; Vega/BlindFold accepts a logarithmic-size R1CS, but in the
  lattice setting the verifier's per-round checks include
  multiplicative constraints over the commitment scheme that don't
  encode as cleanly.

The masking-polynomial + commit-to-$\sigma\theta$ approach proposed
here keeps the entire ZK overhead **inside the lattice
homomorphism**: one extra commitment ($C_h$), one extra sum-check
challenge ($\rho$), and a constant-round sigma protocol on the
existing commitments. No second proof system, no R1CS encoding of the
verifier, no extra cyclotomic ring.

## Cost summary

Let $s = \log(dn)$ be the number of sum-check rounds, $d_Q$ the
sum-check polynomial degree, $kt$ the count of $\sigma, \theta$ values.

| Step                         | Original Neo                 | $\Pi^{\mathrm{ZK}}_{\mathrm{CCS}}$ delta |
|------------------------------|------------------------------|------------------------------------------|
| Pre-sumcheck commit          | none                         | one commit to $h$ (size of $Q$ in coefficients) |
| Sumcheck rounds              | $s$ field-element univariates| same length, sample from $\widetilde{Q} = Q + \rho h$ |
| Sumcheck round polynomials   | clear                        | clear (uniform, no leakage)              |
| Final $\sigma, \theta$       | $kt$ scalars in clear        | $kt$ hiding commitments                   |
| Final binding check          | one field equation           | constant-round sigma protocol on $kt$ commitments |
| Folded output                | clear $\sigma, \theta$ folded| folded commitments                         |
| Soundness loss               | $O(s \cdot d_Q / |K|)$       | $+ O(1/|K|)$ for the masking step         |

## Cross-reference: Dall'Ava's parallel exploration

A complementary unpublished memo by Luca Dall'Ava (ICME Labs),
*Thoughts on lattice-based analogies for Nova's BlindFold*, attacks
the same problem and arrives at protocol sketches that overlap with
Approach A and Approach B above; it adds several technical
observations that sharpen the picture and identifies a number of
open questions, which we record and address here.

### The randomizing-vs-blinding distinction (sharpened)

The HyperNova paper defines two adjacent properties:

- **Randomizing** (Definition 13): the joint distribution of the
  *folded instance and witness* is identical to a freshly sampled one;
  HyperNova's Construction 1 satisfies this (Lemma 6/10).
- **Blinding** (Definition 14): in addition to the above, the
  *verifier's view* (its full output $\mathsf{st}_2$) is simulatable
  without the witness. Only Construction 3 (Nova R1CS folding) is
  proved blinding, in Lemma 5.

The gap is exactly the verifier's view of the sum-check transcript and
of $\sigma, \theta$: those messages are deterministic functions of the
witness even after the folded outputs are randomised. Dall'Ava makes
this concrete by computing the distribution of the folded witness
$\widetilde{w} = \sum_k \rho^k \mathcal{L}_k.w + \cdots$ and observing
that, taken alone, it *is* uniform (the random folding scalar $\rho$
acts as a one-time pad over the random sampled instance), so
randomising is genuinely the easy half. The hard half is precisely the
$\sigma, \theta$ leakage, which is what the masking polynomials and
the commit-and-prove subprotocol of the previous sections address.

This sharpens the framing of the entry: Approach A and Approach B
together turn HyperNova/Neo's *randomizing* multi-folding into a
*blinding* one, without delegating to a separate Nova R1CS folding.

### Two protocol sketches in Dall'Ava's memo

**(D-A) Per-$\sigma_{j,k}, \theta_{j,k}$ masking polynomial.**
Instead of one global masking polynomial $h$ for $Q$ as in our
Approach A, Dall'Ava proposes a finer-grained mask: the prover
samples $t \cdot \mu$ random polynomials $F_{j,k}(X)$ and
$t \cdot \nu$ polynomials $G_{j,k}(X)$, with the same individual
degrees as $\sigma_{j,k}(X) := \sum_y \widetilde{M}_j(X, y)\,
\widetilde{z}_{1,k}(y)$ and $\theta_{j,k}(X)$ respectively, and
commits to all of them. After the verifier sends a fresh challenge
$\xi$, the prover replaces every occurrence of $\widetilde{z}_{1,k}$
inside $L_{j,k}$ by a "blinded witness MLE" whose effect amounts to
$L_{j,k} \mapsto L_{j,k} + \xi^{j+k} F_{j,k}$, and similarly for
$Q_k$. The clear-text $\sigma_{j,k}$ now equals the original value
plus $\xi^{j+k} F_{j,k}(r'_x)$, which is uniform conditioned on the
verifier's view by the hiding of the commitment to $F_{j,k}$.

**(D-B) Mask the sum-check polynomial *and* commit to $\sigma$.**
A second, more conservative protocol drops the per-$\sigma$ masking
in favour of (i) a single global mask $p(X)$ on $Q$ (verifier
challenge $\zeta$), and (ii) replacing $\sigma_{j,k}$ by a hiding
commitment $C_{\sigma_{j,k}}$. The final binding equation
$$
c \;=\; \sum_{j,k} \gamma^{(k-1)t + j} \cdot e_1 \cdot \sigma_{j,k}
\;+\; \zeta \cdot p(r'_x)
$$
is then checked **as an equality of commitments** using the
homomorphism of the commitment scheme. The folded $v_j$ output by
the protocol is similarly a commitment that the verifier validates
via $\mathrm{Com}(v_j) = \sum_k \rho^k \cdot C_{\sigma_{j,k}}$.

Approach (D-B) matches our Approach B almost line-for-line; the
difference is in how the final binding check is structured (Dall'Ava
uses a commitment-equality check, we propose a sigma-protocol on the
commitments). The two are equivalent up to bookkeeping.

### Lattice-binding caveat: choose the right commitment scheme

The most useful technical observation in the memo: in the lattice
setting, **Ajtai commitments lose binding when the committed scalar
or vector has large $\ell_\infty$ norm**. The σ, θ values are inner
products of (potentially short) witnesses with the matrix MLE
evaluated at random challenge points, and the resulting scalar can be
arbitrarily large in $\mathbb{F}$ but its lift to $\mathbb{Z}$ is
*not* bounded. Committing to such scalars with Ajtai breaks binding.

Dall'Ava proposes the **BDLOP commitment scheme** ([Baum-Damgård-Lyubashevsky-Oechsner-Peikert, ePrint 2016/997](https://eprint.iacr.org/2016/997),
improved in Lyubashevsky-Khoshakhlagh-Maller-Pointcheval, ePrint
2022/284) as the right primitive: it is SIS-based, additively
homomorphic, and **binding even when the committed values are
unbounded**, at the cost of being non-succinct (commitment size
linear in the message length rather than constant). For our use
(committing to a constant-size scalar per $\sigma, \theta$), the
non-succinctness is irrelevant: there are only $O(kt)$ commitments,
each constant-size in the BDLOP setup parameters.

This addresses an open issue in our entry's "Lattice instantiation of
the homomorphic commitment" question: Neo's matrix commitment is
fine for the masking polynomial $h$ (which lives in a structured
small-norm space) but is *not* the right primitive for the
$\sigma, \theta$ commitments; switch to BDLOP for those.

### Norm-blowup tradeoffs

The memo also flags a more strategic concern: **adding masks
increases the norm**, and lattice-based folding schemes already burn
their norm budget on the decomposition step. Two competing demands:

- *Hiding $\sigma, \theta$* requires combining them with a random
  mask whose magnitude is large enough for soundness, which inflates
  the effective norm budget the commitment scheme has to absorb.
- *Restricting the random mask to small norm* hurts soundness,
  because the mask challenge then comes from a small subset of the
  ring (already a constraint in lattice schemes that avoid
  zero-divisor challenge sets).

Dall'Ava's tentative conclusion: HyperNova-with-masking is plausibly
blinding in spirit (sketch unfinished), but the *concrete* lattice
instantiation may not be practical without further work. Switching
to BDLOP for the σ, θ commitments breaks the norm-blowup constraint
on those specific scalars (BDLOP doesn't care about norm), so the
remaining issue is the $h$ commitment, and there the natural fix is
to keep $h$ in the "short-coefficient" world that the existing
matrix commitment supports.

### Status of Dall'Ava's open questions, addressed

The memo lists several explicit open questions, with provisional
answers below.

1. **"Is masked HyperNova actually blinding?"** Dall'Ava sketches a
   simulator but the construction is incomplete (one step is
   labelled "I think this works"). Our analysis in the "Why the
   modifications work" section above gives a more complete sketch
   for the masking-plus-commit-to-$\sigma\theta$ variant; the
   simulator's key step is to interpret the σ, θ commitments as
   trapdoor commitments and use HVZK simulation of the final sigma
   protocol. The masking-only variant (D-A) does not obviously
   simulate without the witness, because $\sigma_{j,k} +
   \xi^{j+k} F_{j,k}(r'_x)$ is uniform but the verifier eventually
   *evaluates* this expression in the binding check, so the
   simulator must produce a fake $F_{j,k}$ whose evaluation matches.
   This is fine if $F_{j,k}$ is committed via a hiding commitment
   the simulator can equivocate (trapdoor commitments solve this);
   without equivocation the simulator is stuck. **Conclusion:** the
   masking-only variant needs a trapdoor commitment to be
   simulatable; the masking-plus-commit variant works with any
   hiding additively-homomorphic commitment.

2. **"Can we use BDLOP without the norm-binding issue?"** Yes for
   the $\sigma, \theta$ commitments; BDLOP is binding regardless of
   message norm. The masking polynomial $h$ does not need BDLOP if
   its coefficients are kept short by construction (sample $h$ from
   the same coefficient distribution as legitimate witnesses).

3. **"Is masking sufficient or do we also need to handle
   linearization checks?"** $\Pi_{\mathrm{RLC}}$ and
   $\Pi_{\mathrm{DEC}}$ each have their own sum-check or
   commitment messages. The same recipe (mask, then commit to
   summary statistics, then sigma-prove the binding) applies to
   both, but the details have not been worked out in either
   exploration. This stays on the open list.

4. **"Norm-bound checks reveal witness via $g(X) = \prod_i (X - i)$
   structure."** This concerns how lattice folding schemes prove
   that decomposed witness chunks have entries in $[-b, b]$: the
   product polynomial $g$ vanishes on this set, and the prover
   reduces "$\widetilde{w}(x) \in [-b, b]$ for $x$ in the cube" to
   sum-check on $\prod_i (\widetilde{w}(x) - i) = 0$. The sum-check
   transcript leaks the witness through the round polynomials. Fix
   the same way: mask this norm-bound sum-check with its own
   masking polynomial. The cost is one extra commit and one extra
   sum-check round per norm-bound check; the soundness analysis is
   the same.

5. **"Could LaBRADOR-style random projections give partial
   blinding?"** The memo explores this and concludes (correctly, in
   our reading) that random projections discard information needed
   for extraction, so they are useful for *succinctness* of norm
   checks but not for *blinding* of the witness. The masking-plus-
   commit recipe stays the right tool for blinding.

6. **"Speculative ideas (homotopy folding, non-archimedean norms,
   modular forms, isogenies of abelian varieties)."** These are
   genuinely speculative directions in the memo, marked as
   experimental. We do not address them here; they would each
   warrant their own separate research note.

### What the memo adds vs what this entry adds

Roughly:

- The memo's main contribution is the **randomizing-vs-blinding
  framing** and the BDLOP suggestion; both are technical
  refinements that should be folded back into the main thrust of
  this entry. It also enumerates lattice-specific obstructions
  (norm blow-up, zero-divisor challenge sets) more explicitly than
  we do.
- This entry's main contributions on top of the memo are: a clean
  modular structure separating "mask the sum-check" from "commit to
  $\sigma, \theta$"; explicit comparison to NovaBlindFold and Vega
  on the cost axis; and a more complete simulator sketch for the
  commit-and-prove variant, including the sigma-protocol step that
  the memo's simulator stops short of.

A merged write-up would: open with the randomising/blinding
distinction; describe Approach B (mask + commit + sigma-protocol)
as the main protocol; instantiate the $\sigma, \theta$ commitments
with BDLOP; treat Approach A (per-$\sigma$ masking) as a variant
that buys smaller commitments at the cost of needing trapdoor
commitments; and end with the lattice-specific norm-blowup
discussion as a section on instantiation.

## Open questions

1. **Tightness of the masking polynomial size.** We chose $h$ to have
   the *full* degree profile of $Q$, which is conservative but
   wasteful; in many sum-check ZK constructions, only a smaller $h$
   (degree-1 in some variables, full-degree in others) is needed,
   exploiting which round polynomial coefficients actually leak
   witness information. The exact minimal $h$ for Neo's $Q$ is worth
   working out.
2. **Compatibility with $\Pi_{\mathrm{RLC}}$ and $\Pi_{\mathrm{DEC}}$.**
   The sketch above only modifies $\Pi_{\mathrm{CCS}}$. The other two
   reductions in Neo ($\Pi_{\mathrm{RLC}}$ for random-linear-combination
   folding, $\Pi_{\mathrm{DEC}}$ for decomposition) also have their own
   sum-check or commitment messages. The same masking + commit-and-prove
   recipe should apply but the details need to be checked.
3. **Lattice instantiation of the homomorphic commitment.** We assumed
   $\mathrm{Com}$ is hiding additively-homomorphic over the working
   ring. Ajtai's commitment is binding under SIS but is *not* hiding by
   default; one needs to add a Gaussian noise term to get hiding (see
   the standard HMC trick). Neo's matrix commitment scheme already
   supports a hiding variant (Section 3.3); whether this directly
   supports the linear/sigma-protocol checks of step 4 is the main
   technical thing to verify.
4. **Soundness against malicious verifiers.** The honest-verifier
   analysis above is the easy half; we need to either argue Fiat-Shamir
   or apply Cramer-Damgard. Both should go through standardly.
5. **Comparison with NovaBlindFold++ in the lattice setting.** It is
   conceivable that a "small lattice-R1CS folding scheme" could do the
   blinding more cheaply than the masking polynomial. A direct
   asymptotic and concrete comparison would be valuable.
6. **Fitting into Neo's IVC layer.** Neo composes its folding scheme
   into an NIVC chain. The proposed ZK variant must propagate the
   "hidden $\sigma, \theta$" property across folds; this is exactly
   what the homomorphic folded-commitments machinery in step 5 buys,
   but the IVC-level argument deserves a careful write-up.

## References

- HyperNova (Kothapalli, Setty), ePrint
  [2023/573](https://eprint.iacr.org/2023/573). Construction 1
  (multi-folding for CCS); Section 7 (NovaBlindFold); Lemma 5 (Nova
  folding is blinding).
- Neo (Nguyen, Setty), ePrint
  [2025/294](https://eprint.iacr.org/2025/294). Section 4.4
  ($\Pi_{\mathrm{CCS}}$); Section 3 (matrix commitment scheme).
- Vega (Kaviani, Setty), ePrint
  [2025/2094](https://eprint.iacr.org/2025/2094). Introduces the
  BlindFold protocol used in Jolt.
- Chiesa, Forbes, Spielman, *A Zero Knowledge Sumcheck and its
  Applications* ([arXiv 1704.02086](https://arxiv.org/abs/1704.02086)),
  Section 5: the masking-polynomial technique.
- Thaler, *Proofs, Arguments, and Zero-Knowledge*, Section 13.2.
- LatticeFold (Boneh, Chen), ePrint
  [2024/257](https://eprint.iacr.org/2024/257). Comparison baseline
  for "what doing this with a second lattice folding scheme would cost."
- [Hachi PCS](../papers/hachi.md): the natural lattice multilinear
  PCS to instantiate the hiding commitment to $h$ if you want a
  succinct opening of $h(\alpha', r')$.
