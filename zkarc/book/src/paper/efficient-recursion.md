# Efficient recursion for proof aggregation

This page applies the techniques from "Efficient Recursion for the Jolt
zkVM" (Dao, Dhawan, Georghiades, Thaler, Tretyakov, Zhu, 2026) to the
zkARc proof aggregation problem. The paper solves the core bottleneck that
makes Jolt-to-Jolt recursion expensive; the same solution unblocks
Jolt-to-Jolt-Atlas aggregation in zkARc.

## The recursion bottleneck

The zkARc proof stack produces proofs from two systems:

- **Jolt Atlas** (Dory or HyperKZG PCS) for model inference proofs
  ($\pi_{\mathsf{PMC}}$, $\pi_{\mathsf{AV}}$, $\pi_{\mathsf{Provenance}}$).
- **Jolt** (Dory PCS) for the SMT solver proof ($\pi_{\mathsf{SMTsolver}}$).

To aggregate these into a single receipt, we need to verify one proof
inside another. The naive approach (run the Jolt verifier as a RISC-V
guest program inside Jolt) is bottlenecked by **Dory evaluation proof
verification**, which requires exponentiations in the pairing target group
$G_T$ (a degree-12 extension of the base field). This translates to
billions of VM cycles and massive non-native field arithmetic.

The efficient recursion paper eliminates this bottleneck.

## The key idea: auxiliary proof $\pi'$

Instead of making the recursive verifier perform the heavy pairing-related
computation, make the prover prove it was done correctly.

The Jolt proof $\pi$ is extended with an auxiliary proof $\pi'$ that
certifies the expensive Dory checks. $\pi'$ is carried out over a second
curve (Grumpkin for BN254) whose scalar field is the base field of the
pairing target group. The resulting extended verifier consists of:

- A few million native-field (BN254 scalar field) multiplications.
- Almost no non-native arithmetic.
- Almost no target-group operations.

This is an order of magnitude cheaper than the naive approach.

## Concrete numbers

From the paper's benchmarks (Apple M4 Max):

| Metric | Standard Jolt | Extended Jolt | Improvement |
|--------|--------------|---------------|-------------|
| Verifier cycles (trace $2^{20}$) | 1,587M | 177M | 9x |
| Verifier time (trace $2^{20}$) | 87 ms | 10 ms | 8.7x |
| Recursion prover overhead | N/A | 1.5-1.9s | (constant across trace lengths) |
| Combined proof size | 87 KB | 223 KB | (base + recursion artifact) |

The recursion prover overhead is **constant** (1.5-1.9s) regardless of
trace length, because $\pi'$ depends on $\sigma = O(\log T)$, not on $T$
itself.

## Three techniques that make $\pi'$ efficient

### 1. PIOPs for elliptic-curve arithmetic

The paper develops specialized Polynomial Interactive Oracle Proofs for
the EC operations that dominate Dory verification:

- **$G_T$ exponentiation**: a base-$b$ representation reduces the number of
  steps from $\lceil \log_2 r \rceil$ to $\lceil \log_b r \rceil$.
  Batching across all $G_T$ operation instances (up to 256 for large
  traces) runs a single zero-check over a stacked constraint index,
  saving per-instance claims.
- **$G_1$ scalar multiplication**: double-and-add encoded as a zero-check
  with a shift sum-check for the running accumulator.
- **$G_2$ arithmetic**: same structure as $G_1$ but with coordinates in
  $\mathbb{F}_{q^2}$, splitting every coordinate MLE into two base-field
  polynomials.

All PIOPs have soundness error polynomial in $m = \lceil \log_2 n \rceil$
and inverse-polynomial in $|\mathbb{F}_q|$.

### 2. Copy constraints via sum-check

When composing the EC PIOPs (the output of one operation must equal the
input of the next), the paper uses a random-linear-combination
permutation check directly in the sum-check, avoiding the need to commit
an accumulator polynomial.

This works because the number of wiring edges $E$ is small (~1-2
thousand): the verifier pays $O(E)$ directly, which is cheaper than the
cost of committing and verifying an accumulator polynomial inside the
recursion.

