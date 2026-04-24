# One-time signatures from ComSIS

> Research sketch. A Lamport/LM18-style one-time signature scheme using
> ABBA's sum-of-commutators function, with security under the ComSIS
> assumption. This is the building block needed for the
> [ComSIS-based blind signature](./blind-signatures.md) (the LNP22
> template).
>
> References:
> - ABBA: ePrint [2026/148](https://eprint.iacr.org/2026/148), Section 8
>   (commented-out text, now restored)
> - LM18: Lyubashevsky, Micciancio, *Asymptotically Efficient
>   Lattice-Based Digital Signatures*, J. Cryptology 2018
> - LNP22: ePrint [2022/006](https://eprint.iacr.org/2022/006)

## Background: what is a one-time signature?

See [glossary: one-time signature](../glossary/one-time-signature.md).
In brief: a signature scheme secure for exactly one signing query per
key. Two signatures on distinct messages leak the secret key. OTS
schemes are building blocks for blind signatures (LNP22 uses $N$
independent OTS keys) and Merkle-tree signatures (XMSS, SPHINCS+).

The standard lattice OTS is LM18, where signing is
$\mathbf{z} = \mathbf{s}\mu + \mathbf{y}$ and verification checks
$A\mathbf{z} = \mathbf{v}\mu + \mathbf{w}$ using the SIS-based Ajtai
commitment $A$. The question is whether the commutator map $F_{\mathbf{a}}$
can replace $A$.

---

## The scheme

Working over the quaternion order $\Lambda_q$ with component field
$K = \mathbb{Q}(\zeta_n)$ and real subfield $K^+$ as in
[ABBA](../papers/abba.md). The message $\mu$ is a ring element from
$\mathcal{O}_{K_q}$ (the commutative centre of $\Lambda_q$), so that
$\mu$ commutes with all quaternion elements.

- **Setup** $(1^\lambda)$. Sample a public ComSIS vector
  $\mathbf{c} \in \Lambda_q^m$.
- **KeyGen** $(1^\lambda)$. Sample
  $sk = (\mathbf{a}, \mathbf{b}) \leftarrow D_{\Lambda^m, \sigma}^2$
  (short Gaussians over the quaternion order). Publish:
  $$pk = (pk_1, pk_2) := \bigl(F_{\mathbf{a}}(\mathbf{c}),\; F_{\mathbf{b}}(\mathbf{c})\bigr) \in T_0^2$$
- **Sign** $(sk, \mu)$. Output $\rho := \mathbf{a}\mu + \mathbf{b} \in \Lambda_q^m$.
- **Verify** $(pk, \rho, \mu)$. Check:
  1. $\|\rho\|_\infty \leq B$ (norm bound, essential for ComSIS hardness).
  2. $F_{\rho}(\mathbf{c}) = pk_1 \cdot \mu + pk_2$.

---

## Correctness

Using centrality of $\mu$ (it commutes with each $c_i$):

$$F_{\rho}(\mathbf{c}) = F_{\mathbf{a}\mu + \mathbf{b}}(\mathbf{c}) = F_{\mathbf{a}\mu}(\mathbf{c}) + F_{\mathbf{b}}(\mathbf{c}) = F_{\mathbf{a}}(\mathbf{c})\,\mu + F_{\mathbf{b}}(\mathbf{c}) = pk_1\,\mu + pk_2$$

The second equality uses bilinearity of the commutator in the first
argument. The third uses $[a_i \mu, c_i] = [a_i, c_i]\mu$, which holds
because $\mu \in \mathcal{O}_{K_q}$ commutes with $c_i$.

---

## Security

### Strong forgery ($\mu' = \mu$, different $\rho'$)

If $\rho' \neq \rho$ satisfies $F_{\rho'}(\mathbf{c}) = pk_1\,\mu + pk_2$,
then:

$$0 = F_{\rho}(\mathbf{c}) - F_{\rho'}(\mathbf{c}) = F_{\rho - \rho'}(\mathbf{c}) = -F_{\mathbf{c}}(\rho' - \rho) = F_{-\mathbf{c}}(\rho' - \rho)$$

by bilinearity and antisymmetry ($[a, b] = -[b, a]$). Since
$\|\rho\|_\infty, \|\rho'\|_\infty \leq B$, the vector $\rho' - \rho$ is
short ($\|\rho' - \rho\|_\infty \leq 2B$) and nonzero, giving a ComSIS
solution for $-\mathbf{c}$.

### Existential forgery ($\mu' \neq \mu$)

This requires the Lyubashevsky-Micciancio entropy argument (LNP22, page
5; LM18, Section 3). The reduction:

1. Receives $A$ (here $\mathbf{c}$) from the ComSIS challenger.
2. Samples its own $(\mathbf{a}, \mathbf{b})$ to answer the signing query.
3. If the forger outputs $(\mu', \rho')$ with $\mu' \neq \mu$, subtracts:
   $A(\mathbf{z} - \mathbf{z}') = A(\mathbf{s}(\mu - \mu'))$ (in
   LM18 notation). Either $\mathbf{z} - \mathbf{z}' \neq \mathbf{s}(\mu - \mu')$
   (ComSIS solution), or the reduction has extracted $\mathbf{s}$ (since
   $\mu - \mu'$ is invertible in $\mathcal{O}_{K_q}$).

The crucial step (LNP22, page 5): the adversary *cannot
information-theoretically determine* $\mathbf{a}$ from $pk$, because
the kernel of $F_{\mathbf{c}}$ is nontrivial (any
$\tilde{\mathbf{a}} = \mathbf{a} + \mathbf{u}$ with
$F_{\mathbf{c}}(\mathbf{u}) = 0$ is an equally valid secret key).
This forces $\mathbf{z} - \mathbf{z}' \neq \mathbf{s}(\mu - \mu')$
with probability at least $\delta$, so the reduction solves ComSIS with
probability $\epsilon\delta$.

