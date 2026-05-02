# Efficient recursion for proof aggregation

This page applies the techniques from "Efficient Recursion for the Jolt
zkVM" (Dao, Dhawan, Georghiades, Thaler, Tretyakov, Zhu, 2026) to the
zkARc proof aggregation problem. A detailed breakdown of the paper itself
is in the [sum-check wiki](../../sum-check/book/src/papers/efficient-recursion.md).

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

### Why it is expensive: the field mismatch

The Jolt SNARK is native to $\mathbb{F}_r$ (BN254's scalar field). But
the Dory verifier performs group operations whose coordinates are in
$\mathbb{F}_q$ (BN254's base field). Since $q \ne r$, every
$\mathbb{F}_q$ multiplication must be emulated non-natively, costing
hundreds of constraints each. For $G_T$ operations in
$\mathbb{F}_{q^{12}}$, the cost is roughly $12^2 = 144$ times worse.

This field mismatch is the fundamental reason the Dory verifier cannot
be cheaply run inside the Jolt SNARK, and it is what the efficient
recursion paper eliminates.

## The key idea: two proofs, two fields

Instead of forcing everything into one field, the proof is split in two:

**$\pi$ (base Jolt proof, over $\mathbb{F}_r$).** Proves program execution
and the information-theoretic component of Dory verification (sum-check
rounds, evaluation claims). Everything here is $\mathbb{F}_r$ arithmetic.

**$\pi'$ (auxiliary proof, over $\mathbb{F}_q$ via Hyrax on Grumpkin).**
Proves the cryptographic component of Dory verification: the $G_T$
exponentiations, $G_1$/$G_2$ scalar multiplications, and the
multi-pairing. By using Hyrax over Grumpkin (whose scalar field is
$\mathbb{F}_q$), these become native arithmetic. No non-native emulation.

The BN254/Grumpkin curve cycle ensures each proof operates over its
natural field:

| Curve | Base field | Scalar field |
|---|---|---|
| BN254 | $\mathbb{F}_q$ | $\mathbb{F}_r$ |
| Grumpkin | $\mathbb{F}_r$ | $\mathbb{F}_q$ |

### What the verifier of $(\pi, \pi')$ does

1. **Checks $\pi$'s information-theoretic component directly.** Pure
   $\mathbb{F}_r$ arithmetic.
2. **Verifies $\pi'$** by checking the Hyrax opening over Grumpkin (two
   MSMs of size $\sim 2^{10.5}$). Since Grumpkin $G_1$ coordinates are in
   $\mathbb{F}_r$, this is also native.

No non-native arithmetic at either step.

### The verifier is not a third SNARK

The verifier of $(\pi, \pi')$ is not itself a SNARK. It is just a
computation. What happens to it depends on the application:

- **Simple recursion:** the verifier runs as a RISC-V guest program inside
  the next Jolt instance (~177M cycles instead of ~1.6B).
- **Wrapping:** the verifier is compiled to R1CS (~10M constraints instead
  of ~100M+) and proved by a wrapper SNARK.

The proof structure is two layers, not three:

```
π   ← Jolt SNARK (over F_r, with Dory PCS)
π'  ← auxiliary SNARK (over F_q, with Hyrax PCS on Grumpkin)

Verifier of (π, π') ← not a SNARK; just a computation
  ├─ run inside the zkVM  → simple recursion
  └─ compile to R1CS      → wrapping
```

## The three stages of $\pi'$

All three stages belong to $\pi'$. The base proof $\pi$ is already done;
what remains is proving that Dory's cryptographic component was correct.

| Stage | What it proves | Rounds | Degree |
|---|---|---|---|
| **Stage 1** | Packed $G_T$ exponentiation zero-check | 11 | 8 |
| **Stage 2** | Everything else batched: $G_T$ mul, $G_1$/$G_2$ scalar mul and add, shift consistency, copy constraints | ~25 | $\le$ 8 |
| **Stage 3** | Prefix packing: reduces all claims to a single Hyrax opening | N/A (not a sum-check) | N/A |

Stage 1 is separate because the $G_T$ exponentiation zero-check has
different variables ($m_{G_T} + 4 = 11$) and a different challenge point.
Stage 2 batches everything else into one sum-check. Stage 3 is not a
sum-check at all; it is the prefix-packing claim reduction that produces
one dense polynomial opening.

## Three techniques that make $\pi'$ efficient

### 1. PIOPs for elliptic-curve arithmetic

The paper develops specialized Polynomial Interactive Oracle Proofs for
the EC operations that dominate Dory verification:

- **$G_T$ exponentiation**: elements are represented as 4-variate MLEs
  (12 $\mathbb{F}_q$ coefficients padded to 16). A base-$b$ decomposition
  of the scalar reduces the number of steps from $\lceil \log_2 r \rceil$
  to $\lceil \log_b r \rceil$. The recurrence
  $\rho_{s+1} = \rho_s^b \cdot g^{d_s}$ is verified via a zero-check,
  with the shifted accumulator handled by a shift sum-check using
  EqPlusOne. The digit selector is a virtual polynomial (not committed).
  Batching across all $G_T$ instances (up to 256 for large traces) runs a
  single zero-check over a stacked constraint index.
- **$G_1$ scalar multiplication**: double-and-add encoded as a zero-check
  with a shift sum-check for the running accumulator.
- **$G_2$ arithmetic**: same structure as $G_1$ but with coordinates in
  $\mathbb{F}_{q^2}$, splitting every coordinate MLE into two base-field
  polynomials (doubling constraint counts).

### 2. Copy constraints via sum-check

When composing the EC PIOPs (the output of one operation must equal the
input of the next), the paper uses a random-linear-combination
permutation check directly in the sum-check. For example, in one
reduce-and-fold round:

```
[Exp PIOP]    D₂, βᵢ   →  D₂^βᵢ      =: t₁
[Mul PIOP]    m₁, t₁   →  m₁·t₁      =: m₂
```

The copy constraint enforces that the exponentiation's output polynomial
equals the multiplication's input polynomial. A single wiring sum-check
per group proves all ~1,000-2,000 edges at once, avoiding the need to
commit an accumulator polynomial. This works because $E$ is small: the
verifier pays $O(E)$ directly, which is cheaper than the cost of
committing and verifying an accumulator polynomial inside the recursion.

### 3. Prefix packing

The witness polynomials for the EC PIOPs are packed using prefix codes so
that the total committed data stays under $2^{21}$ coefficients (~2.1M)
for all trace lengths. Since the dominant recursion verifier cost is the
Hyrax opening check (which scales with $\sqrt{\text{committed data}}$),
minimizing committed data minimizes verification time. Prefix packing is
a folklore technique first formalized by Peshawaria (2025).

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
itself. The Hyrax PCS opening check dominates at ~62% of cycles (~109M),
with the sum-check stages contributing ~10% and the pairing check ~9%.

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

The copy-constraint technique from the paper directly solves the
cross-proof binding problem:

- The output of $\pi_{\mathsf{PMC}}$ ($P_{\mathsf{SMT}}$) must equal the
  policy input of $\pi_{\mathsf{SMTsolver}}$.
- The output of $\pi_{\mathsf{AV}}$ ($C_{\mathsf{SMT}}$) must equal the
  claims input of $\pi_{\mathsf{SMTsolver}}$.

These are wiring edges between the R1CS instances of adjacent component
verifiers. Since the number of edges is small (one per cross-proof
binding), the $O(E)$ direct check is cheap.

### Step 4: Wrapping for on-chain verification

| Wrapper | Proof size | Setup | Estimated wrapper time |
|---------|-----------|-------|----------------------|
| Groth16 | ~128 bytes | Per-circuit trusted setup | ~10s |
| Spartan + HyperKZG | ~5 KB | Universal SRS | < 1s |

For zkARc's agentic commerce use case, the Spartan + HyperKZG wrapper is
the pragmatic choice: universal SRS (no per-circuit ceremony), sub-second
wrapper proving, and 5 KB proofs that fit in a single HTTP header.

## Mapping to the prototype

The [folding-sessions prototype](./folding-sessions-prototype.md) at
`code/zkarc-folding-sessions/` implements the session-level orchestration
that efficient recursion slots into. Here is how the paper's techniques
map to the prototype's types and code paths.

### The `BlindFoldBackend` trait and the two proofs

The `BlindFoldBackend` trait (`crates/sessions/src/backend.rs`) abstracts
the proving backend with three associated types:

```rust
pub trait BlindFoldBackend {
    type Instance: Clone;         // one component's proof
    type FoldedInstance: Clone;   // running fold across components
    type Decider: Clone;          // final Spartan proof + PCS opening
    // ...
}
```

In the efficient recursion picture, each `Instance` would be an
**extended** proof $(\pi_i, \pi'_i)$: the base BlindFold instance plus
its auxiliary Grumpkin proof. The `fold` method then folds the *verifiers*
of extended proofs (which are cheap R1CS instances) rather than the raw
Dory-heavy proofs.

The real adapter `JoltAtlasBackend<F, C>`
(`crates/sessions-jolt-atlas/src/lib.rs`) currently wraps
`RelaxedR1CSInstance<F, C>` directly. Integrating efficient recursion
means:

1. Extend `AtlasInstance<F, C>` to carry both the base BlindFold instance
   and the auxiliary Grumpkin proof artifact.
2. In `JoltAtlasBackend::fold`, fold the extended verifier's R1CS (not
   the original Dory-heavy verifier's R1CS).
3. The `decide` method (currently a placeholder) produces a real Spartan
   decider over the folded extended-verifier instances.

### Tier 2 and the in-circuit verifier

The `Session::attach` method (`crates/sessions/src/session.rs`) consumes
a `RecursiveCredential` from a previous session. The credential carries a
folded instance and a decider but no witness. The current session must
verify this credential by adding a verifier subcircuit to its R1CS.

This is exactly where the efficient recursion paper's techniques are
needed. Without the paper, the verifier subcircuit would contain the full
Dory verification (billions of cycles worth of non-native arithmetic).
With the paper, the credential also carries $\pi'$, and the subcircuit
verifies the *extended* verifier instead:

```rust
pub struct RecursiveCredential<B: BlindFoldBackend> {
    pub folded: B::FoldedInstance,
    pub decider: B::Decider,
    pub exposed: Vec<NamedCommitment>,
    // With efficient recursion, add:
    // pub auxiliary_proof: AuxiliaryProof,  // π' over Grumpkin
}
```

The in-circuit verifier subcircuit (the main open item in the prototype)
then becomes:

1. Check the Spartan decider over the folded instance (field arithmetic).
2. Check $\pi'$: verify the Hyrax opening over Grumpkin (two MSMs).
3. Both checks are native $\mathbb{F}_r$ arithmetic, so the subcircuit
   fits in ~10M R1CS constraints.

### Copy constraints at the session level

The prototype's `Session::bind` method records cross-component wiring:

```rust
session.bind(plan.output("C_SMT"), smt.input("C_SMT"))?;
```

This is the session-level analogue of the paper's copy constraints. The
paper's sum-check-based wiring technique can be applied here: instead of
emitting `OutputClaimConstraint`s into the R1CS (which is the current
partial implementation), the session can batch all bindings into a single
wiring sum-check as part of the folded R1CS.

The number of wiring edges is even smaller at the session level (one per
`bind` call, typically 2-3 per session) than in the paper's Dory verifier
(~1,000-2,000), so the $O(E)$ direct check is trivially cheap.

### Prefix packing and committed data

The prototype's `unified_r1cs` helper
(`crates/sessions-jolt-atlas/src/pad.rs`) reconciles different R1CS shapes
by concatenating stage configurations. This is the prototype's analogue of
shape unification.

With efficient recursion, the committed polynomials in $\pi'$ are
prefix-packed into a single 21-variable polynomial (~2.1M coefficients).
The session-level folding operates on the extended verifier's R1CS, whose
committed data is already minimal thanks to prefix packing. This means
the Hyrax commitment cost in the session's Spartan decider remains
bounded regardless of how many components are folded.

### Concrete integration path

| Step | What changes | Where in the prototype |
|---|---|---|
| 1. Extend `AtlasInstance` | Carry $(\pi, \pi')$ instead of just $\pi$ | `crates/sessions-jolt-atlas/src/lib.rs` |
| 2. Implement Grumpkin Hyrax adapter | New backend for the auxiliary proof | New crate `crates/sessions-grumpkin/` |
| 3. Extended verifier circuit | R1CS for checking $(\pi, \pi')$ | Verifier subcircuit in `session.rs` |
| 4. Wire `Session::attach` | Credential carries $\pi'$; subcircuit verifies extended verifier | `crates/sessions/src/session.rs`, `credential.rs` |
| 5. Real Spartan decider | Produce decider over the folded extended-verifier R1CS | `JoltAtlasBackend::decide` |
| 6. Cross-component wiring | Emit `OutputClaimConstraint`s or batch via wiring sum-check | `Session::bind` |

Steps 1-2 are the core integration. Steps 3-4 unlock Tier 2. Steps 5-6
complete the end-to-end pipeline.

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
| Architecture | Requires cycle-of-curves folding | Simple $\pi'$ over Grumpkin, no folding needed |

## Open questions for zkARc

1. **HyperKZG recursion.** The paper focuses on Dory. Jolt Atlas uses
   HyperKZG. The pairing check in HyperKZG is simpler (a constant number
   of pairings vs Dory's logarithmic inner-product argument), so the
   recursion extension should be even cheaper. But the concrete circuit
   for HyperKZG's Gemini-based pairing check needs to be worked out.

2. **Heterogeneous PCS.** If we keep Dory in Jolt and HyperKZG in Jolt
   Atlas, each component's $\pi'$ is different. The folding step needs
   both R1CS shapes to be compatible. The prototype's `unified_r1cs`
   helper handles this at preprocessing time, but the extended verifier
   circuits for Dory-based and HyperKZG-based $\pi'$s will have different
   shapes that must also be reconciled.

3. **Streaming + recursion.** Jolt Atlas supports streaming proving
   (prefix-suffix decomposition) to bound prover memory. Does the
   recursion extension compose with streaming, or does $\pi'$ require
   the full trace in memory? The paper's [small-space](../../sum-check/book/src/papers/small-space.md)
   companion (Nair, Thaler, Zhu, 2025/611) suggests streaming and
   recursion are complementary, not conflicting.

4. **Groth16 vs Spartan + HyperKZG.** For on-chain verification, Groth16
   gives 128-byte proofs but requires a per-circuit trusted setup. Spartan
   + HyperKZG gives ~5 KB proofs with a universal SRS. Which wrapper is
   appropriate depends on the deployment: on-chain (Groth16) vs off-chain
   API verification (Spartan + HyperKZG).

5. **Credential size.** With efficient recursion, each
   `RecursiveCredential` grows by the size of $\pi'$ (~136 KB). For
   credentials that are stored and reused many times (like the policy
   credential), this is acceptable. For one-shot credentials, the
   overhead needs to be weighed against the proving-time savings.

## References

- Dao, Dhawan, Georghiades, Thaler, Tretyakov, Zhu. "Efficient Recursion
  for the Jolt zkVM." 2026. (`raw/zkps/2026-0-efficient-recursion-jolt-zkvm/`)
- Peshawaria. "Batching evaluation claims without homomorphic
  commitments." Manuscript, 2025.
- See also: [Proof aggregation](./proof-aggregation.md),
  [Folding sessions](./folding-sessions.md),
  [Prototype](./folding-sessions-prototype.md).
