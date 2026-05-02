# Efficient Recursion for the Jolt zkVM

| | |
|---|---|
| **Authors** | Quang Dao, Sagar Dhawan, Markos Georghiades, Justin Thaler, Andrew Tretyakov, Michael Zhu |
| **Affiliations** | Carnegie Mellon University, a16z crypto research, Georgetown University |
| **Status** | Preprint (no eprint number) |

## Motivation

Recursion (proving knowledge of proofs) is central to many zkVM applications: proof aggregation, succinct blockchains, incremental verification, and "proofs of long computations" built from smaller chunks. For Jolt, recursion appears in two forms:

1. **Simple (self-)recursion.** The Jolt verifier runs inside the zkVM itself.
2. **Wrapping.** The verifier is compiled to R1CS and proved in a second SNARK, typically to reduce proof size for on-chain verification.

Both require the verifier to be cheap when executed inside a SNARK or a VM.

### The Bottleneck: Field Mismatch

Jolt uses [Dory](../pcs/dory.md) for polynomial commitments. A modern SNARK verifier has two components:

- **Information-theoretic component.** Checks sum-check rounds and evaluation claims. This is pure [native field arithmetic](../fundamentals/field-arithmetic.md) ($\mathbb{F}_r$ operations), and is already cheap.

- **Cryptographic component.** Checks the Dory evaluation proof: scalar multiplications in $G_1$, $G_2$, exponentiations in $G_T$ (a degree-12 extension of the base field), and a final multi-pairing.

The cryptographic component is the problem. The Dory verifier performs group operations whose coordinates are in $\mathbb{F}_q$ (BN254's base field), but the Jolt SNARK is native to $\mathbb{F}_r$ (BN254's scalar field). Since $q \ne r$, every $\mathbb{F}_q$ operation must be emulated non-natively, costing hundreds of constraints each. For $G_T$ operations in $\mathbb{F}_{q^{12}}$, the situation is roughly $12^2 = 144$ times worse again. The result: **1-2 billion VM cycles** when compiled to RISC-V, or an exploding constraint count when compiled to R1CS.

## Core Idea: Why Two Proofs?

The central idea is to split verification across two proofs, each operating over a different field, so that **no non-native arithmetic appears anywhere**.

### $\pi$ (base Jolt proof, over $\mathbb{F}_r$)

Proves correct execution of the program (sum-check, Lasso lookups, memory checking) and the information-theoretic component of Dory verification. Everything here is $\mathbb{F}_r$ arithmetic.

### $\pi'$ (auxiliary proof, over $\mathbb{F}_q$ via Hyrax on Grumpkin)

Proves the **cryptographic component** of Dory verification: the $G_T$ exponentiations, $G_1$/$G_2$ scalar multiplications, and the multi-pairing. These operations have coordinates in $\mathbb{F}_q$. By using [Hyrax](../pcs/hyrax.md) over Grumpkin (whose scalar field is $\mathbb{F}_q$), they become **native arithmetic**. No non-native emulation.

### Why not a single proof?