This argument requires:
- ABBA's Corollary 2 (the distribution
  $(\mathbf{c}, F_{\mathbf{c}}(\mathbf{a}))$ is near-uniform on
  $\Lambda_q^m \times T_0$ when $\mathbf{a}$ is Gaussian) to play the
  role of [MP12]'s trapdoor indistinguishability.
- The smoothing parameter $\sigma \geq \eta_\varepsilon(\Lambda)$ to
  ensure the kernel coset argument holds statistically.

### One-time-ness

Two signatures $\rho_1 = \mathbf{a}\mu_1 + \mathbf{b}$ and
$\rho_2 = \mathbf{a}\mu_2 + \mathbf{b}$ on distinct messages give:

$$\rho_1 - \rho_2 = \mathbf{a}(\mu_1 - \mu_2)$$

If $\mu_1 - \mu_2$ is invertible in $\mathcal{O}_{K_q}$ (the generic
case when the message space is a strong set with invertible differences),
the attacker recovers $\mathbf{a} = (\rho_1 - \rho_2)(\mu_1 - \mu_2)^{-1}$
and then $\mathbf{b} = \rho_1 - \mathbf{a}\mu_1$, forging on any
message. The scheme is *strictly* one-time.

---

## Technical requirements

| Requirement | Why | ABBA reference |
|---|---|---|
| $\mu \in \mathcal{O}_{K_q}$ (central) | $[a_i\mu, c_i] = [a_i, c_i]\mu$ needs $\mu c_i = c_i \mu$ | ABBA homomorphic property, Eq. (1) |
| $\mathbf{a}, \mathbf{b} \leftarrow D_{\Lambda^m, \sigma}$ | Short keys for ComSIS norm budget | ABBA hiding, Theorem 7 |
| $\sigma \geq \eta_\varepsilon(\Lambda)$ | Coset-hiding: $pk$ is near-uniform | ABBA Corollary 2 |
| $\|\rho\|_\infty \leq B$ in Verify | ComSIS is hard only for bounded-norm solutions | Standard in LM18/LNP22 |
| $\mu_1 - \mu_2$ invertible | Ensures one-time-ness (key recovery from two signatures) | Neo strong-set (Definition 10) |

---

## Comparison with LM18 (SIS-based OTS)