### 3. Prefix packing

The witness polynomials for the EC PIOPs are packed using prefix codes so
that the total committed data stays under $2^{21}$ coefficients for all
trace lengths. Since the dominant recursion verifier cost is the Hyrax
opening check (which scales with $\sqrt{\text{committed data}}$),
minimizing committed data minimizes verification time.

## Application to zkARc: how aggregation works

### Step 1: Each component proof produces an extended proof

Each Jolt or Jolt Atlas proof in the zkARc stack is extended with its own
$\pi'$:

| Proof | System | Extension |
|-------|--------|-----------|
| $\pi_{\mathsf{PMC}}$ | Jolt Atlas (HyperKZG) | $\pi'_{\mathsf{PMC}}$ over Grumpkin (if using Dory path) or native (if HyperKZG) |
| $\pi_{\mathsf{AV}}$ | Jolt Atlas (HyperKZG) | $\pi'_{\mathsf{AV}}$ (same) |
| $\pi_{\mathsf{SMTsolver}}$ | Jolt (Dory) | $\pi'_{\mathsf{SMTsolver}}$ over Grumpkin |

For Jolt Atlas with HyperKZG, the recursion bottleneck is different (a
small number of pairing checks rather than Dory's logarithmic inner-product
argument), but the same technique applies: prove the pairing check in
$\pi'$ and leave the verifier with native-field arithmetic.

### Step 2: Verify the extended proofs recursively

The extended verifier for each component is now cheap: a few million
native-field multiplications. This verifier can be:

- **Compiled to R1CS** (~10M constraints) and proved by a Spartan +
  HyperKZG wrapper in under a second.
- **Run as a RISC-V guest** inside Jolt (~177M cycles, provable in a
  few seconds).

For zkARc aggregation, we verify each component's extended proof inside
a single aggregation circuit:

```
π_PMC extended  ──→  Verify(π_PMC, π'_PMC)  ──→  R1CS instance
π_AV extended   ──→  Verify(π_AV, π'_AV)    ──→  R1CS instance   ──→  Fold  ──→  Spartan  ──→  π_agg
π_solver ext.   ──→  Verify(π_solver, π'_s) ──→  R1CS instance
```

### Step 3: Cross-proof data flow

The copy-constraint technique from the paper (Section 3.2) directly solves
the cross-proof binding problem:

- The output of $\pi_{\mathsf{PMC}}$ ($P_{\mathsf{SMT}}$) must equal the
  policy input of $\pi_{\mathsf{SMTsolver}}$.
- The output of $\pi_{\mathsf{AV}}$ ($C_{\mathsf{SMT}}$) must equal the
  claims input of $\pi_{\mathsf{SMTsolver}}$.

These are wiring edges between the R1CS instances of adjacent component
verifiers. Since the number of edges is small (one per cross-proof
binding), the $O(E)$ direct check is cheap.

### Step 4: Wrapping for on-chain verification

The aggregated R1CS instance can be wrapped:

| Wrapper | Proof size | Setup | Estimated wrapper time |
|---------|-----------|-------|----------------------|
| Groth16 | ~128 bytes | Per-circuit trusted setup | ~10s |
| Spartan + HyperKZG | ~5 KB | Universal SRS | < 1s |

For zkARc's agentic commerce use case, the Spartan + HyperKZG wrapper is
the pragmatic choice: universal SRS (no per-circuit ceremony), sub-second
wrapper proving, and 5 KB proofs that fit in a single HTTP header.

## The two-tier design with efficient recursion

The [two-tier aggregation](./proof-aggregation.md) design maps cleanly onto
efficient recursion:

**Tier 1 (same-session, folding).** Per-action proofs ($\pi_{\mathsf{AV}}$,
$\pi_{\mathsf{SMTsolver}}$) are produced in the same session. Their
extended verifiers are compiled to R1CS and folded via Nova. The session
ends with a single Spartan decider.

