# Fields and Cyclotomic Rings

How to choose a prime field $\mathbb{F}_q$ for a cyclotomic ring
$R_q = \mathbb{F}_q[X]/(\Phi_n)$ in lattice-based cryptography.

## The splitting condition

The cyclotomic polynomial $\Phi_n(X)$ (degree $\varphi(n)$) factors over
$\mathbb{F}_q$ according to the multiplicative order of $q$ modulo $n$:

- **Complete splitting** ($q \equiv 1 \pmod{n}$): $\Phi_n$ splits into
  $\varphi(n)$ linear factors. This enables NTT, CRT decomposition, and
  the quaternion isomorphism $\Lambda_q \cong \prod M_2(\mathbb{F}_q)$.
- **Partial splitting** ($\text{ord}_n(q) = f > 1$): $\Phi_n$ factors
  into $\varphi(n)/f$ irreducible polynomials of degree $f$. NTT is not
  directly available; polynomial multiplication uses schoolbook
  $O(\varphi(n)^2)$ or Karatsuba.
- **Inert** ($f = \varphi(n)$): $\Phi_n$ is irreducible over
  $\mathbb{F}_q$. The ring $R_q \cong \mathbb{F}_{q^{\varphi(n)}}$ is a
  field extension.

### Why splitting matters

| Application | Needs complete splitting? | Why |
|---|---|---|
| NTT-based polynomial multiplication | **yes** | NTT evaluates at $n$-th roots of unity, which exist iff $n \mid q-1$ |
| Schoolbook ring multiplication | no | Just polynomial arithmetic mod $\Phi_n$, any $q$ works |
| ABBA quaternion CRT ($\Lambda_q \cong \prod M_2(\mathbb{F}_q)$) | **yes** (in $O_K$) | Wedderburn decomposition requires $q$ completely split in $O_K$ |
| Ajtai binding (MSIS hardness) | no | Hardness holds for any $q$, but NTT makes it practical |

Nightstream uses $\Phi_{81}$ with Goldilocks, where $q \bmod 81 = 4$
(NOT completely split). This is deliberate: Nightstream uses schoolbook
multiplication with the `rot_step` optimization, avoiding NTT entirely.

### The 2-adic valuation

For power-of-two cyclotomics $\Phi_{2^k}(X) = X^{2^{k-1}} + 1$, the
splitting condition becomes $2^k \mid q - 1$. The key parameter is the
**2-adic valuation** $v_2(q-1)$: the largest power of 2 dividing $q-1$.
This determines the largest power-of-two NTT the field supports.

## Common fields

| Field | $q$ | Bits | $v_2(q-1)$ | Max NTT size | Used by |
|---|---|---|---|---|---|
| Goldilocks | $2^{64} - 2^{32} + 1$ | 64 | 32 | $2^{32}$ | Plonky2, Nightstream |
| Baby Bear | $2^{31} - 2^{27} + 1$ | 31 | 27 | $2^{27}$ | Plonky3, SP1, RISC Zero |
| Koala Bear | $2^{31} - 2^{24} + 1$ | 31 | 24 | $2^{24}$ | Plonky3 (alt) |
| Mersenne31 | $2^{31} - 1$ | 31 | 1 | $2^1$ | Circle STARKs (no NTT!) |
| BN254 scalar | $\approx 2^{254}$ | 254 | 28 | $2^{28}$ | Ethereum, Groth16 |
| BLS12-381 scalar | $\approx 2^{255}$ | 255 | 32 | $2^{32}$ | Zcash |

### Cyclotomic compatibility matrix

For $\Phi_{2^k}$ (power-of-two cyclotomics), complete splitting requires
$2^k \mid q - 1$, i.e. $k \leq v_2(q-1)$:

| Conductor $\eta = 2^k$ | $D = 2^{k-1}$ | Goldilocks ($v_2 = 32$) | Baby Bear ($v_2 = 27$) | Mersenne31 ($v_2 = 1$) |
|---|---|---|---|---|
| 4 | 2 | complete | complete | **inert** |
| 8 | 4 | complete | complete | **inert** |
| 64 | 32 | complete | complete | **inert** |
| 128 | 64 | complete | complete | **inert** |
| 256 | 128 | complete | complete | **inert** |
| $2^{28}$ | $2^{27}$ | complete | complete | **inert** |
| $2^{33}$ | $2^{32}$ | complete | **partial** | **inert** |

Mersenne31 ($q = 2^{31} - 1$) is the outlier: $q - 1 = 2(2^{30} - 1)$
with $v_2 = 1$. It cannot support any power-of-two cyclotomic beyond
$\Phi_2$ (trivial). This is why Mersenne31-based proof systems (Circle
STARKs) use the **circle group** $\{(x,y) : x^2 + y^2 = 1\}$ over
$\mathbb{F}_q$ instead of cyclotomic rings.

### Non-power-of-two cyclotomics

For odd-prime-power conductors like $\eta = 81 = 3^4$ ($\Phi_{81}$,
degree 54), complete splitting requires $q \equiv 1 \pmod{81}$:

- Goldilocks: $q \bmod 81 = 4$ (**not split**; Nightstream uses
  schoolbook)
- Baby Bear: $q \bmod 81 = 59$ (**not split**)
- BN254 scalar: can check, but irrelevant for SNARK applications

Complete splitting is not required for correctness of Ajtai/ABBA
commitments, only for NTT-accelerated arithmetic. Nightstream's choice
of $\Phi_{81}$ prioritises the S-action algebraic structure over NTT
compatibility.

## Goldilocks vs Baby Bear for ABBA

### Arithmetic cost

Baby Bear is a 31-bit prime; Goldilocks is 64-bit. On modern CPUs:

- Baby Bear addition/subtraction: 1 cycle (native 32-bit)
- Goldilocks addition/subtraction: 1-2 cycles (native 64-bit)
- Baby Bear multiplication: 1 `imul` + reduction (2-3 cycles)
- Goldilocks multiplication: 1 `imul` + Goldilocks-specific reduction
  (3-4 cycles)

For $\Phi_{128}$ with $D_K = 32$: each ring operation touches 32 field
elements. Baby Bear processes these ~1.5$\times$ faster per element,
giving a system-level speedup of roughly 1.5$\times$ for the same
algorithmic complexity.

### ABBA benchmarks: Goldilocks vs Baby Bear at $\Phi_{128}$

Simulated folding (commit + s_mul + accumulate), $\kappa = 16$, 50%
density:

| Config | Goldilocks | Baby Bear | Speedup |
|---|---|---|---|
| 2 $\times$ 16 (1,024 bits) | 2.58 ms | 1.29 ms | 2.0$\times$ |
| 5 $\times$ 16 (1,024 bits) | 6.31 ms | 2.91 ms | 2.2$\times$ |
| 10 $\times$ 16 (1,024 bits) | 13.14 ms | 6.08 ms | 2.2$\times$ |
| 5 $\times$ 32 (2,048 bits) | 13.11 ms | 5.99 ms | 2.2$\times$ |

Baby Bear is consistently **2$\times$ faster** for the same ABBA
workload, as expected from the field-size ratio.

### ABBA/Ajtai overhead: consistent across fields

| Field | ABBA/Ajtai ratio (folding) | Commitment size |
|---|---|---|
| Goldilocks | 1.06-1.38$\times$ | 768 vs 1,024 (25% smaller) |
| Baby Bear | 1.40-1.63$\times$ | 768 vs 1,024 (25% smaller) |

The overhead ratio is similar in both fields (~1.4$\times$), confirming
that the ABBA construction is field-independent: the 3-vs-1
sparse-operations-per-column ratio does not depend on the field size.

### When to use which

| Criterion | Goldilocks | Baby Bear |
|---|---|---|
| Security bits per element | ~64 | ~31 |
| Arithmetic speed | baseline | ~2$\times$ faster |
| Nightstream compatibility | native ($\Phi_{81}$, $D = 54$) | needs porting |
| ABBA $\Phi_{128}$ folding | 2.6-13.1 ms | 1.3-6.0 ms |
| Recursive proof depth | better (larger field, fewer repetitions) | needs extension field for security |
| Hardware (GPU/ASIC) | 64-bit word | 32-bit word (cheaper) |

**Goldilocks** is the right choice when Nightstream compatibility
matters (since everything is hardcoded for $D = 54$) or when the proof
system needs high per-element security (e.g., recursive SNARKs where
each recursion step amplifies the soundness error).

**Baby Bear** is the right choice for throughput-critical applications
where the 2$\times$ arithmetic speedup matters more than per-element
security margin, and where the proof system already handles the 31-bit
field (e.g., SP1, RISC Zero, Plonky3).

For ABBA specifically, both fields confirm the 25% commitment size
reduction at $\Phi_{128}$ with moderate overhead (~1.4$\times$). The
algebraic properties (theta involution, tracelessness, sparse
commutator, linearity) are field-independent and verified by tests
over both fields.

## Independence of field and cyclotomic ring

The choice of field $\mathbb{F}_q$ and cyclotomic conductor $\eta$
(defining $R_q = \mathbb{F}_q[X]/(\Phi_\eta)$) are **algebraically
independent** but **practically coupled**.

### What is independent

Three properties of ABBA depend only on the conductor $\eta$, not on
the field $\mathbb{F}_q$:

