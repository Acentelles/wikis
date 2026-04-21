# Efficient Lattice-Based Blind Signatures via Gaussian One-Time Signatures

> Vadim Lyubashevsky, Ngoc Khanh Nguyen, Maxime Plancon
> *Efficient Lattice-Based Blind Signatures via Gaussian One-Time Signatures*
> ePrint [2022/006](https://eprint.iacr.org/2022/006), PKC 2022

The PDF and full markdown live in
`raw/lattices/2022-006-efficient-lattice-blind-signatures-gaussian-one-time/`.

Referenced by the [blind signatures from ABBA](../ideas/blind-signatures.md)
idea as the template for a ComSIS-based blind signature.

## TL;DR

The first practical round-optimal (2-round) lattice-based blind signature
scheme. Signatures are ~150 KB, the signing protocol is stateful (the
signer keeps a counter), and the running time of verification is linear
in the maximum number of signatures $N$ that can be issued per public
key (practical for $N \lesssim 2^{20}$). Security reduces to SIS and
LWE in the random oracle model.

The core idea: replace the interactive Schnorr-style blind signing
(which is broken in the lattice setting per [BLL+21]) with a
one-time-signature-based approach where the signer pre-commits to $N$
one-time key pairs and uses a trapdoor to sign each one.

## The construction

### Setup

The signer generates:
- A random matrix $A \in R_q^{k \times \ell}$ with a trapdoor $T$
  (via the [GPV08] framework, refined by [MP12])
- For each $i \in \{1, \ldots, N\}$: a one-time key pair
  $(s_i, y_i)$ with short coefficients, satisfying
  $A s_i = v_i$ and $A y_i = w_i$, where
  $(v_i, w_i) = H(i)$ are derived from a random oracle

The public key is $(A, H)$; the secret is the trapdoor $T$.

#### What is a trapdoor $T$ for $A$?

A trapdoor for a matrix $A$ is a short basis of the lattice
$\Lambda^\perp(A) = \{x : Ax = 0 \bmod q\}$ that lets the holder
efficiently solve the **preimage sampling problem**: given any target
$v$, find a short $s$ with $As = v \bmod q$. Without the trapdoor,
finding such a short $s$ is the SIS problem (hard).

The [MP12] construction generates $A$ with embedded structure:

$$
A = [\bar{A} \mid -\bar{A}R + G]
$$

where $\bar{A}$ is uniform, $R$ is a short random matrix (the
trapdoor), and $G$ is a **gadget matrix** with known simple structure
(e.g. powers of 2 along the diagonal). The resulting $A$ is
computationally indistinguishable from uniform, but the signer knows
$R$.

To find a short $s$ with $As = v$:

1. Solve $Gz = v + \bar{A}w$ for short $z$ (easy, because $G$ has
   trivial structure; just binary decomposition of the target)
2. Combine using $R$ to get $s$ from $z$ and $w$
3. Add discrete Gaussian noise so that the output distribution is
   independent of $R$ (this is the smoothing step)

Anyone without $R$ sees a uniform-looking $A$ and cannot find short
preimages. The Gaussian noise ensures the output $s$ does not leak $R$.

#### How the private keys are generated

The public keys $(v_i, w_i)$ are uniformly random (derived from
$H(i)$) and do not need to be stored. The private keys $(s_i, y_i)$
are generated using the **GPV trapdoor** for $A$:

1. Compute $(v_i, w_i) = H(i)$ (deterministic, from the random oracle)
2. Use the trapdoor $T$ to sample a short $s_i$ with $A s_i = v_i$,
   and a short $y_i$ with $A y_i = w_i$

The trapdoor sampling produces vectors whose coefficients follow a
**discrete Gaussian distribution** with standard deviation above the
smoothing parameter $\eta_\varepsilon(\Lambda)$. This is why the
paper is called "Gaussian One-Time Signatures": the secret keys are
Gaussian-distributed, not uniform.

#### The GPV paradigm and the security proof

The trapdoor serves two roles:

- **Functionality**: it lets the signer efficiently find short
  preimages of arbitrary targets $v_i, w_i$ (without the trapdoor,
  this is as hard as SIS).
- **Simulatability**: in the security proof, the simulator does the
  *reverse*. It receives a random $A$ from the SIS challenger (no
  trapdoor), samples short $(s_i, y_i)$ directly, and programs the
  random oracle as $H(i) = (A s_i, A y_i)$. The two distributions
  are indistinguishable because:
  - The trapdoored $A$ is computationally indistinguishable from
    uniform [MP12].
  - The Gaussian $(s_i, y_i)$ above the smoothing parameter produce
    statistically uniform images $A s_i, A y_i$.

This is the standard GPV paradigm: the trapdoor lets the signer find
short preimages of arbitrary targets, while in the proof the simulator
avoids the trapdoor by choosing the targets to match pre-selected
short vectors.

The paper's main technical contribution is proving the one-time
signature secure under Gaussian keys without Renyi entropy arguments
(which would be the standard tool but introduces complications with
the blind signature reduction).

### Signing protocol (2 rounds)

**Round 1 (User → Signer):** The user picks message $\mu$ (a polynomial
with $\{-1, 0, 1\}$ coefficients), encrypts it via an LWE public-key
scheme: $c = \text{enc}(\mu)$, and sends $c$ together with a
zero-knowledge proof that $c$ is well-formed (small coefficients in the
randomness and message).

**Round 2 (Signer → User):** The signer, at counter value $i$,
homomorphically evaluates the function $z = s_i \mu + y_i$ on the
ciphertext. Concretely, for each coordinate $j$ of $s_i$ and $y_i$, the
signer computes masked ciphertexts:

$$
t_j = B(r s_j + y_j^{(1)}) + p(e s_j + y_j^{(2)})
$$
$$
t'_j = b^T(r s_j + y_j^{(1)}) + p(e_1 s_j + y_j^{(3)}) + p(\mu s_j + y_j)
$$

where $y_j^{(1)}, y_j^{(2)}, y_j^{(3)}$ are fresh masking polynomials.
The signer sends the masked ciphertexts and increments the counter.

**User output:** The user decrypts to obtain $z = s_i \mu + y_i$ and
outputs a zero-knowledge proof of knowledge of a short $z$ satisfying

$$
A z = v_i \mu + w_i
$$

for some $(v_i, w_i) \in \{H(1), \ldots, H(N)\}$. The proof uses the
compact set-membership argument from [LNS21b], which is logarithmic in
$N$.

### The signature

A signature on $\mu$ is a NIZK proof that the signer knows a short $z$
and an index $i$ such that $Az = v_i \mu + w_i$. The proof hides both
$z$ and $i$, giving blindness. The set-membership proof over
$\{H(1), \ldots, H(N)\}$ is what makes verification linear in $N$.

## Security

**Blindness:** The masked ciphertexts (equations 4-5 in the paper)
hide $\mu$ because the LWE encryption is semantically secure and the
masking polynomials $y_j^{(1)}, y_j^{(2)}, y_j^{(3)}$ prevent the
signer from learning anything beyond the fact that a valid signing
request was made.

**One-more unforgeability:** Reduces to the hardness of the one-time
signature scheme, which in turn reduces to SIS. The key insight is that
each $(s_i, y_i)$ pair is used exactly once, and producing a second
valid $z'$ for the same $(v_i, w_i)$ requires solving a short-vector
problem. The Gaussian distribution of the secret keys (sampled via the
GPV trapdoor) requires a dedicated security proof that avoids Renyi
entropy arguments; this is the paper's main technical contribution.

**ROS connection:** The reason earlier Schnorr-style lattice blind
signatures failed is the ROS (Random inhomogeneities in a Overdetermined
Solvable system) problem. [BLL+21] showed that the full Schnorr blind
signature is broken, and [HKLN20] showed that fixing the proof for the
lattice variants makes them impractical. LNP22 sidesteps ROS entirely
by using one-time signatures instead of interactive sigma protocols.

## Parameters and costs

| Metric | Value |
|---|---|
| Rounds | 2 (optimal) |
| Signature size | ~150 KB |
| Communication | ~16 MB (user ↔ signer) |
| Public key | ~1 MB (dominated by $A$) |
| Max signatures per key | $N \lesssim 2^{20}$ |
| Verification time | $O(N)$ (linear in max signatures) |
| Ring | $R_q = \mathbb{Z}_q[X]/(X^d + 1)$, power-of-two cyclotomic |
| Security assumption | SIS + LWE |

The $O(N)$ verification is the main practical limitation: for
$N = 2^{20}$, verifying involves checking membership in a set of
$\sim 10^6$ public keys. The compact proof from [LNS21b] keeps the
signature logarithmic in $N$, but the verifier must still evaluate the
set-membership check.

## The one-time signature (Section 3)

The one-time signature at the core of the scheme is a variant of
[LM18] (Lyubashevsky-Micciancio). Given a key pair $(s, y)$ with
$As = v$, $Ay = w$, the signature on $\mu$ is $z = s\mu + y$.
Verification checks $Az = v\mu + w$ and $\|z\| \leq \beta$.

The paper's contribution is proving this secure when $(s, y)$ are
sampled from a discrete Gaussian (via the GPV trapdoor), not uniform.
The standard uniform analysis from [LM18] does not apply because the
Gaussian tails introduce correlations that a Renyi entropy argument
would need to handle. Instead, the paper gives a direct reduction to
SIS that avoids Renyi entirely.

**This is the component that [ABBA's open question](../ideas/blind-signatures.md)
asks to replace with a ComSIS-based variant.** The question is whether
$z = s\mu + y$ can be replaced by a commutator-based construction
$z = [s, \mu] + [y, r]$ (or similar) that satisfies the same one-time
unforgeability under ComSIS instead of SIS, while the Gaussian sampling
is replaced by sampling from the quaternion order $\Lambda$.

The key obstacle: the GPV paradigm requires a **trapdoor for the
commitment function** that lets the signer sample short preimages of
arbitrary targets. For SIS, this is the [MP12] construction
$A = [\bar{A} \mid -\bar{A}R + G]$ described above. For ComSIS, one
would need a trapdoor for the commutator function
$F_a(x) = \sum [a_i, x_i]$: given a key $a$ and an arbitrary
traceless target $t \in T_0$, efficiently sample a short quaternion
vector $x \in \Lambda^m$ with $F_a(x) = t$.

The commutator is linear in $x$ for fixed $a$ (bilinear in $(a, x)$),
so the map $x \mapsto F_a(x)$ is a linear map from $\Lambda^m$ to
$T_0$, structurally similar to $x \mapsto Ax$. A gadget-matrix
approach might therefore work: decompose the key $a$ as
$a = [\bar{a} \mid -\bar{a}R + g]$ for some "commutator gadget" $g$
with trivially invertible commutator map. The complication is that the
quaternion multiplication is non-commutative, so the gadget structure
must respect the $u \cdot \ell = \theta(\ell) \cdot u$ relation. This
is unexplored territory.

## Relation to other lattice blind signature work

| Paper | Approach | Rounds | Status |
|---|---|---|---|
| [Ru10] Ruckert (Asiacrypt 2010) | Schnorr-style, lattice | 3 | proof broken [HKLN20] |
| [ABB20] (FC 2020) | improved Ruckert | 3 | proof broken [HKLN20] |
| [HKLN20] (Crypto 2020) | fixed proof | 3 | works, but $N < 12$ |
| [ASY21] | FHE-based homomorphic signing | 2 | theoretical (needs FHE + ZK) |
| **[LNP22] (this paper)** | **OTS + trapdoor** | **2** | **practical, $N \lesssim 2^{20}$** |

## References in raw/

| Paper | ePrint | Raw |
|---|---|---|
| LNP22 (this paper) | [2022/006](https://eprint.iacr.org/2022/006) | `raw/lattices/2022-006-efficient-lattice-blind-signatures-gaussian-one-time/` |
| FHKS19 (modular treatment) | [2019/260](https://eprint.iacr.org/2019/260) | `raw/lattices/2019-260-modular-treatment-blind-signatures-identification/` |
| ROS insecurity | [2020/945](https://eprint.iacr.org/2020/945) | `raw/lattices/2020-945-insecurity-of-ros/` |
| LM18 (OTS from SIS) | [2008/328](https://eprint.iacr.org/2008/328) | in ABBA bib as `OGefficientlatticesigs` |