**Tier 2 (cross-session, recursion).** The setup-time proof
($\pi_{\mathsf{PMC}}$) was produced earlier and stored as a credential.
The per-action session verifies the credential recursively: the extended
verifier for $\pi_{\mathsf{PMC}}$ runs inside the aggregation circuit,
binding $\bar{P}_{\mathsf{SMT}}$ to the policy input. Thanks to efficient
recursion, this adds only ~1.5-1.9s to the per-action proving time.

**What the verifier sees.** A single aggregated proof $\pi_{\mathsf{agg}}$
(5 KB with Spartan + HyperKZG, or 128 bytes with Groth16) attesting
that all component proofs verified and all cross-proof bindings hold.

## Estimated end-to-end aggregation costs

| Component | Proving time | Notes |
|-----------|-------------|-------|
| $\pi_{\mathsf{AV}}$ (SLM extraction) | 1-2s | Jolt Atlas, small model |
| $\pi_{\mathsf{SMTsolver}}$ (50 rules) | ~2s | Jolt, oxiz in RISC-V |
| Recursion extensions ($\pi'$ per component) | ~1.5-1.9s each | Constant, independent of trace length |
| Folding (Tier 1) | ~100ms | One MSM + hash per fold step |
| Credential verification (Tier 2, $\pi_{\mathsf{PMC}}$) | ~1.5-1.9s | Recursive, once per action |
| Spartan decider | < 1s | Over the folded instance |
| **Total per-action overhead** | **~7-9s** | Dominated by component proving, not aggregation |

The aggregation overhead itself (folding + credential verification +
Spartan decider) is ~3-4s. The rest is component proving time. As
component proving gets faster (hardware acceleration, algorithmic
improvements), the aggregation overhead will dominate, and the efficient
recursion techniques keep it bounded.

## Why this is better than naive recursion

| | Naive recursion | Efficient recursion |
|---|---|---|
| Verifier cycles (per component) | ~1.5B | ~177M |
| Non-native arithmetic | Pervasive ($G_T$ in $\mathbb{F}_{q^{12}}$) | Eliminated (pushed to prover) |
| R1CS constraints (wrapper) | ~100M+ | ~10M |
| Recursion prover overhead | Minutes | 1.5-1.9s |
| Curve dependency | Requires cycle-of-curves folding | Simple $\pi'$ over Grumpkin |

## Open questions for zkARc

1. **HyperKZG recursion.** The paper focuses on Dory. Jolt Atlas uses
   HyperKZG. The pairing check in HyperKZG is simpler (a constant number
   of pairings vs Dory's logarithmic inner-product argument), so the
   recursion extension should be even cheaper. But the concrete circuit
   for HyperKZG's Gemini-based pairing check needs to be worked out.

2. **Heterogeneous PCS.** If we keep Dory in Jolt and HyperKZG in Jolt
   Atlas, each component's $\pi'$ is different. The folding step needs
   both R1CS shapes to be compatible. The copy-constraint wiring
   technique handles cross-proof binding, but the R1CS shapes must either
   be padded or folded via CCS/HyperNova.

3. **Streaming + recursion.** Jolt Atlas supports streaming proving
   (prefix-suffix decomposition) to bound prover memory. Does the
   recursion extension compose with streaming, or does $\pi'$ require
   the full trace in memory?

4. **Groth16 vs Spartan + HyperKZG.** For on-chain verification, Groth16
   gives 128-byte proofs but requires a per-circuit trusted setup. Spartan
   + HyperKZG gives ~5 KB proofs with a universal SRS. Which wrapper is
   appropriate depends on the deployment: on-chain (Groth16) vs off-chain
   API verification (Spartan + HyperKZG).

## References

- Dao, Dhawan, Georghiades, Thaler, Tretyakov, Zhu. "Efficient Recursion
  for the Jolt zkVM." 2026. (`raw/zkps/2026-0-efficient-recursion-jolt-zkvm/`)
- See also: [Proof aggregation](./proof-aggregation.md),
  [Folding sessions](./folding-sessions.md).