1. **The 25% compression ratio.** This comes from
   $\dim(T_0)/\dim(\Lambda) = 3/4$, a consequence of the eigenspace
   decomposition of the theta automorphism on $\mathcal{O}_K$. It
   depends on the Galois structure of $\mathbb{Q}(\zeta_n)$ and on the
   [parity of $n$](../papers/abba.md#the-parity-obstruction), not on $q$.
2. **The ABBA/Ajtai overhead ratio** (~1.4$\times$). This comes from the
   3-vs-1 sparse-operations-per-column structure of the commutator. The
   ratio is the same over Goldilocks and Baby Bear because the operation
   count is identical; only the cost per operation changes.
3. **The algebraic identities** (theta is an involution, commutators are
   traceless, bilinearity). These hold over any $\mathbb{F}_q$.

Conversely, **arithmetic speed** depends only on $q$ (field size), not
on $\eta$: Baby Bear is ~2$\times$ faster than Goldilocks regardless
of the cyclotomic polynomial.

### What is coupled

Despite the algebraic independence, three practical constraints link the
choice of $q$ and $\eta$:

1. **NTT compatibility.** NTT-based ring multiplication requires $\eta$-th
   roots of unity in $\mathbb{F}_q$, i.e. $\eta \mid q - 1$ (complete
   splitting). If this fails, ring multiplication falls back to schoolbook
   $O(d^2)$ or Karatsuba. The
   [compatibility matrix](#cyclotomic-compatibility-matrix) above captures
   this constraint.

2. **Security level.** Module-SIS/ComSIS hardness depends on both $q$ and
   $d = \varphi(\eta)$ together. The lattice dimension seen by an
   attacker is $n_{\text{rows}} \cdot d$ over $\mathbb{Z}_q$. A smaller
   $q$ (fewer bits per element) reduces the lattice dimension in bits,
   requiring either larger $d$ or more rows $\kappa$ to maintain the same
   security level.

3. **Extension-field soundness.** If the proof system samples challenges
   from $\mathbb{F}_{q^k}$ with $q^k \geq 2^{128}$, a smaller $q$
   forces a larger extension degree $k$, which increases proof size
   (each challenge is $k$ field elements) and verifier cost.

### Ideal $(q, \eta)$ pairs for ABBA

| Pair | ABBA viable? | Notes |
|---|---|---|
| Goldilocks + $\Phi_{128}$ | **best** | Even $n = 64$: 25% compression confirmed. $128 \mid q-1$ ($v_2 = 32$): NTT available. 64-bit field: strong per-element security. Already benchmarked (1.4$\times$ overhead). |
| Baby Bear + $\Phi_{128}$ | **good** | Same 25% compression. $128 \mid q-1$ ($v_2 = 27$): NTT available. 2$\times$ faster arithmetic. Needs extension field ($k = 5$) for 128-bit soundness. Benchmarked (1.4-1.6$\times$ overhead). |
| Goldilocks + $\Phi_{256}$ | **good** | Even $n = 128$: 25% compression holds. $256 \mid q-1$: NTT available. Larger $D_K = 64$ means more work per operation but fewer rows $\kappa$ needed for the same security. Not yet benchmarked. |
| Baby Bear + $\Phi_{256}$ | **good** | Same as above but faster per element. $256 \mid q-1$: NTT available. |
| Goldilocks + $\Phi_{81}$ | **bad** | Odd $n = 81$: parity obstruction, ABBA is 50% *larger* than Ajtai. $81 \nmid q-1$: no NTT. This is Nightstream's current configuration; ABBA does not help here. |
| Baby Bear + $\Phi_{81}$ | **bad** | Same parity obstruction. $81 \nmid q-1$: no NTT. |
| Mersenne31 + $\Phi_{128}$ | **problematic** | Even $n$: 25% compression holds algebraically. But $v_2(q-1) = 1$: no NTT for any power-of-two cyclotomic beyond $\Phi_2$. Ring multiplication is $O(d^2)$ schoolbook, which is prohibitively slow for $d = 64$. |
| Goldilocks + $\Phi_{64}$ | **good** | Even $n = 32$: compression holds. $D_K = 16$, $T_0 = 24$: very small ring, fast operations. But $n^+ = 8$, so ComSIS security per row is low; needs more rows $\kappa$. NTT available ($64 \mid q-1$). |
| BN254 scalar + $\Phi_{128}$ | **poor fit** | Even $n$: compression holds. But 254-bit field eliminates the small-field advantage that motivates lattice commitments. ABBA's pay-per-bit is irrelevant when each field element is already 254 bits. |

### Selection criteria

A good $(q, \eta)$ pair for ABBA should satisfy:

1. **$n$ is even** (so $\eta$ is a power of two, or more generally
   $4 \mid \eta$). This is the hard requirement for the 25% compression.
2. **$\eta \mid q - 1$** (NTT compatibility). Without this, ring
   multiplication is too slow for practical commitment schemes.
3. **$q$ fits in a machine word** (31--64 bits). This is the small-field
   advantage that makes lattice commitments competitive with
   hash-based ones.
4. **$D_K = \varphi(\eta)/2 \geq 16$** (sufficient ComSIS security per
   row). Smaller $D_K$ means the ComSIS lattice dimension $3n^+$ is too
   small for 128-bit security without an impractical number of rows.
5. **$q^k \geq 2^{128}$ for moderate $k$** (extension-field soundness
   without excessive proof blowup). Goldilocks ($k = 2$) is better than
   Baby Bear ($k \geq 5$) on this axis.

The sweet spot is **Goldilocks + $\Phi_{128}$**: it satisfies all five
criteria with $D_K = 32$, $n^+ = 16$, NTT available, and $k = 2$ for
128-bit soundness. Baby Bear + $\Phi_{128}$ is the throughput-optimised
alternative, trading extension-field cost for 2$\times$ faster
arithmetic.
