# ABBA: Lattice-based Commitments from Commutators

> Alberto Centelles, Andrew Mendelsohn
> *ABBA: Lattice-based Commitments from Commutators*
> ePrint [2026/148](https://eprint.iacr.org/2026/148), accepted at PQCrypto 2026

The PDF and full markdown of the paper live in
`raw/lattices/2026-148-abba-lattice-based-commitments-from-commutators/`.

A Rust implementation (`neo-abba`) lives in `code/Nightstream/crates/neo-abba/`,
with benchmarks comparing ABBA to Ajtai for Neo.

## TL;DR

ABBA is a linearly homomorphic commitment scheme based on sums of
commutators of quaternions modulo $q$. For a key vector
$\mathbf{a} \in \Lambda_q^m$ and input $\mathbf{x} \in \Lambda_q^m$, the
commitment is

$$
F_{\mathbf{a}}(\mathbf{x}) = \sum_{i=1}^{m} [a_i, x_i] \in T_0
$$

where $[a, b] = ab - ba$ is the ring commutator and $T_0$ is the traceless
subspace of the quaternion algebra. Commutators compress: a 4-dimensional
quaternion element maps into the 3-dimensional traceless subspace, giving
a 25% compression over the full algebra. Under the ComSIS assumption
(a commutator-analogue of SIS on quaternion orders), the scheme is
computationally binding and computationally hiding.

The paper demonstrates a drop-in replacement for Ajtai commitments in
Neo (a lattice-based folding scheme), claiming a 25% reduction in
commitment size. This claim depends on a parity condition on the
cyclotomic conductor (see [the parity obstruction](#the-parity-obstruction)
below).

## The quaternion algebra

The paper defines the field tower as $L = \mathbb{Q}(\zeta_{2n})$,
$K = \mathbb{Q}(\zeta_n)$, $K^+ = K \cap \mathbb{R}$. The quaternion
algebra is

$$
A = (K/K^+, \theta, -1)
$$

where $\theta$ is complex conjugation on $K/K^+$ and $u^2 = -1$. Elements
are $a = a_0 + u a_1$ with $a_0, a_1 \in \mathcal{O}_K$, and the
multiplication rule is $u \cdot \ell = \theta(\ell) \cdot u$ for
$\ell \in K$. The natural order is
$\Lambda = \mathcal{O}_K \oplus u \mathcal{O}_K$.

### Key dimensions (general)

Let $D_L = \varphi(2n) = [L:\mathbb{Q}]$, $D_K = \varphi(n) = [K:\mathbb{Q}]$,
and $n^+ = [K^+:\mathbb{Q}] = D_K/2$.

| Object | $\mathbb{F}_q$-dimension |
|--------|-----|
| $\mathcal{O}_{L,q}$ (Ajtai's ring) | $D_L$ |
| $\mathcal{O}_{K,q}$ (quaternion component ring) | $D_K$ |
| $\mathcal{O}_{K^+,q}$ (real subfield) | $n^+$ |
| $\Lambda_q = \mathcal{O}_K \oplus u \mathcal{O}_K$ | $2 D_K$ |
| $T_0$ (traceless subspace) | $3 n^+ = \tfrac{3}{2} D_K$ |

### How $\varphi$ and $\theta$ determine every dimension

The totient $\varphi$ and the automorphism $\theta$ are two aspects of
the same Galois structure. $\varphi$ counts the total degrees of freedom;
$\theta$ determines how they split between "real" and "imaginary" parts.

**$\varphi(n)$ defines the Galois group.** The Galois group
$\text{Gal}(\mathbb{Q}(\zeta_n)/\mathbb{Q}) \cong (\mathbb{Z}/n\mathbb{Z})^\times$
has order $\varphi(n)$. Each $a \in (\mathbb{Z}/n\mathbb{Z})^\times$
acts as $\sigma_a : \zeta_n \mapsto \zeta_n^a$.

**$\theta$ is one element of this group.** Complex conjugation is
$\sigma_{n-1} : \zeta_n \mapsto \zeta_n^{-1}$. It has order 2 and
generates a subgroup $\{1, \theta\}$. Its fixed field is the maximal
real subfield $K^+ = \mathbb{Q}(\zeta_n + \zeta_n^{-1})$ with
$[K^+:\mathbb{Q}] = \varphi(n)/2$.

**$\theta$ splits $\mathcal{O}_K$ into eigenspaces.** The ring
$\mathcal{O}_K$ (dimension $D_K = \varphi(n)$ over $\mathbb{F}_q$)
decomposes under $\theta$ into:

- Fixed space $\{a : \theta(a) = a\} = \mathcal{O}_{K^+}$ (dimension
  $D_K/2 = n^+$), where ABBA's folding scalars live.
- Anti-fixed space $\{a : \theta(a) = -a\} = \ker(1+\theta)$ (dimension
  $D_K/2 = n^+$), which forms the "$u^0$" part of $T_0$.

**Together they give $T_0$.** The traceless subspace of the quaternion
combines the anti-fixed eigenspace of $\theta$ (for the $a_0$ component)
with the full $\mathcal{O}_K$ (for the $a_1$ component):

$$
\dim(T_0) = \underbrace{n^+}_{\ker(1+\theta)} +
\underbrace{D_K}_{u\text{-part}} =
\frac{D_K}{2} + D_K = \frac{3}{2} D_K
$$

And $\dim(\Lambda) = 2 D_K$, so:

$$
\frac{\dim(T_0)}{\dim(\Lambda)} = \frac{3 D_K / 2}{2 D_K} = \frac{3}{4}
$$

This is the 25% compression: $\varphi$ sets the denominator ($2D_K$)
and $\theta$'s eigenspace splitting sets the numerator ($\frac{3}{2}D_K$).

**The parity condition connects $\varphi$ and $\theta$ to $O_L$.**
For the 25% to beat Ajtai (not just the full quaternion), we need
$\dim(O_L) = \dim(\Lambda)$, i.e. $\varphi(2n) = 2\varphi(n)$. This
holds iff $n$ is even (see
[the parity obstruction](#the-parity-obstruction) below).

### The theta automorphism

Theta is complex conjugation: $\theta(\zeta_n) = \zeta_n^{-1}$.
On the coefficient representation $f(X) = \sum c_i X^i$ in
$\mathbb{F}_q[X]/(\Phi_n)$, theta acts as a **signed coefficient
permutation**: each monomial $X^i$ maps to at most 2 monomials with
$\pm 1$ coefficients. No $\mathbb{F}_q$-multiplications are needed.

For $\Phi_{64}(X) = X^{32} + 1$ (the quaternion component ring when
$\eta = 128$): $\theta(X^i) = (-1)^{i+q} X^r$ where $31i = 32q + r$. Each
monomial maps to exactly one monomial with a sign flip.

### The traceless subspace

$T_0 = \ker(1 + \theta) \oplus u \cdot \mathcal{O}_{K,q}$:

- $a_0$ must satisfy $a_0 + \theta(a_0) = 0$ (the "anti-fixed" subspace of
  $\theta$, dimension $n^+$).
- $a_1$ is unrestricted in $\mathcal{O}_{K,q}$ (dimension $D_K$).
- Total: $n^+ + D_K = \tfrac{3}{2} D_K$ dimensions over $\mathbb{F}_q$.

## The commitment scheme

### Key generation

$$
\text{Gen}(1^\lambda): \quad \mathbf{a}, \mathbf{a}' \leftarrow U(\Lambda_q^m), \quad \text{output } (\mathbf{a}, \mathbf{a}')
$$

Two key vectors: $\mathbf{a}$ for the message, $\mathbf{a}'$ for the
randomness.

### Commit

$$
\text{Com}(\mu, r) = F_{\mathbf{a}}(\mu) + F_{\mathbf{a}'}(r) = \sum_i [a_i, \mu_i] + \sum_i [a_i', r_i] \in T_0
$$

where $\mu$ is binary and $r \leftarrow D_{\Lambda^m, \sigma}$ is sampled
from a discrete Gaussian.

### Security

**Hiding** (Theorem 7): for $m \geq 16(n^+)^2$ and
$\sigma \geq \eta_\varepsilon(\Lambda)$, the commitment is hiding with
probability $1 - \text{neg}(n^+)$.

**Binding** (Theorem 7): computationally binding under the
$\text{ComSIS}_{q, 2m, \beta}$ assumption.

### Homomorphic property

For any $\alpha \in \mathcal{O}_{K^+,q}$ (the real subfield, fixed by
$\theta$):

$$
\text{Com}(\mu, r) + \alpha \cdot \text{Com}(\mu', r') = \text{Com}(\mu + \alpha\mu', r + \alpha r')
$$

## The parity obstruction

The paper's Table 1 claims ABBA commitments in Neo are 25% smaller than
Ajtai: output size $\tfrac{3}{2}nkN$ vs $2nkN$, ratio 3/4. This relies
on the identity stated in Section 8:

> $|O_{L,q}| = |\Lambda_q|$ and $N = [O_{L,q} : \mathbb{F}_q] = [\Lambda_q : \mathbb{F}_q]$

This identity, $D_L = 2 D_K$, holds **if and only if $n$ is even**:

$$
D_L = \varphi(2n), \quad 2 D_K = 2\varphi(n)
$$

| Parity of $n$ | $\varphi(2n)$ | $2\varphi(n)$ | $D_L = 2D_K$? |
|---|---|---|---|
| $n$ even | $2\varphi(n)$ | $2\varphi(n)$ | **yes** |
| $n$ odd | $\varphi(n)$ | $2\varphi(n)$ | **no** ($\Lambda$ is $2\times$ bigger than $O_L$) |

When $n$ is odd, $\gcd(2, n) = 1$, so
$\varphi(2n) = \varphi(2)\varphi(n) = \varphi(n)$. The number field $L$ and
the quaternion order $\Lambda$ then have different dimensions, and $T_0$ at
$\tfrac{3}{4}\dim(\Lambda)$ ends up $\tfrac{3}{2}\dim(O_L)$: **50% larger,
not 25% smaller.**

### Nightstream's $\eta = 81$ (odd): the problem

Nightstream uses $\Phi_{81}$ with $n = 81$ (odd):

| Object | Dim | For $\kappa = 16$ |
|--------|-----|---|
| $O_L$ (Ajtai) | 54 | **864 Fq** |
| $T_0$ (ABBA) | 81 | 1,296 Fq |
| $\Lambda$ (full quaternion) | 108 | 1,728 Fq |

ABBA is 25% smaller than the full quaternion (81 vs 108), but **50%
larger than Ajtai** (81 vs 54).

### $\Phi_{128}$ (even $n$): the fix

With $\eta = 128$, $n = 64$ (even): $D_L = \varphi(128) = 64$ and
$2D_K = 2\varphi(64) = 64$. The identity holds.

| Object | Dim | For $\kappa = 16$ |
|--------|-----|---|
| $O_L$ (Ajtai) | 64 | 1,024 Fq |
| $T_0$ (ABBA) | 48 | **768 Fq** |
| $\Lambda$ | 64 | 1,024 Fq |

ABBA commitment is genuinely **25% smaller** than Ajtai (768 vs 1,024).

Goldilocks ($q = 2^{64} - 2^{32} + 1$) is compatible: $64 \mid q - 1$
since $q - 1 = 2^{32}(2^{32} - 1)$, so 64th roots of unity exist in
$\mathbb{F}_q$.

This is verified by the standalone $\Phi_{128}$ test suite in
`crates/neo-abba/tests/phi128_test.rs` (10 tests: theta involution,
tracelessness, sparse commutator correctness, linearity, dimension checks,
and the size comparison table).

## Neo integration

### The naive embedding fails

In Neo, the witness $Z$ consists of binary elements $\{0, 1\}$ after
base-2 decomposition. The naive quaternion embedding $0 \mapsto 0$,
$1 \mapsto 1$ is useless: both $0$ and $1$ lie in $\mathcal{O}_{K^+}$
(the centre of the commutator), so $[a, 0] = [a, 1] = 0$ for all $a$.

### The {0, u} embedding

Map $0 \mapsto 0$, $1 \mapsto u$. Since $u \notin \mathcal{O}_{K^+}$,
the commutator $[a, u]$ is generically nonzero:

$$
[a_0 + u a_1, \; u] = (a_1 - \theta(a_1)) + u(\theta(a_0) - a_0)
$$

This uses only the theta automorphism (a signed permutation) and two
$\mathcal{O}_K$-subtractions, for a total of $2 D_K$ field additions and
**zero field multiplications**. The result automatically lies in $T_0$.

### Column-based sparse commit (the practical approach)

Rather than giving each bit its own key (which causes a $d \times$ key
blowup), we pack $d$ bits per column into one $R_q$ element
$z_j = \sum_t z_t X^t$ and embed as the quaternion $(0, z_j)$. The
commitment for $\kappa$-row $i$ is:

$$
C[i] = \sum_{j=0}^{m-1} [A_{i,j}, (0, z_j)]
$$

This uses $m$ keys per $\kappa$-row (matching Ajtai). For binary
$z_j$ with Hamming weight $w_j$, the commutator $[a, (0, z_j)]$
decomposes by bilinearity into $w_j$ terms, each computed via
`commutator_with_uz_sparse` using only `mul_by_monomial` (zero
$\mathbb{F}_q$-multiplications).

Closed form:

$$
[a, (0, z)]_0 = a_1 \cdot \theta(z) - \theta(a_1) \cdot z
$$
$$
[a, (0, z)]_1 = z \cdot (\theta(a_0) - a_0)
$$

For binary $z$, each multiplication by $z$ is $w$ additions of rotated
copies. Total per column: $O(4 w D_K)$ additions, zero multiplications.

### Benchmark results (Apple M-series, b=2)

All benchmarks use the column-based sparse commit with precomputed theta
values (see `crates/neo-abba/tests/phi128_test.rs`). Both Ajtai and ABBA
are pay-per-bit for binary witnesses: they skip zero entries, so commit
time scales linearly with Hamming weight.

#### $\Phi_{128}$: the apples-to-apples comparison

With even $n = 64$, the parity obstruction vanishes and we get a fair
comparison. Both schemes commit to the same 8,192 binary bits;
$\kappa = 16$.

|  | Ajtai ($O_L$) | ABBA ($T_0$) |
|---|---|---|
| Ring dimension | $D_L = 64$ | $D_K = 32$ |
| Bits per column | 64 | 32 |
| Columns ($m$) | 128 | 256 |
| Keys per $\kappa$-row | 128 Rq128 | 256 QuatK |

**Density sweep** (8,192 bits, $\kappa = 16$):

| Density | Ajtai | ABBA (optimized) | Ratio |
|---|---|---|---|
| 10% | 1.53 ms | 2.64 ms | 1.7$\times$ |
| 25% | 3.64 ms | 5.39 ms | 1.5$\times$ |
| 50% | 7.12 ms | 10.10 ms | 1.4$\times$ |
| 75% | 10.02 ms | 14.27 ms | 1.4$\times$ |
| 100% | 12.90 ms | 18.66 ms | 1.4$\times$ |

Both scale linearly with density. ABBA's overhead stabilises at
$\sim$1.4$\times$ (from 3 sparse-by-dense operations per column vs
Ajtai's 1 rotation accumulation). The optimizations that bring the ratio
from $\sim$13$\times$ (unoptimized, $\Phi_{81}$) to 1.4$\times$ are:

1. **Precomputed theta**: $\theta(a_0)$, $\theta(a_1)$, and
   $\text{diff} = \theta(a_0) - a_0$ are computed once at setup and stored
   alongside each key. Eliminates redundant theta evaluations across
   commits.
2. **Stack-allocated bit lists**: nonzero positions are collected into a
   fixed-size `[usize; D_K]` array instead of a heap-allocated `Vec`,
   avoiding allocation in the hot loop.
3. **Correct ring dimension**: $\Phi_{128}$ uses $D_K = 32$ (vs
   $\Phi_{81}$'s $D_K = 54$), so each `mul_by_monomial` and addition
   touches fewer elements.

**Size comparison**:

| Metric | Ajtai | ABBA | Ratio |
|---|---|---|---|
| Commitment output | 1,024 Fq (8.0 KB) | **768 Fq (6.0 KB)** | **0.75$\times$** |
| PP keys ($m = 128$ / $256$) | 128 $\times$ 64 Fq | 256 $\times$ 64 Fq | 2$\times$ |

ABBA's commitment is **25% smaller** than Ajtai's, confirming the paper's
claim. The PP is 2$\times$ larger because ABBA needs twice as many columns
(32 bits/column vs 64) and each QuatK key has the same 64 Fq elements as
an Rq128 key (since $\dim(\Lambda) = \dim(O_L) = 64$ for even $n$).

#### $\Phi_{81}$: parity-obstructed (for reference)

On Nightstream's current $\Phi_{81}$ ($n = 81$, odd), ABBA cannot
achieve smaller commitments (see [the parity
obstruction](#the-parity-obstruction)). Column-based benchmarks with
$m = 256$, $\kappa = 16$, $d = 54$, witness size 13,824 bits:

| Density | Ajtai | ABBA (column) | Ratio |
|---|---|---|---|
| 10% | 3.8 ms | 12.3 ms | 3.2$\times$ |
| 50% | 2.8 ms | 38.2 ms | 13.5$\times$ |
| 100% | 2.8 ms | 70.7 ms | 25.3$\times$ |

Ajtai's time is constant here because Nightstream's `neo-ajtai` does not
skip zero entries (it processes all $d$ positions per column via
`rot_step`). ABBA is 3-25$\times$ slower and produces 1.5$\times$ larger
commitments (1,296 vs 864 Fq). The higher overhead (vs $\Phi_{128}$'s
1.4$\times$) comes from the larger $D_K = 54$ and the unoptimized
implementation (no precomputed theta).

## Folding integration: the O_K challenge constraint

### Why folding challenges must live in O_K

Neo's RLC phase computes a folded commitment $c_{\text{acc}} = \sum \rho_i
\cdot c_i$ and a folded witness $Z_{\text{acc}} = \sum \rho_i \cdot Z_i$,
where the $\rho_i$ are Fiat-Shamir challenges sampled from a "strong set"
$S \subset R_q$.

For the folding to be sound, we need:

$$
\text{Com}(\textstyle\sum \rho_i \cdot Z_i) = \textstyle\sum \rho_i \cdot \text{Com}(Z_i)
$$

For **Ajtai**, this holds for any $\rho \in R_q$ because the commitment
is $R_q$-homomorphic.

For **ABBA**, Equation (1) of the paper restricts the scalar to
$\alpha \in \mathcal{O}_{K,q}$ (the real subfield, fixed by $\theta$).
A general $\rho \in R_q$ that is NOT fixed by $\theta$ breaks the
identity: $\rho \cdot [A, (0, z)] \neq [A, (0, \rho \cdot z)]$ in general,
because the commutator's internal $\theta$ applications do not commute
with arbitrary $\rho$.

Concretely, with $\rho \in R_q \setminus \mathcal{O}_K$:
- The witness side computes $Z_{\text{acc}} = \sum \rho_i \cdot Z_i$
  (ring multiplication in $R_q$, always valid).
- The commitment side computes $\sum \rho_i \cdot C_i$ via `s_mul`, which
  scales the $T_0$ element by $\rho_i$.
- Recommitting $Z_{\text{acc}}$ gives a DIFFERENT result because the
  commutator $[A, (0, \rho \cdot z)]$ is not the same as
  $\rho \cdot [A, (0, z)]$ when $\rho$ does not commute with $\theta$.

This manifests as a $\Pi_{\text{DEC}}$ verification failure: `c=false`.

### The O_K projection fix

The fix is to project each sampled challenge to $\mathcal{O}_K$ before
it is used for both the witness folding and the commitment scaling.
Given a sampled $\rho \in R_q$, the projection is:

$$
\rho_K = \rho + \theta(\rho) \;\in\; \mathcal{O}_K
$$

Since $\theta(\rho_K) = \theta(\rho) + \theta^2(\rho) = \theta(\rho) +
\rho = \rho_K$, the projected element is indeed fixed by $\theta$.

In the implementation (`neo-reductions/src/common.rs`, gated behind
`#[cfg(feature = "abba")]`), this projection is applied inside
`sample_rot_rhos_n` immediately after drawing the coefficients from the
strong set, before building the rotation matrix. Both prover and
verifier sample identically via Fiat-Shamir, so they agree on the
projected $\rho_K$.

### Soundness implications

The projection halves the effective challenge set: $\mathcal{O}_K$ has
$\mathbb{F}_q$-dimension $n^+ = D_K / 2$ (27 for $\Phi_{81}$, 16 for
$\Phi_{128}$), compared to $D_K$ for the full $R_q$. This means:

- **Challenge entropy drops by a factor of 2** per challenge.
- The knowledge-soundness analysis from Neo (Section 4.3, Theorem 3)
  assumes challenges from a set of size $|S|^{D}$; with the O_K
  restriction the effective size is $|S|^{D_K/2}$.
- For $|S| \geq 3$ (the typical strong-set alphabet) and $D_K = 32$
  ($\Phi_{128}$), this gives $3^{16} \approx 4.3 \times 10^7$ possible
  challenges per round, still far above the $2^{-\lambda}$ soundness
  target for reasonable $\lambda$.

The norm growth from the projection ($\|\rho_K\| \leq 2\|\rho\|$) also
affects the DEC norm bound: the accumulated witness after $k$ RLC rounds
grows by an extra factor of 2 per round compared to Ajtai, requiring a
slightly larger $k_\rho$ (the decomposition depth). For the fibonacci
tests this is absorbed by the existing $k_\rho$ margin.

### End-to-end folding benchmarks ($\Phi_{81}$)

The full Π_CCS → Π_RLC → Π_DEC pipeline with ABBA commitments is
verified and benchmarked by the fibonacci integration test
(`neo-fold-next/tests/fibonacci_abba.rs`, run with `--features abba`).
Matching Ajtai metrics are in `fibonacci_ccs.rs`.

| Config (steps $\times$ trans/chunk) | | Prove (ms) | | Verify (ms) | | Proof size (bytes) | |
|---|---|---|---|---|---|---|---|
| | Ajtai | ABBA | Ajtai | ABBA | Ajtai | ABBA | Size ratio |
| 2 $\times$ 5 ($k_\rho = 12$) | 21.3 | 23.4 | 2.5 | 3.4 | 659,852 | 846,474 | 1.28$\times$ |
| 5 $\times$ 5 ($k_\rho = 12$) | 35.7 | 50.7 | 6.9 | 12.0 | 1,649,588 | 2,116,143 | 1.28$\times$ |
| 10 $\times$ 5 ($k_\rho = 12$) | 66.1 | 112.3 | 14.9 | 25.2 | 3,299,148 | 4,232,258 | 1.28$\times$ |
| 5 $\times$ 10 ($k_\rho = 13$) | 39.9 | 62.1 | 9.6 | 13.2 | 2,078,148 | 2,579,263 | 1.24$\times$ |

**Prove time**: 1.1-1.7$\times$ slower (overhead grows with step count,
dominated by commitment operations in $\Pi_{\text{RLC}}$).

**Verify time**: 1.4-1.7$\times$ slower (same cause: $T_0$ scaling via
`s_mul` costs more than $R_q$ rotation).

**Proof size**: consistently 1.24-1.28$\times$ larger on $\Phi_{81}$.

#### Proof size breakdown

Each Neo proof contains commitments (from $\Pi_{\text{CCS}}$,
$\Pi_{\text{RLC}}$, $\Pi_{\text{DEC}}$) plus non-commitment data
(sumcheck rounds, evaluation claims, CCS outputs). Each commitment
costs $\kappa \times d_{\text{slot}} \times 8$ bytes (Goldilocks):

| Scheme | $d_{\text{slot}}$ ($\Phi_{81}$) | $d_{\text{slot}}$ ($\Phi_{128}$) | Bytes/commit ($\Phi_{81}$) | Bytes/commit ($\Phi_{128}$) |
|---|---|---|---|---|
| Ajtai | $D = 54$ | $D_L = 64$ | 6,912 | 8,192 |
| ABBA | $T_0 = 81$ | $T_0 = 48$ | 10,368 | **6,144** |

On $\Phi_{81}$, ABBA commitments are 1.5$\times$ larger per slot (81 vs
54). Commitment data accounts for ~93% of the proof size difference
(measured diff matches predicted diff within 8%).

On $\Phi_{128}$, ABBA commitments are **25% smaller** per slot (48 vs
64), so the proof size flips:

| Config | Ajtai $\Phi_{81}$ | Ajtai $\Phi_{128}$ | ABBA $\Phi_{128}$ | ABBA/Ajtai |
|---|---|---|---|---|
| 2$\times$5 ($k_\rho = 12$) | 660 KB | 724 KB | **621 KB** | **0.86$\times$** |
| 5$\times$5 ($k_\rho = 12$) | 1,650 KB | 1,810 KB | **1,554 KB** | **0.86$\times$** |
| 10$\times$5 ($k_\rho = 12$) | 3,299 KB | 3,619 KB | **3,107 KB** | **0.86$\times$** |
| 5$\times$10 ($k_\rho = 13$) | 2,078 KB | 2,251 KB | **1,974 KB** | **0.88$\times$** |

ABBA $\Phi_{128}$ proofs are **12-14% smaller** than Ajtai $\Phi_{128}$
proofs (not the full 25% commitment reduction, because non-commitment
data dilutes the saving).

#### Per-commitment cost across fields and conductors

| | $\Phi_{81}$ (bytes) | $\Phi_{128}$ (bytes) |
|---|---|---|
| Ajtai (Goldilocks) | 6,912 | 8,192 |
| ABBA (Goldilocks) | 10,368 | **6,144** |
| Ajtai (Baby Bear) | 3,456 | 4,096 |
| ABBA (Baby Bear) | 5,184 | **3,072** |

On Baby Bear (4 bytes/element instead of 8), all sizes halve.
ABBA $\Phi_{128}$ on Baby Bear achieves the smallest commitment at
**3,072 bytes**, a 55% reduction vs Goldilocks Ajtai $\Phi_{81}$
(6,912 bytes) from the combined field + conductor + traceless savings.

At the full-proof level, ABBA $\Phi_{128}$ on Baby Bear projects to
~311 KB for a 2-step fold (vs 660 KB for Goldilocks Ajtai $\Phi_{81}$,
a **53% reduction**).

### Simulated folding performance for $\Phi_{128}$

The full Neo folding pipeline cannot currently run on $\Phi_{128}$
because `neo-math` is hardcoded for $\Phi_{81}$ ($D = 54$). However,
`phi128_test.rs` includes a simulated folding benchmark that exercises
the same operations as the real pipeline: commit all steps, then
accumulate via $\rho \cdot c$ (s_mul) with O_K-projected challenges.

| Config (steps $\times$ $m$, bits) | Ajtai | ABBA | Ratio | Commitment size |
|---|---|---|---|---|
| 2 $\times$ 16 (1,024 bits) | 2.42 ms | 2.58 ms | **1.06$\times$** | 768 vs 1,024 Fq (**25% smaller**) |
| 5 $\times$ 16 (1,024 bits) | 4.97 ms | 6.31 ms | **1.27$\times$** | 768 vs 1,024 Fq (**25% smaller**) |
| 10 $\times$ 16 (1,024 bits) | 10.00 ms | 13.14 ms | **1.31$\times$** | 768 vs 1,024 Fq (**25% smaller**) |
| 5 $\times$ 32 (2,048 bits) | 9.53 ms | 13.11 ms | **1.38$\times$** | 768 vs 1,024 Fq (**25% smaller**) |

ABBA's folding overhead at $\Phi_{128}$ is **1.06-1.38$\times$**
(vs 1.1-1.7$\times$ on $\Phi_{81}$), while commitment size is
**25% smaller** (vs 28% larger on $\Phi_{81}$). The improvement
comes from $D_K = 32$ (vs 54): each field operation touches fewer
elements, and the s_mul ring multiplication is cheaper.

### Comparison: $\Phi_{81}$ vs $\Phi_{128}$

| Metric | $\Phi_{81}$ (odd, obstructed) | $\Phi_{128}$ (even, clean) |
|---|---|---|
| Commitment (ABBA/Ajtai) | 1.28$\times$ larger | **0.75$\times$ (25% smaller)** |
| Folding overhead | 1.1-1.7$\times$ | **1.06-1.38$\times$** |
| Commit overhead (standalone) | 3.2-25$\times$ (unoptimized) | **1.4$\times$** |
| PP size ratio | 2$\times$ | 2$\times$ |
| The 25% claim | **fails** (parity) | **confirmed** |

$\Phi_{128}$ is the right conductor for ABBA: the commitment is
genuinely smaller, the overhead is moderate, and all algebraic
properties hold. Porting would require:

- `neo-math/src/ring.rs`: $D = 64$, rot_step for $X^{64} + 1$
  (simpler: just negate on wrap)
- `neo-math/src/quaternion.rs`: theta for $\Phi_{64}$, $D_K = 32$
- `neo-params`: new Goldilocks preset for $\eta = 128$
- `neo-ajtai`: update compile-time assertions from $D = 54$ to $D = 64$
- SuperNeo bar transform re-derivation

## ComSIS: the hardness assumption

The binding reduction relies on a quaternion-analogue of SIS:

> **ComSIS$_{q,m,\beta}$**: given $\mathbf{a} \in \Lambda_q^m$, find
> $\mathbf{z} \in \Lambda^m$ with
> $F_{\mathbf{a}}(\mathbf{z}) = \sum [a_i, z_i] = 0 \bmod q$ and
> $0 < \|\mathbf{z}\| \leq \beta$.

The paper gives a worst-case reduction:

$$
\text{approx-SVP}_{I, 2\gamma} \;\longrightarrow\; \text{ComSIS}_{q, m, \beta}
$$

for invertible ideals $I$ in the natural order of the quaternion algebra,
when $q$ is completely split in $\mathcal{O}_K$.

### Why the structure cuts both ways

It is tempting to read ComSIS as plain Module-SIS at
$3n^+ \times 4n^+ m$ instead of $4n^+ \times 4n^+ m$, i.e., a structured
SIS where the codomain has shrunk by 25% (the "row-rank reduction"; you
can see it directly by fixing a $\mathbb{Z}$-basis of $\Lambda$ and
writing $F_{\mathbf{a}}$ as an $\mathbb{F}_q$-matrix, which has rows
constrained to $T_0$). That view captures the *concrete*-security side
of the trade; it is what the Lattice-Estimator drop in [Security
level](#security-level) below measures.

But the row-rank reduction does not come from arbitrary structure: it
comes from the ring identity $\operatorname{Tr}(ab) = \operatorname{Tr}(ba)$,
which forces every commutator into the traceless subspace. That same
ring origin pays a *structural dividend* on the worst-case side, and
the dividend is what the worst-case-to-average-case reduction relies
on. The chain has three steps, all of which need ring arithmetic:

**1. The commutator subgroup and the commutator lattice (Definition 26).**
For an ideal $\mathcal{I}\subset\Lambda$, the paper defines

$$
[\mathcal{I}, \Lambda] \;=\; \left\{\sum_i [x_i, y_i]\;:\;x_i\in\mathcal{I},\, y_i\in\Lambda\right\} \;\subset\; \mathcal{I},
$$

an additive subgroup of $\mathcal{I}$. Under the canonical embedding it
is a sublattice of the ambient ideal lattice $\mathcal{I}$, the
*commutator lattice*. There is no analogue of this construction in plain
Module-SIS; it is purely ring-arithmetic.

**2. ComSIS solutions live in the commutator lattice (Corollary 3).**
A ComSIS solution is, by definition, a vector $\mathbf{z}$ with
$\sum_i [a_i, z_i] = 0$, hence the components of $\mathbf{z}$ produce a
sum-of-commutators that vanishes mod $q$. Lifted appropriately, this
places $\mathbf{z}$ inside the commutator lattice of an ideal built
from $\mathbf{a}$. A bare $3n^+$-row SIS instance not derived from a
ring would have no such constraint on its solutions.

**3. Short vectors in the commutator lattice are short vectors in the
ideal (Proposition 3).** The commutator sublattice is not far from the
ideal it sits inside:

$$
\lambda_1([\mathcal{I},\Lambda]) \;\leq\; 2\,\lambda_1(\mathcal{I}).
$$

Concretely, given a shortest $v = v_0 + uv_1 \in \mathcal{I}$, the
element $w = [v, u] = \xi(\theta(v_1) - v_1) + u(\theta(v_0) - v_0)$
lies in $[\mathcal{I}, \Lambda]$ with $\|w\| \leq 2\|v\|$. So a short
vector in the commutator lattice is automatically a short vector in
$\mathcal{I}$, up to a constant approximation factor.

**Composition.** Combining the three steps:

```
ComSIS solution
   => short vector in [I, Λ]       (step 2; ring arithmetic)
   => short vector in I, factor 2  (step 3; geometric tightness)
   => approx-SVP_I solution         (definition of approx-SVP)
```

This is the engine behind the SIVP/approx-SVP $\to$ ComSIS reduction.
The naive "row-rank reduction" framing loses this chain entirely: it
sees only a smaller codomain. A *naked* SIS at $3n^+$ rows,
unconstrained by ring arithmetic, would have the same row-rank
reduction without earning the dividend, and would be strictly easier to
attack than ComSIS.

**I-ComSIS uses the same lever.** The reduction from I-CSIS$^\times$
(inhomogeneous CSIS in the natural order, with invertible keys) to
I-ComSIS conjugates a CSIS instance by an invertible quaternion to turn
the linear equation $\mathbf{a}\cdot\mathbf{z} = v$ into a commutator
equation involving the map $a \mapsto [a, u]$. Without ring
multiplication this conjugation has no analogue; with it, I-CSIS$^\times$
maps into I-ComSIS in polynomial time, transferring CSIS's worst-case
hardness across.

The 25% row-rank reduction is therefore the price the ring structure
charges; the commutator-lattice/ideal-lattice tightness is the dividend
it pays. Both flow from the *same* identity $[a,b] = ab - ba$, which is
exactly why ComSIS is harder than a "naked SIS at $3n^+$ rows" would be:
the structure that compresses the codomain also pins ComSIS solutions to
a useful sublattice of an ideal.

### Security level

ComSIS instances correspond to SIS instances on $3n^+ \times 4n^+ m$
matrices (vs $4n^+ \times 4n^+ m$ for standard module-SIS), due to the
commutator's compression from $4n^+$ to $3n^+$ dimensions. Running the
Lattice Estimator on ML-DSA parameters ($N = 2048$) while varying
dimension from 2048 to 1536 gives estimated bit-security of 288.2 and
209.6, respectively: a drop from NIST category V to category III. This
is the *concrete-security price* paid for the row-rank drop discussed
above; the worst-case-to-average-case reduction continues to bite at
this dimension because the ring-arithmetic dividend (Steps 1-3) is
unaffected.

## Why Nightstream uses $\Phi_{81}$ and whether it can change

### Why $\Phi_{81}$

The choice comes from Neo's concrete parameterization (Neo, Section 6.2).
The constraint is **Module-SIS security over Goldilocks**:

- Goldilocks ($q = 2^{64} - 2^{32} + 1$) has $v_2(q-1) = 32$, so any
  power-of-two cyclotomic $\Phi_{2^k}$ with $k \leq 33$ **splits
  completely** over $\mathbb{F}_q$: $R_q \cong \mathbb{F}_q^d$. An
  attacker then works over individual $\mathbb{F}_q$ components, getting
  only 64 bits of security regardless of $d$ (Neo, Remark 2).
- With $\eta = 81 = 3^4$, $q \bmod 81 = 4 \neq 1$, so $\Phi_{81}$ does
  **not** split over $\mathbb{F}_q$. The ring retains the algebraic
  structure that makes Module-SIS hard. Neo achieves ~128 bits of
  security with $\kappa = 16$, $d = 54$.
- The trade-off: $81 \nmid q - 1$ means NTT is unavailable, so
  Nightstream uses schoolbook multiplication with the `rot_step`
  optimization. This is slower than NTT but acceptable because Neo's
  pay-per-bit property keeps commit cost manageable for binary witnesses.

### Option 1: Almost Goldilocks (AGL) + $\Phi_{128}$

The Neo paper already provides this parameterization (Section 6.1):
$q_{\text{AGL}} = (2^{64} - 2^{32} + 1) - 32$ with $\eta = 128$,
$d = 64$, $\kappa = 13$, ~127 bits of security. Because
$q_{\text{AGL}} \bmod 128 \neq 1$, the ring does not split. This is the
cleanest path for ABBA (even $n = 64$, 25% compression confirmed). The
field arithmetic is nearly identical to Goldilocks (a small tweak to
the Solinas reduction). See the
[fields appendix](../appendix/fields-and-cyclotomics.md) for the
$(q, \eta)$ compatibility analysis.

### Option 2: Goldilocks + non-power-of-two even conductor

The natural idea is to keep Goldilocks and use an even conductor that
avoids the splitting problem. The simplest candidate is
$\eta = 162 = 2 \times 81$. **This does not work**, for a fundamental
reason.

**The field isomorphism.** For any odd $m$, $\gcd(2, m) = 1$, so
$\zeta_{2m} = -\zeta_m$ is a primitive $2m$-th root of unity iff
$\zeta_m$ is a primitive $m$-th root. This means
$\mathbb{Q}(\zeta_{2m}) = \mathbb{Q}(\zeta_m)$ and
$\Phi_{2m}(X) = \Phi_m(-X)$. Concretely:

$$\Phi_{162}(X) = \Phi_{81}(-X) = X^{54} - X^{27} + 1$$

The ring $\mathbb{F}_q[X]/(\Phi_{162})$ is isomorphic to
$\mathbb{F}_q[X]/(\Phi_{81})$ via $X \mapsto -X$. Same degree ($d = 54$),
same security, same everything.

**The parity obstruction persists.** In ABBA's framework, $\eta = 2n$
where $n$ is the conductor of the quaternion component field $K$. For
$\eta = 162$: $n = 81$ (odd). The parity condition $\varphi(2n) = 2\varphi(n)$
fails. $\dim(T_0) = 81 > 54 = \dim(O_L)$. ABBA is still 50% larger
than Ajtai.

Doubling an odd conductor does not escape the parity obstruction because
it does not change the underlying field.

**What about other even conductors?** To get ABBA's 25% compression, we
need $n$ even (i.e. $4 \mid \eta$) AND the ring must not split over
Goldilocks. Candidates:

| $\eta$ | $n = \eta/2$ | $d = \varphi(\eta)$ | $D_K = \varphi(n)$ | $n^+$ | $\dim(T_0)$ | Splits over GL? | ABBA viable? |
|---|---|---|---|---|---|---|---|
| 108 | 54 | 36 | 18 | 9 | 27 | no ($108 \nmid q-1$) | marginal ($n^+ = 9$, weak ComSIS) |
| 156 | 78 | 48 | 24 | 12 | 36 | no | moderate ($n^+ = 12$) |
| 204 | 102 | 64 | 32 | 16 | 48 | no ($204 \nmid q-1$) | good ($n^+ = 16$, same as $\Phi_{128}$) |
| 324 | 162 | 108 | 54 | 27 | 81 | no ($324 \nmid q-1$) | good but $d = 108$ is large |

These all satisfy $4 \mid \eta$ (so $n$ is even) and do not split over
Goldilocks. However, they introduce complications compared to
$\Phi_{128}$:

1. **Non-power-of-two $\Phi$**: the cyclotomic polynomial has more than
   2 terms (e.g. $\Phi_{108} = X^{36} - X^{18} + 1$), so the shift
   matrix $F$ and rot_step are more complex than the simple negacyclic
   case.
2. **No NTT**: since $\eta \nmid q - 1$ for all candidates, ring
   multiplication is schoolbook (same as current $\Phi_{81}$).
3. **Theta re-derivation**: the coefficient permutation implementing
   $\theta(\zeta_n) = \zeta_n^{-1}$ must be recomputed for each new
   $\Phi_n$. For non-power-of-two conductors, the permutation has a
   more complex structure (not just $i \mapsto d - i$).
4. **Security re-estimation**: the Lattice Estimator must be rerun for
   each $(q, \eta, \kappa)$ triple.

The most promising candidate is $\eta = 204$ ($n = 102 = 2 \times 3 \times 17$),
which gives $d = 64$ (same as $\Phi_{128}$), $n^+ = 16$ (same ComSIS
security dimension), and $\dim(T_0) = 48$ (25% compression). It is
essentially the "non-power-of-two analogue" of $\Phi_{128}$ that avoids
Goldilocks splitting. However, the polynomial $\Phi_{204}$ has a complex
multi-term structure, making the arithmetic and theta derivation
significantly harder than $X^{64} + 1$.

### Option 3: stay on Goldilocks + $\Phi_{81}$

Accept that ABBA does not help at this conductor. The current setup
achieves ~128 bits of security with well-optimized Ajtai commitments.
ABBA's value would then be limited to other applications (e.g., blind
signatures, standalone commitments) rather than Nightstream integration.

### Recommendation

**AGL + $\Phi_{128}$ (Option 1) is the best path for ABBA.** It is
already parameterized in the Neo paper, uses the simplest possible
cyclotomic ($X^{64} + 1$), and avoids all the complications of
non-power-of-two conductors. The field change from Goldilocks to AGL is
minimal (a 32-unit shift in the modulus). The porting effort is
dominated by the ring-layer changes (`neo-math`), not the field change.

Option 2 ($\eta = 204$ over Goldilocks) is a fallback if AGL is
unacceptable for ecosystem reasons (e.g., if the proof system must
compose with Goldilocks-native components that cannot tolerate the
modulus change). It gives identical dimensions and security but at the
cost of a harder cyclotomic polynomial.

## Open questions

1. Can the Lyubashevsky-Micciancio one-time signature be adapted to use the
   sum-of-commutators function, yielding ComSIS-based blind signatures?
   (Explored in [blind signatures idea](../ideas/blind-signatures.md).)

2. **Porting Nightstream to $\Phi_{128}$**: the ring layer in `neo-math` is
   currently hardcoded for $\Phi_{81}$ ($D = 54$). Switching to
   $\Phi_{128}$ ($D_L = 64$, $D_K = 32$) would unlock the 25% commitment
   size reduction (confirmed by benchmarks: 768 vs 1,024 Fq at only
   1.4$\times$ speed overhead). The arithmetic is simpler ($X^{64} + 1$ vs
   $X^{54} + X^{27} + 1$), but the change propagates through rot_step,
   the SuperNeo bar transform, parameter selection, and the folding
   pipeline.

3. **Closing the 1.4$\times$ speed gap**: the remaining overhead comes from
   ABBA needing 3 sparse-by-dense operations per column vs Ajtai's 1
   rotation accumulation. Parallelising the $\kappa$-row loop with rayon
   could hide this, since the 16 $\kappa$-rows are independent. On a
   machine with $\geq 16$ cores the commit would be close to Ajtai's
   wall-clock time with 25% smaller output.

4. **Security parameter re-derivation for $\Phi_{128}$**: the commutator
   compression ($4n^+ \to 3n^+$) trades security margin for size. With
   $\Phi_{128}$ ($n^+ = 16$), the ComSIS dimension per row is
   $3 \cdot 16 = 48$ vs Ajtai's $64$. The concrete security level under the
   Lattice Estimator needs to be checked for $\kappa = 16$ at this
   dimension to confirm it still meets the target (NIST category III or
   above).

5. **Tightening the O_K challenge analysis**: the current O_K projection
   $\rho_K = \rho + \theta(\rho)$ doubles the norm and halves the
   challenge entropy. A cleaner approach would sample directly from a
   strong set inside $\mathcal{O}_K$ (i.e., sample $n^+$ coefficients
   in the real-subfield basis and embed), avoiding the norm doubling.
   This requires a dedicated `RotRing` configuration for O_K and a
   re-derivation of the expansion factor $T$ from Neo's Theorem 3.
   Whether the halved challenge space affects the concrete soundness
   level for Nightstream's $\lambda = 127$ target needs explicit
   computation.
