# From Interactive Proofs to SNARKs

## The Two-Component View

A modern SNARK verifier has two components (Thaler, 2025):

1. **Information-theoretic component.** Checks sum-check round messages and evaluation claims using field arithmetic. Cost: $O(\ell \cdot d)$ per sum-check instance.

2. **Cryptographic component.** Checks polynomial commitment opening proofs using group operations (scalar multiplications, pairings, multi-exponentiations). This is almost always the verifier's bottleneck.

## The Pipeline

```
Program  →  Arithmetisation  →  Polynomial IOP  →  PCS  →  SNARK
             (R1CS, CCS, ...)    (sum-check)        (Dory, Hyrax, ...)
```

1. **Arithmetisation.** The computation is encoded as a constraint system. For zkVMs, this includes instruction execution (via Lasso lookups in Jolt), memory checking (via Twist/Shout), and continuity constraints.

2. **Polynomial IOP.** Sum-check reduces the constraint-satisfaction claim to a small number of polynomial evaluation claims. The prover's cost here is $O(d \cdot 2^\ell)$ field operations per sum-check, which is quasi-linear in the witness size.

3. **Polynomial commitment scheme.** Each evaluation claim is verified cryptographically. The PCS choice determines proof size, verifier time, setup assumptions, and whether recursion is practical.

## Fiat-Shamir

The interactive sum-check is made non-interactive via the Fiat-Shamir transformation: the verifier's random challenges are replaced by hash evaluations over the transcript. This yields a publicly verifiable argument in the random oracle model.

(*Proofs, Arguments, and Zero-Knowledge*, Chapter 5)

## Two PIOP Families

Thaler's survey (2025/2041) identifies two main PIOP families:

| | Quotienting (PLONK-style) | Sum-check-based |
|---|---|---|
| Core technique | Polynomial division | Sum-check protocol |
| Prover bottleneck | $O(n \log n)$ FFTs + commitments | $O(n)$ field ops + commitments |
| Commitment overhead | 1 commitment per constraint column | Fewer commitments (virtual polys) |
| Representative systems | PLONK, Marlin, Halo2 | Spartan, Lasso, Jolt, HyperNova |

The sum-check family achieves the fastest known provers because the information-theoretic phase is linear and commitment costs can be minimised through virtual polynomials and batching.

## Recursion

Recursion (proving knowledge of proofs) requires the verifier to be cheap when executed inside a SNARK. This is where the PCS choice becomes critical:

- **Hash-based PCS** (e.g., FRI): verifier is hash-heavy, leading to large constraint counts when arithmetised.
- **Curve-based PCS** (e.g., Dory, HyperKZG): verifier involves group operations, which can be expensive if they require non-native field arithmetic.

The [Efficient Recursion for the Jolt zkVM](../papers/efficient-recursion.md) paper shows how to restructure the verifier so that nearly all expensive group work is offloaded to an auxiliary proof, leaving a recursion-friendly verifier consisting of native-field multiplications.