Because the two components live in different fields. Forcing Dory verification into the $\mathbb{F}_r$-native SNARK means non-native emulation. Forcing sum-check verification into an $\mathbb{F}_q$-native SNARK would have the same problem in reverse. The [curve cycle](../fundamentals/field-arithmetic.md#curve-cycles-and-why-they-eliminate-non-native-arithmetic) (BN254/Grumpkin) lets each proof operate over its natural field.

### What the verifier of $(\pi, \pi')$ does

After receiving $(\pi, \pi')$, the verifier:

1. **Checks $\pi$'s information-theoretic component directly.** Sum-check round messages, evaluation claims. This is pure $\mathbb{F}_r$ arithmetic.
2. **Verifies $\pi'$** to confirm that Dory's cryptographic component would have accepted. The $\pi'$ verifier performs Grumpkin $G_1$ operations (two [Hyrax](../pcs/hyrax.md) MSMs), whose coordinates are in $\mathbb{F}_r$, so this is also native.

A **Schwartz-Zippel random check** reduces target-group and extension-field relations to base-field identities, ensuring all remaining checks are native-field multiplications.

### The verifier is not a SNARK

The verifier of $(\pi, \pi')$ is not itself a SNARK. It is just a computation. What happens to it depends on the application:

- **Simple recursion:** the verifier runs as a RISC-V guest program inside the next Jolt instance. The cycle counts in the benchmarks (e.g., ~177M cycles at $2^{20}$) are RISC-V instructions, not SNARK constraints.
- **Wrapping:** the verifier is compiled to an R1CS constraint system over $\mathbb{F}_r$, and a wrapper SNARK (Groth16 or Spartan + HyperKZG) proves it is satisfied. The paper's "under 10 million constraints" target refers to this R1CS.

In both cases, the proof structure is two layers, not three:

```
π   ← Jolt SNARK (over F_r, with Dory PCS)
π'  ← auxiliary SNARK (over F_q, with Hyrax PCS on Grumpkin)

Verifier of (π, π') ← not a SNARK; just a computation
  ├─ run inside the zkVM  → simple recursion
  └─ compile to R1CS      → wrapping
```

### Why not two running recursive instances?

An alternative approach (used in some other systems) maintains two parallel recursive accumulators, one per curve. The auxiliary-proof approach is simpler: $\pi'$ is a one-shot proof appended to $\pi$, sharing the same Fiat-Shamir transcript. There is no need to track two recursive states across multiple iterations.

## Techniques

### 1. PIOPs for Elliptic Curve Arithmetic

Dedicated sum-check PIOPs prove the EC operations that dominate Dory verification. Full details are in [Group Arithmetic](../fundamentals/group-arithmetic.md); here is a summary.

#### $G_T$ Multiplication

[Elements of $G_T$ are represented as 4-variate MLEs](../fundamentals/group-arithmetic.md#g_t-elements-4-variate-mles) over $x \in \{0,1\}^4$, encoding 12 $\mathbb{F}_q$ coefficients padded to 16. The [polynomial division identity](../fundamentals/field-arithmetic.md#the-polynomial-division-identity) reduces extension-field multiplication to a base-field zero-check:

$$C(x) := \tilde{a}(x) \cdot \tilde{b}(x) - \tilde{c}(x) - \tilde{q}(x) \cdot \tilde{p}(x) = 0, \quad \forall x \in \{0,1\}^4.$$

Soundness error: $16/|\mathbb{F}_q|$.

#### $G_T$ Exponentiation

To prove $h = g^\alpha$, decompose $\alpha$ into base-$b$ digits and compute $g^\alpha$ via the recurrence $\rho_{s+1} = \rho_s^b \cdot g^{d_s}$. The step constraint is verified via a zero-check over $(m+4)$ variables. The shifted accumulator $\tilde{\rho}_{\text{shift}}(s,x) := \tilde{\rho}(s+1,x)$ is a [virtual polynomial](../fundamentals/virtual-polynomials.md) (not committed), handled via a **shift sum-check** using [EqPlusOne](../fundamentals/mle.md#eqplusone):

$$\tilde{\rho}_{\text{shift}}(r_s, r_x) = \sum_{(s,x) \in \{0,1\}^{m+4}} \text{EqPlusOne}(r_s, s) \cdot \widetilde{\text{eq}}(r_x, x) \cdot \tilde{\rho}(s, x).$$

The digit selector $\tilde{b}(s,x)$ is also virtual.

Concrete sizing with base $b = 4$: 127 steps, $m_{G_T} = 7$ step variables, 11 total variables.

#### $G_1$ and $G_2$ Arithmetic

Point addition and scalar multiplication follow the standard affine Weierstrass group law. $G_2$ arithmetic mirrors $G_1$ but over $\mathbb{F}_{q^2} = \mathbb{F}_q[u]/(u^2+1)$, doubling the number of coordinate constraints.

See [Group Arithmetic](../fundamentals/group-arithmetic.md) for the full constraint listings, witness counts, and soundness proofs.

#### Batching

All instances of the same operation type (up to 256 for large traces) are batched into a single zero-check by stacking over a constraint-index domain $c \in \{0,1\}^k$. Claims are per operation *type*, not per instance.

### 2. Copy Constraints via Sum-Check

The [PIOPs above prove individual operations in isolation](../fundamentals/group-arithmetic.md#composing-piops-copy-constraints). To prove the full Dory verifier, their inputs and outputs must be wired together. For example, in one reduce-and-fold round:

```
[Exp PIOP]    D₂, βᵢ   →  D₂^βᵢ      =: t₁
[Mul PIOP]    m₁, t₁   →  m₁·t₁      =: m₂
```

The copy constraint enforces that the exponentiation's output polynomial $\tilde{\rho}(\text{final})$ equals the multiplication's input polynomial $\tilde{b}(\text{input})$.

A single wiring sum-check per group ($G_T$, $G_1$, $G_2$) proves all ~1,000-2,000 edge equalities at once, using random edge-batching challenges $\lambda_e$. No accumulator polynomial is needed (unlike PLONK-style permutation arguments), because $E$ is small enough for the verifier to pay $O(E)$ directly.

### 3. Prefix Packing

[Prefix packing](../fundamentals/prefix-packing.md) reduces all $c$ evaluation claims on committed polynomials of varying sizes to a single dense polynomial opening. Each polynomial is assigned a unique binary prefix forming a prefix-free code, and they are packed into disjoint subcubes of a single $n$-variable hypercube.

For $\sigma = 19$: all committed polynomials pack into a single 21-variable dense polynomial ($\sim$2.1M coefficients). The packed size is at most 2x the total size of the individual polynomials.

## The Three Stages of $\pi'$

All three stages belong to $\pi'$ (the auxiliary proof). The base proof $\pi$ is already done at this point; what remains is proving that Dory's cryptographic component was performed correctly.

| Stage | What it proves | Rounds | Degree |
|---|---|---|---|
| **Stage 1** | Packed $G_T$ exponentiation zero-check | 11 | 8 |
| **Stage 2** | Everything else batched: $G_T$ multiplication, $G_1$/$G_2$ scalar mul and addition, shift consistency, copy constraints | ~25 | $\le$ 8 |
| **Stage 3** | Prefix packing: fresh $r_{\text{pack}}$, weighted claim folding, one dense PCS opening | N/A (not a sum-check) | N/A |

**Why Stage 1 is separate.** The $G_T$ exponentiation zero-check uses different variables ($m_{G_T} + 4 = 11$) and a different challenge point than the remaining constraints, so it runs as its own sum-check.

**Stage 2 batches everything else** into a single sum-check sharing one challenge point: claim reductions, shift consistency, $G_T$ multiplication, $G_1$/$G_2$ EC operations, and copy constraints. This is where the bulk of the PIOP work happens (~$2^{28}$ operations, dominating Stage 1's ~$2^{14}$).

**Stage 3 is not a sum-check.** It is the prefix-packing reduction that produces one dense polynomial opening, verified by a single Hyrax opening proof (two MSMs over Grumpkin).

### Commitment

All witness polynomials are family-packed (extending each polynomial's domain by a constraint-index suffix) and then [prefix-packed](../fundamentals/prefix-packing.md) into a single 21-variable dense polynomial. The prover commits via [Hyrax](../pcs/hyrax.md) over Grumpkin: a vector of $2^{n_{\text{pack}}/2}$ Pedersen commitments, each computed as an MSM of size $2^{n_{\text{pack}}/2}$.

This is the only cryptographic operation in the extended prover; all other prover work is pure field arithmetic.

### Evaluation Proof

The Hyrax opening is a [vector-matrix-vector product](../pcs/dory.md#the-vmv-vector-matrix-vector-structure). The verifier performs two MSMs over Grumpkin of size $2^{n_{\text{pack}}/2}$. This is the **dominant verifier cost**, accounting for ~62% of total cycles. Since Grumpkin $G_1$ coordinates are in $\mathbb{F}_r$, this check is native to the Jolt SNARK when running inside the VM for recursion.

## Results

### Simple Recursion (M4 Max, 16 cores, 64GB RAM)

| Trace length | Standard verifier (ms) | Extended verifier (ms) | Standard cycles | Extended cycles | Speedup |
|---|---|---|---|---|---|
| $2^{16}$ | 60 | 9 | 1,414M | 171M | 8.1x |
| $2^{18}$ | 81 | 9 | 1,499M | 174M | 8.7x |
| $2^{20}$ | 87 | 10 | 1,587M | 177M | 9.0x |
| $2^{22}$ | 91 | 12 | 1,695M | 182M | 9.3x |
| $2^{24}$ | 92 | 12 | 1,784M | 191M | 9.4x |
| $2^{26}$ | 120 | 10 | 1,879M | 198M | 9.5x |

### Prover Overhead

The recursion step adds only **1.5-1.9s** across all trace lengths, decomposed as:
- Hyrax commitment over Grumpkin: ~1s
- Three sum-check stages: ~0.4s
- Witness generation: ~0.3s
- Opening proof: < 5ms

The recursion prover's cost is nearly independent of trace length because all three phases depend on $\sigma = O(\log T)$, not on $T$ itself.

### Cycle Budget Breakdown (at $2^{20}$)

| Component | Cycles | Share |
|---|---|---|
| Base stages 1-7 (checking $\pi$) | ~24M | 13% |
| Recursion stage-8 preparation (deriving constraint system from Dory hints) | ~6M | 4% |
| Recursion SNARK verification, stages 1-3 (checking $\pi'$'s sum-checks) | ~17M | 10% |
| **Hyrax PCS opening check** (checking $\pi'$'s commitment, two Grumpkin MSMs) | **~109M** | **62%** |
| External pairing check | ~16M | 9% |

The Hyrax cost is nearly independent of trace length because the packed polynomial size ($n_{\text{pack}} \approx 20$ variables) grows only with $\sigma = O(\log T)$, not with $T$.

### Proof Sizes

| Trace length | Base proof $\pi$ | Recursion artifact $\pi'$ | Combined |
|---|---|---|---|
| $2^{16}$ | 79 KB | 135 KB | 214 KB |
| $2^{20}$ | 87 KB | 136 KB | 223 KB |
| $2^{26}$ | 93 KB | 136 KB | 229 KB |

The base proof grows logarithmically with trace length (driven by the Dory opening proof). The recursion artifact is essentially constant for $\sigma \ge 9$ (trace lengths $\ge 2^{18}$).

### Wrapping

| Wrapper | Proof size | Expected proving time | Setup |
|---|---|---|---|
| Groth16 | ~128 bytes | ~10s | Per-circuit trusted setup |
| Spartan + [HyperKZG](../pcs/hyperkzg.md) | ~5 KB | < 1s | Universal SRS |

The extended verifier's R1CS representation targets **under 10 million constraints**, roughly an order of magnitude fewer than wrappers for hash-based zkVMs, largely because non-native field arithmetic is eliminated.

## Key Takeaways

1. **Two proofs, two fields, no non-native arithmetic.** The field mismatch between $\mathbb{F}_r$ (SNARK) and $\mathbb{F}_q$ (Dory coordinates) is resolved by splitting into $\pi$ and $\pi'$, each operating over its natural field via the BN254/Grumpkin curve cycle.

2. **Make $\pi$ larger to make the verifier faster.** Appending $\pi'$ increases total proof size but dramatically reduces what the recursive verifier must do: from billions of cycles (non-native $G_T$ arithmetic) to under 200M cycles (native Grumpkin MSMs).

3. **The verifier of $(\pi, \pi')$ is not a SNARK.** It is a computation that is either run inside the zkVM (simple recursion) or compiled to R1CS (wrapping). The two-layer structure ($\pi$, $\pi'$, then a verifier) avoids the complexity of maintaining two running recursive accumulators.

4. **Minimise committed data to minimise verifier cost.** [Prefix packing](../fundamentals/prefix-packing.md), [virtual polynomials](../fundamentals/virtual-polynomials.md), and sum-check-based wiring were originally developed to make provers fast. Here they are repurposed to make the *verifier* fast, since the [Hyrax](../pcs/hyrax.md) verifier cost grows with $\sqrt{n_{\text{pack}}}$.

5. **The approach generalises.** The same idea (proving expensive verification steps in an auxiliary SNARK over a cycle curve) applies beyond Dory, including SNARKs using lattice-based PCS or algebraic hash functions. It also ports directly to higher-security curves (BLS12-381/Bandersnatch).

6. **No folding needed.** The direct recursion approach avoids folding schemes, yielding a simpler architecture. Folding could further optimise some steps but is not necessary.