| Component | LM18 (SIS) | ComSIS-OTS (this scheme) |
|---|---|---|
| Public matrix | $A \in R_q^{n \times m}$ | $\mathbf{c} \in \Lambda_q^m$ |
| Commitment map | $\mathbf{v} = A\mathbf{s}$ | $pk_1 = F_{\mathbf{a}}(\mathbf{c})$ |
| Output space | $R_q^n$ | $T_0$ (traceless subspace) |
| Output dimension | $n \cdot D_L$ | $\frac{3}{2}D_K$ (25% smaller for even $n$) |
| Signing | $\mathbf{z} = \mathbf{s}\mu + \mathbf{y}$ | $\rho = \mathbf{a}\mu + \mathbf{b}$ |
| Verification | $A\mathbf{z} = \mathbf{v}\mu + \mathbf{w}$ | $F_{\rho}(\mathbf{c}) = pk_1\mu + pk_2$ |
| Hardness assumption | SIS | ComSIS |
| Homomorphism | Full $R_q$ | $\mathcal{O}_{K^+}$ only |
| Message space | $R_q$ (or a short subset) | $\mathcal{O}_{K_q}$ (must be central) |

The key structural difference: the message $\mu$ must be **central** in
$\Lambda_q$ (i.e., live in $\mathcal{O}_{K_q}$, the commutative part).
For LM18, $\mu$ can be any short ring element. This restricts the
message space but is not a problem in the blind signature application,
where $\mu$ is derived from a hash and can be constrained to any
desired subset.

---

## What does ComSIS-OTS buy over LM18?

### The two benefits

**1. Smaller public keys per OTS instance (25%).** Each of $pk_1, pk_2$
lives in $T_0$ (dimension $\frac{3}{2}D_K$) instead of $R_q^n$
(dimension $n \cdot D_L$). For $\Phi_{128}$: 96 vs 128 field elements
per key pair, a 25% reduction.

However, in the LNP22 blind signature, per-instance public keys are
derived from $H(i)$ and never stored, so this saving is marginal for
the blind signature's actual public key size (dominated by the shared
matrix $\mathbf{c}$ or $A$).

**2. Smaller NIZK transcripts (25% on commitment material).** This is
the benefit that matters. The dominant cost in the LNP22 blind
signature is the **final NIZK transcript**, which proves knowledge of a
short $\rho$ and an index $i$ satisfying
$F_{\rho}(\mathbf{c}) = pk_1\mu + pk_2$. The NIZK's commitment layer
commits to elements whose target space is $T_0$. Since $T_0$ is 25%
smaller than $R_q$, the commitment material in the transcript shrinks
by 25%. LNP22's signature is ~150 KB, and the commitment portion is the
dominant term; a 25% reduction on that portion could bring the
transcript to ~115-125 KB (the exact number depends on what fraction of
the 150 KB is commitment vs sumcheck/evaluation data).

### The three costs

**1. Prover speed (1.4$\times$ to $\gg$2$\times$ slower, depending on
the operation).** The 1.4$\times$ overhead from the ABBA benchmarks
applies only to **binary commit** ($b = 2$, the sparse $\{0, u\}$
embedding path). The OTS involves two distinct computational regimes:

- **Binary commit** (NIZK layer): when the NIZK decomposes the witness
  $\rho$ via $\Pi_{\text{DEC}}$ into base-$b$ binary layers before
  committing, those layers use the sparse ABBA commit at ~1.4$\times$
  overhead.
- **General commutator evaluation** (verification equation): computing
  $F_\rho(\mathbf{c}) = \sum [\rho_i, c_i]$ for a Gaussian-distributed
  $\rho$ requires **full quaternion commutators**, not the sparse path.
  Each $[\rho_i, c_i] = \rho_i c_i - c_i \rho_i$ costs 2 quaternion
  multiplications, each expanding to ~4 ring multiplications in
  $\mathcal{O}_K$. This gives ~$8m$ ring multiplications in
  $\mathcal{O}_K$ vs Ajtai's $m$ ring multiplications in
  $\mathcal{O}_L$. For even $n$, $D_K = D_L/2$, so each
  $\mathcal{O}_K$ multiplication is ~$4\times$ cheaper (schoolbook), net
  ~$2\times$ slower. With NTT (if available), the gap may differ.

  The sumcheck arithmetization of the quadratic relation
  $F_\rho(\mathbf{c}) = pk_1\mu + pk_2$ also involves full commutator
  evaluations at intermediate points, so this cost appears in the NIZK's
  sumcheck rounds as well, not just in direct verification.

**2. Weaker security assumption.** ComSIS instances correspond to SIS
on $3n^+ \times 4n^+ m$ matrices (vs $4n^+ \times 4n^+ m$ for
Module-SIS). At the same ring dimension, ComSIS gives fewer bits of
security. To compensate, one needs more rows $\kappa$ in the commitment
matrix, partially offsetting the 25% size saving.

