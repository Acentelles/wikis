# Project idea: blind signatures from ABBA commutator commitments

> The open problem left in
> [ABBA, ePrint 2026/148](https://eprint.iacr.org/2026/148) is whether the
> Lyubashevsky, Nguyen, Plancon blind signature
> ([LNP22](https://eprint.iacr.org/2022/006)) can be re-based on ComSIS
> using a commutator one-time signature in place of its SIS-based one. This
> note gathers the literature, extracts the structural ideas common to
> lattice-based blind signatures, and sketches a concrete route to a
> ComSIS-based blind signature that plugs ABBA into the LNP22 template.

## Why blind signatures, and why post-quantum

A blind signature scheme, originally introduced by Chaum [Cha82], is a
two-party protocol where a user obtains the signer's signature on a
message $\mu$ without the signer learning $\mu$ or being able to recognise
the resulting signature when it surfaces later. The standard security
notions are:

- **Blindness**, the signer, even if malicious, cannot link a finished
  signature back to the interaction that produced it;
- **One-more unforgeability**, after $k$ concluded interactions with the
  signer, the user cannot produce $k + 1$ valid signature/message pairs.

Blind signatures are the canonical privacy primitive for e-cash, private
payment tokens (Privacy Pass, VOPRF-style authentication), anonymous
credentials, and e-voting. All major deployments today rely on RSA or
discrete-log / pairing assumptions and therefore need a post-quantum
replacement.

## Literature snapshot

I keep the PDFs and markdown conversions of the papers below in
`raw/lattices/` and `raw/isogenies/`.

### Classical, foundational

| Year | Paper | Ref |
|------|-------|-----|
| 1982 | Chaum, *Blind Signatures for Untraceable Payments* (Crypto '82) | Cha82, origin of the primitive |
| 1996 | Abe, Fujisaki, *How to Date Blind Signatures* | AF96, introduces the partially-blind variant |
| 2000 | Pointcheval, Stern, *Security Arguments for Digital Signatures and Blind Signatures* | PS00, Schnorr-blind proof under ROS |
| 2021 | Benhamouda, Lepoint, Loss, Orru, Raykova, *On the (in)Security of ROS* ([ePrint 2020/945](https://eprint.iacr.org/2020/945)) | BLL+21, breaks full Schnorr blind |

### Lattice-based

| Year | Paper | ePrint | Raw |
|------|-------|--------|-----|
| 2020 | Hauck, Kiltz, Loss, Nguyen, *Lattice-Based Blind Signatures, Revisited* | [2020/769](https://eprint.iacr.org/2020/769) | `raw/lattices/2020-769-lattice-blind-signatures-revisited/` |
| 2022 | Lyubashevsky, Nguyen, Plancon, *Efficient Lattice-Based Blind Signatures via Gaussian One-Time Signatures* (PKC '22) | [2022/006](https://eprint.iacr.org/2022/006) | `raw/lattices/2022-006-efficient-lattice-blind-signatures-gaussian-one-time/` |
| 2022 | Agrawal, Kirshanova, Stehle, Yadav, *Practical, Round-Optimal Lattice-Based Blind Signatures* (CCS '22) | [2021/1565](https://eprint.iacr.org/2021/1565) | `raw/lattices/2021-1565-practical-round-optimal-lattice-blind-signatures/` |
| 2023 | Beullens, Lyubashevsky, Nguyen, Seiler, *Lattice-Based Blind Signatures: Short, Efficient, and Round-Optimal* (CCS '23) | [2023/077](https://eprint.iacr.org/2023/077) | `raw/lattices/2023-077-lattice-blind-signatures-short-round-optimal/` |

### Isogeny- and group-action-based

| Year | Paper | ePrint | Raw |
|------|-------|--------|-----|
| 2023 | Katsumata, Lai, LeGrow, Qin, *CSI-Otter: Isogeny-Based (Partially) Blind Signatures from the Class Group Action with a Twist* (Crypto '23) | [2023/1239](https://eprint.iacr.org/2023/1239) | `raw/isogenies/2023-1239-csi-otter-isogeny-blind-signatures/` |
| 2025 | *Tanuki: New Frameworks for (Concurrently Secure) Blind Signatures from Post-Quantum Group Actions* | [2025/1100](https://eprint.iacr.org/2025/1100) | `raw/isogenies/2025-1100-tanuki-blind-signatures-group-actions/` |

## The main ideas, distilled

Three design templates recur in the literature. Understanding which one
fits your algebra is the hardest part of any blind-signature project.

### 1. Schnorr blind signature (the "module" template)

The canonical construction [CP93, PS00, HKL19, HKLN20]: take a
$\Sigma$-protocol for the discrete-log (or Ring-SIS) relation and
*randomise* the transcript so that the signer's view is independent of
the message. Concretely, the user blinds the commitment by a random shift,
applies the challenge, and unblinds the response. Two abstractions capture
this:

- **Linear identification schemes** (HKL19), which require the underlying
  relation to live in a *module*. Pedersen commitments and Ring-SIS
  satisfy this; isogeny group actions do not.
- **ROS-style unforgeability reduction**, which converts a one-more
  forgery into solving the ROS problem. As BLL+21 showed, ROS is polynomial
  for $\ell \geq \log q$ concurrent sessions, which kills all plain Schnorr
  blind signatures at high concurrency.

This is why the Ruckert (2010) and the follow-up [ABB20] schemes had
buggy proofs, and why [HKLN20] has to cap the number of signatures per key
to O(polylog).

### 2. Encrypt-then-sign-homomorphically (the "LNP22" template)

Introduced by [LNP22] and the basis for the ABBA open question. The idea
is:

1. The signer's public key is a *collection* of $N$ one-time signature
   public keys $\{(\mathbf{v}_i, \mathbf{w}_i)\}_{i=1}^N$ of a
   Lyubashevsky-Micciancio [LM18] style OTS: $\mathbf{A s}_i = \mathbf{v}_i$,
   $\mathbf{A y}_i = \mathbf{w}_i$, with $(\mathbf{s}_i, \mathbf{y}_i)$
   small and Gaussian.
2. The user encrypts $\mu$ under a (Ring-LWE) encryption scheme and sends
   the ciphertext plus a NIZK that it is well-formed.
3. The signer homomorphically computes an encryption of the OTS signature
   $\mathbf{z}_i = \mathbf{s}_i \mu + \mathbf{y}_i$, using the fact that
   the needed function is linear over small inputs.
4. The user decrypts $\mathbf{z}_i$, and publishes a NIZK of knowledge of
   a small $\mathbf{z}$ and an index $i$ such that
   $\mathbf{A z} = \mathbf{v}_i \mu + \mathbf{w}_i$.

Blindness comes from the encryption; one-more unforgeability reduces to
SIS (forging on a fresh index requires finding a second short preimage of
the same OTS equation, which is SIS). The dominant cost is the final NIZK
hiding both $\mathbf{z}$ and $i$, since it must hide *which* of $N$ public
keys was used.

Running time is linear in $N$, so practical $N$ caps at roughly $2^{20}$.
This is the trade the scheme makes to avoid ROS entirely.

### 3. NIZK-of-a-signature (the "Fischlin" template)

[AKSY22], [dK22], [BLNS23]: the signer simply issues a standard lattice
signature (Dilithium-style) on an encryption of $\mu$, and the user
publishes a NIZK proving knowledge of a valid signature of a decryption
that matches. [BLNS23] cuts size to 22 KB by using vanishing-SIS and a
tailor-made NIZK; [AKSY22] leans on trapdoor sampling. This template
avoids the SIS-to-OTS reduction entirely but needs a very expressive NIZK.

### Isogeny detour

[CSI-Otter] sidesteps the module obstruction by exploiting the quadratic
*twist* of supersingular curves: the twist gives isogenies "slightly more
than a group action but still less than a module", enough to run a
Schnorr-like protocol with OR-proofs. Signature size is 8 KB (128 B public
key), but concurrency is again polylogarithmic. [Tanuki] generalises and
achieves polynomial concurrency under a strengthened group-action
assumption.

The takeaway: the underlying algebra dictates which template you can
instantiate. Modules give you (1) with a ROS cap; group actions can
sometimes give you (1) with a twist; anything else needs (2) or (3).

## Where ABBA fits

[ABBA, §8] already points at this:

> It follows from the above that one can create a Lamport-style one-time
> signature scheme using the sums-of-commutators function, with security
> based on ComSIS, analogously to how [23, 24] build a one-time signature
> scheme from SIS. This raises a further open question, to study whether
> the blind signature scheme of [LNP22] can be adapted to hold with
> security based on ComSIS, using such a one-time signature scheme.

The reason the LNP22 template is the natural target for ABBA, rather than
(1) or (3), is structural:

- $F_{\mathbf{a}}(\mu) = \sum_i [a_i, \mu_i]$ is **linearly homomorphic**
  when one argument is fixed. This is exactly the property the Schnorr
  template destroys (commutators of two *random* inputs don't behave like
  a module, which is why plain Schnorr-blind over commutators is a
  non-starter) and exactly the property the LNP22 template uses.
- ABBA already establishes that the map
  $F_{\mathbf{a}}: \Lambda_q^m \to T_0$ compresses by a factor of $3/4$
  relative to the natural order, which is precisely the size win the
  blind signature needs (the 22 KB BLNS23 transcript is dominated by
  commitment material).
- ComSIS is quantitatively a weaker assumption than SIS at the same
  dimension (category-III vs category-V in the paper's ML-DSA-sized
  estimate), but the *qualitative* property that the LNP22 one-time
  signature proof needs is short-preimage hardness of a linear map on
  Gaussians, which ComSIS provides.

## A concrete proposal: ComSIS-OTS plus LNP22 plumbing

### Commutator one-time signature (cOTS)

Working in the natural quaternion order
$\Lambda = \mathcal{O} \oplus i\mathcal{O} \oplus j\mathcal{O} \oplus k\mathcal{O}$
with $\Lambda_q \cong \prod_{i=1}^n M_2(\mathbb{F}_q)$ as in ABBA:

- **Key generation.** Sample $\mathbf{a} \leftarrow \mathcal{U}(\Lambda_q^m)$
  uniformly. Sample $\mathbf{y}, \mathbf{s} \leftarrow D_{\Lambda^m, \sigma}$.
  Publish $(\mathbf{a}, \mathbf{v}, \mathbf{w})$ where
  $\mathbf{v} = F_{\mathbf{a}}(\mathbf{s}) \in T_0$ and
  $\mathbf{w} = F_{\mathbf{a}}(\mathbf{y}) \in T_0$. Secret key is
  $(\mathbf{y}, \mathbf{s})$.
- **Sign.** On a short challenge $\mu \in C \subset \mathcal{O}_q$ (an
  ABBA-style "ball of short elements"), output
  $\mathbf{z} = \mathbf{y} + \mu \cdot \mathbf{s}$. Here $\mu \cdot \mathbf{s}$
  is scalar multiplication by $\mathcal{O}_q$, under which commutators are
  linear: $[a_i, \mu s_i] = \mu [a_i, s_i]$, which is exactly the
  homomorphism ABBA uses for its commitment's linear combinations.
- **Verify.** Check that $\mathbf{z}$ is short and
  $F_{\mathbf{a}}(\mathbf{z}) = \mathbf{w} + \mu \cdot \mathbf{v}$.

Correctness follows from bilinearity of $[\cdot, \cdot]$ in the second
argument and the ABBA homomorphism
$F_{\mathbf{a}}(\mathbf{y} + \mu \mathbf{s}) = F_{\mathbf{a}}(\mathbf{y}) + \mu F_{\mathbf{a}}(\mathbf{s})$
whenever $\mu, \mathbf{s}, \mathbf{y}$ remain in the "short" regime where
ABBA is binding.

Unforgeability under a single query reduces to ComSIS exactly as in
LM18/LNP22: a forgery
$(\mu', \mathbf{z}') \neq (\mu, \mathbf{z})$ with
$F_{\mathbf{a}}(\mathbf{z}') = \mathbf{w} + \mu' \mathbf{v}$ yields
$F_{\mathbf{a}}(\mathbf{z} - \mathbf{z}' - (\mu - \mu') \mathbf{s}) = 0$
with a short argument, a ComSIS solution once the Gaussian-preimage
coset-hiding argument of LNP22 §3 is ported to the quaternion setting
(the statistical step uses ABBA's Corollary 2, that
$(\mathbf{a}, \mathrm{im} F_{\mathbf{a}})$ is near-uniform on
$\Lambda_q^m \times T_0$, in place of the Micciancio-style argument).

### Minimalist ComSIS-OTS (LM18-style restatement)

A cleaner, "closest-to-LM18" restatement of the commutator one-time
signature, keeping only the essentials; this is the form the
co-authored draft notes use. The message $\mu$ is a ring element
drawn from $\mathcal{O}_{K_q}$ (the commutative centre of the
quaternion order $\Lambda_q$, so that $\mu$ commutes with everything
in $\Lambda_q$).

- **Setup** $(1^\lambda)$. Sample a public ComSIS vector
  $\mathbf{c} \in \Lambda_q^m$.
- **KeyGen** $(1^\lambda)$. Sample the secret key
  $sk = (sk_1, sk_2) := (\mathbf{a}, \mathbf{b}) \in \Lambda_q^{2m}$
  (from the short-element distribution used in ABBA). Publish
  $$
  pk \;=\; (pk_1, pk_2) \;:=\;
  \bigl(F_{\mathbf{a}}(\mathbf{c}),\; F_{\mathbf{b}}(\mathbf{c})\bigr).
  $$
- **Sign** $(sk, \mu)$. Output $\rho := \mathbf{a}\mu + \mathbf{b}$.
- **Verify** $(pk, \rho, \mu)$. Check
  $F_{\rho}(\mathbf{c}) = pk_1 \, \mu + pk_2$.

Correctness. Using that $\mu$ is central in $\Lambda_q$ (it commutes
with every $c_i$),
$$
F_{\rho}(\mathbf{c})
\;=\; F_{\mathbf{a}\mu + \mathbf{b}}(\mathbf{c})
\;=\; F_{\mathbf{a}\mu}(\mathbf{c}) + F_{\mathbf{b}}(\mathbf{c})
\;=\; F_{\mathbf{a}}(\mathbf{c})\,\mu + F_{\mathbf{b}}(\mathbf{c})
\;=\; pk_1\,\mu + pk_2.
$$

Security sketch. Given a forgery $\rho' \ne \rho$ on the *same*
message $\mu$ (i.e., a strong forgery),
$$
0 \;=\; F_{\rho}(\mathbf{c}) - F_{\rho'}(\mathbf{c})
\;=\; F_{\rho - \rho'}(\mathbf{c})
\;=\; -F_{\mathbf{c}}(\rho' - \rho)
\;=\; F_{-\mathbf{c}}(\rho' - \rho)
$$
by bilinearity and antisymmetry of the commutator form, so
$\rho' - \rho$ is a nonzero short element satisfying
$F_{-\mathbf{c}}(\rho' - \rho) = 0$, i.e., a ComSIS solution for the
public vector $-\mathbf{c}$ (equivalently $\mathbf{c}$, by sign
symmetry of ComSIS).

### Errors and gaps in the minimalist restatement

Auditing the scheme above, I see the following issues worth
addressing before using it in a security proof:

1. **Verify omits a norm check on $\rho$.** ComSIS is hard only for
   solutions of *bounded* norm. Without a norm bound in Verify,
   an adversary could return a valid-but-huge $\rho'$ whose
   difference $\rho' - \rho$ is large, and the reduction produces a
   vector that is not a valid ComSIS instance. Verify should
   additionally check $\|\rho\|_\infty \le B$ for a prescribed
   bound $B$ compatible with the ComSIS hardness assumption and
   the honest signing distribution. This is routine in LM18 and
   LNP22 and needs to be stated explicitly here.

2. **The reduction only covers strong forgery ($\mu' = \mu$).** The
   derivation $0 = F_{\rho}(\mathbf{c}) - F_{\rho'}(\mathbf{c})$
   implicitly uses
   $F_{\rho}(\mathbf{c}) - F_{\rho'}(\mathbf{c})
   = pk_1 \mu + pk_2 - (pk_1 \mu' + pk_2) = pk_1(\mu - \mu')$,
   which is zero iff $\mu = \mu'$ (assuming $pk_1$ is not annihilated
   by $\mu - \mu'$). For an **existential** forgery
   $(\mu', \rho')$ with $\mu' \ne \mu$, the reduction needs the
   standard Lyubashevsky-Micciancio entropy argument: the simulator
   samples its own secret $\mathbf{a}$, computes the signing answer,
   and uses that the forger's response is independent of
   $\mathbf{a}$ conditioned on the public key (since multiple
   $\mathbf{a}$s give the same $F_{\mathbf{a}}(\mathbf{c})$), so
   $\rho - \rho' - \mathbf{a}(\mu - \mu')$ is a nontrivial short
   element with noticeable probability, yielding a ComSIS solution.
   The present sketch skips this step.

3. **Centrality of $\mu$ is used without being stated.** The step
   $F_{\mathbf{a}\mu}(\mathbf{c}) = F_{\mathbf{a}}(\mathbf{c}) \mu$
   requires $\mu c_i = c_i \mu$ for all $i$, i.e., $\mu$ commutes
   with the entries of $\mathbf{c}$. In ABBA this holds because
   messages live in $\mathcal{O}_{K_q}$, the commutative centre of
   $\Lambda_q$, but the write-up should say so explicitly; without
   centrality, the correctness derivation fails at
   $a_i \mu c_i \stackrel{?}{=} a_i c_i \mu$.

4. **One-time-ness is not argued.** Two signatures on distinct
   messages $\mu_1 \ne \mu_2$ give
   $\rho_1 - \rho_2 = \mathbf{a}(\mu_1 - \mu_2)$; if $\mu_1 - \mu_2$
   is invertible in $\mathcal{O}_{K_q}$ (the generic case for a
   well-chosen message space), the attacker recovers $\mathbf{a}$
   directly, and then $\mathbf{b} = \rho_1 - \mathbf{a}\mu_1$.
   So the scheme is strictly one-time. The $C$-set of allowable
   messages must therefore be structured so that differences are
   invertible (ABBA-style "ball of short elements with invertible
   differences"), and a single-signature assumption must be part of
   the security statement.

5. **Sampling distributions for $\mathbf{a}, \mathbf{b}$ are not
   specified.** For $\rho$ to be short (and therefore a candidate
   ComSIS preimage if differenced with a forgery), both keys must
   be drawn from a short distribution, typically a discrete
   Gaussian $D_{\Lambda^m, \sigma}$ with appropriate parameters.
   The statistical argument that $pk$ is near-uniform on
   $\mathrm{im}(F)$ (ABBA Corollary 2) relies on this choice of
   distribution.

6. **Norm budget and the signature norm bound $B$.** With
   $\mathbf{a}, \mathbf{b}$ short and $\mu$ from a short central
   subset, the signature norm satisfies
   $\|\rho\|_\infty \le C \cdot \|\mathbf{a}\|_\infty \cdot
   \|\mu\|_\infty + \|\mathbf{b}\|_\infty$
   for a modest expansion constant $C$ depending on the quaternion
   basis. $B$ should be set at (or slightly above) this bound so
   honest signatures pass, and so ComSIS for vectors of norm
   $\le 2B$ governs forgery security.

Modulo fixing these six points, the reduction goes through and the
scheme is the ComSIS-based analogue of LM18, with $F$ (sum-of-
commutators) taking the role of $\mathbf{A} \cdot (-)$ and
$F_{\mathbf{a}}(\mathbf{c})$ taking the role of $\mathbf{A}\mathbf{a}$.

### cBlind: blind signature

The LNP22 plumbing carries over with one change: the "target" space of
the homomorphic evaluation is $T_0$, the traceless-quaternion submodule,
not $R_q$. Concretely:

1. **Setup.** The signer generates $N$ independent cOTS keys
   $\{(\mathbf{a}, \mathbf{v}_i, \mathbf{w}_i)\}_{i=1}^N$. The matrix
   $\mathbf{a}$ is shared across the OTS instances (as in LNP22) and the
   $(\mathbf{v}_i, \mathbf{w}_i)$ are derived from $H(i)$, so the public
   key size is independent of $N$.
2. **User step.** User encrypts $\mu$ using the LNP22 Ring-LWE encryption
   (unchanged) and sends a NIZK that the ciphertext is well-formed.
3. **Signer step.** On session $i$, the signer homomorphically computes
   an encryption of $\mathbf{z}_i = \mathbf{y}_i + \mu \mathbf{s}_i$.
   This is where the algebra changes: the homomorphic evaluation runs
   inside $\Lambda_q$, with addition and $\mathcal{O}_q$-scaling lifted
   to the ciphertext. The encryption scheme of LNP22 is defined over
   $R = \mathbb{Z}[\zeta_{2n}]$ and since
   $\Lambda \cong \mathcal{O}_{K^+}^4$ with $K^+ = \mathbb{Q}(\zeta_n + \zeta_n^{-1})$
   (the lifted cyclotomic used by ABBA in its Neo instantiation), the
   underlying ring matches up to a tensor. This is the piece that needs
   care; the Gaussian-preimage analysis of LNP22 §3 works
   coordinate-wise over $\mathcal{O}_{K^+}$, and ABBA already does its
   uniformity analysis on $\Lambda$ coordinate-wise via
   $\Lambda_q \cong \prod_n M_2(\mathbb{F}_q)$.
4. **User decrypts and proves.** User recovers $\mathbf{z}_i$, discards
   the session index, and outputs a NIZK of knowledge of a short
   $\mathbf{z}$ and an index $i$ such that
   $F_{\mathbf{a}}(\mathbf{z}) = \mathbf{w}_i + \mu \mathbf{v}_i$
   (the ABBA-analog of LNP22 equation (1)).

### What this buys, back-of-envelope

Assuming the LNP22 costs port linearly:

- The cOTS signature $\mathbf{z}$ lives in $\Lambda^m$ (4n-dim per ring
  element), the same size as an SIS-OTS signature at matched $n$. No win
  here.
- The OTS *public key* $(\mathbf{v}_i, \mathbf{w}_i)$ lives in $T_0^2$,
  compressing from $R_q^2$ to $T_0^2$, i.e., by the ABBA factor $3/4$.
  LNP22 already hashes these from $i$ so they do not inflate the public
  key, but the same factor propagates into the final NIZK's commitments,
  which dominate signature size.
- The LNP22 signature hides $\mathbf{z}$ via an LNS21b-style NIZK whose
  commitment layer is Ajtai. Replacing that with ABBA commitments gives,
  per Table 1 of ABBA when applied to Neo, a $3/4$ compression on the
  commitment part of the signature. The transcript would drop from
  ~150 KB to somewhere between 115 and 125 KB if the NIZK-commitment
  portion is the dominant term. (This needs a careful accounting; the
  LNP22 signature has many other components.)

The security trade is explicitly the same one ABBA makes in its Neo case
study: fixed $N = 4n$, a drop from NIST-V to NIST-III. For a blind
signature this is probably acceptable.

## Open questions and risks

1. **Unforgeability under concurrent signing.** LNP22's proof handles a
   single signing session, and the paper's analysis of Gaussian
   coset-hiding is the step most sensitive to the algebra. Porting it
   requires proving a version of LNP22 Lemma 3.2 over $\Lambda$, using
   the traceless-target quotient. ABBA's Corollary 2 is the statistical
   ingredient, but the one-more reduction needs it *conditioned* on the
   OTS secrets, which is a slightly different claim.
2. **Encryption algebra.** LNP22's encryption is Ring-LWE over
   $\mathbb{Z}[\zeta_{2n}]$. We need an encryption whose plaintexts can
   be scaled by $\mathcal{O}_q$ and whose ciphertexts support the
   non-commutative lift needed by the signer. One option: encrypt each
   $M_2(\mathbb{F}_q)$ coordinate of $\Lambda_q$ independently under
   LNP22-style Ring-LWE. This increases the ciphertext size by a factor
   of 4, which partially eats the $3/4$ commitment win; the question is
   whether the NIZK of well-formedness then gets cheaper (the target
   space is smaller, so fewer norm-bound constraints).
3. **NIZK for the final proof.** LNP22 uses LNS21b, which proves
   knowledge of a short preimage of an affine relation over $R_q$. We
   need a variant proving knowledge of a short preimage of
   $F_{\mathbf{a}}(\cdot) - \mathbf{w}_i - \mu \mathbf{v}_i = 0$ with
   a bilinear-in-$[\cdot, \cdot]$ structure. The ABBA paper's Neo
   instantiation (section 8) already sketches how to do this kind of
   ComSIS-flavoured NIZK; adapting it to the cOTS relation should be
   mechanical if the Neo folding analysis transfers.
4. **Concurrency.** Neither LNP22 nor the cBlind sketch here aim beyond
   polylog concurrency. If the target application needs polynomial
   concurrency, we would have to layer in the Tanuki-style technique
   from [2025/1100], or switch template entirely to (3).

## Plan of attack

1. Write out the cOTS scheme formally, prove single-query unforgeability
   under ComSIS. This is the isolated piece ABBA flags as analogous to
   LM18 and should be the first paper out.
2. Port LNP22's Gaussian coset-hiding lemma (LNP22 §3, Lemma 3.2) to
   $\Lambda$. This is the real mathematical content.
3. Instantiate the encryption and the homomorphic evaluation, and
   compare transcript sizes against LNP22 and BLNS23.
4. A follow-up paper on concurrency (Tanuki-style or via trapdoor
   sampling) if (2) caps out at polylog.

## References

- [ABBA]  Centelles, Mendelsohn, *ABBA: Lattice-based Commitments from Commutators*, ePrint 2026/148.
- [LNP22] Lyubashevsky, Nguyen, Plancon, *Efficient Lattice-Based Blind Signatures via Gaussian One-Time Signatures*, PKC 2022. ePrint 2022/006.
- [BLNS23] Beullens, Lyubashevsky, Nguyen, Seiler, *Lattice-Based Blind Signatures: Short, Efficient, and Round-Optimal*, CCS 2023. ePrint 2023/077.
- [HKLN20] Hauck, Kiltz, Loss, Nguyen, *Lattice-Based Blind Signatures, Revisited*, Crypto 2020. ePrint 2020/769.
- [AKSY22] Agrawal, Kirshanova, Stehle, Yadav, *Practical, Round-Optimal Lattice-Based Blind Signatures*, CCS 2022. ePrint 2021/1565.
- [CSI-Otter] Katsumata, Lai, LeGrow, Qin, *CSI-Otter*, Crypto 2023. ePrint 2023/1239.
- [Tanuki] *Tanuki: New Frameworks for (Concurrently Secure) Blind Signatures from Post-Quantum Group Actions*, ePrint 2025/1100.
- [LM18] Lyubashevsky, Micciancio, *Asymptotically Efficient Lattice-Based Digital Signatures*, J. Cryptology 2018.
- [Cha82] Chaum, *Blind Signatures for Untraceable Payments*, Crypto 1982.
- [BLL+21] Benhamouda, Lepoint, Loss, Orru, Raykova, *On the (in)Security of ROS*, Eurocrypt 2021. ePrint 2020/945.
