# Revisiting the IPA-Sumcheck Connection

| | |
|---|---|
| **Authors** | Liam Eagen, Ariel Gabizon |
| **eprint** | 2025/1325 |
| **Affiliations** | Alpen Labs, Aztec Labs |

## Summary

A new perspective viewing the Inner Product Argument (IPA) as a sum-check protocol where the polynomial coefficients lie in a discrete-log hard group $G$. This yields a multilinear PCS with Halo-style accumulation and a polylogarithmic verifier.

## Key Observation

In standard IPA, the prover sends cross-term commitments $L_j, R_j$ and the verifier folds generators. The recursive halving of the commitment is structurally identical to the sum-check protocol's recursive variable elimination, but operating "in the exponent" over a group $G$ rather than over a field.

## Contributions

### Multilinear PCS with Accumulation

The paper constructs a PCS for multilinear polynomials (not just univariate ones) by lifting the sum-check protocol to operate over group elements. Multiple evaluation claims can be accumulated into a single deferred claim, following the Halo paradigm.

### Polylogarithmic Verifier

The standard IPA verifier is $O(n)$ because it must reconstruct the folded generator vector. This paper replaces that step with a **group variant of BaseFold**, reducing verifier time from $O(n)$ to $O(\lambda \cdot \log^2 n)$ where $\lambda$ is the security parameter.

### Cost Tradeoff

The prover performs an additional $4n$ scalar multiplications for deciding accumulated claims, which is a constant-factor overhead on top of the standard IPA prover.

## Connection to the Broader Landscape

This paper bridges two lines of work:

- The **IPA/Bulletproofs** line (transparent, no pairings, $O(n)$ verifier).
- The **sum-check + PCS** line (Spartan, Jolt).

By showing they are instances of the same algebraic structure, the paper opens the door to combining techniques from both worlds, e.g., applying sum-check prover optimisations to IPA-based systems.
