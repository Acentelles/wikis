# Virtual Polynomials

## Definition

A **virtual polynomial** is a multilinear polynomial that the prover does not explicitly commit to, but whose evaluations the verifier can still check. To evaluate a virtual polynomial $q$ at a point $r$, the prover provides the claimed value $q(r)$ and proves correctness via a subsequent sum-check, reducing to evaluation claims on the underlying committed polynomials.

## Why They Matter

Virtual polynomials avoid additional commitment costs while still enabling succinct verification. Since polynomial commitments (MSMs, pairings) are typically the prover's bottleneck, every polynomial that can be made virtual represents a direct speedup.

## Examples

- **The equality polynomial** $\widetilde{\text{eq}}(r, \cdot)$: given $r$ from the transcript, both prover and verifier can evaluate it in $O(\ell)$ field operations. No commitment is needed.

- **Shifted accumulators** $\tilde{\rho}_{\text{shift}}(s, x) := \tilde{\rho}(s+1, x)$: in the Jolt recursion PIOPs, the shifted accumulator is not committed separately. Instead, its evaluation is derived from the original accumulator $\tilde{\rho}$ via a shift sum-check using EqPlusOne.

- **Digit selectors** $\tilde{b}(s, x)$: the base-power selector for $G_T$ exponentiation interpolates $b$ pre-computed base-power MLEs over digit-bit MLEs, producing degree $\lceil\log_2 b\rceil + 1$ per variable.

- **Bit MLEs**: in scalar multiplication PIOPs, the bits of the scalar $\alpha$ form a public polynomial $\tilde{b}(i)$ that is not committed.

## Interaction with Prover Cost

The techniques in [Speeding Up Sum-Check Proving](../papers/speeding-up-sum-check.md) and [Twist and Shout](../papers/twist-and-shout.md) use virtual polynomials systematically to push the prover's dominant cost from commitments toward field arithmetic, where hardware-level optimisations (SIMD, small-value specialisation) can apply.
