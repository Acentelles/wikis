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

The paper's 25% claim compares $T_0$ to $\Lambda$:
$\dim(T_0) / \dim(\Lambda) = \frac{3n^+}{4n^+} = 3/4$.

For this to also be a reduction compared to Ajtai (which uses $O_L$, not
$\Lambda$), we need $\dim(O_L) = \dim(\Lambda)$, i.e.
$D_L = 2 D_K$.

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

### Security level

ComSIS instances correspond to SIS instances on $3n^+ \times 4n^+ m$
matrices (vs $4n^+ \times 4n^+ m$ for standard module-SIS), due to the
commutator's compression from $4n^+$ to $3n^+$ dimensions. Running the
Lattice Estimator on ML-DSA parameters ($N = 2048$) while varying
dimension from 2048 to 1536 gives estimated bit-security of 288.2 and
209.6, respectively: a drop from NIST category V to category III.

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
