# ABBA Commitments in Hachi's Multilinear PCS

> Research sketch. Goal: evaluate whether Hachi's inner Ajtai commitment
> layer can be replaced by ABBA's commutator-based commitment, and
> whether the 25% inner compression propagates to the full ~55 KB proof.
>
> This is an ARIA Track 3.3 Phase 1 milestone: "Evaluate ABBA vs Ajtai
> in Hachi."
>
> References:
> - Hachi: ePrint [2026/156](https://eprint.iacr.org/2026/156),
>   Nguyen, O'Rourke, Zhang
> - ABBA: ePrint [2026/148](https://eprint.iacr.org/2026/148),
>   Centelles, Mendelsohn
> - See also: [Hachi wiki entry](../papers/hachi.md),
>   [ABBA wiki entry](../papers/abba.md)

## TL;DR

- Hachi is a two-level Ajtai commitment: inner commitments
  $t_i = A \cdot s_i \in R_q^{n_A}$, outer commitment
  $u = B \cdot (\hat{t}_1, \ldots, \hat{t}_{2^r})^T \in R_q^{n_B}$.
  Binding under Module-SIS (Hachi Lemma 7).
- ABBA replaces the inner layer:
  $t_i = F_{\mathbf{a}}(s_i) = \sum_j [a_j, s_{i,j}] \in T_0^{n_A}$,
  changing output dimension from $n_A \cdot D_L$ to
  $n_A \cdot \frac{3}{2}D_K$ (25% smaller for even $n$). The outer
  layer can remain standard Ajtai.
- **Two critical obstacles**: (a) ABBA is homomorphic over
  $\mathcal{O}_{K^+}$ only, but Hachi's evaluation protocol uses
  challenges from $\mathbb{F}_{q^k}$; (b) Hachi's gadget decomposition
  produces integer coefficients in $\{0, \ldots, b-1\}$, which all lie
  in $\mathcal{O}_{K^+}$, so commutators vanish (same "naive embedding
  fails" problem as Neo).
- **What is straightforward**: ComSIS binding (structurally parallel to
  Module-SIS), dimension cascade to outer commitment (fewer bits to
  decompose), and ring-switching/sumcheck components (orthogonal to
  commitment choice).
- The restricted homomorphism is the harder obstacle. Three approaches
  are sketched below.

---

## Hachi's protocol at the commitment level

This section summarises the four components of Hachi that the ABBA
substitution touches. All figure/lemma references are to ePrint 2026/156.

### Inner commitment (Hachi Section 4.1, Eq. 13--14)

The prover commits to $2^r$ vectors $(f_i)_i$ of length $2^m$:

1. Gadget-decompose each $f_i$ in base $b$:
   $s_i := G_{2^m}^{-1}(f_i)$, producing a vector in
   $R_q^{\delta \cdot 2^m}$ with coefficients in $\{0, \ldots, b-1\}$,
   where $\delta = \lceil \log_b q \rceil$.
2. Compute the inner Ajtai commitments:
   $t_i := A \cdot s_i \in R_q^{n_A}$ for $i \in [2^r]$, where
   $A \in R_q^{n_A \times \delta \cdot 2^m}$ is a uniformly random
   public matrix.

**Binding** (Hachi Lemma 7): two distinct weak openings $(s_i)_i$ and
$(s'_i)_i$ for the same $u$ yield a vector $z$ with $[A \; B] z = 0$
and $0 < \|z\|_\infty \leq \max(2\bar\omega\bar\beta, 2\bar\gamma)$,
breaking Module-SIS.

### Outer commitment (Hachi Section 4.1, Eq. 14)

Binary-decompose each inner commitment:
$\hat{t}_i := G_{n_A}^{-1}(t_i)$. Then:

$$
u := B \cdot (\hat{t}_1, \ldots, \hat{t}_{2^r})^T \in R_q^{n_B}
$$

The outer commitment sees the *output representation* of the inner
commitment. Its binding (Hachi Lemma 8) also reduces to Module-SIS.

### Folding with challenges (Hachi Section 4.2, Eq. 18--19)

The verifier sends a challenge vector
$c = (c_1, \ldots, c_{2^r}) \leftarrow \mathcal{C}^{2^r}$ where
$\mathcal{C} \subseteq \{c \in R_q : \|c\| \leq \omega\}$. The prover
computes a folded witness:

$$
z := \sum_{i=1}^{2^r} c_i \cdot s_i \in R_q^{\delta \cdot 2^m}
$$

The key identity for soundness is the **homomorphic folding**:

$$
A \cdot z = \sum_i c_i \cdot (A \cdot s_i) = \sum_i c_i \cdot G_{n_A} \hat{t}_i = (c^T \otimes G_{n_A}) \hat{t}
$$

This relies on Ajtai's full $R_q$-homomorphism:
$A \cdot (\sum c_i s_i) = \sum c_i \cdot (A \cdot s_i)$ for arbitrary
$c_i \in R_q$.

### Norm proof via sumcheck (Hachi Figures 5--6, Lemmas 10--11)

After folding, the prover must demonstrate that the committed values
satisfy norm bounds. Hachi runs a sumcheck protocol over the extension
field $\mathbb{F}_{q^k}$ (via ring-switching, Section 3) to prove
$H_0(\tau_0) = 0$ (range/normality) and $H_\alpha(\tau_1) = 0$ (linear
consistency). The commitment enters only through its binding property;
the sumcheck arithmetic operates on evaluations, not on committed
elements. This part is **orthogonal to the commitment scheme choice**.

---

## The substitution: ABBA for Ajtai at the inner layer

### The swap

Replace the inner commitment:

| Component | Ajtai (current) | ABBA (proposed) |
|-----------|----------------|-----------------|
| Key | $A \in R_q^{n_A \times \delta 2^m}$ | $\mathbf{a} \in \Lambda_q^{\delta 2^m}$, $n_A$ copies |
| Commitment | $t_i = A \cdot s_i \in R_q^{n_A}$ | $t_i = F_{\mathbf{a}}(s_i) \in T_0^{n_A}$ |
| Output dim per entry | $D_L$ (e.g. 64 for $\Phi_{128}$) | $\frac{3}{2}D_K$ (e.g. 48 for $\Phi_{128}$) |
| Binding | Module-SIS | ComSIS |

The base-$b$ gadget decomposition (step 1) is unchanged; it operates on
MLE coefficients, not on the commitment.

### Output dimension change and cascade

For $\Phi_{128}$ ($n = 64$, even):

- Inner output per entry: 48 $\mathbb{F}_q$-elements (ABBA) vs 64
  (Ajtai), a 25% reduction.
- The outer commitment $B$ operates on binary decompositions of the
  inner output. Fewer dimensions means fewer bits to decompose, so $B$'s
  column count decreases by 25%.
- The outer layer can remain standard Ajtai (its input is a binary
  vector, not a quaternion element), so its binding stays under
  Module-SIS.

### ComSIS binding (replacing Hachi Lemma 7)

The reduction is structurally parallel to Module-SIS:

1. An adversary opens $t_i$ to two distinct preimages $s_i \neq s'_i$.
2. Then $F_{\mathbf{a}}(s_i - s'_i) = 0$ with
   $0 < \|s_i - s'_i\|_\infty \leq 2(b-1)$, which is a ComSIS solution.
3. The norm bound on $s_i - s'_i$ carries over from the gadget
   decomposition, same as the Ajtai case.

The concrete security level changes: ComSIS instances correspond to SIS
on $3n^+ \times 4n^+ m$ matrices (vs $4n^+ \times 4n^+ m$ for
Module-SIS), as documented in the
[ABBA ComSIS section](../papers/abba.md#comssis-the-hardness-assumption).
For $\Phi_{128}$ ($n^+ = 16$), the ComSIS dimension per row is $48$ vs
Ajtai's $64$. The Lattice Estimator analysis needs to confirm this meets
NIST Category III or above at $\kappa = n_A$.

Hachi Lemma 8 (outer binding) is unchanged if the outer layer stays
Ajtai. Lemmas 9--11 (evaluation and sumcheck binding) use inner binding
in black-box fashion: if ComSIS binding holds, the rest follows.

---

## The restricted homomorphism obstacle

This is the central technical difficulty.

### Where homomorphism is used

In the folding step (Hachi Eq. 18--19), the protocol computes:

$$
A \cdot z = A \cdot \left(\sum_i c_i \cdot s_i\right) = \sum_i c_i \cdot (A \cdot s_i)
$$

This identity, $A \cdot (c \cdot s) = c \cdot (A \cdot s)$ for any
$c \in R_q$, is what makes Ajtai's commitment $R_q$-homomorphic. The
challenges $c_i$ are short elements of $R_q$ (the challenge space
$\mathcal{C}$), and in the extension-field sumcheck rounds (Figure 7),
the evaluation challenges $a_i$ are sampled from $\mathbb{F}_{q^k}$.

For ABBA, the homomorphic identity
$\alpha \cdot F_{\mathbf{a}}(s) = F_{\mathbf{a}}(\alpha \cdot s)$ holds
**only when $\alpha \in \mathcal{O}_{K^+}$** (fixed by $\theta$). A
general $c \in R_q$ or $\alpha \in \mathbb{F}_{q^k}$ breaks the
identity because the commutator's internal $\theta$ applications do not
commute with arbitrary scalars (see
[ABBA folding integration](../papers/abba.md#folding-integration-the-ok-challenge-constraint)).

### Approach (a): restrict challenges to $\mathcal{O}_{K^+}$

Sample folding challenges $c_i$ from a strong set
$S \subset \mathcal{O}_{K^+}$ instead of $R_q$. This is the same
$\mathcal{O}_K$ projection technique already implemented for Neo folding
($\rho_K = \rho + \theta(\rho)$), further restricted to the real
subfield.

**Soundness cost.** The effective challenge space has
$\mathbb{F}_q$-dimension $n^+ = D_K/2$ instead of $D_K$ (or $D_L$).
For $\Phi_{128}$: $n^+ = 16$. With strong-set alphabet $|S| = 3$:

$$
|S|^{n^+} = 3^{16} \approx 4.3 \times 10^7 \approx 2^{22}
$$

This is ~22 bits of soundness per challenge, which is likely too low for
Hachi's target $\lambda = 128$. To compensate:

- Increase $|S|$ (e.g. $|S| = 5$: $5^{16} \approx 2^{37}$, still
  insufficient for single-round extraction).
- Use repetition: $k$ independent challenges give
  $|S|^{k \cdot n^+}$ soundness, but this multiplies proof size by $k$.
- Accept a weaker extraction guarantee and rely on the overall protocol
  composition for security (Hachi's soundness is CWSS-based, so each
  component contributes independently).

**Cross-reference**: Hachi Section 4.2 (Figure 3) defines the challenge
space as $\mathcal{C} \subseteq \{c \in R_q : \|c\| \leq \omega\}$. The
CWSS analysis in Lemma 8 requires that $\mathcal{C}$ is a
$(2^r, \omega)$-challenge space; whether $\mathcal{O}_{K^+}$-restricted
challenges satisfy this needs explicit verification against Definition 1
(special soundness sets).

### Approach (b): challenge decomposition

Write any $c \in R_q$ as $c = c^+ + c^-$ where
$c^+ = \frac{c + \theta(c)}{2} \in \mathcal{O}_{K^+}$ and
$c^- = \frac{c - \theta(c)}{2} \in \ker(1 + \theta)$. Then:

$$
c \cdot F_{\mathbf{a}}(s) = c^+ \cdot F_{\mathbf{a}}(s) + c^- \cdot F_{\mathbf{a}}(s)
$$

The first term equals $F_{\mathbf{a}}(c^+ \cdot s)$ by homomorphism.
The second term does **not** equal $F_{\mathbf{a}}(c^- \cdot s)$ in
general. The question is whether there exists a correction identity of
the form:

$$
c \cdot [a, x] = [a, c^+ \cdot x] + \text{correction}(a, c^-, x)
$$

where the correction term can be expressed as a function the prover can
commit to and the verifier can check.

**Concrete algebra.** For a single commutator $[a, x]$ with
$a = a_0 + u a_1$ and $x = x_0 + u x_1$:

$$
c \cdot [a, x] = c \cdot (a x - x a)
$$

Expanding $c = c^+ + c^-$ and using $c^+ \cdot u = u \cdot c^+$
(since $\theta(c^+) = c^+$) and
$c^- \cdot u = -u \cdot c^-$ (since $\theta(c^-) = -c^-$):

The $c^+$ part commutes with $u$, so
$c^+ \cdot [a, x] = [c^+ \cdot a, x] = [a, c^+ \cdot x]$ (using
bilinearity of the commutator and centrality of $c^+$).

The $c^-$ part anti-commutes with $u$. We get:

$$
c^- \cdot [a, x] = c^- \cdot (ax - xa)
$$

Since $c^- \in \ker(1+\theta)$, left-multiplication by $c^-$ does not
preserve the traceless subspace in general, because $c^-$ does not
commute with the quaternion structure.

More precisely, for a traceless element $t = t_0 + u t_1$ with
$t_0 + \theta(t_0) = 0$:

$$
c^- \cdot t = c^- t_0 + u \cdot \theta(c^-) \cdot t_1 = c^- t_0 - u \cdot c^- \cdot t_1
$$

The $u$-component $-c^- t_1$ is unconstrained, but the scalar component
$c^- t_0$ needs $c^- t_0 + \theta(c^- t_0) = 0$ to stay in $T_0$. We
have $\theta(c^- t_0) = \theta(c^-)\theta(t_0) = (-c^-)(-t_0) = c^- t_0$,
so $c^- t_0 + \theta(c^- t_0) = 2 c^- t_0 \neq 0$ in general.

**Conclusion**: $c^- \cdot T_0 \not\subseteq T_0$. Scaling a traceless
element by an anti-fixed scalar does not preserve tracelessness. This
means the folded commitment $\sum c_i \cdot t_i$ would not live in
$T_0^{n_A}$, breaking the algebraic structure.

This rules out a simple decomposition fix. A more involved approach
would introduce an auxiliary commitment to the non-traceless "overflow"
component, but this adds complexity and likely negates the size saving.

### Approach (c): hybrid architecture

Use ABBA for the inner commitment (getting 25% compression on the
commitment itself) but convert to an Ajtai-compatible representation
before the folding step:

1. The prover commits $t_i = F_{\mathbf{a}}(s_i) \in T_0^{n_A}$ (25%
   smaller commitments).
2. Before folding, the prover "lifts" each $t_i$ to a representation in
   $R_q^{n_A'}$ by embedding $T_0 \hookrightarrow \Lambda_q$ (zero-padding
   the trace component).
3. Folding proceeds over $R_q$ with unrestricted challenges.
4. The verifier uses the lifted representation for the folding check.

**Trade-off**: the commitment is 25% smaller, but the evaluation proof
operates on the lifted $R_q$ representation, so the proof transcript is
the same size as Ajtai. The net saving depends on what fraction of the
55 KB proof is the commitment vs. the evaluation proof (see
[next section](#does-the-25-propagate-to-proof-size)).

### Comparison with Neo's $\mathcal{O}_K$ projection

| Aspect | Neo (implemented) | Hachi (proposed) |
|--------|------------------|-----------------|
| Where ABBA sits | CCS witness commitment | Inner multilinear commitment |
| Homomorphism needed for | RLC folding challenges | Evaluation-protocol folding challenges |
| Challenge domain (before projection) | $R_q$ | $R_q$ (short challenges) and $\mathbb{F}_{q^k}$ (sumcheck) |
| Projected domain | $\mathcal{O}_K$ (full CM field) | $\mathcal{O}_{K^+}$ (real subfield) |
| Challenge entropy loss | Factor 2 ($D_K/2$ vs $D_K$) | Factor 4 ($n^+$ vs $D_L$) for $\Phi_{128}$ |
| Soundness impact | Absorbed by margin ($3^{16} \gg 2^{-\lambda}$ per round) | Needs explicit computation; may be too tight |
| Norm growth | $\|\rho_K\| \leq 2\|\rho\|$ | Same, plus potential interaction with Hachi's $\beta$ bound |

The key difference: Neo's folding has many independent rounds, each with
moderate soundness requirements, so the halved challenge space is
tolerable. Hachi's CWSS extraction requires large challenge spaces in a
single round (Lemma 8), making the entropy loss more painful.

---

## The gadget-decomposition embedding problem

Hachi's inner commitment takes base-$b$ gadget-decomposed coefficients
$s_i$ with entries in $\{0, 1, \ldots, b-1\} \subset \mathbb{Z}$. For
Ajtai, these are just ring elements and $A \cdot s_i$ works directly.

For ABBA, integer scalars lie in
$\mathcal{O}_{K^+} \subset \mathcal{O}_K \subset \Lambda$, so
$[a, c] = 0$ for any integer $c$. This is the same "naive embedding
fails" problem documented in the
[ABBA wiki](../papers/abba.md#the-naive-embedding-fails).

### The $u$-embedding fix

Map each coefficient $c_j \in \{0, \ldots, b-1\}$ to the quaternion
$(0, c_j) = c_j \cdot u$. Since $u \notin \mathcal{O}_{K^+}$, the
commutator $[a, c_j u]$ is generically nonzero:

$$
[a, c_j u] = c_j \cdot [a, u] = c_j \cdot \big((a_1 - \theta(a_1)) + u(\theta(a_0) - a_0)\big)
$$

where the scalar $c_j \in \mathbb{Z}$ factors out by bilinearity. This
uses only the theta automorphism (signed permutation) and
$\mathcal{O}_K$-additions, with zero field multiplications for binary
$c_j$. For $b > 2$, the scalar multiplication $c_j \cdot [\,]$ costs
$O(\log c_j)$ additions.

### Norm analysis

The $u$-embedding maps $s_i \in \{0, \ldots, b-1\}^{\delta 2^m}$ to
quaternion elements $(0, s_{i,j}) \in \Lambda$ with
$\|(0, s_{i,j})\|_\infty = \|s_{i,j}\|_\infty \leq b-1$. The ComSIS
norm bound in the binding reduction becomes:

$$
\|s_i - s'_i\|_\infty \leq 2(b-1)
$$

same as the Ajtai case, because the $u$-embedding preserves the
$\ell_\infty$ norm. The ComSIS parameters ($q$, $m$, $\beta$) can
therefore be set identically to the Module-SIS parameters in Hachi,
modulo the dimension change ($3n^+$ rows vs $n_A \cdot D_L$ for Ajtai).

### Comparison with Neo's binary embedding

In Neo, the witness is binary ($b = 2$), so only the $\{0, u\}$
embedding is needed. In Hachi, $b$ is typically 4 or 16 (Hachi uses
$b = O(1)$ with $\delta = \lceil \log_b q \rceil \approx 5$ for
$q \approx 2^{32}$). The multi-valued embedding $c_j \mapsto c_j u$
generalises the binary case; the commit cost per column becomes
$O(4 w D_K)$ additions where $w$ is the number of nonzero coefficients
(pay-per-bit still holds: zero entries contribute nothing).

---

## Does the 25% propagate to proof size?

Hachi's ~55 KB proof (for $\ell = 30$ variables at 128-bit security)
breaks down as (Hachi Section 5.2):

| Component | Size | What it is |
|-----------|------|-----------|
| Ring-switching transformation | ~7.3 KB | $d + k(k-1)$ $\mathbb{Z}_q$-elements from Section 3 |
| Commitment $v$ (first iteration) | ~4.8 KB | $n_D \cdot d$ $\mathbb{Z}_q$-elements (Section 4.2) |
| Greyhound evaluation proof | ~43 KB | Recursive evaluation via [NS24] |
| **Total** | **~55.1 KB** | |

### Where ABBA compression applies

- **Commitment $v$** (~4.8 KB): This is the commitment to the
  intermediate witness $\hat{w}$. If this uses the inner commitment
  scheme, ABBA saves 25%, i.e. ~1.2 KB. This is only ~2% of the total.
- **Greyhound evaluation proof** (~43 KB): This is the dominant
  component. Greyhound internally uses its own Ajtai commitments
  (inner/outer) for the recursive iterations. If those are also replaced
  by ABBA, the savings depend on what fraction of Greyhound's 43 KB is
  Ajtai commitments vs sumcheck transcripts. From [NS24, Table 1], the
  commitment portion is roughly half, so ABBA could save
  ~$0.25 \times 21.5 \approx 5.4$ KB.
- **Ring-switching** (~7.3 KB): Unchanged (independent of commitment
  scheme).

**Estimated total saving**: ~1.2 + ~5.4 = ~6.6 KB, bringing the proof
from ~55 KB to ~48.5 KB, a **~12% reduction**. This aligns with the
Neo experience, where ABBA $\Phi_{128}$ proofs were 12--14% smaller than
Ajtai $\Phi_{128}$ proofs (the 25% commitment saving is diluted by
non-commitment data).

### Caveat

This estimate assumes the Greyhound component's Ajtai commitments can
also be replaced by ABBA. Greyhound has its own internal folding steps
that rely on $R_q$-homomorphism, so the same restricted-homomorphism
obstacle applies recursively. If only Hachi's first-iteration commitment
uses ABBA (with Greyhound's internal commitments remaining Ajtai), the
saving drops to ~1.2 KB (~2%).

---

## Comparison: ABBA-in-Neo vs ABBA-in-Hachi

| Aspect | Neo (done) | Hachi (proposed) |
|--------|-----------|-----------------|
| Where ABBA sits | CCS witness commitment | Inner multilinear commitment |
| Embedding | Binary $\{0, u\}$ | Multi-valued $c_j \mapsto c_j u$ |
| Base $b$ | 2 | 4--16 |
| Homomorphism constraint | RLC folding ($\mathcal{O}_K$ projection) | Evaluation folding ($\mathcal{O}_{K^+}$ restriction or hybrid) |
| Soundness impact | Tolerable (many rounds, moderate per-round) | Tight (CWSS extraction needs large challenge space) |
| End-to-end proof saving | 12--14% ($\Phi_{128}$) | Estimated 2--12% depending on scope |
| Implementation complexity | Drop-in (same `SModuleHomomorphism` trait) | Requires resolving homomorphism obstacle |

---

## Open questions

1. **Exact soundness for $\mathcal{O}_{K^+}$-restricted challenges in
   Hachi's CWSS.** What is the per-round extraction probability? Does it
   meet $\lambda = 128$ with $|S|^{n^+}$ for practical $|S|$ and
   $n^+ = 16$? Cross-reference Hachi Definition 1 (special soundness
   sets) and Lemma 8.

2. **Can $c^- \cdot T_0$ be "absorbed" into the protocol?** The analysis
   in approach (b) shows $c^- \cdot T_0 \not\subseteq T_0$. But if the
   non-traceless overflow is small and bounded, an extended commitment
   (committing to both the traceless and overflow parts) might recover
   full-ring homomorphism at the cost of a larger commitment. Whether the
   net effect is still smaller than Ajtai is unclear.

3. **Algebraic identity for $c \cdot [a, x]$.** Is there a clean
   decomposition of $c \cdot [a, x]$ into commutator-like terms that the
   verifier can check? This would be independently interesting for the
   lattice commitment literature.

4. **Greyhound's internal homomorphism requirements.** Does Greyhound's
   recursive evaluation (the 43 KB component) use the same folding
   pattern as Hachi's first iteration? If so, the restricted-homomorphism
   obstacle applies to every recursive level, not just the first.

5. **Concrete parameters for $\Phi_{128}$.** Derive $n_A$, $n_B$, $b$,
   $\delta$, $\omega$, and $\beta$ for Hachi-with-ABBA at $\lambda = 128$.
   Check the ComSIS security level at the resulting dimensions.

6. **Is the hybrid approach (c) sufficient for ARIA's goals?** If the
   target is "proof size $\leq 40$ KB at 128-bit security" (ARIA R1),
   saving ~1.2 KB on the first-iteration commitment alone is marginal.
   The value of ABBA in Hachi may instead be **conceptual**: demonstrating
   that commutator-based commitments compose with the Greyhound/Hachi
   framework, even if the concrete savings require further optimization
   (e.g. a fully ABBA-native recursive scheme that avoids the Greyhound
   component entirely).