**3. Restricted message space.** Messages $\mu$ must be **central** in
$\Lambda_q$ (i.e., live in $\mathcal{O}_{K_q}$, the commutative part
of the quaternion order) for the correctness derivation
$[a_i\mu, c_i] = [a_i, c_i]\mu$ to hold. In LM18, $\mu$ can be any
short ring element. This is not a problem for the blind signature
application (messages are hash outputs and can be constrained to any
desired subset), but it limits generality.

### The signature itself is not smaller

The signature $\rho = \mathbf{a}\mu + \mathbf{b} \in \Lambda_q^m$ has
the same dimension as the LM18 signature $\mathbf{z} \in R_q^m$: for
even $n$, $\dim(\Lambda) = 2D_K = D_L$, so $m \cdot D_L$ field
elements either way.

### Summary table

| Aspect | LM18 (SIS) | ComSIS-OTS | Winner |
|---|---|---|---|
| Public key / OTS instance | $2 \cdot D_L$ | $\frac{3}{2} D_L$ | ComSIS (25% smaller) |
| Signature size | $m \cdot D_L$ | $m \cdot D_L$ | tie |
| NIZK commitment output | over $R_q$ | over $T_0$ | ComSIS (25% smaller) |
| Blind sig transcript | ~150 KB | ~115-125 KB (est.) | ComSIS |
| Prover speed (binary commit) | baseline | ~1.4$\times$ slower | LM18 |
| Prover speed (general elements) | baseline | ~2$\times$ slower | LM18 |
| Security / dimension | Module-SIS | ComSIS (weaker) | LM18 |
| Message flexibility | any short $\mu \in R_q$ | $\mu \in \mathcal{O}_{K_q}$ (central) | LM18 |

### Net assessment

The ComSIS-OTS is worth pursuing when **transcript size is the primary
bottleneck** (e.g., on-chain verification of blind signatures, where
every kilobyte costs gas). The ~25 KB saving on a 150 KB transcript is
meaningful in that setting. If prover speed or security margin is the
bottleneck, LM18 is the safer choice.

---

## Application: building block for cBlind

The ComSIS-OTS plugs into the LNP22 blind signature template as a
drop-in replacement for the SIS-based OTS. The signer generates $N$
independent ComSIS-OTS key pairs; each blind signing session consumes
one key. The user encrypts $\mu$ under Ring-LWE, the signer
homomorphically computes an encryption of $\rho = \mathbf{a}_i\mu + \mathbf{b}_i$,
and the user decrypts and proves knowledge via a NIZK.

See [blind signatures from ABBA](./blind-signatures.md) for the full
construction and open questions.

---

## Open questions

1. **Formal security proof.** The sketch above follows LM18/LNP22 but
   needs a self-contained proof that ports the Gaussian coset-hiding
   argument (LNP22 Lemma 3.2) to the quaternion setting, using ABBA's
   Corollary 2 in place of [MP12].

2. **Concrete parameters.** Derive $m$, $\sigma$, $B$, and the message
   space $\mathcal{C} \subset \mathcal{O}_{K_q}$ for $\Phi_{128}$
   at 128-bit security. The norm budget
   $\|\rho\|_\infty \leq C \cdot \|\mathbf{a}\|_\infty \cdot \|\mu\|_\infty + \|\mathbf{b}\|_\infty$
   determines $B$, which in turn sets the ComSIS parameters.

3. **Signature size.** The signature is $\rho \in \Lambda_q^m$ (same
   dimension as LM18). The public key lives in $T_0^2$ (25% smaller
   than $R_q^2$ for even $n$). Whether this propagates to the blind
   signature's total transcript size depends on the NIZK layer.

4. **Non-central messages.** Can the centrality requirement on $\mu$
   be relaxed? If $\mu$ is only in $\mathcal{O}_K$ (not $\mathcal{O}_{K^+}$),
   the correctness step $[a_i\mu, c_i] = [a_i, c_i]\mu$ may still hold
   if $\mu$ commutes with elements of $\Lambda_q$ for other reasons
   (e.g., $\mu$ is scalar). Exploring the exact commutativity requirement
   could widen the message space.
